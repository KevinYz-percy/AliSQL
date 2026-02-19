# AliSQL: DuckDB + MySQL Storage Engine Architecture

## High-Level Architecture

```
+------------------------------------------------------------------+
|                        Client Layer                               |
|                  MySQL Client / Application                       |
+-------------------------------+----------------------------------+
                                | SQL
                                v
+------------------------------------------------------------------+
|                      MySQL SQL Layer                              |
|                                                                   |
|  +----------------------------+   +----------------------------+  |
|  |    Parser + Optimizer      |   |     sql/duckdb/            |  |
|  +------------+---------------+   |  - duckdb_query            |  |
|               |                   |  - duckdb_context           |  |
|               v                   |  - duckdb_manager           |  |
|  +----------------------------+   |  - duckdb_config            |  |
|  |   Executor (sql_executor)  +-->|  - charset / timezone       |  |
|  +------+----------+-----+---+   +----------------------------+  |
+---------|----------|-----|---------------------------------------+
          |          |     |
          | handler API    |
          v          v     v
+------------------------------------------------------------------+
|                   Storage Engine Layer (handlerton)               |
|                                                                   |
|  +-----------+   +---------------------------+   +-------------+  |
|  |  InnoDB   |   |  storage/duckdb/          |   |  MyISAM /   |  |
|  | ha_innodb |   |                           |   |  Others     |  |
|  +-----------+   |  ha_duckdb (handler)       |   +-------------+  |
|                  |  +----------+ +----------+ |                   |
|                  |  | DDL Conv | | DML Conv | |                   |
|                  |  +----------+ +----------+ |                   |
|                  |  +----------+ +----------+ |                   |
|                  |  | duckdb   | | Delta    | |                   |
|                  |  | _select  | | Appender | |                   |
|                  |  +----------+ +----------+ |                   |
|                  |  +----------------------+  |                   |
|                  |  |    duckdb_types      |  |                   |
|                  |  +----------------------+  |                   |
|                  +------------+--------------+                    |
+---------------------------|--------------------------------------+
                            | DuckDB C++ API
                            v
+------------------------------------------------------------------+
|                  DuckDB Engine (extra/duckdb/)                    |
|                                                                   |
|  +----------------+  +------------------+  +-----------------+   |
|  | Query Processor|  | Columnar Storage |  | Transaction Mgr |   |
|  +----------------+  +--------+---------+  +-----------------+   |
+--------------------------------|---------------------------------+
                                 |
                                 v
                        +----------------+
                        |  Disk (.duckdb)|
                        +----------------+
```

---

## Directory Structure

```
AliSQL/
├── storage/duckdb/           # Storage engine handler implementation
│   ├── ha_duckdb.cc/h        # Handler class (core entry point)
│   ├── ddl_convertor.cc/h    # CREATE/ALTER/DROP TABLE translation
│   ├── dml_convertor.cc/h    # INSERT/UPDATE/DELETE translation
│   ├── duckdb_select.cc/h    # SELECT result conversion
│   ├── delta_appender.cc/h   # Batch write optimization
│   └── duckdb_types.cc/h     # MySQL <-> DuckDB type mapping
│
├── sql/duckdb/               # SQL layer integration
│   ├── duckdb_query.cc/h     # Query execution entry point
│   ├── duckdb_context.cc/h   # Per-thread connection & transaction state
│   ├── duckdb_manager.cc/h   # DuckDB instance lifecycle (singleton)
│   ├── duckdb_config.cc/h    # System variables & configuration
│   ├── duckdb_table.cc/h     # Table metadata & partition support
│   ├── duckdb_charset_collation.cc/h
│   ├── duckdb_timezone.cc/h
│   └── duckdb_mysql_udf.cc/h # MySQL UDF implementations in DuckDB
│
├── extra/duckdb/             # Bundled DuckDB source code
├── cmake/duckdb.cmake        # Build configuration
├── mysql-test/suite/duckdb/  # Test cases
└── wiki/duckdb/              # Documentation
```

---

## Key Interaction Flows

### 1. Plugin Registration

```
mysql_declare_plugin(duckdb)
        |
        v
duckdb_init_func()                    [ha_duckdb.cc:240]
        |
        +-- Sets handlerton function pointers:
            duckdb_hton->create    = duckdb_create_handler
            duckdb_hton->commit    = duckdb_commit
            duckdb_hton->rollback  = duckdb_rollback
            duckdb_hton->prepare   = duckdb_prepare
            ...
```

### 2. SELECT Query Flow

```
Client: SELECT * FROM t1 WHERE id > 10;
        |
        v
MySQL Parser + Optimizer
        |
        v
Executor calls handler methods
        |
        v
ha_duckdb::rnd_init()                [ha_duckdb.cc:868]
  |-- Builds DuckDB SELECT SQL via duckdb_select
  |-- Executes via duckdb_query()
  |-- Materializes DuckDB result set
        |
        v
ha_duckdb::rnd_next()                [ha_duckdb.cc:914]
  |-- For each row:
  |   store_duckdb_field_in_mysql_format()  [duckdb_select.cc:77]
  |     |-- Converts DuckDB column values -> MySQL field format
  |     |-- Handles type mapping (timestamptz->TIMESTAMP, etc.)
  |-- Returns HA_ERR_END_OF_FILE when done
```

### 3. INSERT Flow

```
Client: INSERT INTO t1 VALUES (1, 'hello');
        |
        v
ha_duckdb::write_row()               [ha_duckdb.cc:608]
        |
        +-- Batch mode ON (duckdb_dml_in_batch=ON)?
        |       |
        |       YES --> DeltaAppender::append_row()
        |       |       (accumulates rows in memory,
        |       |        flushed at COMMIT)
        |       |
        |       NO  --> InsertConvertor::translate()
        |               |-- "INSERT INTO `db`.`t1` VALUES (1, 'hello')"
        |               |-- duckdb_query() executes immediately
        |
        v
    Update stats: duckdb_rows_insert++
```

### 4. UPDATE Flow

```
Client: UPDATE t1 SET name='world' WHERE id=1;
        |
        v
ha_duckdb::update_row()              [ha_duckdb.cc:663]
        |
        +-- Primary key changed?
        |       |
        |       YES --> DELETE old row + INSERT new row
        |       NO  --> UpdateConvertor::translate()
        |               |-- Only modified columns (if duckdb_update_modified_column_only=ON)
        |               |-- "UPDATE `db`.`t1` SET name='world' WHERE id=1"
        |               |-- duckdb_query()
```

### 5. CREATE TABLE (DDL) Flow

```
Client: CREATE TABLE t1 (id INT, name VARCHAR(100)) ENGINE=DUCKDB;
        |
        v
ha_duckdb::create()                   [ha_duckdb.cc:1305]
        |
        v
CreateTableConvertor::translate()     [ddl_convertor.cc]
        |
        +-- For each column:
        |   FieldConvertor::convert_type()  [ddl_convertor.cc:212]
        |     MySQL INT       --> DuckDB "integer"
        |     MySQL VARCHAR   --> DuckDB "varchar"
        |     MySQL TIMESTAMP --> DuckDB "timestamptz"
        |     MySQL DECIMAL   --> DuckDB "decimal(p,s)"
        |     ... (see type mapping table below)
        |
        v
    Generates: CREATE TABLE `db`.`t1` (id integer, name varchar)
        |
        v
    duckdb_query() executes DDL in DuckDB
```

### 6. Transaction Flow

```
BEGIN (implicit on first DML)
        |
        v
duckdb_register_trx()
  |-- Creates DuckDB connection (per-thread)
  |-- Executes "BEGIN" in DuckDB
        |
        +-- DML operations (INSERT/UPDATE/DELETE)
        |   |-- Appender accumulates rows (batch mode)
        |   |-- Or individual SQL execution
        |
        v
COMMIT
        |
        v
duckdb_commit()                       [ha_duckdb.cc:122]
  |-- flush_appenders()  --> Flush all batched rows to DuckDB
  |-- duckdb_trans_commit()  --> "COMMIT" in DuckDB
  |-- srv_duckdb_status.duckdb_commit++

        --- OR ---

ROLLBACK
        |
        v
duckdb_rollback()                     [ha_duckdb.cc:146]
  |-- Discard appenders
  |-- duckdb_trans_rollback() --> "ROLLBACK" in DuckDB
```

---

## MySQL to DuckDB Type Mapping

Source: `ddl_convertor.cc:212-336`

| MySQL Type | DuckDB Type | Notes |
|---|---|---|
| `TINYINT` | `tinyint` / `utinyint` | Based on UNSIGNED flag |
| `SMALLINT` | `smallint` / `usmallint` | |
| `INT` / `MEDIUMINT` | `integer` / `uinteger` | |
| `BIGINT` | `bigint` / `ubigint` | |
| `FLOAT` | `float` | |
| `DOUBLE` | `double` | |
| `DECIMAL(p,s)` | `decimal(p,s)` | Capped at 38 digits; overflow uses DOUBLE |
| `TIMESTAMP` | `timestamptz` | Timezone-aware |
| `DATETIME` | `datetime` | |
| `DATE` | `date` | |
| `TIME` | `time` | |
| `YEAR` | `integer` | |
| `BIT` | `blob` | |
| `JSON` | `json` | |
| `ENUM` / `SET` | `varchar` | |
| `VARCHAR` / `CHAR` | `varchar` | |
| `BLOB` / `BINARY` | `blob` | |
| `GEOMETRY` | `blob` | |

---

## Configuration System Variables

Source: `ha_duckdb.cc:274-287`, `duckdb_config.h`

### Storage Engine Variables

| Variable | Default | Description |
|---|---|---|
| `duckdb_dml_in_batch` | ON | Batch INSERT/UPDATE/DELETE for performance |
| `duckdb_copy_ddl_in_batch` | ON | Batch mode for DDL copy operations |
| `duckdb_update_modified_column_only` | ON | Only update changed columns |

### DuckDB Engine Variables

| Variable | Default | Description |
|---|---|---|
| `duckdb_mode` | ON | Enable/disable DuckDB engine |
| `duckdb_memory_limit` | ~80% RAM | Max memory for DuckDB |
| `duckdb_threads` | auto (CPU cores) | Worker thread count |
| `duckdb_temp_directory` | - | Temp file location |
| `duckdb_max_temp_directory_size` | ~90% free | Max temp disk usage |
| `duckdb_require_primary_key` | ON | Enforce PRIMARY KEY on tables |
| `duckdb_use_direct_io` | OFF | Use O_DIRECT for I/O |

---

## Batch Processing: Delta Appender

Source: `delta_appender.cc/h`

The Delta Appender is the key optimization for write-heavy workloads (especially replication replay):

```
                        Normal Mode              Batch Mode
                     (1 row at a time)        (Delta Appender)

INSERT               SQL per row              Appender::append_row()
                         |                         |
                         v                    accumulate in memory
                    duckdb_query()                  |
                         |                    ... more rows ...
                         v                         |
                      COMMIT                  flush_appenders()
                                                   |
                                              bulk write to DuckDB
                                                   |
                                                COMMIT

Performance:         ~10K rows/sec            ~300K rows/sec
```

### Batch State Machine

```
UNDEFINED --> NOT_IN_BATCH -----> IN_INSERT_ONLY_BATCH
                  |                       |
                  |                       v
                  +-------------> IN_MIX_BATCH
                                (INSERT + UPDATE + DELETE)
```

---

## Replication / CDC: Binlog-Based Data Sync

DuckDB replicas stay in sync with InnoDB primaries via MySQL's row-based binlog replication. DuckDB does **not** have its own replication protocol — it reuses MySQL's relay log infrastructure.

### Overall Flow

```
InnoDB Primary                          DuckDB Replica
+------------------+                    +------------------+
| Client writes    |                    | SQL Thread reads |
| INSERT/UPDATE/   |  binlog stream     | relay log events |
| DELETE to InnoDB  +------------------>+                  |
+------------------+  (row-based)       | Log_event::      |
                                        | do_apply_event() |
                                        |       |          |
                                        |       v          |
                                        | ha_duckdb::      |
                                        | write_row()      |
                                        | update_row()     |
                                        | delete_row()     |
                                        |       |          |
                                        |       v          |
                                        | DeltaAppender    |
                                        | (batch accumulate)|
                                        |       |          |
                                        |       v          |
                                        | Batch COMMIT     |
                                        | (2PC with binlog)|
                                        +------------------+
```

### Multi-Transaction Batch Replay

The key performance innovation: instead of committing each replicated transaction individually, multiple transactions are batched into a single DuckDB commit.

```
Relay Log Events:           DuckDB Processing:

TX1: INSERT (3 rows)  ---+
TX2: INSERT (5 rows)     |---> DeltaAppender accumulates
TX3: UPDATE (2 rows)     |     all rows in memory
TX4: INSERT (4 rows)     |
TX5: DELETE (1 row)   ---+
                          |
    Batch boundary hit    |   (size > 256MB OR timeout > 5s OR DDL seen)
                          v
                    flush_appenders()
                          |
                    2PC: prepare -> binlog sync -> COMMIT
                          |
                    All 5 TXs committed atomically in DuckDB
                    All 5 GTIDs marked as executed
```

**Controlled by:**
- `duckdb_multi_trx_in_batch` — enable multi-TX batching
- `duckdb_multi_trx_max_batch_length` — max batch size (default 256MB)
- `duckdb_multi_trx_timeout` — max batch wait time (default 5000ms)

### Batch State Machine (DML Type Handling)

```
                       +-- INSERT only --> IN_INSERT_ONLY_BATCH
                       |                   (DuckDB Appender API, fastest)
dml_in_batch=ON? --YES-+
                       |
                       +-- Mixed DML ----> IN_MIX_BATCH
                                           (temp table with delete flags:
                                            #alibaba_rds_delete_flag
                                            #alibaba_rds_row_no
                                            #alibaba_rds_trx_no)

dml_in_batch=OFF? ---> NOT_IN_BATCH
                       (individual SQL per row via DML Convertor)
```

**Mixed DML temp table flush strategy:**
1. All rows appended to temp table with metadata columns
2. On flush: DELETE phase (remove rows marked for deletion by PK)
3. On flush: INSERT phase (insert remaining rows)
4. Drop temp table

### Two-Phase Commit (2PC) with Binlog

Source: `ha_duckdb.cc:102-160`

```
                  MySQL 2PC Sequence
                  ==================

ha_duckdb::             Binlog              DuckDB

1. duckdb_prepare()                         flush_appenders()
   (flush to DuckDB,                        (data written but
    don't commit yet)                        NOT committed)
         |
         v
2. [Binlog sync]        fsync() to disk
   (binlog is now
    crash-safe)
         |
         v
3. duckdb_commit()                          COMMIT
   (now safe to commit                      (data visible)
    because binlog is
    already on disk)


CRASH SCENARIOS:

Point A (before prepare):
  Binlog: NOT synced    DuckDB: nothing flushed
  Recovery: skip → SAFE (nothing to recover)

Point B (during prepare):
  Binlog: NOT synced    DuckDB: partial flush
  Recovery: DuckDB rolls back uncommitted → SAFE

Point C (after binlog sync, before commit):
  Binlog: synced        DuckDB: flushed but NOT committed
  Recovery: replay from binlog with idempotent=true → SAFE

Point D (after commit):
  Binlog: synced        DuckDB: committed
  Recovery: nothing needed → SAFE
```

### Idempotent Replay (Crash Recovery)

Source: `ha_duckdb.cc:627-705`, `rpl_rli.h:677-693`

When recovering from a crash (scenario C above), transactions may be replayed that DuckDB already partially has. Idempotent replay makes this safe:

```
Normal INSERT:           Idempotent INSERT:

INSERT INTO t            DELETE FROM t WHERE pk=X   (remove if exists)
VALUES (X, ...)          INSERT INTO t VALUES (X, ...)  (safe now)


Normal UPDATE:           Idempotent UPDATE (with PK change):

UPDATE t SET ...         DELETE FROM t WHERE pk=OLD_PK
WHERE pk=X               DELETE FROM t WHERE pk=NEW_PK
                         INSERT INTO t VALUES (NEW_PK, ...)


Normal DELETE:           Idempotent DELETE:

DELETE FROM t            DELETE FROM t WHERE pk=X   (safe if already gone)
WHERE pk=X
```

**Controlled by:**
- `m_duckdb_idempotent_flag` — set true during crash recovery
- `m_duckdb_idempotent_cnt` — number of TXs to replay idempotently

### Initial Sync: Bootstrapping a DuckDB Replica

```
Step 1: Start MySQL with InnoDB tables on primary
Step 2: Set up replica with duckdb_mode=ON
Step 3: Initial data migration:
        Option A: convert_all_to_duckdb at startup (auto-converts all tables)
        Option B: parallel_copy_ddl (multi-threaded, 7x faster)
        Option C: duckdb_data_import_mode (manual bulk import)
Step 4: Start replication (GTID-based)
Step 5: Batch replay catches up (300K rows/sec)
Step 6: DuckDB replica serves analytical queries
```

**Notes on conversion:**
- Foreign keys are automatically stripped (DuckDB doesn't support them)
- Indexes become metadata only (uniqueness not enforced)
- `duckdb_copy_ddl_threads` controls parallel copy thread count

---

## Suggested Reading Order

| Step | File | Lines | What You'll Learn |
|---|---|---|---|
| 1 | `storage/duckdb/ha_duckdb.h` | 347 | Handler class definition, all virtual methods |
| 2 | `storage/duckdb/ha_duckdb.cc` | 240-272 | Plugin registration (how DuckDB becomes a MySQL engine) |
| 3 | `storage/duckdb/ddl_convertor.cc` | 212-336 | Type mapping table |
| 4 | `storage/duckdb/ha_duckdb.cc` | 608 | `write_row()` - write path entry |
| 5 | `storage/duckdb/delta_appender.cc` | - | Batch write optimization |
| 6 | `storage/duckdb/ha_duckdb.cc` | 868 | `rnd_init()` - read path entry |
| 7 | `storage/duckdb/duckdb_select.cc` | 77-236 | Result conversion (DuckDB -> MySQL format) |
| 8 | `sql/duckdb/duckdb_context.cc` | - | Per-thread connection & transaction lifecycle |
| 9 | `sql/duckdb/duckdb_manager.cc` | - | DuckDB instance startup/shutdown |
| 10 | `sql/duckdb/duckdb_query.cc` | - | Query execution gateway |
| 11 | `mysql-test/suite/duckdb/` | - | Practical usage examples & test cases |
| 12 | `wiki/duckdb/duckdb-en.md` | - | Design rationale & benchmarks |
| **Replication / CDC** | | | |
| 13 | `sql/log_event.cc` + `ha_duckdb.cc:536-750` | - | Binlog event -> DuckDB handler dispatch |
| 14 | `duckdb_context.cc:234-520` + `delta_appender.cc` | - | Multi-TX batching (300K rows/sec) |
| 15 | `ha_duckdb.cc:102-183` + `rpl_rli.h:677-693` | - | 2PC crash recovery + idempotent replay |
| 16 | `ha_duckdb_rpl_batch*.test` | - | Batch replication test cases |
| 17 | `convert_all_to_duckdb*.test` + `parallel_copy_ddl.test` | - | Initial sync & bootstrapping |

---

## Limitations

- No foreign key support
- No auto-increment
- PRIMARY KEY / UNIQUE KEY are metadata only (uniqueness not enforced)
- No savepoint support (stub implementation)
- No InnoDB recycle bin
- Designed primarily as an analytical read-replica engine

---

## Performance Reference

From `wiki/duckdb/duckdb-en.md`:

| Benchmark | DuckDB | InnoDB | Improvement |
|---|---|---|---|
| TPC-H SF100 avg query | 15.3s | 25,234s | **~200x** |
| Replication replay | 300K rows/sec | - | Zero lag |
| DDL copy (parallel) | - | - | **7x** speedup |
| Storage footprint | ~20% of main | 100% | **5x** smaller |
