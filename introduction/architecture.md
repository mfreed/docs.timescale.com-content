# Architecture & Concepts

## Overview [](overview)

TimescaleDB is a **relational database for time-series data**.

It is implemented as an extension to PostgreSQL, which means that it runs
within an PostgreSQL server as part of the same process, with code that
introduces new capabilities for time-series data management, new functions
for data analytics, new query planner and query execution optimizations, and
new storage mechanisms for more cost effective and performant analytics.
Any operations to a PostgreSQL database that includes TimescaleDB (whether
SELECTs or INSERTs, or schema management like creating indexes) are first
processed by TimescaleDB to determine how they should be planned or
executed against TimescaleDB's data structures.

This extension model allows the database to take advantage of the richness of
PostgreSQL, from 40+ data types (from integers, floats, strings, timestamps,
to arrays and JSON data types), to a dozen types of indexes, to complex
schemas, to an advanced query planner, and to a larger extension ecosystem
that plays nicely with TimescaleDB (including geo-spatial support through
PostGIS, monitoring with pg_stat_statements, foreign data wrappers, and more).

This design also allows TimescaleDB to take advantage of PostgreSQL's maturity
from a reliability, robustness, security, and ecosystem perspective.  One can
use PostgreSQL's physical replication to create multiple replicas for higher-
availability and scaling read queries; snapshots and incremental WAL streaming
for continuous backups to support full point-in-time recovery; role-based
access control at the schema, table, or even row-level; and integrations with
a huge number of third-party connectors and systems that speak the standard
PostgreSQL protocol, such as popular programming languages, ORMs, data science
tools, streaming systems, ETL tools, visualization libraries and dashboards,
and more.

At the same time, TimescaleDB leverages the high degree of customization
available to extensions by adding hooks deep into PostgreSQL's query planner,
data and storage model, and execution engine. This allows for new,
advanced capabilities designed specifically for time-series data, including:

- **Transparent and automated time partitioning**, where time-series tables are
  automatically and continuously "chunked" into smaller intervals to improve
  performance and to unlock various data-management capabilities.  Data
  and indexes for the latest chunks naturally remain in memory,
  insuring fast inserts and performant queries to recent data.

- **Native columnar compression** with advanced datatype-specific compression,
  employing various best-in-class algorithms based on whether the data are
  timestamps, integers, floats, strings, or others.  Users typically report 94-97\%
  storage reduction and faster queries to compressed data.

- **Continuous and real-time aggregations**, in which the database continually
  and incrementally maintains a materialized view of aggregate time-series data
  to improve query performance, while properly handling late data or data
  backfills.  TimescaleDB even enables queries to transparently merge pre-
  computed aggregates with the latest raw data to ensure always up-to-date
  answers.

- **Automated time-series data management features**, such as explicit or
  policy-based data retention policies, data reordering policies, aggregation
  and compression policies, downsampling policies, and more.

- **In-database job-scheduling framework** to power both native policies and to
  support user-defined actions, including those written in SQL or PL/pgSQL.

- **Horizontally-scalable multi-node operation** to automatically and
  elastically scale your time-series data across many TimescaleDB databases,
  while still giving the abstraction of a single time-series table.

To better understand TimescaleDB, we first want to better explain two main
concepts of how TimescaleDB scales: its data abstractions of **hypertables**
and **chunks** (and how these are stored and processed), and how TimescaleDB
can deployed either in single-node mode, with physical replicas, or in multi-
node mode to enable distributed hypertables.

## Hypertables and Chunks [](hypertable-chunks)

From a user's perspective, TimescaleDB exposes what look like singular tables,
called **hypertables**. A hypertable is your primarily point of interaction
with you data, as it provides the standard table abstraction that you can query
via standard SQL.  Creating a hypertable in TimescaleDB takes two simple
SQL commands: `CREATE TABLE` (with standard SQL syntax),
followed by `SELECT create_hypertable()`.

Virtually all user interactions with TimescaleDB are with hypertables.
Inserting, updating, or deleting data, querying data via SELECTs, altering
tables, adding new columns or indexes, JOINs with other tables or hypertables,
and so forth can (and should) all be executed on the hypertable.

However, hypertables are actually an abstraction or virtual view of
many individual tables that actually store the data, called **chunks**.

<img class="main-content__illustration" src="https://assets.iobeam.com/images/docs/illustration-hypertable-chunk.svg" alt="hypertable and chunks"/>

### Partitioning in Hypertables[](hypertable-partitioning)

Chunks are created by partitioning a hypertable's data into one
(or potentially multiple) dimensions. All hypertables are partitioned
by the values belonging to a time column, which may be in timestamp,
date, or various integer forms.  If the time partitioning interval is one
day, for example, then rows with timestamps that belong to the same
day are colocated within the same chunk, while rows belonging to
different days belong to different chunks.

TimescaleDB creates these chunks automatically as rows are inserted into the
database.  If the timestamp of a newly-inserted row belongs to a day not yet
present in the database, TimescaleDB will create a new chunk corresponding to
that day as part of the INSERT process. Otherwise, TimescaleDB will
determine the existing chunk(s) to which the new row(s) belong, and
insert the rows into the corresponding chunks.  The interval of a hypertable's
partitioning can also be changed over time (e.g., to adapt to changing workload
conditions, so in one example, a hypertable could initially create a new chunk
per day and then change to a chunk every 6 hours as the workload increases).

A hypertable can be partitioned by additional columns as well -- such as a device
identifier, server or container id, user or customer id, location, stock ticker
symbol, and so forth.  Such partitioning on this additional column typically
employs hashing (mapping all devices or servers into a specific number of hash
buckets), although interval-based partitioning can be employed here as well.
We sometimes refer to hypertables partitioned by both time and this additional
dimension as "time and space" partitions.

This time-and-space partitioning is primarily used for *distributed hypertables*.
With such two-dimensional partitioning, each time interval will also be
partitioned across multiple nodes comprising the distributed hypertables.
In such cases, for the same hour, information about some portion of the
devices will be stored on each node.  This allows multi-node TimescaleDB
to parallelize inserts and queries for data during that time interval.

[//]: # (Comment: We should include an image that shows a chunk picture of a
[//]: # partition pointing at multiple chunks, each chunk have some range of
[//]: # data, and an index (binary tree-like data structure) associated with it

Each chunk is implemented using a standard database table.  (In PostgreSQL
internals, the chunk is actually a "child table" of the "parent" hypertable.)
A chunk includes constraints that specify and enforce its partitioning ranges,
e.g., that the time interval of the chunk covers
['2020-07-01 00:00:00+00', '2020-07-02 00:00:00+00'),
and all rows included in the chunk must have a time value within that
range. Any space partitions will be reflected as chunk constraints as well.
As these ranges and partitions are non-overlapping, all chunks in a
hypertable are disjoint in their partitioning dimensional space.

Knowledge of chunks' constraints are used heavily in internal database
operations.  Rows inserted into a hypertable are "routed" to the right chunk
based on the chunks' dimensions.  And queries to a hypertable use knowledge
of these chunks' constraints to only "push down" a query to the proper
chunks: If a query specifies that `time > now() - interval '1 week'`, for
example, the database only executes the query against chunks covering
the past week, and excludes any chunks before that time.  This happens
transparently to the user, however, who simply queries the hypertable with
a standard SQL statement.

### Benefits of Hypertables and Chunks[](hypertable-benefits)

This chunk-based architecture benefits many aspects of time-series data
management. These includes:

- **In-memory Data**. Chunks can be configured (based on their time intervals)
  so that the recent chunks (and their indexes) fit in memory.  This helps ensure that inserts to
  recent time intervals, as well as queries to recent data, typically accesses
  data already stored in memory, rather than from disk.  But TimescaleDB
  doesn't *require* that chunks fit solely in memory (and otherwise error);
  rather, the database follows LRU caching rules on disk pages to maintain
  in-memory data and index caching.

- **Local Indexes**. Indexes are built on each chunk independently, rather than
  a global index across all data. This similarly ensures that *both* data and
  indexes from the latest chunks typically reside in memory, so that updating
  indexes when inserting data remains fast.  And TimescaleDB can still ensure
  global uniqueness on keys that include any partitioning keys, given the
  disjoint nature of its chunks, i.e., given a unique (device_id, timestamp)
  primary key, first identify the corresponding chunk given constraints, then
  use one of that chunk's index to ensure uniqueness.  But this remains simple
  to use with TimecaleDB's hypertable abstraction: Users simply create an index
  on the hypertable, and these operations (and configurations) are pushed down
  to both existing and new chunks.

- **Easy Data Retention**.  In many time-series applications, whether based on
  cost, storage, compliance, or other reasons, users often only want to retain
  data only for a certain amount of time. With TimescaleDB, users can quickly
  delete chunks based on their time ranges (e.g., all chunks whose data has
  timestamps more than 6 months old). Even easier, useres can create a data
  retention policy within TimescaleDB to make this automatic, which employs its
  internal job-scheduling framework. Chunk-based deletion is fast -- it's simply
  deleting a file from disk -- as opposed to deleting individual rows, which
  requires more expensive "vacuum" operations to later garbage collect and
  defragment these deleted rows.

- **Age-Based Compression, Data Reordering, and More**.  Many other data
  management features can also take advantage of this chunk-based architecture,
  which allows users to execute specific commands on chunks or employ
  hypertable policies to automate these actions.  These include TimescaleDB's
  native compression, which convert chunks from their traditional row-major
  form into a layout that is more columnar in nature, while employing
  type-specific compression on each column. Or data reordering, which
  asynchronously rewrites data stored on disk from the order it was inserted
  into an order specified by the user based on a specified index. By reordering
  data based on (device_id, timestap), for example, all data associated with a
  specific device becomes written contiguously on disk, making "deep and
  narrow" scans for a particular device's data much faster.

- ** Instant Multi-Node Elasticity**.  TimescaleDB supports horizontally
  scaling across multiple nodes. Unlike traditional one-dimensional
  database sharding, where shards must be migrated to a newly-added
  server as part of the process of expanding the cluster, TimescaleDB
  supports the elastic addition (or removal) of new servers without
  requiring any immediate rebalancing. When a new server is added,
  existing chunks can remain at their current location, while chunks
  created for future time intervals are partitioned across the new set
  of servers.  The TimescaleDB planner can then handle queries
  across these reconfigurations, always knowing which nodes are
  storing which chunks.  Server load subsequently can be rebalanced
  either by asynchronously migrating chunks or handled via data
  retention policies if desired.

- **Data Replication**.  Chunks can be individually replicated across
  nodes transactionally, either by configuring a replication factor on a
  distributed hypertable (which occurs as part of a 2PC transaction at
  insert time) or by copying an older chunk from one node to another
  to increase its replication factor, e.g., after a node failure (coming soon).

- **Data Migration**.  Chunks can be individually migrated transactionally.
  This migration can be across tablespaces (disks) residing on a single
  server, often as a form of data tiering; e.g., moving older data from a
  faster, more expensive disks to slower, cheaper storage. This migration
  can also occur across nodes in a distributed hypertable, e.g., in order to
  asynchronous rebalance a cluster after adding a server or to prepare for
  retiring a server (coming soon).

[//]: # (Comment:  Should we add some pointers to other key capabilities,
[//]: # such as native compression and continuous aggs?)


## Deployment Options: Single node, physical replicas, multi-node [](deployment-options)
[//]: # (Comment:  Not sure "deployment options" is the right term...  "Scaling:" ?)


[//]: # (Comment: Add image comparing single node, physical replication, multi-node)

TimescaleDB supports three main deployment options:  as an single database server,
in a tradition primary/replica deployment, or in a multi-node deployment with horizontal
scalability.

[//]: # (Comment: Is there a section on single node here)

### Primary / Backup Replication [](primary-backup-replication)

[//]: # (Comment: Update this image: https://blog.timescale.com/content/images/2018/12/image-12.png )

TimescaleDB supports streaming replication from a "primary" database server
to separate "replica" servers, using the standard PostgreSQL physical
replication protocol.  The protocol works by streaming records of database
modifications from the primary server to one or more replicas, which can then
be used as read-only nodes (to scale queries) or as failover servers (for high availability).

PostgreSQL streaming replication leverages the Write Ahead Log (WAL), which is
an append-only series of instructions that captures every atomic database change.
The replication works by continuously shipping segments of the WAL from the primary
to any connected replicas. Each replica then applies the WAL changes and makes them
available for querying, ensuring that operations to the primary are applied atomically
and in an identical order to each replica.

TimescaleDB (from underlying PostgreSQL functionality) supports various
options for physical replication that trade off performance and consistency.

Replication can occur synchronously.  When an insert or schema modification
operation is executed on the primary, the operation is replicated (or even
applied) to replicas before the primary returns any result.  This ensures that
replicas always stay perfectly up-to-date.

Replication can also occur asynchronously.  Operations to the primary return
to a client as soon as they are executed, but any changes are queued for
asynchronous transmission to any replicas.  Asynchronous replicas are
often used for ad-hoc data exploration, to avoid heavy query load on replicas
from interfering with a production primary server.

In fact, a single TimescaleDB primary can have both synchronous and
asynchronous replicas, for a mix of HA failover and read scaling.  The main
limitation of primary/backup replication is that each server stores a *full copy*
of the database.

[//]: # ( Link to section on distributed hypertables?  Timescale Cloud? )

### Multi-node TimescaleDB and Distributed Hypertables [](multi-node)

TimescaleDB 2.0 also supports horizontally scaling across many servers.
Instead of a primary node (and each replica) which stores the full copy
of the data, a *distributed hypertable* can be spread across multiple
nodes, such that each node only stores a portion of the distributed
hypertable (namely, a subset of the chunks). This allows TimescaleDB
to scale storage, inserts, and queries beyond the capabilities of a single
TimescaleDB instance.  Distributed hypertables and regular hypertables
look very similar, with the main difference being that distributed chunks
are not stored locally (and some other [current limitations][distributed-hypertable-limitations]).

In a multi-node deployment of TimescaleDB, a database can assume the
role of either an **access node** or a **data node**; both run the identical
TimescaleDB software for operational simplicity.

[//]: # (Comment: Picture of access nodes and data nodes )

A client connects to an access node database.  The client can
create a distributed hypertable on the access node, which stores
cluster-wide information about the different data nodes as well as
how chunks belonging to distributed hypertables are spread
across those data nodes. Access nodes can also store non-distributed
hypertables, as well as regular PostgreSQL tables.

Data nodes do not store cluster-wide information, and otherwise look
just as if they were stand-alone TimescaleDB instances.

Clients interact with the distributed hypertable all through the access
node, performing schema operations, inserting data, or querying the
data as if it's a standard table. When receiving a query, the access
node uses local information about chunks to determine to which data
nodes to push down queries. Query optimizations across the cluster
attempt to minimize data transferred between data nodes and the
access node, such that aggregates are performed on data nodes
whenever possible, and only aggregated or filtered results are passed
back to the access node for merging and returning to a client.

[//]: # ( Link to section on distributed hypertables?  Setting up on Cloud/Forge? )

[data model]: /introduction/data-model
[chunking]: https://www.timescale.com/blog/time-series-data-why-and-how-to-use-a-relational-database-instead-of-nosql-d0cd6975e87c#2362
[jumpSQL]: /using-timescaledb/hypertables
[TvsP]: /introduction/timescaledb-vs-postgres
[Compression Operational Overview]: /using-timescaledb/compression
[compression blog post]: https://blog.timescale.com/blog/building-columnar-compression-in-a-row-oriented-database
[contact]: https://www.timescale.com/contact
[slack]: https://slack.timescale.com/
[distributed-hypertable-limitations]: /using-timescaledb/limitations#distributed-hypertable-limitations
[multi-node-basic]: /getting-started/setup-multi-node-basic
