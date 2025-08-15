
# PostgreSQL Maintenance, Monitoring and Optimization Cheatsheet

## Table of Contents
0. [PostgreSQL Performance Factors Summary](#postgresql-performance-factors-summary)
	- [I/O Impact Factors](#io-impact-factors)
	- [Memory Impact Factors](#memory-impact-factors)

1. [Maintenance and Monitoring](#1-maintenance-and-monitoring)
   - [Routine Maintenance](#routine-maintenance)
   - [Monitoring and Diagnostics](#monitoring-diagnostics)
	 - [Query Execution Analysis](#query-execution-analysis)
	 - [Table Clustering](#table-clustering)
	 - [Index Health](#index-health)
	 - [Complete System Assessment](#complete-system-assessment)

2. [Parallel Query Execution](#2-parallel-query-execution)
   - [Gather Nodes](#21-definition-gather-nodes)
   - [Limitations of Parallelism](#22-limitations-of-parallelism)

3. [NUMA Awareness](#3-numa-awareness)

4. [Checklist](#4-checklist)
   - [Memory Settings](#42-memory-settings)
   - [Parallelism Settings](#43-parallelism-settings)
   
5. [Profiling](#5-profiling)
   - [Extensions and Metrics](#51-extensions-and-metrics)
   - [Monitoring Parallel Queries Performance](#52-monitoring-parallel-queries-performance)

6. [Deploy a PostgreSQL disk-spill monitoring system on EC2](#6-deploy-a-postgresql-disk-spill-monitoring-system-on-ec2)
   - [Overview](#61-overview)
   - [Files and Configuration](#62-files-and-configuration)


## PostgreSQL Performance Factors Summary

### 1. I/O Impact Factors

| Factor | Worst-Case Impact | Best Practice |
|--------|-------------------|------------------|
| **Table Access Methods** | **Sequential Scans**: When queries don't use indexes and scan entire tables, causing excessive disk reads and significantly increased I/O load. | **Appropriate Indexing**: Well-chosen indexes dramatically reduce disk reads by allowing PostgreSQL to access only the relevant pages rather than scanning entire tables. Different index types (B-tree, GIN, GiST) are optimized for different query patterns. |
| **Index Implementation** | **B-tree Index Operations**: Sequential scan + sorting requires temporary files on disk, creating storage and I/O overhead. | **Specialized Indexes**: Using the right index type for specific query patterns minimizes I/O (GiST for geometric data, GIN for full-text, BRIN for ordered data). |
| **Data Organization** | **Lack of Table Clustering**: When related data is physically scattered across storage, more disk blocks must be read. | **Table Clustering**: `CLUSTER` command physically reorders table data to match index order, reducing random I/O by ensuring related data is stored sequentially on disk. |
| **Large Table Management** | **Unpartitioned Large Tables**: Without partitioning, queries need to scan entire large tables rather than targeting specific partitions. | **Partitioning**: Breaking large tables into smaller chunks enables partition pruning, allowing queries to skip irrelevant partitions entirely. |
| **Maintenance Operations** | **VACUUM Operations**: Background cleanup processes generate significant I/O, especially with high update/delete rates. | **Strategic VACUUM Scheduling**: Regular but appropriately timed VACUUM operations prevent bloat without causing I/O spikes during peak hours. Autovacuum with optimized settings balances maintenance needs with I/O impact. |
| **Checkpointing** | **Excessive Checkpoints**: Too frequent checkpoints cause I/O spikes. | **Optimal Checkpointing**: Setting appropriate `checkpoint_timeout` and `max_wal_size` spreads I/O over time rather than causing spikes. |
| **Parallel Processing** | **Parallel Query Spills**: Parallel hash joins or nested loop joins degrade due to spill to disk or duplicated work. | **Parallel Query Operations**: When properly configured, parallel query execution distributes I/O load across multiple workers. |
| **Memory Management** | **Large Sort Operations**: When sort operations exceed work_mem, PostgreSQL spills to disk, causing additional I/O operations. | **Buffer Cache Warming**: Using pg_prewarm extension to explicitly load frequently accessed data into the buffer cache before it's needed reduces I/O during peak usage. |
| **Join Implementations** | **Inefficient JOIN Operations**: Particularly nested loop joins on large datasets that can't fit in memory, causing repeated table scans. | **Connection Pooling**: Using pgBouncer reduces the overhead of connection establishment/teardown, minimizing associated I/O operations. |
| **Write-Ahead Logging** | **WAL Volume**: High transaction rates generate excessive WAL files, increasing I/O for both writing and archiving. | |
| **Schema Changes** | **Full Table Rewrites**: Operations like ALTER TABLE that require rewriting entire tables. | |


### 2. Memory Impact Factors

| Factor | Worst-Case Impact | Best Practice |
|--------|-------------------|------------------|
| **shared_buffers** | **Insufficient**: When buffer cache is too small, PostgreSQL must frequently read from disk instead of memory. | **Appropriate**: Keeping frequently accessed data in memory eliminates disk reads. For OLTP workloads, allocate ~25–40% of system RAM; for OLAP workloads, up to 50–75%. |
| **work_mem** | **Inadequate**: When set too low, sort and hash operations spill to disk instead of completing in memory. | **Adequate**: Sufficient per-operation memory prevents spilling sorts and hash operations to disk. Start with 32MB–64MB per operation to avoid "Sort Method: external merge Disk." |
| **maintenance_work_mem** | **Poorly Sized**: When too low, affects vacuum and index creation performance. | **Proper**: Higher values for maintenance operations like VACUUM and CREATE INDEX significantly reduce their I/O impact. For transactional DBs: 256MB–512MB; for bulk loads/indexing: 1GB–2GB (or more). |
| **effective_cache_size** | **Inadequate**: When set incorrectly, can lead to suboptimal query plans that don't account for OS cache. | **Optimized**: Accurately representing available OS cache helps the planner choose more efficient execution strategies. For OLTP: ~50% of system RAM; for OLAP: ~75% of system RAM. |
| **Connection Management** | **Too Many Connections**: Each connection consumes memory; excessive connections can exhaust available RAM. | **Connection Pooling**: Reduces total memory footprint by reusing connections instead of creating new ones. |
| **Parallel Query Operations** | **Unoptimized Parallel Queries**: Each worker process requires its own memory allocation, potentially multiplying memory usage. | **Proper Parallelism Settings**: Configuring max_parallel_workers appropriately ensures efficient memory utilization across multiple cores without oversubscription. |
| **Indexing Strategy** | **Too Many Indexes**: Each index consumes memory in the shared buffer pool, reducing space for table data.<br>**Inefficient GIN Indexes**: While powerful for full-text search and JSONB, they have higher memory requirements than B-tree indexes. | **Proper Index Selection**: Using the right index type for each use case minimizes memory requirements. For example, Hash indexes for equality-only lookups use less memory than B-tree indexes. |
| **Query Execution** | **Poor Query Planning**: Inefficient execution plans may use more memory than necessary for operations.<br>**Large Temporary Result Sets**: Queries that generate large intermediate results can exhaust available memory. | **Prepared Statements**: Reusing query plans reduces the memory overhead of planning and optimizing repeated queries.<br>**temp_buffers Optimization**: For queries using temporary tables, adequate temp_buffers reduces disk I/O. |
| **Data Types** | **Complex JSONB Data**: Large JSON documents consume significant memory for parsing, storage, and indexing. | **JIT Compilation**: For complex queries, Just-In-Time compilation can improve CPU efficiency and reduce memory pressure. |
| **Architecture** | **Transaction ID Wraparound**: Can cause memory bloat if not addressed with regular vacuuming. | **NUMA-Aware Configuration**: For multi-socket servers, binding PostgreSQL processes to specific NUMA nodes reduces memory access latency.<br>**Logical Replication Over Physical**: Logical replication is often more memory-efficient than physical replication as it propagates only the necessary data changes. |


## 1. Maintenance and Monitoring

### Maintenance Routine 
```sql
-- 1. Vacuuming (for heavily updated/truncated tables)
-- Note: Vacuum daemon (autovacuum) runs automatically to reclaim storage and maintain database health.
VACUUM table_name;      	-- cleans up dead tuples; non-blocking operation; reclaims space within the same data pages for reuse
							-- but does not shrink the table file size on disk (just marks space as reusable)
							-- and generates I/O contention if run frequently
							-- Use case: maintenance

VACUUM FULL table_name; 	-- rewrites the entire table; shrinks physical file size
							-- but acquires an exclusive lock on the table
							-- and has high disk I/O and CPU usage
							-- and is slow for large tables
							-- and must rebuild all indexes afterwards
							-- Use case: after large deletions or updates, defragmentation, before archival
							-- Alternatively, use pg_repack extension for online table reorganization
							-- or pg_squeeze for online table compression						

VACUUM ANALYZE table_name; 	-- Use case: after significant data changes, to update statistics for the query planner						

VACUUM FREEZE table_name; 	-- marks all tuples as frozen; prevents transaction ID wraparound issues
							-- Use case: for tables with high transaction ID churn, to prevent wraparound

-- After significant data changes, analyze table to update statistics
ANALYZE my_table;

-- 2. Reindexing
-- Rebuilding indexes to improve performance after significant data changes or fragmentation
-- Note: too many indexes can slow down write operations
REINDEX TABLE my_table;  		-- rebuilds the index for a specific table
REINDEX DATABASE my_database;  	-- rebuilds all indexes in a specific database
REINDEX SYSTEM my_database;  	-- rebuilds all system indexes in a specific database
REINDEX INDEX my_index;  		-- rebuilds a specific index

-- Check tables needing VACUUM based on dead tuple percentage
SELECT relname, n_live_tup, n_dead_tup, 
       round(n_dead_tup * 100.0 / nullif(n_live_tup, 0), 2) AS dead_percentage,
       last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE round(n_dead_tup * 100.0 / nullif(n_live_tup, 0), 2) > 10
ORDER BY dead_percentage DESC;

-- Clustering: physically reorders the table based on the specified index
-- Clustering is a one-time operation and does not maintain the order after updates or inserts.
CLUSTER my_table USING my_index;

-- 3. Log Rotation
-- PostgreSQL logs can be rotated using the log_rotation_size and log_rotation_age parameters in postgresql.conf.
-- Log rotation helps manage log file size and prevents excessive disk usage.
-- Example:
log_rotation_size = 100MB
log_rotation_age = 7d
-- Or:
ALTER SYSTEM SET log_rotation_size = '100MB';
ALTER SYSTEM SET log_rotation_age = '7d';

-- 4. Backup and Restore
-- see section 4, PostgreSQL.md
```

### Monitoring & Diagnostics

#### Query Execution Analysis
```sql
-- Enable partition pruning to skip irrelevant partitions
-- Critical for partitioned tables to avoid scanning unnecessary data
SET enable_partition_pruning = on;  -- Default is on in modern PostgreSQL

-- Control scan methods (useful for testing different execution plans)
SET enable_indexscan = on;      -- Enable index scans
SET enable_bitmapscan = on;     -- Enable bitmap scans (better for larger result sets)
SET enable_seqscan = off;       -- Temporarily disable sequential scans to force index usage
                                -- Warning: Only use during testing, not in production
								
-- Explain query plan
EXPLAIN SELECT * FROM products WHERE price > 100;  				-- shows the query plan without executing it
EXPLAIN ANALYZE SELECT * FROM products WHERE price > 100;  		-- shows the execution plan and actual execution time in milliseconds
EXPLAIN (FORMAT JSON) SELECT * FROM products WHERE price > 100; -- shows the execution plan in JSON format
EXPLAIN (BUFFERS) SELECT * FROM products WHERE price > 100;  	-- shows buffer usage statistics
EXPLAIN (VERBOSE) SELECT * FROM products WHERE price > 100; 	-- shows detailed information about the query plan

-- Find queries with disk spills
SELECT query, calls, rows, 
       total_exec_time/1000 AS total_seconds,
       temp_blks_read, temp_blks_written
FROM pg_stat_statements 
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC
LIMIT 10;

-- Monitor active queries
SELECT pid, 
       age(clock_timestamp(), query_start) AS duration, 
       usename, datname, state, 
       query
FROM pg_stat_activity
WHERE state != 'idle' AND query != '<IDLE>' 
ORDER BY duration DESC;

-- Find slow queries (requires pg_stat_statements)
CREATE EXTENSION pg_stat_statements;
SELECT dbid, query, calls, total_exec_time, rows, mean_exec_time
    FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;

-- Check for problematic idle transactions (uncommitted work holding locks)
SELECT pid, usename, datname, 
       age(clock_timestamp(), state_change) AS idle_time,
       query 
FROM pg_stat_activity 
WHERE state = 'idle in transaction' 
  AND state_change < now() - interval '5 minutes';

-- Examine lock conflicts
SELECT blocked.pid AS blocked_pid, 
       blocking.pid AS blocking_pid,
       blocked.query AS blocked_query,
       blocking.query AS blocking_query,
       age(now(), blocked.query_start) AS blocked_duration
FROM pg_stat_activity blocked
JOIN pg_locks blocked_locks ON blocked.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
  AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
  AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
  AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
  AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
  AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
  AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
  AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
  AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
  AND blocking_locks.pid != blocked_locks.pid
JOIN pg_stat_activity blocking ON blocking.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

#### Table Clustering
```sql
-- Check the clustering status of an index
SELECT indexrelname, indisclustered FROM pg_index WHERE indexrelname = 'my_index';

-- Check the clustering status of a table
SELECT relname, relhasoids, reloptions FROM pg_class WHERE relname = 'my_table';

-- Map physical ordering (ctid) to logical ordering (id)
-- ctid = (block/page number, index within the block)
SELECT ctid, id FROM my_table ORDER BY id DESC; 

-- Measure correlation between physical and logical order
-- Value interpretation: 1.0 = perfect correlation, 0 = random ordering
-- High values (>0.75) indicate good physical ordering for the column
SELECT correlation FROM pg_stats 
WHERE tablename = 'my_table' AND attname = 'id';  -- Higher is better for B-tree indexes
```

#### Index Health
```sql
-- Check index corruption for critical B-tree indexes
-- This performs logical verification of index structure
-- Returns true for each valid index, or error details for corrupted ones
SELECT bt_index_check(index => c.oid, heapallindexed => i.indisunique),
       c.relname,
       c.relpages
FROM pg_index i
JOIN pg_opclass op ON i.indclass[0] = op.oid
JOIN pg_am am ON op.opcmethod = am.oid
JOIN pg_class c ON i.indexrelid = c.oid
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE am.amname = 'btree'
  AND c.relpersistence != 't'  -- Skip temp tables from other sessions
  AND c.relkind = 'i' AND i.indisready AND i.indisvalid
ORDER BY c.relpages DESC LIMIT 10;  -- Check the 10 largest indexes first

-- Find unused indexes consuming space without benefit
SELECT s.schemaname, 
       s.relname AS table_name, 
       s.indexrelname AS index_name, 
       pg_size_pretty(pg_relation_size(idx.oid)) AS index_size,
       s.idx_scan AS scan_count
FROM pg_stat_user_indexes s
JOIN pg_index i ON s.indexrelid = i.indexrelid
JOIN pg_class idx ON i.indexrelid = idx.oid
WHERE s.idx_scan = 0      -- No scans since last stats reset
AND NOT i.indisunique     -- Not a unique constraint
AND NOT EXISTS            -- Not a primary key
    (SELECT 1 FROM pg_constraint c 
     WHERE c.conindid = i.indexrelid AND c.contype = 'p')
ORDER BY pg_relation_size(idx.oid) DESC;

-- Check for index bloat
SELECT schemaname, 
	   relname AS index_name, 
	   pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
	   pg_size_pretty(pg_relation_size(indexrelid) - pg_relation_size(indexrelid, 'main')) AS bloat_size,
	   idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- No scans since last stats reset
ORDER BY pg_relation_size(indexrelid) DESC;
```

#### Complete System Assessment
```sql
-- Check database statistics
SELECT datname, tup_inserted, tup_updated, tup_deleted, tup_fetched, tup_returned FROM pg_stat_database;

-- Check database size
SELECT pg_size_pretty(pg_database_size('myDB')) AS database_size;

-- Check table statistics
SELECT relname, n_live_tup, n_dead_tup, last_vacuum
    FROM pg_stat_user_tables;  

-- Check replication status
SELECT * FROM pg_stat_replication;

-- Check for replication lag (if using streaming replication)
-- Critical for read scalability and failover readiness
SELECT application_name, client_addr, state, 
	   sync_state, pg_last_xact_replay_timestamp() AS last_replay_time,
	   now() - pg_last_xact_replay_timestamp() AS replication_lag
	FROM pg_stat_replication;

-- Emergency: Terminate blocking connections
SELECT pg_terminate_backend(pid) 
    FROM pg_stat_activity 
    WHERE datname = 'mydb' AND pid <> pg_backend_pid();

-- Comprehensive system check - evaluate multiple health indicators
WITH table_stats AS (
    SELECT
        relname,
        n_live_tup,
        n_dead_tup,
        CASE WHEN n_live_tup = 0 THEN 0 
             ELSE n_dead_tup::float / n_live_tup::float 
        END AS dead_ratio,
        last_vacuum,
        last_autovacuum,
        last_analyze
    FROM pg_stat_user_tables
),
index_stats AS (
    SELECT
        schemaname,
        relname,
        indexrelname,
        idx_scan
    FROM pg_stat_user_indexes
)
SELECT 
    ts.relname AS table_name,
    pg_size_pretty(pg_total_relation_size(ts.relname::regclass)) AS total_size,
    ts.n_live_tup AS live_rows,
    ts.n_dead_tup AS dead_rows,
    ROUND(ts.dead_ratio * 100, 2) AS dead_pct,
    (SELECT COUNT(*) FROM index_stats WHERE relname = ts.relname) AS index_count,
    (SELECT COUNT(*) FROM index_stats WHERE relname = ts.relname AND idx_scan = 0) AS unused_indexes,
    ts.last_vacuum,
    ts.last_autovacuum,
    ts.last_analyze
FROM table_stats ts
ORDER BY pg_total_relation_size(ts.relname::regclass) DESC
LIMIT 20;  -- Focus on the largest tables
```


## 2. Parallel Query Execution

### 2.1. Definition: Gather Nodes
| Component | Description |
|-----------|-------------|
| Purpose | Used for Parallel Seq Scan, Parallel Index Scan, or Parallel Hash Join operations |
| Worker Operation | Workers operate independently on different portions of the data and send results via shared memory queues |
| Result Assembly | The final result is assembled by the leader process, which merges (and sorts) the results |

### 2.2. Limitations of Parallelism

#### ***Unsupported Query Types***

| Query Type | Limitation |
|------------|------------|
| CRUD Operations | Not supported for parallel execution |
| CTEs/Subqueries | Disable parallelism by default or must be materialized early to run in serial mode |
| Materialized Views | Typically not parallelized |
| Complex JOINs | Parallel hash joins or nested loop joins may degrade due to spill to disk or duplicated work |
| Functions | Window/aggregate/user-defined functions with non-deterministic or side-effect-inducing behavior not supported |
| Supported Queries | Primarily limited to SELECT queries |

#### ***Query Design***

| Consideration | Best Practice |
|---------------|--------------|
| Result Set Size | Avoid small result sets (e.g., LIMIT 1, FILTER, OFFSET) as workers process the entire table anyway |
| Table Size | Use large tables with significant data to process (parallelism adds overhead > fast serial queries on small tables) |
| Data Organization | Leverage partitioning and parallel query capabilities for optimal performance |

#### ***System factors***  
| Factor                  | Impact                                                                              |
| ----------------------- | ----------------------------------------------------------------------------------- |
| CPU cores           | More cores = more parallel workers available. Ideally N workers + 1 leader.         |
| Disk I/O throughput | Parallelism can become disk-bound. Use fast SSDs or RAID.                           |
| RAM                 | Parallel operations may spill to disk. Ensure sufficient memory.                    |
| NUMA topology       | For multi-socket servers, memory locality matters. PostgreSQL isn't NUMA-aware yet. |


## 3. NUMA Awareness
```bash
# Check CPU cores and NUMA topology
lscpu | grep -i cores
lscpu | grep -i numa

# Check memory usage by NUMA node
numactl --show
numactl --hardware

# Check disk I/O throughput
iostat -x 1 10 | grep -i sda
# Check memory usage
free -h

# Profile NUMA performance of PostgreSQL processes
numastat -p `pidof postgres`

# Bind PostgreSQL processes and memory to a specific NUMA node (here, node 0)
numactl --cpunodebind=0 --membind=0 postgres
# Bind per query
numactl --cpunodebind=0 --membind=0 psql -c "SELECT * FROM my_table WHERE id = 1"

# Distribute memory allocations round-robin across NUMA nodes
# Reducing worst-case latency, at the cost of average-case speed
numactl --interleave=all postgres

```

## 4. Checklist

### 4.1. Indexing
- Use appropriate index types (B-tree, GIN, GiST, etc.) based on query patterns.
- Avoid over-indexing; too many indexes can slow down write operations.
- Regularly analyze and reindex tables to maintain index health.

### 4.2. Memory Settings

- Estimating worst-case RAM requirements:
```
Max Memory = max_connections × work_mem [× joins/sorts per query]
        	+ autovacuum × maintenance_work_mem
        	+ background workers × work_mem
        	+ shared_buffers
```
- `pgtune` helps generate a configuration based on your system's resources and workload:  
https://pgtune.leopard.in.ua/

- Memory parameters in `postgresql.conf` / scopes:
```sql
-- Memory used for sorting, hashing and bitmap operations (GROUP BY, ORDER BY, DISTINCT, JOIN)
ALTER SYSTEM SET work_mem = '64MB';          		
	-- Default = 4MB
	-- Recommended:
	--		Start with 32MB–64MB per operation, but beware of multiplication across queries and connections
	--  	i.e. if a query has 10 joins + 2 sorts = 12×work_mem.
	SELECT pg_size_pretty(sum(work_mem)) AS total_work_mem FROM pg_stat_activity; 
	SELECT * FROM pg_stat_statements WHERE query LIKE '%Sort%' ORDER BY total_exec_time DESC;
	-- if you see: Sort Method: external merge Disk: ...
	-- ==> Increase work_mem to avoid disk-based sorting

-- Memory used for maintenance operations like VACUUM, CREATE INDEX
ALTER SYSTEM SET maintenance_work_mem = '256MB'; 	
	-- Default = 64MB
	-- Recommended:
	--		transactional DBs: 256MB–512MB
	--		bulk loads/indexing: 1GB–2GB (or more)
	SELECT pg_size_pretty(sum(maintenance_work_mem)) AS total_maintenance_work_mem FROM pg_stat_activity;

-- Memory used for caching data in shared memory (buffer pool)
ALTER SYSTEM SET shared_buffers = '1GB';    	
	-- Default = 128MB	
	-- Recommended:
	--		OLTP: ~25–40% of system RAM
	--		OLAP: More aggressive, up to 50–75% (if PostgreSQL is the only service)
	SELECT pg_size_pretty(sum(blks_hit) * 8192) AS cached_data FROM pg_stat_database;

-- Memory used for caching I/O operations
ALTER SYSTEM SET effective_io_concurrency = 200; 
	-- Default = 1
	-- Recommended:
	--		OLTP: 100–200 (for SSDs)
	--		OLAP: 200–400 (for SSDs)
	SELECT pg_size_pretty(sum(effective_io_concurrency)) AS total_effective_io_concurrency FROM pg_stat_activity;

-- Memory used for temporary tables
ALTER SYSTEM SET temp_buffers = '64MB'; 
	-- Default = 8MB
	-- Recommended:
	--		if frequent SELECT INTO TEMP => increase to 64MB+
	SELECT pg_size_pretty(sum(temp_buffers)) AS total_temp_buffers FROM pg_stat_activity;

-- Planner heuristic, used to decide between index scan vs seq scan
ALTER SYSTEM SET effective_cache_size = '2GB';
	-- Default = 4GB
	-- Recommended:
	--		OLTP: ~50% of system RAM
	--		OLAP: ~75% of system RAM
	SELECT pg_size_pretty(sum(effective_cache_size)) AS total_effective_cache_size FROM pg_stat_activity;

-- JIT compilation is used to speed up execution of expressions in queries, e.g. WHERE clauses, JOIN conditions, etc.
ALTER SYSTEM SET jit = on;
	-- Default = off
	-- Recommended:
	--		if queries are CPU-bound and have complex expressions
	--		if queries are I/O-bound, JIT may not help much
	ALTER SYSTEM SET jit_above_cost = 100000; -- Minimum cost for JIT to kick in; cost is estimated by the planner
	-- Default = 100000
	-- Recommended: 10000–50000
	SELECT pg_size_pretty(sum(jit)) AS total_jit FROM pg_stat_activity;
	SELECT * FROM pg_stat_statements WHERE query LIKE '%JIT%' ORDER BY total_exec_time DESC;

-- Restart PostgreSQL to apply changes

```

### 4.3. Parallelism Settings
```sql
-- Ensure max_worker_processes ≥ max_parallel_workers
SET max_worker_processes = 8; 				-- Total background worker slots (incl. autovacuum) (default is 8)
SET max_parallel_workers = 4;   			-- Global max for parallel workers (for any kind of parallel task) (default is 8)

-- Increase depending on the number of CPU cores and workload
SET max_parallel_workers_per_gather = 2; 	-- Max number of workers per Gather node (default is 2-4)

-- Lower costs to encourage parallelism; start with lower values and adjust based on performance
SET parallel_setup_cost = 1000; 			-- Penalty/Overhead cost of setting up parallel workers (default is 1000)
SET parallel_tuple_cost = 0.1;  			-- Per-tuple cost added to parallel plans (default is 0.1)

-- set off to improve performance on small tables or in-memory datasets only
SET parallel_leader_participation = on; 	-- Whether leader participates in work (default is on)

-- Restart PostgreSQL to apply changes
```

## 5. Profiling

### 5.1. Extensions and Metrics
- `pg_stat_progress_parallel_query`: shows progress of parallel queries
- `pg_stat_activity`: shows active queries and their states
- `pg_stat_user_tables`: shows statistics for user tables, including number of rows processed by parallel workers
- `pg_stat_activity`: See if multiple workers are active.
- `pg_stat_progress_*`: Show progress for VACUUM, ANALYZE, etc.
- `pg_stat_statements`: Track most expensive queries (serial vs parallel).
- `EXPLAIN ANALYZE`: Detailed performance per node.

| Metric                       | What to Watch                                                            |
| ---------------------------- | ------------------------------------------------------------------------ |
| `Execution Time`             | Compare serial vs parallel runs                                          |
| `Buffers`, `Shared Hit/Read` | See memory vs disk usage                                                 |
| `Workers Launched`           | If fewer workers launched than planned, may indicate resource exhaustion |
| `Loops`, `Actual Rows`       | Compare distribution of work across workers                              |


### 5.2. Monitoring Parallel Queries Performance
```sql
CREATE TEMP TABLE analyzed_plans AS (
  	SELECT queryid,
		query,
		(EXPLAIN (ANALYZE, VERBOSE, BUFFERS, FORMAT TEXT)
		FROM pg_stat_statements
		WHERE calls > 5
		ORDER BY total_exec_time DESC
		LIMIT 10
		) AS explain_output
);

-- Identify heavy queries that likely spilled to disk, 
-- i.e. having Sort Method: external or Hash Method: external in their plans
SELECT
  queryid,
  query,
  explain_output
FROM analyzed_plans
WHERE explain_output LIKE '%external%'
   OR explain_output LIKE '%Disk:%'
   OR explain_output LIKE '%I/O%'
LIMIT 10;

-- Identify queries that are not using parallelism effectively
SELECT
  queryid,
  query,
  explain_output
FROM analyzed_plans
WHERE explain_output NOT LIKE '%Parallel%'
LIMIT 10;
```
### 5.3. Benchmarking
```bash
# Create a benchmark script file
echo "
\set aid random(1, 100000 * :scale)
\set bid random(1, 1 * :scale)
\set delta random(-5000, 5000)
BEGIN;
UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
INSERT INTO pgbench_history (bid, aid, delta, mtime) VALUES (:bid, :aid, :delta, CURRENT_TIMESTAMP);
END;
" > benchmark_script.sql

# Run the benchmark
pgbench -h localhost -p 5432 -U postgres -d myDB -f benchmark_script.sql\
	-D scale=100000 \
	-j 4 -c 8 -T 60 \  # 4 threads, 8 clients, 60 seconds duration testing
  	--no-vacuum --failures-detailed --log --max-tries=1000 --debug
```


## 6. Deploy a PostgreSQL disk-spill monitoring system on EC2
(Using Docker, Prometheus, Grafana, pg_exporter, and Slack alerts)

### 6.1. Overview
Steps:
  - Launch an EC2 Ubuntu instance
  - Configure environment variables in `.env` file
  - Configure Grafana (API Key + Slack Webhook)
  - Run Ansible role to finalize provisioning and register Slack integration
  - Run the bootstrap script to set up and start the monitoring stack

### 6.2. Files and Configuration

##### docker-compose.yml
```yaml
version: '3.7'

services:
  postgres:
    image: postgres:15
    container_name: postgres
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  postgres_exporter:
    image: quay.io/prometheuscommunity/postgres-exporter
    container_name: postgres_exporter
    restart: always
    environment:
      DATA_SOURCE_NAME: "postgresql://postgres:postgres@postgres:5432/postgres?sslmode=disable"
    volumes:
      - ./postgres_exporter/queries.yaml:/etc/postgres_exporter/queries.yaml
    command:
      - '--extend.query-path=/etc/postgres_exporter/queries.yaml'
    ports:
      - "9187:9187"
    depends_on:
      - postgres

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    restart: always
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    container_name: grafana
    restart: always
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards

volumes:
  pgdata:
  grafana_data:
```

##### prometheus.yml
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'postgres_exporter'
    static_configs:
      - targets: ['postgres_exporter:9187']
```

##### postgres_exporter/queries.yaml
```yaml
pg_stat_statements:
  query: |
    SELECT
      queryid,
      calls,
      total_exec_time,
      shared_blks_hit,
      shared_blks_read,
      shared_blks_dirtied,
      shared_blks_written,
      temp_blks_read,
      temp_blks_written,
      query
    FROM pg_stat_statements
    ORDER BY temp_blks_written DESC
    LIMIT 10;
  metrics:
    - queryid:
        usage: "LABEL"
        description: "Query ID"
    - calls:
        usage: "COUNTER"
        description: "Number of times executed"
    - total_exec_time:
        usage: "GAUGE"
        description: "Total execution time in ms"
    - temp_blks_read:
        usage: "COUNTER"
        description: "Temporary blocks read"
    - temp_blks_written:
        usage: "COUNTER"
        description: "Temporary blocks written"

```

##### roles/postgres_monitoring/tasks/main.yml (Ansible Tasks/Playbook)
```yaml
- name: Ensure Docker Compose is present
  apt:
    name: docker-compose
    state: present

- name: Copy monitoring stack files
  copy:
    src: "{{ item.src }}"
    dest: /opt/spill-monitoring/{{ item.dest }}
    owner: root
    group: root
  with_items:
    - { src: 'docker-compose.yml', dest: 'docker-compose.yml' }
    - { src: 'prometheus/prometheus.yml', dest: 'prometheus/prometheus.yml' }
    - { src: 'postgres_exporter/queries.yaml', dest: 'postgres_exporter/queries.yaml' }
    - { src: 'grafana/provisioning/dashboards/disk_spill_dashboard.json', dest: 'grafana/provisioning/dashboards/disk_spill_dashboard.json' }

- name: Launch monitoring stack
  command: docker-compose up -d
  args:
    chdir: /opt/spill-monitoring

- name: Configure Slack contact point in Grafana
  uri:
    url: "http://localhost:3000/api/contact-points"
    method: POST
    headers:
      Authorization: "Bearer {{ grafana_api_key }}"
      Content-Type: "application/json"
    body_format: json
    body:
      name: "slack-alert"
      type: "slack"
      settings:
        url: "{{ slack_webhook_url }}"
  register: contact_point

- name: Register notification policy for dashboard alert
  uri:
    url: "http://localhost:3000/api/folders/uid/__home/alert-rules"
    method: POST
    headers:
      Authorization: "Bearer {{ grafana_api_key }}"
      Content-Type: "application/json"
    body_format: json
    body:
      dashboardUid: "spill_monitoring_dashboard"
      name: "Disk Spill Alert Rule"
      folderUid: "__home"
      panelId: 1
      condition:
        type: "query"
        evaluator:
          type: "gt"
          params: [0]
        operator: "and"
        query:
          params: ["A", "5m", "now"]
        reducer:
          type: "avg"
        reducerOptions: {}
      noDataState: "no_data"
      execErrState: "alerting"
      for: "5m"
      notifications:
        - uid: "{{ contact_point.json.uid }}"
```

##### Configuration Files (.env)
```yaml
grafana_api_key: "<YOUR_GRAFANA_API_KEY>"
slack_webhook_url: "<YOUR_SLACK_WEBHOOK_URL>"
contact_point_uid.json.uid: "<CONTACT_POINT_UID>"
```

##### grafana/provisioning/dashboards/dashboard.yaml
```yaml
apiVersion: 1
providers:
  - name: 'PostgreSQL Monitoring'
    orgId: 1
    folder: 'PostgreSQL'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    options:
      path: /var/lib/grafana/dashboards
```

##### grafana/provisioning/datasources/datasource.yaml
```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```
##### grafana/provisioning/notifiers/slack.yaml
```yaml
notifiers:
  - name: "Slack Notifications"
    type: "slack"
    uid: "slack-webhook"
    org_id: 1
    is_default: true
    send_reminder: false
    frequency: "1m"
    disable_resolve_message: false
    settings:
      url: "https://hooks.slack.com/services/XXX/YYY/ZZZ"  
      recipient: "#your-channel"
      title: "Disk Spill Alert"
      text: "Temp blocks written is above threshold in PostgreSQL"

```

##### grafana/provisioning/dashboards/disk_spill_dashboard.json 
```json
{
  "id": null,
  "uid": "spill_monitoring_dashboard",
  "title": "PostgreSQL Disk Spill Monitoring",
  "panels": [
    {
      "id": 1,
      "type": "timeseries",
      "title": "Disk Spills per Hour",
	  "dataSource": "Prometheus",
      "targets": [
        {
          "expr": "pg_spill_queries_spills_last_hour",
          "legendFormat": "Spills",
          "interval": "1m"
        }
      ],
      "alert": {
        "name": "Disk Spill Alert",
        "conditions": [
          {
            "type": "query",
            "evaluator": { "type": "gt", "params": [0] },
            "query": { "params": ["A", "5m", "now"] },
            "reducer": { "type": "avg" }
          }
        ],
        "for": "5m",
        "frequency": "1m",
        "notifications": [{ "uid": "slack-webhook" }],
        "noDataState": "no_data",
        "executionErrorState": "alerting"
      },
      "datasource": "-- Grafana --",
      "gridPos": { "x": 0, "y": 0, "w": 24, "h": 8 }
    }
  ],
  "schemaVersion": 36,
  "version": 1
}
```

##### Docker Bootstrap Script (deploy_monitoring.sh)
```bash
#!/bin/bash
set -e

# Install Docker & Compose
apt update
apt install -y apt-transport-https ca-certificates curl gnupg lsb-release 
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor \	
  > /usr/share/keyrings/docker-archive-keyring.gpg 						
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  > /etc/apt/sources.list.d/docker.list 

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-compose 

# Copy config files into /opt/spill-monitoring
cd /opt
mkdir -p spill-monitoring/{prometheus,postgres_exporter,roles/postgres_monitoring/tasks,grafana/provisioning/{datasources,dashboards,notifiers}}

cd spill-monitoring
cp /path/to/docker-compose.yml .
cp /path/to/prometheus/prometheus.yml ./prometheus/
cp /path/to/postgres_exporter/queries.yaml ./postgres_exporter/
cp /path/to/roles/postgres_monitoring/tasks/main.yml ./roles/postgres_monitoring/tasks/
cp /path/to/grafana/provisioning/datasources/datasource.yaml ./grafana/provisioning/datasources/
cp /path/to/grafana/provisioning/dashboards/dashboard.yaml \
   /path/to/grafana/provisioning/dashboards/disk_spill_dashboard.json \
   ./grafana/provisioning/dashboards/
cp /path/to/grafana/provisioning/notifiers/slack.yaml ./grafana/provisioning/notifiers/

# Run Docker Compose to start the monitoring stack
docker-compose pull
docker-compose up -d
```