# Snowflake Paper and Architecture Review

- [Snowflake Paper](https://www.snowflake.com/wp-content/uploads/2019/06/Snowflake_SIGMOD.pdf)
- Andi Pavlo's lecture from spring 2023:
  - [Youtube, Lecture](https://www.youtube.com/watch?v=bveqnSk15JQ)
  - [Slides] (https://15721.courses.cs.cmu.edu/spring2023/slides/21-snowflake.pdf)
    
## Intro

Snowflake was a wilful investment decision to make a bleeding edge from scratch OLAP database built for SaaS.
Hired very senior engineers from Oracle and Vectorwise.
Developed in C++, intention to have a scalable, vectorized OLAP database built on top of cloud.
Fully built from scratch, full control. (This is known as a "greenfield" design, no baggage, no legacy)

Some quick highlights:
- Distributed Database, multiple worker nodes
- Disaggregated Storage
- Separates fully data from metadata
- In C++, full control of memory, compiled code (no JVM)
- Columnar Storage (PAX-style format)
- Unified Query Optmizer + Adaptive planner (can change plans)
- Pre-compiled code for vectorized operations on specific primitives (with type specialization)
- No concept of a shared cache or buffer-pool (more on this later)

(Credit to Andi Pavlo's lectures on this)


## High Level Architecture

- Data Storage // Storage
  
 Remote, S3 etc.. blob store of data files, immutable fuiles, in PAX format (like parquet), micro-partitions ~50-500MB of raw data.
 Concurrency control logic was engineered accounting for write-once files and ability to range-read of files.
 Engine can re-write (optimize) data files to for popular workloads.
 Any type of table mutation is handled by Copy-on-Write.

- Virtual Warehouses // Compute

  Nodes are fully ephemeral & stateless but contain flash-storage which is critical for performance & cost.
  Designed for agreessive caching of data files, w/ consistent hashing
  
  A note about temp data: they mention that they spill-over to s3 for temp-data. So temp-data can be as large as needed.
  A side-effect of this is: the flash storage is *always* seen as a disposable cache.

  Compute nodes have SSD drives used solely for caching and temp-data. S3 is also used as spill-over of temp-data so that full disks are handled gracefully.


- Cloud Services // Control-Plane.

  All business logic is on the Control-Plane:
   - metadata mgmt, schemas, DDL operations
   - ACL metadata, encryption, metering, usage metrics..
  Control-Plane uses a fast, transactional Key-Value store (FoundationDB) for the catalog and all metadata. the "DB" of this OLAP-DB is a NoSQL KV-storage. heheeh

  This component doesn't have SPOFs and will handle all the multitenancy.
  A single tenant will deploy  "virtual warehouses" where it will run their queries.
  Tenants have to IMPORT their DATA into snowflake, they can share data across warehouses.

  

### Data and Metadata

Because files are immutable caching is straightforward, their sizes are also optimized for high granularity on caching. Caching a whole file is fine.
Metadata, schema, file statistics etc.. they are all stored in a fast NoSQL Key-Value storage (Foundation DB).
So, query planning has access to very low-latency, high-TPS, consistent KV-storage to provide all the information required for planning an execution.
Workers available and probably even the mapping of data-files to worker-node(s) that is/are responsible for caching that are all available to the scheduler
This type of information makes sense to keep alive and fresh on their KV-storage, allowing for more optimal task placement and scheduling.

This contrasts with DeltaLog or basic HiveMetastore that only keep some part of this information on the catalog and the catalog is not in a fast-KV storage
Also not strongly consistent (like files on S3)

So, there are is a huge technical advantage here:
- the catalog information is in a fast consistent, transactional storage
- table -> files mapping is present on catalog, fast to read and atomically change
- the file metadata and statistics is kept on catalog.
  - on Parquet files this is kept on the file footer.
- transaction logs, logs, etc.. everything is on FoundationDB.

On worker-nodes:
- local cache of files and related metadata it has seen before
- simple LRU is "good enough"
- brain-less, single "fork" process to process query, knows of sibling worker nodes
- Data produced can go to temp-files (s3 + cached) or pushed to other nodes (probably just "sinks")
- "cluster membership" is per-query and is dynamic as workers can die or have other faults.


### Elasticity and Isolation

Note that snowflake now has a "serverless" offering, the documented design as a clear allocation and segregation of compute per "warehouse".

The membership worker-nodes of the warehouse is elastic, but there is a membership group across which the caching of data is divided.
The cache placement uses a consistent-hashing algo, so that we increase the cache hit rate and reduce duplication/wasted flash storage.
The algo is lazy, when group-membership changes happen, they rely on the cache-replacement algo (LRU) of the local worker node to evict stale files.
Compute tasks are split per file and sent to the respective owner of that file according to the consistent-hashing.

A Work stealing scheduler used to deal with stragglers or workload in-balances. Dynamically during query execution, workers that finish early will steal work from straggling nodes (taking over remaining files that haven't been processed yet)

They are flexible on the worker-count of a warehouse, if query is taking a long time, more worker-nodes are "borrowed" temporarily to finish the query faster.

Compute elasticity is possible because storage is only a caching optimization, loosely coupled. Any worker-node can work on any file.
The performance advantage of caching comes from the cache-aware scheduler and having long-lived worker nodes that hold hot-data.



Data format is columnar, will include column-vector compression and there are additional tricks to work well with nested types.

### The Execution Engine.

(Shorter and more summarized)

Push-based engine, vectorized and columnar data-layout.
The data-format and RPCs are built in-house and optimized with care.

Push-based engines are common because they maintain a top-coordinator that centralized the plan/control-flow of the execution and parallelize it better.
In the paper they emphasise better cache efficiency and creating a better DAG-plan instead of a discovered-tree that results from the pull-model.
This is in contrast with the classically used "vulcano-style" pull-model.

All queries run against a static-map of a files. the workers just work on files and operations over vectors essentially.
The coordinator/planner is responsible for exploding the catalog, table-versions, schema, logical-plans, file-memberships etc.. and possibly creates:
- a static, denormalized plan of dags of operations of known discrete types over specific sets of files.
- the planner uses catalog metadata statistics for pruning the file-list, no secondary-indices, zero user-input.
- maps the vertices of the dag into worker-nodes, passing "sources" and "sinks", either files or other workers..
- operations are designed to spill-to-disk, handling mem exhaustion (they expliclity rejected the design of a in-mem-only engine)



### "Cloud Services"

The highlight of this section for me is:
- query management and planning relies on pruning and delays "decisions" based on table statistics.
  I imagine that they delay deciding on the hash/probe side of joins and things like that and re-plan
  if they discover data-skew or things like that.
- they combine this w/ prunning and "alway spill-to-disk" choices wisely.


### Left overs

They used a bunch of pages on data 


