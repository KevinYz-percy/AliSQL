# AliSQL S3 Lakehouse Integration — High-Level Design

## 1. Vision

Turn a single AliSQL instance into a **MySQL-compatible lakehouse** — applications write to InnoDB using standard MySQL protocol, while analytical queries transparently read from S3 (Parquet/Iceberg) via DuckDB. No application changes required.

```
                        ┌─────────────────────────────────┐
                        │         MySQL Client             │
                        │  (any MySQL-compatible app/tool) │
                        └──────────┬──────────────────────┘
                                   │ MySQL Protocol
                                   ▼
                        ┌─────────────────────────────────┐
                        │         AliSQL Instance          │
                        │                                  │
                        │  ┌─────────┐    ┌────────────┐  │
                        │  │ InnoDB  │    │  DuckDB    │  │
                        │  │ (write) │    │  (read)    │  │
                        │  └────┬────┘    └─────▲──────┘  │
                        │       │               │         │
                        │       │  CDC Engine   │         │
                        │       └──────►────────┘         │
                        │              │                   │
                        └──────────────┼───────────────────┘
                                       │ S3 API
                                       ▼
                        ┌─────────────────────────────────┐
                        │     S3 / Object Storage          │
                        │  (Parquet / Iceberg format)      │
                        │                                  │
                        │  Optional: AWS Glue Data Catalog │
                        └─────────────────────────────────┘
```

### 1.1 Key Principles

1. **Full MySQL compatibility** — Applications see a standard MySQL instance. No new SQL dialect, no DuckDB-specific syntax required. Standard `SELECT`, `INSERT`, `UPDATE`, `DELETE` all work as expected.

2. **Write to InnoDB, read from S3** — All writes go through InnoDB (ACID, row-level locking, crash recovery). Analytical reads are transparently routed to DuckDB reading Parquet/Iceberg on S3.

3. **Automatic query routing** — The MySQL optimizer decides whether to use InnoDB or DuckDB+S3 based on query cost. No explicit hints or session variables needed for most use cases.

4. **Single instance** — No separate analytics cluster. One `mysqld` process serves both OLTP and OLAP workloads. This simplifies operations, reduces cost, and eliminates data staleness from external ETL.

### 1.2 Target Use Cases

| Use Case | Write Path | Read Path |
|----------|-----------|-----------|
| OLTP (point lookups, small transactions) | InnoDB | InnoDB |
| OLAP (full scans, aggregations, joins on large tables) | InnoDB | DuckDB + S3 |
| Data exploration on historical data | — | DuckDB + S3 |
| External Glue catalog queries (cross-account data sharing) | — | DuckDB + S3 (Iceberg) |

---

## 2. Architecture Overview

### 2.1 Component Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AliSQL Server                               │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │                 MySQL Optimizer                          │       │
│  │                                                          │       │
│  │  ① Cost-based engine selection                           │       │
│  │     InnoDB cost < threshold → use InnoDB                 │       │
│  │     InnoDB cost ≥ threshold → use DuckDB+S3              │       │
│  │                                                          │       │
│  │  ② SECONDARY ENGINE framework integration                │       │
│  └──────────────────────┬──────────────────┬────────────────┘       │
│                         │                  │                         │
│              ┌──────────▼──────┐  ┌────────▼──────────┐             │
│              │    InnoDB       │  │  DuckDB Engine     │             │
│              │  (Primary)      │  │  (Secondary)       │             │
│              │                 │  │                     │             │
│              │  • OLTP writes  │  │  • Columnar scans  │             │
│              │  • Point reads  │  │  • Aggregations    │             │
│              │  • Transactions │  │  • S3 Parquet/     │             │
│              │  • WAL/redo     │  │    Iceberg reads   │             │
│              └────────┬────────┘  └────────▲───────────┘             │
│                       │                    │                          │
│              ┌────────▼────────────────────┘──────────┐              │
│              │         CDC / Sync Engine               │              │
│              │                                         │              │
│              │  ③ Captures InnoDB changes (binlog)     │              │
│              │  ④ Converts to Parquet / Iceberg        │              │
│              │  ⑤ Writes to S3                         │              │
│              │  ⑥ Updates Iceberg catalog metadata     │              │
│              └────────────────────┬────────────────────┘              │
│                                   │                                   │
└───────────────────────────────────┼───────────────────────────────────┘
                                    │ S3 API (httpfs)
                                    ▼
                          ┌─────────────────────┐
                          │     Object Storage   │
                          │                     │
                          │  • AWS S3            │
                          │  • Alibaba Cloud OSS │
                          │  • MinIO             │
                          │                     │
                          │  Format:            │
                          │  • Apache Parquet    │
                          │  • Apache Iceberg    │
                          └─────────┬───────────┘
                                    │
                          ┌─────────▼───────────┐
                          │  Data Catalog        │
                          │  (optional)          │
                          │                     │
                          │  • AWS Glue          │
                          │  • DuckLake          │
                          │  • Iceberg REST      │
                          └─────────────────────┘
```

### 2.2 How It Works — End to End

**Write (INSERT/UPDATE/DELETE):**
```
App → MySQL → InnoDB → commit → binlog
                                   │
                          CDC Engine picks up change
                                   │
                          Batches rows → converts to Parquet
                                   │
                          Writes to S3 → updates Iceberg metadata
```

**Read (SELECT):**
```
App → MySQL → Optimizer estimates cost
                │
                ├── Low cost (point lookup, indexed scan)
                │   → InnoDB → return rows
                │
                └── High cost (full scan, aggregation, join)
                    → DuckDB → reads Parquet from S3
                    → columnar scan → return rows via MySQL protocol
```

---

## 3. Phased Implementation

### Phase 1: S3 Foundation

**Goal**: Build the plumbing — DuckDB can read from S3, credentials are managed via MySQL system variables.

**What ships:**
- `iceberg`, `aws`, `httpfs` extensions statically linked into DuckDB
- MySQL system variables for S3 credentials (`duckdb_s3_region`, `duckdb_s3_access_key_id`, etc.)
- MySQL system variables for Glue catalog (`duckdb_glue_enabled`, `duckdb_glue_catalog_id`)
- S3 credentials configured at DuckDB init via `CREATE SECRET`
- Optional Glue catalog auto-ATTACH at startup

**User experience after Phase 1:**
```sql
-- Direct S3 query via stored procedure (manual)
CALL dbms_duckdb.query("SELECT count(*) FROM read_parquet('s3://bucket/data.parquet')");

-- Glue catalog query (manual)
CALL dbms_duckdb.query("SELECT * FROM glue_catalog.mydb.events LIMIT 10");
```

**Limitations**: No automatic query routing. Users must explicitly use `dbms_duckdb.query()`.

---

### Phase 2: Secondary Engine Integration

**Goal**: Wire DuckDB into MySQL's SECONDARY ENGINE framework so the optimizer can automatically route queries.

**Background — MySQL's Secondary Engine Framework**:

MySQL 8.0+ has a built-in framework for dual-engine tables (originally designed for HeatWave/RAPID). It works as follows:

1. A table is created with `ENGINE=InnoDB` and `SECONDARY_ENGINE=DUCKDB`
2. `ALTER TABLE t SECONDARY_LOAD` tells the secondary engine to load the table's data
3. For SELECT queries, MySQL's optimizer first estimates cost using InnoDB. If the cost exceeds `secondary_engine_cost_threshold`, it re-optimizes using the secondary engine.
4. The secondary engine's `prepare_secondary_engine`, `optimize_secondary_engine`, and `compare_secondary_engine_cost` callbacks decide whether to accept the query.

**What ships:**
- DuckDB handlerton registers as a secondary engine (`duckdb_hton->prepare_secondary_engine`, etc.)
- `ALTER TABLE t SECONDARY_ENGINE = DUCKDB` marks a table for dual-engine access
- `ALTER TABLE t SECONDARY_LOAD` triggers DuckDB to create a shadow table (initially loading from InnoDB)
- `ALTER TABLE t SECONDARY_UNLOAD` removes the DuckDB shadow
- Cost-based query routing: optimizer compares InnoDB plan cost vs DuckDB plan cost
- `secondary_engine_cost_threshold` controls the routing sensitivity

**User experience after Phase 2:**
```sql
-- Mark table for automatic routing
ALTER TABLE orders SECONDARY_ENGINE = DUCKDB;
ALTER TABLE orders SECONDARY_LOAD;

-- This automatically uses DuckDB (high cost, full scan)
SELECT region, SUM(amount) FROM orders GROUP BY region;

-- This automatically uses InnoDB (low cost, point lookup)
SELECT * FROM orders WHERE id = 42;

-- Force DuckDB for a session
SET use_secondary_engine = FORCED;
SELECT * FROM orders WHERE date > '2025-01-01';
```

**Key design decisions:**

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Initial data load | DuckDB reads from InnoDB via internal scan | No S3 dependency for SECONDARY_LOAD; works offline |
| Query acceptance | Accept all read-only queries DuckDB can handle | Reject writes, DDL, and unsupported SQL features |
| Cost model | DuckDB provides estimated row count; optimizer compares | Leverages MySQL's existing cost framework |
| Fallback | If DuckDB fails, retry on InnoDB | Standard secondary engine behavior |

**Handlerton callbacks to implement:**

| Callback | Purpose |
|----------|---------|
| `prepare_secondary_engine` | Create DuckDB execution context for the query |
| `optimize_secondary_engine` | Validate query is DuckDB-compatible, estimate cost |
| `compare_secondary_engine_cost` | Compare DuckDB cost vs InnoDB cost |
| `secondary_engine_flags` | Declare supported join types, features |
| `secondary_engine_modify_access_path_cost` | Adjust cost estimates for specific access paths |

---

### Phase 3: CDC to S3

**Goal**: Automatically sync InnoDB changes to S3 in Parquet/Iceberg format so DuckDB reads from S3 instead of local storage.

**Background — Current CDC in AliSQL**:

AliSQL already has CDC from InnoDB to DuckDB via binlog replication:
- `storage/duckdb/ha_duckdb.cc` handles `write_row`, `update_row`, `delete_row`
- `storage/duckdb/delta_appender.cc` batches rows into DuckDB appenders
- Changes are applied transactionally (2PC with binlog for crash safety)

Phase 3 extends this pipeline to also write to S3.

**What ships:**
- Background S3 sync thread that periodically exports DuckDB data to S3 as Parquet files
- Configurable sync policy: time-based (every N minutes) or size-based (every N MB of changes)
- Iceberg table format support — writes produce new Iceberg snapshots, not just raw Parquet files
- DuckDB queries transparently read from S3 instead of local storage for synced tables
- MySQL system variables: `duckdb_s3_sync_enabled`, `duckdb_s3_sync_interval`, `duckdb_s3_sync_path`

**Data flow:**
```
InnoDB change → binlog → DuckDB applies locally
                                  │
                         S3 Sync Thread (background)
                                  │
                         ┌────────▼─────────┐
                         │  For each synced  │
                         │  table:           │
                         │                   │
                         │  COPY table TO    │
                         │  's3://bucket/    │
                         │   db/table/'      │
                         │  (FORMAT PARQUET) │
                         └────────┬──────────┘
                                  │
                                  ▼
                         S3: db/table/
                              ├── part-0001.parquet
                              ├── part-0002.parquet
                              └── _metadata/
```

**Iceberg vs raw Parquet:**

| Aspect | Raw Parquet | Iceberg |
|--------|------------|---------|
| Write complexity | Simple COPY TO | Need catalog + manifest management |
| Time travel | No | Yes (snapshot-based) |
| Schema evolution | Manual | Automatic |
| Partition pruning | Manual path filtering | Automatic via metadata |
| Concurrent reads | Possible stale reads during write | MVCC via snapshots |
| Glue compatibility | Limited | Full |

**Recommendation**: Start with raw Parquet export (simpler), add Iceberg support as an option. DuckDB's DuckLake extension could manage the Iceberg catalog internally.

---

### Phase 4: Transparent S3-Backed Secondary Engine

**Goal**: Combine Phase 2 (secondary engine routing) and Phase 3 (S3 sync) so that `SECONDARY_LOAD` automatically means "this table's analytical queries read from S3."

**What ships:**
- `SECONDARY_LOAD` triggers initial full export to S3 + enables ongoing CDC sync
- `SECONDARY_UNLOAD` stops sync and removes S3 data
- DuckDB's secondary engine handler reads from S3 (not local DuckDB storage)
- For tables not yet synced to S3, DuckDB falls back to local storage
- `information_schema.SECONDARY_ENGINE_STATUS` shows sync state per table

**User experience after Phase 4:**
```sql
-- Enable S3-backed analytics for a table
ALTER TABLE orders SECONDARY_ENGINE = DUCKDB;
ALTER TABLE orders SECONDARY_LOAD;
-- Behind the scenes: full export to S3, CDC sync starts

-- Analytical query automatically reads from S3
SELECT DATE(created_at), COUNT(*), SUM(total)
FROM orders
WHERE created_at > '2024-01-01'
GROUP BY DATE(created_at);
-- → Routed to DuckDB → reads Parquet from S3

-- Point lookup still goes to InnoDB
SELECT * FROM orders WHERE id = 12345;
-- → Stays on InnoDB (low cost)

-- Check sync status
SELECT * FROM information_schema.DUCKDB_S3_SYNC_STATUS;
+--------+-----------+---------------------+--------+
| table  | s3_path   | last_sync           | lag_ms |
+--------+-----------+---------------------+--------+
| orders | s3://...  | 2025-06-15 10:30:00 | 1200   |
+--------+-----------+---------------------+--------+
```

---

### Phase 5: External Catalog Federation

**Goal**: Query data from external Iceberg catalogs (Glue, Hive, REST) through standard MySQL SELECTs, not just `dbms_duckdb.query()`.

**What ships:**
- `CREATE SERVER` syntax to register external Iceberg catalogs
- MySQL VIEWs automatically created for Glue catalog tables
- Cross-engine JOINs: InnoDB local data JOIN with S3 Iceberg data
- Query pushdown: predicates pushed to DuckDB/Iceberg for partition pruning

**User experience after Phase 5:**
```sql
-- Register external Glue catalog
CREATE SERVER glue_analytics
  FOREIGN DATA WRAPPER iceberg
  OPTIONS (
    CATALOG_TYPE 'GLUE',
    CATALOG_ID '123456789012',
    REGION 'us-east-1'
  );

-- Import schemas as MySQL databases
CALL dbms_duckdb.import_catalog('glue_analytics', 'analytics_db');

-- Now query Glue tables with standard SQL!
SELECT o.id, o.total, e.event_type
FROM orders o                             -- InnoDB (local)
JOIN analytics_db.click_events e          -- Iceberg on S3 (via Glue)
  ON o.user_id = e.user_id
WHERE e.event_date = '2025-06-15';
```

---

## 4. Data Consistency Model

### 4.1 Write Path Guarantees

| Property | Guarantee |
|----------|-----------|
| Durability | InnoDB WAL + binlog (standard MySQL crash recovery) |
| Atomicity | InnoDB transactions (standard ACID) |
| S3 consistency | Eventual — CDC sync has configurable lag (seconds to minutes) |

### 4.2 Read Path Consistency

```
                    Timeline
    ─────────────────────────────────────────►

    InnoDB write    CDC sync to S3    DuckDB read from S3
         │               │                   │
         ▼               ▼                   ▼
    ┌─────────┐    ┌──────────┐        ┌──────────┐
    │ tx=100  │    │ tx=100   │        │ tx=100   │  ← consistent
    │ (fresh) │    │ exported │        │ visible  │
    └─────────┘    └──────────┘        └──────────┘

                         Staleness window
                    ◄─────────────────────►
                    (configurable: seconds to minutes)
```

**Staleness guarantees:**
- **InnoDB reads**: Always current (latest committed)
- **DuckDB/S3 reads**: Eventually consistent, bounded staleness
  - Configurable via `duckdb_s3_sync_interval` (default: 60 seconds)
  - `information_schema.DUCKDB_S3_SYNC_STATUS` exposes current lag per table
  - For strict freshness: `SET use_secondary_engine = OFF` forces InnoDB

**Conflict handling**: Not applicable — InnoDB is the single source of truth for writes. S3 is a read-only materialized view. If the CDC sync fails, reads fall back to local DuckDB or InnoDB.

### 4.3 Schema Evolution

| Scenario | Behavior |
|----------|----------|
| `ALTER TABLE ADD COLUMN` | InnoDB updates immediately. S3 Parquet files are lazily rewritten on next sync. Iceberg handles via schema evolution metadata. |
| `ALTER TABLE DROP COLUMN` | InnoDB updates immediately. Old S3 Parquet files still have the column; DuckDB ignores it. Iceberg handles via schema evolution. |
| `ALTER TABLE MODIFY COLUMN` (compatible type change) | Works seamlessly for widening conversions (INT→BIGINT). |
| `ALTER TABLE MODIFY COLUMN` (incompatible type change) | Triggers full re-export to S3 on next sync. |

---

## 5. System Variables Summary

### Phase 1 — S3 Foundation
| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `duckdb_s3_region` | string | NULL | AWS region for S3 |
| `duckdb_s3_access_key_id` | string | NULL | AWS access key |
| `duckdb_s3_secret_access_key` | string | NULL | AWS secret key |
| `duckdb_s3_session_token` | string | NULL | STS temp token |
| `duckdb_s3_endpoint` | string | NULL | Custom S3 endpoint (OSS, MinIO) |
| `duckdb_s3_use_credential_chain` | bool | OFF | Use IAM role chain |
| `duckdb_glue_enabled` | bool | OFF | Enable Glue catalog |
| `duckdb_glue_catalog_id` | string | NULL | AWS account ID |
| `duckdb_glue_region` | string | NULL | Glue region |

### Phase 2 — Secondary Engine
| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `use_secondary_engine` | enum | ON | OFF / ON / FORCED (existing MySQL var) |
| `secondary_engine_cost_threshold` | double | 100000 | Cost threshold to route to DuckDB (existing MySQL var) |

### Phase 3 — CDC to S3
| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `duckdb_s3_sync_enabled` | bool | OFF | Enable CDC sync to S3 |
| `duckdb_s3_sync_interval` | int | 60 | Sync interval in seconds |
| `duckdb_s3_sync_path` | string | NULL | S3 path prefix (e.g., `s3://bucket/lakehouse/`) |
| `duckdb_s3_sync_format` | enum | PARQUET | PARQUET or ICEBERG |
| `duckdb_s3_sync_batch_size` | int | 100MB | Flush threshold per table |

---

## 6. Comparison With Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **This design (AliSQL + DuckDB + S3)** | Single instance, full MySQL compat, no external tools, automatic routing | New code in AliSQL, DuckDB extension deps |
| **MySQL + HeatWave** | Oracle-supported, proven secondary engine | Proprietary, cloud-only, expensive, Oracle lock-in |
| **MySQL + Debezium + Spark/Trino** | Standard open-source stack | Multiple systems to operate, ETL lag, no transparent routing |
| **MySQL + ClickHouse (MaterializedMySQL)** | ClickHouse is fast for OLAP | Separate system, different SQL dialect, data duplication |
| **TiDB (HTAP)** | Single system, TiFlash columnar | Not MySQL-compatible for all workloads, complex cluster |

---

## 7. Risks and Mitigations

| Risk | Phase | Probability | Impact | Mitigation |
|------|-------|------------|--------|------------|
| DuckDB secondary engine callbacks are complex to implement correctly | 2 | High | High | Study HeatWave/RAPID patterns; implement incrementally with fallback to InnoDB |
| S3 latency spikes degrade OLTP workloads | 3 | Medium | High | S3 reads run on DuckDB's thread pool, isolated from MySQL's OLTP threads; CDC sync is async |
| CDC sync lag causes stale analytical results | 3 | Medium | Medium | Expose lag metrics; allow `SET use_secondary_engine = OFF` for fresh reads |
| iceberg/aws extension build dependencies (AWS SDK, vcpkg) | 1 | Medium | Medium | Pin versions; add to CI; document build prerequisites |
| Schema changes cause S3 data format mismatches | 3-4 | Medium | Medium | Iceberg handles schema evolution natively; Parquet mode triggers re-export |
| Memory pressure from running both InnoDB and DuckDB | All | Low | High | DuckDB memory bounded by `duckdb_memory_limit`; configurable per workload |

---

## 8. Phase Dependencies and Timeline

```
Phase 1: S3 Foundation
  │  • Build: iceberg + aws extensions
  │  • System variables for S3/Glue credentials
  │  • Manual S3 queries via dbms_duckdb.query()
  │
  ▼
Phase 2: Secondary Engine Integration
  │  • DuckDB registers as SECONDARY_ENGINE
  │  • ALTER TABLE ... SECONDARY_LOAD
  │  • Cost-based automatic query routing
  │  • Reads from local DuckDB (not S3 yet)
  │
  ▼
Phase 3: CDC to S3
  │  • Background sync thread
  │  • InnoDB changes → Parquet/Iceberg on S3
  │  • Configurable sync interval
  │
  ▼
Phase 4: Transparent S3-Backed Secondary Engine     (Phase 2 + Phase 3 combined)
  │  • SECONDARY_LOAD triggers S3 export + CDC
  │  • DuckDB secondary engine reads from S3
  │  • Sync status monitoring
  │
  ▼
Phase 5: External Catalog Federation
     • CREATE SERVER for Glue/Iceberg catalogs
     • Cross-engine JOINs (InnoDB + S3)
     • Schema import from external catalogs
```

**Phase 1 and 2 can be developed in parallel** — Phase 1 focuses on DuckDB extension build + S3 credentials, Phase 2 focuses on secondary engine handlerton callbacks. They converge at Phase 4.

---

## 9. Open Questions

1. **Iceberg catalog management**: Should AliSQL manage its own Iceberg catalog (via DuckLake) or register tables in an external catalog (Glue)? DuckLake is simpler for single-instance use; Glue enables cross-system data sharing.

2. **Partial table sync**: Should CDC sync export entire tables or support partition-level sync? Time-partitioned tables (e.g., events by date) benefit from partition-level export where only the latest partition is synced frequently.

3. **DELETE/UPDATE handling in Parquet**: Parquet is append-only. Updates/deletes require either rewriting entire Parquet files or using Iceberg's merge-on-read approach. Iceberg is strongly preferred for tables with updates.

4. **Resource isolation**: How much memory/CPU should DuckDB use vs InnoDB? Current `duckdb_memory_limit` and `duckdb_threads` provide coarse control. Fine-grained resource isolation (e.g., cgroups, IO priority) may be needed for production HTAP workloads.

5. **Multi-table consistency for S3 reads**: If a query joins two S3-synced tables, they may be at different sync points. Should we enforce cross-table snapshot consistency? Iceberg snapshots make this possible but add complexity.
