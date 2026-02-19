# DuckDB-MySQL Code Exploration Guide

> **How to use**: Each section below contains a self-contained prompt you can copy-paste into a coding agent (Claude Code, Cursor, etc.) to explore that part of the codebase interactively. Work through them in order for a progressive understanding, or jump to any section independently.
>
> **Prerequisite**: Read `duckdb-mysql-architecture.md` first for the high-level architecture overview.

---

# Part 1: Foundation — The Building Blocks

These steps build the foundational knowledge needed for all subsequent
workflows: the handler contract, plugin registration, and type system.

---

## Step 1: Handler Class Definition — The Contract

### Prompt
```
Read the file `storage/duckdb/ha_duckdb.h` in its entirety.

This is the header that defines `class ha_duckdb`, which implements MySQL's
`handler` interface — the contract every storage engine must fulfill.

Please answer the following questions:

1. What classes does `ha_duckdb` inherit from? Why does it inherit from both
   `handler` and `Partition_handler`?

2. List every overridden virtual method grouped by category:
   - Table operations (open, close, create, delete, rename)
   - Row operations (write_row, update_row, delete_row)
   - Scan operations (rnd_init, rnd_next, rnd_end)
   - Index operations (index_read_map, etc.)
   - Transaction hooks (external_lock, start_stmt)
   - Info/metadata (info, table_flags, index_flags)
   - DDL (check_if_supported_inplace_alter, etc.)

3. What private/protected member variables does the class hold? What state
   does each one track (e.g., current result set, row position, batch mode)?

4. Look at `table_flags()` — what capabilities does `ha_duckdb` advertise to
   MySQL? What does each flag mean (HA_BINLOG_ROW_CAPABLE, HA_NO_AUTO_INCREMENT, etc.)?

5. How does the Partition_handler integration work? What partition-related
   methods are defined?

Summarize your findings as a "cheat sheet" of the handler interface that
someone could reference while reading the .cc implementation.
```

---

## Step 2: Plugin Registration — How DuckDB Becomes a MySQL Engine

### Prompt
```
Read `storage/duckdb/ha_duckdb.cc`, focusing on lines 1-290 (the plugin
registration and initialization section).

This is where DuckDB registers itself as a MySQL storage engine via the
`mysql_declare_plugin` macro and the `duckdb_init_func` function.

Please answer the following questions:

1. Walk through `duckdb_init_func()` step by step. What does each line do?
   What function pointers are assigned to the `handlerton` struct?

2. Explain the `mysql_declare_plugin(duckdb) { ... } mysql_declare_plugin_end`
   block. What metadata does it provide? What is `duckdb_storage_engine`?

3. Find and explain every `static` function defined before `duckdb_init_func`:
   - `duckdb_commit()` — what happens during commit?
   - `duckdb_rollback()` — what happens during rollback?
   - `duckdb_prepare()` — what is this for (hint: 2PC/binlog)?
   - `duckdb_close_connection()` — cleanup logic?
   - `duckdb_create_handler()` — what does it return?

4. What system variables are registered (`MYSQL_SYSVAR`)? List each one with
   its type, default value, and purpose.

5. What status variables are exposed (`SHOW_VAR`)? What metrics do they track?

6. Compare this to how InnoDB registers itself (conceptually). What's similar?
   What's different about DuckDB's approach?

Draw a sequence diagram (in text/ASCII) showing what happens from MySQL
startup to the point where DuckDB is ready to handle queries.
```

---

## Step 3: Type Mapping — MySQL Types to DuckDB Types

### Prompt
```
Read `storage/duckdb/ddl_convertor.cc` and `storage/duckdb/ddl_convertor.h`.

Focus specifically on the type conversion logic — the function
`FieldConvertor::convert_type()` (around lines 212-336 in the .cc file) and
the `CreateTableConvertor::translate()` method.

Please answer the following questions:

1. Walk through `convert_type()` case by case. For each MySQL field type
   (MYSQL_TYPE_TINY, MYSQL_TYPE_LONG, MYSQL_TYPE_DECIMAL, etc.), what DuckDB
   type string does it produce? How does it handle the UNSIGNED flag?

2. How are DECIMAL types handled? What happens when precision exceeds 38
   (DuckDB's limit)? Why is DOUBLE used as a fallback?

3. How are string types (VARCHAR, CHAR, TEXT, BLOB) mapped? What role does
   charset play in deciding between `varchar` and `blob`?

4. How are temporal types mapped?
   - Why is TIMESTAMP mapped to `timestamptz` but DATETIME to `datetime`?
   - How is YEAR handled?

5. What types are mapped to `blob` as a catch-all? Why?

6. Look at `CreateTableConvertor::translate()`. How does it assemble a
   complete CREATE TABLE statement? How are PRIMARY KEY and UNIQUE constraints
   handled? How are column defaults translated?

7. Find the `BaseConvertor` class hierarchy. List all convertor subclasses
   and what DDL operation each handles. Show the inheritance tree.

Create a reference table: MySQL type -> DuckDB type -> special notes.
```

---

# Part 2: Direct Write Path — Client INSERT to DuckDB

How data gets written when a client directly INSERTs into a DuckDB table.

---

## Step 4: Write Path — How INSERT/UPDATE/DELETE Reaches DuckDB

### Prompt
```
Read `storage/duckdb/ha_duckdb.cc` focusing on the write path, then read
`storage/duckdb/dml_convertor.cc` and `storage/duckdb/dml_convertor.h`.

Trace the complete journey of an INSERT statement from MySQL to DuckDB.

Start with `ha_duckdb::write_row()` (around line 608 in ha_duckdb.cc):

1. What does `write_row()` receive as input? What is the `buf` parameter?
   How does MySQL's row format work at this level?

2. What is the `dml_in_batch` check? What are the two paths (batch vs
   non-batch)? When is each used?

3. For the non-batch path: trace into `InsertConvertor::translate()`.
   - How does it build the "INSERT INTO `db`.`tbl` VALUES (...)" string?
   - How are individual field values serialized? (Look at the field iteration)
   - How are NULLs, strings, numbers, dates, and BLOBs formatted?

4. How does `duckdb_query()` get called to execute the generated SQL?

5. Now read `ha_duckdb::update_row()` (around line 663) and
   `ha_duckdb::delete_row()` (around line 747):
   - How does UPDATE detect which columns changed?
   - What is `duckdb_update_modified_column_only` and why does it matter?
   - How does UPDATE handle primary key changes (the DELETE + INSERT pattern)?
   - How does DELETE build its WHERE clause? How does it identify the row?

6. What is "idempotent replay"? Where is `duckdb_idempotent_flag` checked?
   Why is this important for replication?

7. What status counters are updated (duckdb_rows_insert, etc.)?

Create an ASCII flow diagram showing the INSERT path from write_row() through
to DuckDB execution.
```

---

## Step 5: Batch Write Optimization — The Delta Appender

### Prompt
```
Read `storage/duckdb/delta_appender.h` and `storage/duckdb/delta_appender.cc`
in their entirety.

The Delta Appender is the key performance optimization that enables 300K
rows/sec replication replay. Understand how it works:

1. Explain the class hierarchy:
   - What is `DeltaAppender`? What DuckDB API does it wrap (`duckdb::Appender`)?
   - What is `DeltaAppenders` (plural)? How does it manage multiple tables?
   - What is `AppenderInfo`? What metadata does it track per table?

2. Walk through the append lifecycle:
   - How is an appender created for a specific table?
   - How is `append_row()` called? What does it do with each field value?
   - How does type conversion work at the appender level vs. SQL string level?

3. Explain the flush mechanism:
   - When does `flush()` get called?
   - What happens during flush? How are accumulated rows written to DuckDB?
   - What is the `appender_allocator_flush_threshold`?

4. How does rollback work?
   - What happens to accumulated rows on ROLLBACK?
   - How does `rollback(ulonglong trx_no)` handle partial rollback
     to a transaction boundary?

5. How does the batch state machine work?
   - Explain the transitions: UNDEFINED -> NOT_IN_BATCH ->
     IN_INSERT_ONLY_BATCH -> IN_MIX_BATCH
   - Why distinguish between INSERT-only and mixed batches?
   - Where is the batch state managed? (Look at `duckdb_context.h`)

6. Compare performance: Why is the Appender API so much faster than
   individual INSERT SQL statements? What overhead does it avoid?

7. Find where `write_row()` in ha_duckdb.cc decides to use the appender
   vs. generating SQL. What conditions trigger each path?
```

---

# Part 3: Query Read Path — MySQL Parser to DuckDB Results

The complete journey of a SELECT query from client to results. This covers
the MySQL parser, optimizer, handler dispatch, DuckDB SQL translation,
DuckDB internal execution, and result conversion back to MySQL format.

---

## Step 6: Complete Query Pipeline — From SQL Text to Result Set

### Prompt
```
Trace the COMPLETE journey of a SELECT query on a DuckDB table, from the
moment the MySQL client sends SQL to the moment results are returned.

Key files to read:
- `sql/sql_parse.cc` (MySQL query entry point — dispatch_command)
- `sql/sql_optimizer.cc` (query optimization)
- `sql/sql_executor.cc` (query execution — calls handler methods)
- `storage/duckdb/ha_duckdb.cc` (rnd_init, rnd_next, index_read_map)
- `storage/duckdb/duckdb_select.cc` (result conversion — the core)
- `storage/duckdb/duckdb_select.h`
- `sql/duckdb/duckdb_query.cc` (query execution gateway to DuckDB)
- `sql/duckdb/duckdb_context.cc` (per-thread DuckDB connection)
- `sql/duckdb/duckdb_timezone.cc` (timezone handling for TIMESTAMP)

Trace this exact query:

    SELECT user_id, COUNT(*) as cnt, SUM(amount) as total
    FROM orders
    WHERE created_at >= '2025-01-01'
    GROUP BY user_id
    HAVING total > 1000
    ORDER BY total DESC
    LIMIT 10;


**Phase 1: MySQL Parser**

1. How does MySQL parse this query?
   - `dispatch_command()` in sql_parse.cc receives the SQL text
   - The parser (Bison-generated) produces a parse tree (SELECT_LEX)
   - What does the parse tree contain? (table refs, WHERE, GROUP BY, etc.)
   - At this point, MySQL doesn't know it's a DuckDB table yet

2. How does MySQL discover the table's storage engine?
   - Table metadata lookup (TABLE_SHARE, frm file or data dictionary)
   - The table was created with ENGINE=DUCKDB
   - How does this affect subsequent optimization?


**Phase 2: MySQL Optimizer**

3. What does the MySQL optimizer do with a DuckDB table?
   - Does it generate an execution plan? (YES — MySQL always optimizes)
   - Does the optimizer push down WHERE, GROUP BY, HAVING, ORDER BY,
     or LIMIT to the storage engine?
   - What does `ha_duckdb::info()` return for row estimates?
   - How does `ha_duckdb::index_flags()` and `table_flags()` affect
     the optimizer's decisions?
   - KEY QUESTION: Does MySQL try to use indexes on DuckDB tables?

4. What execution plan does MySQL generate?
   - Full table scan via rnd_init/rnd_next? Or index scan?
   - Does MySQL plan to do GROUP BY/ORDER BY itself, or expect
     the engine to handle it?


**Phase 3: MySQL Executor → DuckDB Handler**

5. Trace `ha_duckdb::rnd_init()` (around line 868 in ha_duckdb.cc):
   - What does `rnd_init(bool scan)` do?
   - How is the DuckDB SELECT SQL constructed?
     - Does it include WHERE? GROUP BY? ORDER BY? LIMIT?
     - Or does it generate `SELECT * FROM orders` and let MySQL filter?
   - What is the exact DuckDB SQL generated for this query?
   - Find the code that builds this SQL string
   - How is the result set obtained and stored?
   - What is the "materialization" step — are all rows fetched upfront?

6. How is the query sent to DuckDB?
   - `duckdb_query()` in duckdb_query.cc
   - How does it get the current thread's DuckDB connection?
   - What session config is applied (timezone, collation)?
   - How is the result materialized? (MaterializedResult vs streaming?)


**Phase 4: DuckDB Internal Processing**

7. What happens inside DuckDB when it receives the query?
   - DuckDB has its own parser → optimizer → executor
   - DuckDB uses columnar storage — how does this help for analytics?
   - DuckDB generates a physical plan with operators:
     - TableScan → Filter → HashAggregate → Sort → Limit
   - How does DuckDB's vectorized execution differ from MySQL's
     row-at-a-time model?

8. Where is the performance advantage?
   - Columnar storage: only reads `user_id`, `amount`, `created_at` columns
   - Vectorized execution: processes thousands of rows at once
   - Compression: columnar data compresses much better
   - Compare: InnoDB would do a row-oriented full table scan, reading
     ALL columns, processing one row at a time


**Phase 5: Result Conversion — DuckDB Back to MySQL**

9. Trace `ha_duckdb::rnd_next()` (around line 914):
   - How does it iterate through the DuckDB result set?
   - How does it signal end-of-data (HA_ERR_END_OF_FILE)?
   - What is the `buf` parameter and how is it populated?

10. Read `store_duckdb_field_in_mysql_format()` in duckdb_select.cc (line 77+):
    This is the core result conversion function. Walk through EVERY type case:
    - Integer types (TINYINT through BIGINT, signed and unsigned)
    - Float/Double
    - Decimal — how is DuckDB's decimal converted to MySQL's fixed-point?
    - DATE, DATETIME, TIMESTAMP — how is timezone handling done?
      (Cross-reference with `sql/duckdb/duckdb_timezone.cc`)
    - TIME — any special conversion?
    - VARCHAR and BLOB — how is charset handled?
    - JSON, ENUM, SET — special handling?
    - How are NULL values handled?

11. For this specific query, trace the conversion of each result column:
    - DuckDB integer → MySQL INT (user_id)
    - DuckDB bigint → MySQL BIGINT (cnt from COUNT(*))
    - DuckDB decimal/double → MySQL DECIMAL (total from SUM(amount))
    - How are NULL values handled in aggregation results?


**Phase 6: Division of Labor & Wire Protocol**

12. Who does what? Trace the division of labor:
    - WHERE filtering: MySQL or DuckDB?
    - GROUP BY aggregation: MySQL or DuckDB?
    - HAVING filtering: MySQL or DuckDB?
    - ORDER BY sorting: MySQL or DuckDB?
    - LIMIT: MySQL or DuckDB?
    - Answer for EACH: which engine performs it, and why?

13. KEY INSIGHT: What is the "double execution" problem?
    - MySQL optimizer may plan GROUP BY + ORDER BY
    - DuckDB may ALSO do GROUP BY + ORDER BY internally
    - Is there redundant work? How is this handled?
    - Is there any query pushdown optimization to avoid this?

14. Look at the index access methods:
    - What does `index_read_map()` do? Is it fully implemented?
    - What does this mean for WHERE clause evaluation?

15. How does `rnd_end()` clean up?

16. How do results get back to the MySQL client?
    - MySQL sends column metadata (field names, types)
    - MySQL sends rows via the MySQL wire protocol
    - The client sees standard MySQL result set — unaware of DuckDB

17. Also read `storage/duckdb/duckdb_types.cc` — what utility functions
    does it provide? How are database/table names parsed from MySQL's
    path format?

Create an ASCII diagram showing the complete query pipeline:
  Client SQL → MySQL Parser → MySQL Optimizer → MySQL Executor
  → ha_duckdb::rnd_init() → DuckDB SQL construction → DuckDB Parser
  → DuckDB Optimizer → DuckDB Executor → DuckDB Result
  → rnd_next() loop → store_duckdb_field_in_mysql_format()
  → MySQL wire protocol → Client
```

---

# Part 4: Engine Infrastructure

The supporting systems that make everything work: per-thread connections,
the DuckDB singleton instance, and the query execution gateway.

---

## Step 7: Per-Thread Context — Connection & Transaction Lifecycle

### Prompt
```
Read `sql/duckdb/duckdb_context.h` and `sql/duckdb/duckdb_context.cc`
in their entirety.

The `DuckdbThdContext` class is the per-thread "bridge" between a MySQL
connection and DuckDB. Every MySQL thread that touches DuckDB tables gets one.

1. Examine the `DuckdbThdContext` class:
   - What member variables does it hold?
   - What is `m_con` (the DuckDB connection)?
   - What is `m_appenders` (the DeltaAppenders)?
   - What is `batch_state`? What are the possible states?
   - What is `batch_gtid_set`? Why track GTIDs here?

2. Trace the transaction lifecycle methods:
   - `duckdb_trans_begin()` — when is this called? What SQL does it execute?
   - `duckdb_trans_commit()` — what happens? Does it flush appenders first?
   - `duckdb_trans_rollback()` — what cleanup occurs?
   - `duckdb_register_trx()` — how does MySQL's handler framework trigger this?

3. Explain the session configuration (`DuckdbSessionConfig`):
   - What settings are propagated to DuckDB per-session?
   - How is the current database (USE db) synchronized?
   - How is timezone communicated to DuckDB?
   - How are collation settings handled?

4. How is the context attached to a MySQL THD (thread)?
   - Find where `thd->get_duckdb_context()` is defined/used
   - How is the context created and destroyed?
   - Is there one DuckDB connection per MySQL thread, or a pool?

5. How does batch mode coordinate with the context?
   - When does the context enter batch mode?
   - How are multiple tables' appenders managed simultaneously?
   - What happens at batch commit vs. batch rollback?

6. Cross-reference with `ha_duckdb.cc`: find every place that calls
   `thd->get_duckdb_context()` and categorize what operation each caller does.
```

---

## Step 8: DuckDB Instance Management — Startup & Shutdown

### Prompt
```
Read `sql/duckdb/duckdb_manager.h` and `sql/duckdb/duckdb_manager.cc`.

The `DuckdbManager` is the singleton that owns the DuckDB database instance
and manages its lifecycle within the MySQL server process.

1. Examine the `DuckdbManager` class:
   - How is it implemented as a singleton?
   - What does `init()` do? Walk through the initialization sequence.
   - What DuckDB configuration is applied during init
     (memory limit, threads, temp directory, etc.)?
   - Where is the DuckDB database file located?

2. Connection management:
   - How are DuckDB connections created per-thread?
   - Is there a connection pool, or is each connection fresh?
   - How is connection cleanup handled when a MySQL thread exits?

3. Crash recovery:
   - How does the manager detect if DuckDB needs recovery?
   - What recovery steps are performed?

4. Shutdown sequence:
   - What does `deinit()` do?
   - How is a clean shutdown ensured?

5. Cross-reference with `sql/duckdb/duckdb_config.cc`:
   - How do MySQL system variables (SET GLOBAL duckdb_memory_limit, etc.)
     propagate to the running DuckDB instance?
   - Which settings are dynamic (changeable at runtime) vs. static?
   - Read the update callback functions — what validation do they perform?

6. Also look at `cmake/duckdb.cmake` and `storage/duckdb/CMakeLists.txt`:
   - How is the DuckDB library built? (static library? shared?)
   - What build command is used? (`make bundle-library`)
   - How is it linked into the MySQL server binary?
   - What other dependencies are linked (RapidJSON, unordered_dense)?
```

---

## Step 9: Query Execution Gateway

### Prompt
```
Read `sql/duckdb/duckdb_query.h` and `sql/duckdb/duckdb_query.cc`.

This is the central gateway — every DuckDB operation (DDL, DML, SELECT)
ultimately calls through `duckdb_query()`.

1. Examine the `duckdb_query()` function signature and its overloads:
   - What parameters does it accept?
   - How does it obtain the DuckDB connection for the current thread?
   - What session configuration is applied before executing the query?

2. Trace the execution path:
   - How is the SQL string passed to DuckDB for execution?
   - How are results obtained? (DuckDB's `MaterializedResult` vs streaming?)
   - How are errors from DuckDB translated to MySQL error codes?

3. Result handling:
   - How does the function send results back to the MySQL client?
   - How does it handle DDL statements (no result set) vs. SELECT (result set)?

4. Cross-reference: find every caller of `duckdb_query()` in the codebase:
   - In `ha_duckdb.cc` (DDL: create/drop/alter/rename, DML: insert/update/delete)
   - In `duckdb_context.cc` (transaction BEGIN/COMMIT/ROLLBACK)
   - In `delta_appender.cc` (if any)
   - Anywhere else?

5. Read `sql/duckdb/duckdb_mysql_udf.cc`:
   - What MySQL UDFs are re-implemented for DuckDB?
   - How are MySQL-specific SQL functions made available inside DuckDB?
   - Give examples of string, date, and numeric functions ported.

6. Read `sql/duckdb/duckdb_charset_collation.cc` and `duckdb_timezone.cc`:
   - How does MySQL's charset/collation system integrate with DuckDB?
   - How are timezone conversions handled for TIMESTAMP fields?
   - What edge cases exist (e.g., MySQL timezone names vs. UTC offsets)?
```

---

# Part 5: InnoDB to DuckDB Replication — The Complete Data Sync Path

This is the core workflow for the DuckDB replica use case. These steps
trace data from a client INSERT on an InnoDB primary all the way to
it becoming queryable on a DuckDB replica.

**Read these steps in order — each builds on the previous one.**

---

## Step 10: The Complete Replication Path — InnoDB Write to DuckDB Sync

### Prompt
```
Trace the COMPLETE journey of a row written via InnoDB on the primary
to its eventual appearance in DuckDB on the replica. This is the
fundamental data sync mechanism — understanding it end-to-end is
critical for anyone operating or debugging an AliSQL DuckDB replica.

Key files to read:
- `sql/handler.cc` (MySQL handler commit path)
- `sql/binlog.cc` (binlog write path — how row events enter the binlog)
- `sql/log_event.cc` (search for "duckdb" — binlog event application hooks)
- `sql/rpl_rli.h` (search for "duckdb" — replication relay log info)
- `storage/duckdb/ha_duckdb.cc` (lines 536-750 — batch state + DML handlers)
- `sql/duckdb/duckdb_context.h` (lines 42-47 — BatchState enum)
- `sql/duckdb/duckdb_context.cc` (lines 234-286 — duckdb_delay_commit)

Trace this exact scenario step by step:

    -- On InnoDB primary:
    INSERT INTO orders (id, user_id, amount) VALUES (1, 42, 99.99);
    -- With a child table:
    INSERT INTO order_items (order_id, product_id) VALUES (1, 100);


**Phase 1: InnoDB Primary — Write + Binlog Generation**

1. How does MySQL process the INSERT on InnoDB?
   - Parser → Optimizer → Executor → ha_innobase::write_row()
   - At what point does MySQL write to the binlog?
   - What does a WRITE_ROWS binlog event contain? (row image format)
   - How does `binlog_format=ROW` affect what's logged?
   - When is the binlog event fsynced to disk?

2. What about auto-increment?
   - InnoDB resolves auto-inc values BEFORE writing to binlog
   - The binlog contains concrete values, not auto-inc directives
   - Verify: read the WRITE_ROWS event format — it's a full row image

3. What about foreign key cascades? (CRITICAL LIMITATION)
   - If `order_items` has `ON DELETE CASCADE` referencing `orders`,
     and you `DELETE FROM orders WHERE id=1`:
   - In MySQL 5.6-8.4: cascading child deletes happen INSIDE InnoDB
     and are NOT written to the binlog (Bug #32506, 18 years unfixed)
   - In MySQL 9.6+: FK cascades moved to SQL layer, now in binlog
   - Since AliSQL is based on MySQL 5.6: cascading deletes WILL NOT
     reach DuckDB, causing data inconsistency
   - Workaround: avoid ON DELETE/UPDATE CASCADE; use explicit DML


**Phase 2: Binlog Transmission — Primary to Replica**

4. How does the binlog stream reach the replica?
   - Binlog dump thread on primary sends events
   - I/O thread on replica writes to relay log
   - What is the relay log? (local copy of binlog events)


**Phase 3: Relay Log Replay — SQL Thread to DuckDB Handler**

5. Trace the SQL thread replay path:
   - SQL thread reads relay log events
   - `Log_event::do_apply_event()` dispatches each event
   - For WRITE_ROWS: unpacks row image → calls `ha_duckdb::write_row(buf)`
   - For UPDATE_ROWS: unpacks before+after images → `ha_duckdb::update_row()`
   - For DELETE_ROWS: unpacks row image → `ha_duckdb::delete_row()`
   - Find the exact code path in `sql/log_event.cc` where DuckDB is invoked

6. Row-based event translation — for each event type, trace how the
   binary row image is translated:
   - WRITE_ROWS → ha_duckdb::write_row() → InsertConvertor or DeltaAppender
   - UPDATE_ROWS → ha_duckdb::update_row() → UpdateConvertor (with PK
     change detection via calc_pk_difference())
   - DELETE_ROWS → ha_duckdb::delete_row() → DeleteConvertor

7. How does write_row() on the replica differ from a direct write?
   - Is batch mode (DeltaAppender) used? Under what conditions?
   - How does `duckdb_dml_in_batch` affect the path?

8. Batch state determination — read ha_duckdb.cc lines 536-568.
   How does the handler decide between:
   - NOT_IN_BATCH (direct SQL per row)
   - IN_INSERT_ONLY_BATCH (appender, inserts only)
   - IN_MIX_BATCH (temp table with delete flags)
   What conditions trigger each? How do dml_in_batch, idempotent_flag,
   and duckdb_multi_trx_in_batch interact?


**Phase 4: Batch Accumulation + Commit (overview)**

9. Multiple transactions are batched into a single DuckDB commit:
   - TX1 commits on primary → relay log has COMMIT event
   - But DuckDB may DELAY this commit (duckdb_delay_commit)
   - TX2, TX3, TX4 arrive → rows accumulate in DeltaAppender
   - Batch boundary hit (256MB or 5s timeout or DDL seen)
   - flush_appenders() → 2PC prepare → binlog sync → COMMIT
   → **Deep-dive on batch mechanics: see Step 11**
   → **Deep-dive on 2PC + crash recovery: see Step 12**


**Phase 5: Data Consistency & Known Gaps**

10. Relay log coordinate tracking: find where `xid_event_relay_log_pos`
    and `xid_event_relay_log_name` are set in duckdb_context. Why does
    DuckDB need to track these? How are they used during crash recovery?

11. After the batch commits, verify:
    - Row is now queryable in DuckDB
    - GTID is marked as executed
    - Relay log position is advanced

12. What data is NOT synced? List all known gaps:
    - FK cascade effects (MySQL 5.6-8.4 — Bug #32506)
    - Temporary tables
    - Non-deterministic functions in STATEMENT format
    - Any other known limitations?

13. How does this compare to other CDC approaches (Debezium, Maxwell)?
    What are the advantages of being inside the MySQL process?

Create a complete ASCII timeline showing: client write → InnoDB → binlog
→ network → relay log → SQL thread → ha_duckdb → DeltaAppender → batch
commit → queryable in DuckDB. Mark timing at each phase.
```

---

## Step 11: Multi-Transaction Batching — 300K Rows/Sec Replay

### Prompt
```
Deep-dive into the multi-transaction batch mechanism that allows DuckDB
to replay replication at 300K rows/sec. This is the key performance
innovation.

**Prerequisite**: Complete Step 10 first to understand where batching
fits in the overall replication path.

Key files to read:
- `sql/duckdb/duckdb_context.cc` (lines 234-520 — batch orchestration)
- `sql/duckdb/duckdb_context.h` (BatchState, batch_gtid_set, etc.)
- `storage/duckdb/delta_appender.cc` (lines 276-326 — flush and rollback)
- `storage/duckdb/delta_appender.h` (DeltaAppender class)
- `sql/log_event.cc` (lines 3215-3227, 4483-4486 — batch commit triggers)

Please answer the following questions:

1. **Batch accumulation**: Trace `duckdb_delay_commit()` in duckdb_context.cc
   (lines 234-286):
   - When is this called? (Hint: from log_event.cc when COMMIT is processed)
   - What decides whether to delay the commit or flush?
   - What are `duckdb_multi_trx_max_batch_length` (default 256MB) and
     `duckdb_multi_trx_timeout` (default 5000ms)?
   - How does `cur_batch_length` accumulate?

2. **GTID tracking in batches**: Read the `batch_gtid_set` management:
   - `add_gtid_to_batch_set()` — when called, what does it store?
   - `save_batch_gtid_set()` — how are batch GTIDs persisted?
   - `prepare_gtids_for_binlog_commit()` (lines 317-335) — how does this
     ensure proper GTID ordering when the batch finally commits?
   - Why is GTID tracking essential for crash recovery?

3. **Batch boundary detection**: Read `need_implicit_commit_batch()`
   (lines 448-493). Explain the state machine:
   - INITIAL → GTID → GTID_BEGIN → ...
   - What events trigger transitions?
   - When does it decide "this batch must commit now"?

4. **The actual commit**: Trace `commit_if_possible()` (lines 414-446):
   - How does it create a synthetic `Xid_log_event`?
   - Why does it manipulate relay log coordinates before/after?
   - How does this trigger the 2PC chain (prepare → binlog sync → commit)?

5. **DeltaAppender flush during batch commit**: Read delta_appender.cc
   lines 276-309:
   - When `m_use_tmp_table=true` (mixed DML), explain the temp table
     strategy with `#alibaba_rds_delete_flag`, `#alibaba_rds_row_no`,
     `#alibaba_rds_trx_no` columns.
   - What SQL does flush() generate for the DELETE phase? INSERT phase?
   - Why is this approach faster than individual SQL statements?

6. **Partial rollback within a batch**: Read delta_appender.cc lines 311-326.
   If one transaction in a batch fails:
   - How does `rollback(ulonglong trx_no)` isolate just that transaction?
   - What happens to other transactions in the batch?

7. **Performance comparison**: Why is batching 30x faster than row-by-row?
   List every source of overhead that batching eliminates (SQL parsing,
   transaction begin/commit, fsync, etc.).

Create a timeline diagram showing 5 binlog transactions being batched
into a single DuckDB commit, with GTID tracking at each step.
```

---

## Step 12: Two-Phase Commit & Crash Recovery

### Prompt
```
Understand how DuckDB maintains crash-safe consistency with MySQL's binlog
via two-phase commit (2PC) and idempotent replay.

**Prerequisite**: Complete Steps 10-11 first to understand the replication
and batching context.

Key files to read:
- `storage/duckdb/ha_duckdb.cc` (lines 102-183 — 2PC handlers)
- `sql/duckdb/duckdb_context.cc` (lines 381-412 — GTID commit/rollback)
- `sql/rpl_rli.h` (lines 677-693 — idempotent flags)
- `mysql-test/suite/duckdb/t/transaction_crash_recovery.test`
- `mysql-test/suite/duckdb/t/ha_duckdb_rpl_batch_multi_trx_crash.test`

Please answer the following questions:

1. **2PC handler chain**: Read ha_duckdb.cc lines 102-160 and trace
   the exact sequence MySQL calls during commit:

   a. `duckdb_prepare()` (line 102):
      - What does it do? (flush appenders to DuckDB, but don't commit yet)
      - What happens if flush fails?
      - Why is this the "prepare" phase — what guarantee does it provide?

   b. `duckdb_set_prepared_in_tc()` (line 117):
      - What is the transaction coordinator (TC)?
      - Why is this currently a no-op?

   c. Binlog sync happens here (external to DuckDB handler):
      - At this point, binlog is fsync'd to disk
      - DuckDB data is flushed but NOT committed

   d. `duckdb_commit()` (line 122):
      - What does it do? (execute COMMIT in DuckDB)
      - What if COMMIT fails? (rolls back, returns error)
      - Note the debug point `crash_after_duckdb_commit`

   e. `duckdb_rollback()` (line 146):
      - When is this called instead of commit?
      - What cleanup does it perform?

2. **Crash scenario analysis**: For each scenario, explain what happens
   on recovery:

   Scenario A: Crash BEFORE duckdb_prepare()
   - Binlog status? DuckDB status? Recovery action?

   Scenario B: Crash DURING duckdb_prepare() (appender flush)
   - Binlog status? DuckDB status? Recovery action?

   Scenario C: Crash AFTER binlog sync, BEFORE duckdb_commit()
   - Binlog status? DuckDB status? Recovery action?
   - This is the critical case — explain how idempotent replay fixes it

   Scenario D: Crash AFTER duckdb_commit()
   - Everything committed? Any cleanup needed?

3. **Idempotent replay mechanism**: Read rpl_rli.h lines 677-693 and
   the idempotent paths in ha_duckdb.cc:
   - What is `m_duckdb_idempotent_cnt`? How is it set on recovery?
   - What is `m_duckdb_idempotent_flag`? When is it true?
   - For INSERT with idempotent=true: why DELETE before INSERT? (line 638-646)
   - For UPDATE with PK change + idempotent: why DELETE(old) + DELETE(new)
     + INSERT? (line 686-705)
   - Why is "delete then insert" safe for idempotent replay?

4. **GTID consistency on commit/rollback**: Read duckdb_context.cc
   lines 381-412:
   - `update_on_commit()` — how does it mark batch GTIDs as executed?
   - `update_on_rollback()` — how does it undo GTID ownership?
   - Why iterate through each GTID individually?

5. **Transaction registration**: Read `duckdb_register_trx()` (line 164-183):
   - Why does it check for XA transactions? (DuckDB refuses XA)
   - How does `trans_register_ha()` enlist DuckDB in 2PC?
   - What is the difference between statement scope and transaction scope?

6. **Read the crash recovery tests**:
   - `transaction_crash_recovery.test` — what crash points are tested?
   - `ha_duckdb_rpl_batch_multi_trx_crash.test` — how is batch crash tested?
   - What does `rpl_diff` verify after recovery?

Draw an ASCII timeline of the 2PC sequence:
  MySQL TX start → DML → prepare(flush) → binlog sync → commit → cleanup
Mark each crash point and its recovery strategy.
```

---

## Step 13: Bootstrapping a DuckDB Replica — Initial Sync

### Prompt
```
Explore how a DuckDB replica is initially bootstrapped — i.e., how
existing InnoDB data gets into DuckDB before replication starts.

**Context**: Steps 10-12 cover ongoing replication. This step covers the
one-time setup that happens before replication begins.

Key files to read:
- `mysql-test/suite/duckdb/t/convert_all_to_duckdb_at_startup_1.test`
- `mysql-test/suite/duckdb/t/convert_all_to_duckdb_at_startup_2.test`
- `mysql-test/suite/duckdb/t/convert_all_to_duckdb_rpl.test`
- `mysql-test/suite/duckdb/t/convert_all_to_duckdb_at_start_remove_fk.test`
- `mysql-test/suite/duckdb/t/duckdb_data_import_mode.test`
- `mysql-test/suite/duckdb/t/parallel_copy_ddl.test`
- `mysql-test/suite/duckdb/t/supported_copy_ddl.test`
- Search for `convert_all_to_duckdb` and `duckdb_data_import_mode` in
  `sql/duckdb/` and `storage/duckdb/`

Please answer the following questions:

1. **Auto-conversion at startup**: What is the `convert_all_to_duckdb`
   mechanism?
   - Is there a system variable or startup flag that triggers it?
   - How does it iterate over existing tables and convert them?
   - How are foreign keys handled? (They're not supported in DuckDB)
   - What happens to indexes? (DuckDB doesn't enforce uniqueness)

2. **Parallel copy DDL**: Read `parallel_copy_ddl.test` and search for
   `duckdb_copy_ddl_threads` in the source:
   - How does parallel copy work?
   - How many threads can be used?
   - What is the claimed 7x speedup mechanism?
   - How is data consistency maintained during parallel copy?

3. **Copy DDL in batch mode**: How does `duckdb_copy_ddl_in_batch` work?
   - How are rows copied from InnoDB to DuckDB?
   - Does it use the DeltaAppender or direct SQL?
   - What batch size / flush threshold is used?

4. **Data import mode**: What is `duckdb_data_import_mode`?
   - How does it differ from normal operation?
   - What restrictions does it impose (no UPDATE? DELETE only by PK?)?
   - When would you use it vs. the auto-conversion?

5. **End-to-end bootstrap sequence**: Describe the complete flow:
   a. Start MySQL with InnoDB tables on primary
   b. Enable DuckDB mode on replica
   c. Initial data migration (convert_all_to_duckdb or copy DDL)
   d. Start replication (binlog position or GTID-based)
   e. Ongoing CDC via batch replay (→ Steps 10-12)
   f. DuckDB replica serves analytical queries (→ Step 6)

6. **Foreign key removal**: Read `convert_all_to_duckdb_at_start_remove_fk.test`:
   - DuckDB doesn't support FKs. How are they stripped during conversion?
   - Is this automatic or does it require manual intervention?

Draw the complete lifecycle: InnoDB primary → bootstrap DuckDB replica
→ start replication → steady-state CDC → crash → recovery → resume.
```

---

# Part 6: Transaction Isolation

---

## Step 14: Transaction Isolation — What Guarantees Does DuckDB Provide?

### Prompt
```
Understand the transaction isolation model of the DuckDB storage engine
in AliSQL, and how it interacts with MySQL's transaction framework.

Key files to read:
- `storage/duckdb/ha_duckdb.h` (table_flags — transaction capabilities)
- `storage/duckdb/ha_duckdb.cc` (lines 102-183 — 2PC handlers,
  lines 164-183 — duckdb_register_trx)
- `sql/duckdb/duckdb_context.cc` (duckdb_trans_begin/commit/rollback)
- `sql/duckdb/duckdb_context.h` (transaction state)
- Search for "isolation" across all duckdb-related files
- `extra/duckdb/` — DuckDB's own transaction documentation (if available)

Please answer the following questions:

**1. What MySQL Advertises**

1. Look at `ha_duckdb::table_flags()` in ha_duckdb.h:
   - Does it include HA_NO_TRANSACTIONS? (No → DuckDB supports transactions)
   - Does it declare any isolation level flags?
   - Compare with InnoDB's table_flags — what's different?

2. When MySQL sets `SET TRANSACTION ISOLATION LEVEL READ COMMITTED` or
   `REPEATABLE READ`, does DuckDB honor this?
   - Search for isolation level handling in ha_duckdb.cc and duckdb_context.cc
   - Is the MySQL isolation level forwarded to DuckDB?

**2. What DuckDB Actually Provides**

3. DuckDB uses MVCC (Multi-Version Concurrency Control) with
   Snapshot Isolation:
   - Each transaction gets a consistent snapshot at BEGIN time
   - All reads within a transaction see data from that snapshot
   - Writes are isolated from concurrent readers
   - This is equivalent to SNAPSHOT ISOLATION (between READ COMMITTED
     and SERIALIZABLE in SQL standard terms)

4. How does this compare to MySQL's isolation levels?

   | MySQL Level | InnoDB Behavior | DuckDB Behavior |
   |-------------|-----------------|-----------------|
   | READ UNCOMMITTED | See uncommitted rows | NOT supported (snapshot) |
   | READ COMMITTED | See committed rows, new snapshot per statement | NOT matched (single snapshot per TX) |
   | REPEATABLE READ | Consistent snapshot for TX duration | MATCHED (this is what DuckDB does) |
   | SERIALIZABLE | Full serializability | NOT fully guaranteed |

5. What are the practical implications?
   - If MySQL session is set to READ COMMITTED, but DuckDB provides
     snapshot isolation, the behavior is STRICTER than expected
   - If MySQL session is set to SERIALIZABLE, DuckDB may allow
     anomalies that true serializability would prevent
   - In practice, for a read-replica workload, this rarely matters — why?

**3. Transaction Lifecycle**

6. Trace the complete transaction lifecycle:
   - `duckdb_register_trx()` — how/when is DuckDB enlisted in MySQL's
     transaction? What is `trans_register_ha()`?
   - `duckdb_trans_begin()` — what SQL does it execute in DuckDB?
     Is it just "BEGIN"?
   - During the transaction: how do reads and writes interact?
     Is there a consistent snapshot?
   - `duckdb_prepare()` — what happens at 2PC prepare?
   - `duckdb_commit()` / `duckdb_rollback()` — finalization

7. What about autocommit?
   - When `autocommit=ON`, each statement is its own transaction
   - How does DuckDB handle this? Is BEGIN/COMMIT wrapped per statement?
   - What is `OPTION_NOT_AUTOCOMMIT | OPTION_BEGIN` check in
     duckdb_register_trx()?

**4. Replica-Specific Isolation Concerns**

8. On a DuckDB replica receiving binlog events:
   - Multiple primary transactions may be batched into one DuckDB TX
   - What isolation does a reader see during batch accumulation?
   - Can a reader see a partial batch? (i.e., TX1 committed on primary
     but TX2 not yet — can a DuckDB reader see TX1's rows before
     the batch commits?)
   - Answer: No — the entire batch is atomic in DuckDB. Readers see
     either ALL or NONE of the batched transactions.

9. What about concurrent reads during batch replay?
   - DuckDB's MVCC allows readers during writes
   - Readers see the last committed snapshot
   - The batch being accumulated is invisible until COMMIT
   - This provides READ COMMITTED or better for replica readers

10. XA transactions:
    - Read `duckdb_register_trx()` — it checks `check_in_xa(true)`
    - DuckDB does NOT support XA transactions
    - What happens if an XA transaction tries to touch a DuckDB table?

**5. Comparison & Gaps**

11. Create a comparison table:

    | Feature | InnoDB | DuckDB in AliSQL |
    |---------|--------|-----------------|
    | Isolation model | MVCC, configurable | MVCC, snapshot only |
    | Default level | REPEATABLE READ | Snapshot (≈ REPEATABLE READ) |
    | READ UNCOMMITTED | Supported | No (always snapshot) |
    | READ COMMITTED | Supported | No (always snapshot) |
    | REPEATABLE READ | Supported | Yes (native behavior) |
    | SERIALIZABLE | Supported | No (snapshot, not serializable) |
    | Gap locks | Yes | No |
    | Savepoints | Yes | No (stub in AliSQL) |
    | XA | Yes | No |

12. For a read-replica use case (the primary design goal):
    - Why does isolation level matter less?
    - What guarantees does the user actually need?
    - How does batch commit affect perceived consistency?

Summarize: Write a clear statement of "what isolation guarantees you get
when querying a DuckDB replica" that an application developer can rely on.
```

---

# Part 7: Verification & Reference

Test cases that validate the system, design documentation, and quick
reference tables.

---

## Step 15: Test Cases — Practical Usage & Replication Verification

### Prompt
```
Explore the test directory `mysql-test/suite/duckdb/t/`.

These MySQL test files are the best way to understand actual DuckDB behavior
from a user's perspective AND to verify the replication system works correctly.

**General functionality tests:**

1. Read `mysql-test/suite/duckdb/t/create.test`:
   - What CREATE TABLE variations are tested?
   - What column types, constraints, and options are exercised?

2. Read `mysql-test/suite/duckdb/t/ha_duckdb.test`:
   - This is the main handler test. What operations does it exercise?
   - What INSERT, SELECT, UPDATE, DELETE patterns are tested?

3. Read `mysql-test/suite/duckdb/t/transaction.test`:
   - How are transactions tested?
   - What COMMIT, ROLLBACK, and implicit transaction scenarios are covered?

4. Read `mysql-test/suite/duckdb/t/feature_duckdb_data_type.test`:
   - What data type edge cases are tested?
   - What boundary values are verified (MAX_INT, precision limits, etc.)?

5. Read a few of the `alter_duckdb_*.test` files:
   - What ALTER TABLE operations are supported?
   - What operations fall back to copy DDL?

**Replication & batch replay tests:**

6. Read `mysql-test/suite/duckdb/t/ha_duckdb_rpl_batch.test`:
   - What is the test topology? (master InnoDB → slave DuckDB?)
   - What data is inserted on the master?
   - How does the slave verify data consistency?
   - What system variables are set for batch mode?

7. Read `mysql-test/suite/duckdb/t/ha_duckdb_rpl_batch_insert_only.test`:
   - What is `duckdb_insert_only_flag`?
   - How does INSERT-only batch mode differ from mixed batch?

8. Read `mysql-test/suite/duckdb/t/ha_duckdb_rpl_batch_multi_trx.test`:
   - How are multiple transactions batched together?
   - What is `duckdb_multi_trx_in_batch`?
   - How does the test verify batch boundaries?

9. Read `mysql-test/suite/duckdb/t/ha_duckdb_rpl_batch_multi_trx_crash.test`:
   - What crash scenarios are tested?
   - How is crash simulated? (debug sync points?)
   - What is verified after recovery? (data consistency, GTID state?)

10. Read `mysql-test/suite/duckdb/t/ha_duckdb_rpl_batch_multi_trx_with_ddl.test`:
    - What happens when a DDL appears in the middle of a batch?
    - Does the batch flush before the DDL?

11. Read `mysql-test/suite/duckdb/t/ha_duckdb_rpl_batch_multi_trx_with_rotate.test`:
    - What happens when binlog rotates during a batch?

12. Read `mysql-test/suite/duckdb/t/ha_duckdb_rpl_batch_multi_trx_with_stop_slave.test`:
    - What happens if STOP SLAVE is issued during a batch?

13. Read `mysql-test/suite/duckdb/t/transaction_determine_idempotent.test`:
    - How does the system determine when idempotent replay is needed?

**Bootstrap & data import tests:**

14. Read `mysql-test/suite/duckdb/t/duckdb_data_import_mode.test`:
    - What is data import mode? What restrictions does it impose?

15. Read `mysql-test/suite/duckdb/t/convert_all_to_duckdb_at_startup_1.test`:
    - How does auto-convert InnoDB→DuckDB work?

16. Read `mysql-test/suite/duckdb/t/parallel_copy_ddl.test`:
    - How is parallel DDL copy tested?

17. Look at the corresponding result files in `mysql-test/suite/duckdb/r/`
    to see expected outputs.

For each test, summarize:
- The scenario being tested
- Key assertions / verifications
- Any surprising or non-obvious behaviors
- Configuration variables involved

What are the 10 most interesting or surprising behaviors across all tests?
```

---

## Step 16: Design Rationale & Benchmarks

### Prompt
```
Read `wiki/duckdb/duckdb-en.md` (or whatever wiki documentation exists in
`wiki/duckdb/`).

Also explore any other documentation files related to DuckDB in the project.

Please answer the following questions:

1. **Design motivation**: Why was DuckDB chosen as an analytical engine
   alongside InnoDB? What problem does it solve?

2. **Architecture decisions**:
   - Why is DuckDB integrated as a storage engine rather than a separate
     process or sidecar?
   - Why use MySQL's handler interface instead of a query proxy?
   - What are the tradeoffs of this approach?

3. **Replication strategy**:
   - How does the DuckDB replica architecture work?
   - Why is batch replay important? How does it achieve 300K rows/sec?
   - What is the "zero replication lag" claim based on?

4. **Performance benchmarks**:
   - What TPC-H results are reported? At what scale factor?
   - How does DuckDB compare to InnoDB for analytical queries?
   - What is the storage efficiency difference?

5. **SQL compatibility**:
   - What is the claimed SQL compatibility percentage?
   - How was it measured (what test suite)?
   - What are the known incompatibilities?

6. **Operational model**:
   - How is DuckDB intended to be deployed? (read replica? mixed?)
   - What management overhead does it add?
   - How does backup/recovery work?

7. **Future directions**:
   - Are there any TODOs or planned features mentioned?
   - What limitations are acknowledged?

Summarize: Write a 1-page "executive summary" of why this project exists,
what it achieves, and what its boundaries are.
```

---

## Bonus: End-to-End Trace

### Prompt
```
Now that you've explored all the components, let's trace a complete query
end-to-end. Use the knowledge from all previous steps.

Trace this exact scenario:

    CREATE TABLE analytics.events (
        id BIGINT PRIMARY KEY,
        user_id INT,
        event_name VARCHAR(100),
        created_at TIMESTAMP
    ) ENGINE=DUCKDB;

    INSERT INTO analytics.events VALUES (1, 42, 'page_view', NOW());

    SELECT * FROM analytics.events WHERE user_id = 42;

For each statement, trace the COMPLETE path through the code:

1. **CREATE TABLE**:
   - MySQL parser recognizes ENGINE=DUCKDB
   - `ha_duckdb::create()` is called -> file & line
   - `CreateTableConvertor::translate()` -> the exact DuckDB SQL generated
   - Type mappings applied: BIGINT->bigint, INT->integer,
     VARCHAR(100)->varchar, TIMESTAMP->timestamptz
   - `duckdb_query()` executes the DDL

2. **INSERT**:
   - MySQL calls `ha_duckdb::write_row(buf)`
   - Batch mode check — is this a single statement? What path is taken?
   - `InsertConvertor::translate()` -> the exact DuckDB SQL generated
   - How is NOW() resolved? (MySQL evaluates it before passing to handler)
   - `duckdb_query()` executes
   - Implicit transaction: begin + commit wrapping

3. **SELECT**:
   - MySQL calls `ha_duckdb::rnd_init(true)`
   - DuckDB SELECT SQL is built
   - Query executed via `duckdb_query()`
   - `rnd_next()` called in a loop
   - `store_duckdb_field_in_mysql_format()` converts each column:
     - bigint (id=1) -> MySQL BIGINT field
     - integer (user_id=42) -> MySQL INT field
     - varchar (event_name='page_view') -> MySQL VARCHAR field
     - timestamptz (created_at) -> MySQL TIMESTAMP field (timezone conversion!)
   - `rnd_end()` cleans up

For each step, cite the exact file and function. This is your "graduation
exercise" — proving you understand the full integration.
```

---

## Quick Reference: All Key Files

### Storage Engine Handler
| File | Purpose |
|------|---------|
| `storage/duckdb/ha_duckdb.h` | Handler class definition |
| `storage/duckdb/ha_duckdb.cc` | Handler implementation, plugin registration, 2PC handlers |
| `storage/duckdb/ddl_convertor.cc/h` | DDL SQL translation + type mapping |
| `storage/duckdb/dml_convertor.cc/h` | DML SQL translation (INSERT/UPDATE/DELETE) |
| `storage/duckdb/duckdb_select.cc/h` | SELECT result conversion |
| `storage/duckdb/delta_appender.cc/h` | Batch write optimization + temp table merge |
| `storage/duckdb/duckdb_types.cc/h` | Type utilities |

### SQL Layer Integration
| File | Purpose |
|------|---------|
| `sql/duckdb/duckdb_query.cc/h` | Query execution gateway |
| `sql/duckdb/duckdb_context.cc/h` | Per-thread context, batch state machine, GTID tracking |
| `sql/duckdb/duckdb_manager.cc/h` | DuckDB instance lifecycle |
| `sql/duckdb/duckdb_config.cc/h` | System variable configuration |
| `sql/duckdb/duckdb_table.cc/h` | Table metadata & partitions |
| `sql/duckdb/duckdb_charset_collation.cc/h` | Charset/collation mapping |
| `sql/duckdb/duckdb_timezone.cc/h` | Timezone conversion |
| `sql/duckdb/duckdb_mysql_udf.cc/h` | MySQL UDF implementations |

### Replication & CDC Integration
| File | Purpose |
|------|---------|
| `sql/log_event.cc` | Binlog event application — hooks for DuckDB batch commit |
| `sql/rpl_rli.h` | Relay log info — idempotent flags, insert-only flags |
| `sql/rpl_applier_reader.cc` | Relay log reader integration |

### Build & Docs
| File | Purpose |
|------|---------|
| `cmake/duckdb.cmake` | DuckDB build configuration |
| `wiki/duckdb/` | Design documentation |

### Key Test Files
| File | What It Tests |
|------|---------------|
| `mysql-test/suite/duckdb/t/ha_duckdb.test` | Core handler operations |
| `mysql-test/suite/duckdb/t/create.test` | CREATE TABLE variations |
| `mysql-test/suite/duckdb/t/feature_duckdb_data_type.test` | Data type edge cases |
| `mysql-test/suite/duckdb/t/transaction.test` | Transaction COMMIT/ROLLBACK |
| `mysql-test/suite/duckdb/t/ha_duckdb_rpl_batch.test` | Basic replication batch replay |
| `mysql-test/suite/duckdb/t/ha_duckdb_rpl_batch_multi_trx.test` | Multi-transaction batching |
| `mysql-test/suite/duckdb/t/ha_duckdb_rpl_batch_multi_trx_crash.test` | Crash during batch replay |
| `mysql-test/suite/duckdb/t/ha_duckdb_rpl_batch_multi_trx_with_ddl.test` | DDL in middle of batch |
| `mysql-test/suite/duckdb/t/ha_duckdb_rpl_batch_multi_trx_with_rotate.test` | Binlog rotation during batch |
| `mysql-test/suite/duckdb/t/ha_duckdb_rpl_batch_multi_trx_with_stop_slave.test` | STOP SLAVE during batch |
| `mysql-test/suite/duckdb/t/transaction_crash_recovery.test` | Transaction crash recovery |
| `mysql-test/suite/duckdb/t/transaction_determine_idempotent.test` | Idempotent replay detection |
| `mysql-test/suite/duckdb/t/convert_all_to_duckdb_at_startup_1.test` | Auto-convert InnoDB→DuckDB |
| `mysql-test/suite/duckdb/t/duckdb_data_import_mode.test` | Bulk data import mode |
| `mysql-test/suite/duckdb/t/parallel_copy_ddl.test` | Parallel DDL copy |

---

## Step-to-Workflow Map

For quick navigation, here's how the steps map to the two primary workflows:

### Workflow A: Direct DuckDB Usage
```
Step 1-3 (Foundation) → Step 4-5 (Write) → Step 6 (Read) → Step 14 (Isolation)
```

### Workflow B: InnoDB → DuckDB Replication Replica
```
Step 1-3 (Foundation) → Step 13 (Bootstrap) → Step 10 (Replication Path)
→ Step 11 (Batching) → Step 12 (Crash Recovery) → Step 6 (Query on Replica)
→ Step 14 (Isolation)
```

### Infrastructure (supporting both workflows)
```
Step 7 (Per-Thread Context) + Step 8 (Instance Management) + Step 9 (Query Gateway)
```
