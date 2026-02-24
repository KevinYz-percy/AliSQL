# S3/Iceberg Integration — Phase 1 Design Document

## 1. Overview

### 1.1 Purpose

This document provides a detailed technical design for Phase 1 of S3 and AWS Glue Data Catalog integration into AliSQL's DuckDB engine. Phase 1 establishes the foundational capabilities that all subsequent phases build upon.

### 1.2 Scope

Phase 1 delivers four capabilities:

1. **Build integration**: Statically link `iceberg` and `aws` DuckDB extensions into `libduckdb_bundle.a`
2. **S3 credential management**: MySQL system variables to configure AWS credentials for DuckDB's S3 access
3. **Extension initialization**: Load and configure httpfs/aws/iceberg extensions at DuckDB startup
4. **Glue catalog attachment**: Automatically ATTACH AWS Glue Data Catalog as an Iceberg catalog at initialization

### 1.3 Success Criteria

After Phase 1, the following operations work from any MySQL client:

```sql
-- Query S3 Parquet files directly
CALL dbms_duckdb.query("SELECT * FROM read_parquet('s3://my-bucket/data/*.parquet')");

-- Query Iceberg tables via Glue catalog
CALL dbms_duckdb.query("SELECT * FROM glue_catalog.analytics_db.events WHERE date = '2025-01-15'");

-- Browse Glue catalog tables
CALL dbms_duckdb.query("SHOW TABLES FROM glue_catalog.analytics_db");
```

### 1.4 Non-Goals

- Cold data offloading from local DuckDB to S3 (Phase 2)
- Transparent query federation with VIEWs (Phase 3)
- DuckLake catalog integration (Phase 4)
- Cross-engine JOINs between InnoDB and S3/Glue tables (Phase 5)

---

## 2. Architecture

### 2.1 Current State

```
MySQL Client
    │
    ▼
MySQL Server (sql layer)
    │
    ├── InnoDB Storage Engine
    │
    └── DuckDB Storage Engine (storage/duckdb/ha_duckdb.cc)
            │
            ▼
        DuckDB Instance (sql/duckdb/duckdb_manager.cc)
            │
            ├── Local .duckdb file
            ├── Extensions: core_functions, parquet, icu, json, delta, httpfs
            └── NO S3 credential configuration
                NO iceberg/aws extensions
```

### 2.2 Target State (Phase 1)

```
MySQL Client
    │
    ▼
MySQL Server (sql layer)
    │  ┌─ duckdb_s3_region, duckdb_s3_access_key_id, etc.
    │  └─ duckdb_glue_enabled, duckdb_glue_catalog_id, etc.
    │
    ├── InnoDB Storage Engine
    │
    └── DuckDB Storage Engine
            │
            ▼
        DuckDB Instance
            │
            ├── Local .duckdb file (unchanged)
            ├── Extensions: + httpfs, aws, iceberg (newly built)
            ├── S3 Secret: CREATE SECRET (TYPE s3, ...)
            ├── Glue Catalog: ATTACH 'account_id' AS glue_catalog (TYPE iceberg, ...)
            │
            └──► S3 / AWS Glue Data Catalog
                    │
                    ├── read_parquet('s3://...')
                    ├── iceberg_scan('s3://...')
                    └── glue_catalog.namespace.table
```

### 2.3 Data Flow

**S3 Parquet Query**:
```
MySQL Client → CALL dbms_duckdb.query("SELECT * FROM read_parquet('s3://...')")
    → duckdb_proc.cc::send_result()
    → duckdb_query.cc::duckdb_query(thd, sql)
    → DuckDB Connection::Query()
    → httpfs extension handles S3 GET requests using configured credentials
    → Parquet extension reads columnar data
    → Results streamed back via MySQL protocol
```

**Glue Catalog Query**:
```
MySQL Client → CALL dbms_duckdb.query("SELECT * FROM glue_catalog.db.table")
    → duckdb_query.cc::duckdb_query()
    → DuckDB resolves 'glue_catalog' via attached Iceberg catalog
    → Iceberg extension contacts Glue REST endpoint
    → Fetches table metadata + data file locations from Glue
    → httpfs reads Parquet data files from S3
    → Results streamed back via MySQL protocol
```

---

## 3. Detailed Design

### 3.1 Build System Changes

#### 3.1.1 Extension Configuration

**File**: `extra/duckdb/extension/extension_config.cmake`

**Current** (lines 1–21):
```cmake
# Base DuckDB extensions loaded on every build
duckdb_extension_load(core_functions)
duckdb_extension_load(parquet)
duckdb_extension_load(icu)
duckdb_extension_load(json)
```

**Change**: Add `httpfs`, `aws`, and `iceberg` to the base extension configuration:

```cmake
# Base DuckDB extensions loaded on every build
duckdb_extension_load(core_functions)
duckdb_extension_load(parquet)
duckdb_extension_load(icu)
duckdb_extension_load(json)

# S3 and data lake extensions
duckdb_extension_load(httpfs)
duckdb_extension_load(aws
    GIT_URL https://github.com/duckdb/duckdb-aws
    GIT_TAG main
)
duckdb_extension_load(iceberg
    GIT_URL https://github.com/duckdb/duckdb-iceberg
    GIT_TAG main
)
```

**Rationale**:
- `httpfs` is currently loaded transitively via `extra/duckdb/extension/delta/extension_config.cmake` (line 10). Moving it to the base config ensures S3 file access is always available, independent of the delta extension.
- `aws` provides the `credential_chain` provider for IAM-based authentication (EC2 instance profiles, ECS task roles, environment variables).
- `iceberg` provides `iceberg_scan()`, `iceberg_metadata()`, `iceberg_snapshots()`, and — critically — the Iceberg catalog ATTACH with `ENDPOINT_TYPE 'GLUE'` for Glue Data Catalog support.
- Both `aws` and `iceberg` are out-of-tree extensions maintained in separate GitHub repos. DuckDB's cmake system fetches them via `GIT_URL`.
- `GIT_TAG main` should be replaced with the exact tag matching DuckDB v1.3.x (e.g., `v1.3.0` or `v1.3.1`) once the build is tested. The main branch tracks the latest DuckDB development and may have API incompatibilities.

**Secondary change**: Remove `duckdb_extension_load(httpfs)` from `extra/duckdb/extension/delta/extension_config.cmake` (line 10) to avoid duplicate loading.

#### 3.1.2 Build Artifacts

The `aws` extension has a dependency on the AWS SDK C++ libraries (STS, SSO). DuckDB's cmake build system handles this via vcpkg or system packages. When building via `make bundle-library`, these get statically linked into `libduckdb_bundle.a`.

The `iceberg` extension depends on:
- `httpfs` (for S3 file access)
- `parquet` (for reading Parquet data files)
- `avro_cpp` (optional, for Avro metadata — may be handled by bundled third-party)

**Build verification**: After adding extensions, run:
```bash
cd extra/duckdb && make bundle-library
# Verify the bundle includes new extensions:
nm build/release/libduckdb_bundle.a | grep -i "iceberg\|aws" | head
```

#### 3.1.3 Binary Size Impact

| Extension | Estimated Size | Dependencies |
|-----------|---------------|-------------|
| httpfs    | ~2 MB         | OpenSSL (already linked) |
| aws       | ~5-10 MB      | AWS SDK (STS, SSO) |
| iceberg   | ~3-5 MB       | avro_cpp |
| **Total** | **~10-17 MB** | |

This increases the `libduckdb_bundle.a` size by approximately 10-17 MB, which is acceptable for the functionality gained.

---

### 3.2 S3 Credential System Variables

#### 3.2.1 Variable Declarations

**File**: `sql/duckdb/duckdb_config.h`

Add within the `myduck` namespace (after existing declarations at line 52):

```cpp
// S3 configuration
extern const char *global_s3_region;
extern const char *global_s3_access_key_id;
extern const char *global_s3_secret_access_key;
extern const char *global_s3_session_token;
extern const char *global_s3_endpoint;
extern bool global_s3_use_credential_chain;

// Glue Data Catalog configuration
extern bool global_glue_enabled;
extern const char *global_glue_catalog_id;
extern const char *global_glue_region;

// Update callback for S3 credential changes
bool update_s3_credentials(sys_var *sys_var, THD *thd, enum_var_type type);
```

#### 3.2.2 Variable Definitions

**File**: `sql/duckdb/duckdb_config.cc`

Add variable definitions:

```cpp
// S3 configuration defaults
const char *global_s3_region = nullptr;
const char *global_s3_access_key_id = nullptr;
const char *global_s3_secret_access_key = nullptr;
const char *global_s3_session_token = nullptr;
const char *global_s3_endpoint = nullptr;
bool global_s3_use_credential_chain = false;

// Glue Data Catalog configuration defaults
bool global_glue_enabled = false;
const char *global_glue_catalog_id = nullptr;
const char *global_glue_region = nullptr;
```

#### 3.2.3 MySQL System Variable Registration

**File**: `sql/sys_vars_ext.cc`

Add after the existing `duckdb_log_options` block (around line 289):

```cpp
/** S3 configuration variables */

static Sys_var_charptr Sys_duckdb_s3_region(
    "duckdb_s3_region",
    "AWS region for DuckDB S3 access (e.g., 'us-east-1'). "
    "Used by httpfs, aws, and iceberg extensions.",
    GLOBAL_VAR(myduck::global_s3_region), CMD_LINE(REQUIRED_ARG),
    IN_FS_CHARSET, DEFAULT(nullptr), NO_MUTEX_GUARD, NOT_IN_BINLOG,
    ON_CHECK(nullptr), ON_UPDATE(myduck::update_s3_credentials));

static Sys_var_charptr Sys_duckdb_s3_access_key_id(
    "duckdb_s3_access_key_id",
    "AWS access key ID for DuckDB S3 access. "
    "Set together with duckdb_s3_secret_access_key for static credentials.",
    GLOBAL_VAR(myduck::global_s3_access_key_id), CMD_LINE(REQUIRED_ARG),
    IN_FS_CHARSET, DEFAULT(nullptr), NO_MUTEX_GUARD, NOT_IN_BINLOG,
    ON_CHECK(nullptr), ON_UPDATE(myduck::update_s3_credentials));

static Sys_var_charptr Sys_duckdb_s3_secret_access_key(
    "duckdb_s3_secret_access_key",
    "AWS secret access key for DuckDB S3 access.",
    GLOBAL_VAR(myduck::global_s3_secret_access_key), CMD_LINE(REQUIRED_ARG),
    IN_FS_CHARSET, DEFAULT(nullptr), NO_MUTEX_GUARD, NOT_IN_BINLOG,
    ON_CHECK(nullptr), ON_UPDATE(myduck::update_s3_credentials));

static Sys_var_charptr Sys_duckdb_s3_session_token(
    "duckdb_s3_session_token",
    "AWS session token for DuckDB S3 access (temporary credentials from STS).",
    GLOBAL_VAR(myduck::global_s3_session_token), CMD_LINE(REQUIRED_ARG),
    IN_FS_CHARSET, DEFAULT(nullptr), NO_MUTEX_GUARD, NOT_IN_BINLOG,
    ON_CHECK(nullptr), ON_UPDATE(myduck::update_s3_credentials));

static Sys_var_charptr Sys_duckdb_s3_endpoint(
    "duckdb_s3_endpoint",
    "Custom S3 endpoint URL for S3-compatible services "
    "(e.g., MinIO, Alibaba Cloud OSS). Leave empty for standard AWS S3.",
    GLOBAL_VAR(myduck::global_s3_endpoint), CMD_LINE(REQUIRED_ARG),
    IN_FS_CHARSET, DEFAULT(nullptr), NO_MUTEX_GUARD, NOT_IN_BINLOG,
    ON_CHECK(nullptr), ON_UPDATE(myduck::update_s3_credentials));

static Sys_var_bool Sys_duckdb_s3_use_credential_chain(
    "duckdb_s3_use_credential_chain",
    "Use AWS credential provider chain for DuckDB S3 access. "
    "Searches environment variables, ~/.aws/credentials, EC2 instance "
    "profiles, and ECS task roles. When ON, access_key_id and "
    "secret_access_key are ignored.",
    GLOBAL_VAR(myduck::global_s3_use_credential_chain), CMD_LINE(OPT_ARG),
    DEFAULT(false), NO_MUTEX_GUARD, NOT_IN_BINLOG,
    ON_CHECK(nullptr), ON_UPDATE(myduck::update_s3_credentials));

/** Glue Data Catalog configuration variables */

static Sys_var_bool Sys_duckdb_glue_enabled(
    "duckdb_glue_enabled",
    "Enable AWS Glue Data Catalog integration via DuckDB Iceberg extension. "
    "Requires duckdb_glue_catalog_id and S3 credentials to be configured. "
    "The Glue catalog is attached at DuckDB initialization as 'glue_catalog'. "
    "Read-only: requires MySQL restart to change.",
    READ_ONLY GLOBAL_VAR(myduck::global_glue_enabled), CMD_LINE(OPT_ARG),
    DEFAULT(false));

static Sys_var_charptr Sys_duckdb_glue_catalog_id(
    "duckdb_glue_catalog_id",
    "AWS account ID for Glue Data Catalog (12-digit number). "
    "Read-only: requires MySQL restart to change.",
    READ_ONLY GLOBAL_VAR(myduck::global_glue_catalog_id),
    CMD_LINE(REQUIRED_ARG), IN_FS_CHARSET, DEFAULT(nullptr));

static Sys_var_charptr Sys_duckdb_glue_region(
    "duckdb_glue_region",
    "AWS region for Glue Data Catalog endpoint. "
    "Defaults to duckdb_s3_region if not set. "
    "Read-only: requires MySQL restart to change.",
    READ_ONLY GLOBAL_VAR(myduck::global_glue_region), CMD_LINE(REQUIRED_ARG),
    IN_FS_CHARSET, DEFAULT(nullptr));
```

#### 3.2.4 Variable Summary

| Variable | Type | Scope | Mutable | Default | Description |
|----------|------|-------|---------|---------|-------------|
| `duckdb_s3_region` | string | GLOBAL | Yes | NULL | AWS region |
| `duckdb_s3_access_key_id` | string | GLOBAL | Yes | NULL | AWS access key ID |
| `duckdb_s3_secret_access_key` | string | GLOBAL | Yes | NULL | AWS secret key |
| `duckdb_s3_session_token` | string | GLOBAL | Yes | NULL | STS session token |
| `duckdb_s3_endpoint` | string | GLOBAL | Yes | NULL | Custom S3 endpoint |
| `duckdb_s3_use_credential_chain` | bool | GLOBAL | Yes | OFF | Use AWS credential chain |
| `duckdb_glue_enabled` | bool | GLOBAL | No (READ_ONLY) | OFF | Enable Glue catalog |
| `duckdb_glue_catalog_id` | string | GLOBAL | No (READ_ONLY) | NULL | AWS account ID |
| `duckdb_glue_region` | string | GLOBAL | No (READ_ONLY) | NULL | Glue region |

**Design decisions**:

- **S3 credential vars are mutable**: Allows credential rotation without MySQL restart. The `ON_UPDATE` callback re-creates the DuckDB secret immediately. Pattern validated: `Sys_var_charptr` with `GLOBAL_VAR` + `ON_UPDATE` is used by `log_error_services` in `sql/sys_vars.cc:2685`.

- **Glue vars are READ_ONLY**: Catalog ATTACH is a one-time operation at DuckDB initialization. Changing it at runtime would require detaching and re-attaching the catalog, which could break in-flight queries. The restart requirement is an acceptable trade-off for simplicity and safety.

- **String vars use `IN_FS_CHARSET`**: Following the existing pattern from `duckdb_temp_directory`.

- **No SESSION-scoped S3 vars**: S3 credentials are inherently global — all DuckDB connections within the same DuckDB instance share the same secret storage. Per-session credentials would require a fundamentally different approach (per-connection DuckDB secrets), which is a Phase 2+ consideration.

---

### 3.3 S3 Credential Update Callback

#### 3.3.1 Implementation

**File**: `sql/duckdb/duckdb_config.cc`

```cpp
bool update_s3_credentials(sys_var * /*sys_var*/, THD *thd,
                           enum_var_type /*type*/) {
  std::ostringstream oss;

  if (global_s3_use_credential_chain) {
    // IAM role-based authentication
    oss << "CREATE OR REPLACE SECRET duckdb_s3 ("
        << "TYPE s3, PROVIDER credential_chain";
  } else if (global_s3_access_key_id != nullptr &&
             global_s3_secret_access_key != nullptr) {
    // Static credentials
    oss << "CREATE OR REPLACE SECRET duckdb_s3 ("
        << "TYPE s3"
        << ", KEY_ID '" << global_s3_access_key_id << "'"
        << ", SECRET '" << global_s3_secret_access_key << "'";
    if (global_s3_session_token != nullptr) {
      oss << ", SESSION_TOKEN '" << global_s3_session_token << "'";
    }
  } else {
    // Incomplete credentials — not an error, just nothing to configure yet
    return false;
  }

  if (global_s3_region != nullptr) {
    oss << ", REGION '" << global_s3_region << "'";
  }
  if (global_s3_endpoint != nullptr) {
    oss << ", ENDPOINT '" << global_s3_endpoint << "'";
  }
  oss << ")";

  return duckdb_query_and_send(thd, oss.str(), false, true);
}
```

#### 3.3.2 Security Considerations

**SQL injection in credential values**: The credential values are string literals embedded in a SQL statement. If a user sets `duckdb_s3_access_key_id = "'; DROP TABLE x; --"`, this could cause SQL injection in the DuckDB `CREATE SECRET` statement.

**Mitigation options**:

1. **Input validation**: Add an `ON_CHECK` callback that rejects values containing single quotes, semicolons, or other SQL metacharacters. AWS access key IDs are alphanumeric only (`[A-Z0-9]{20}`), secret keys are base64-encoded (`[A-Za-z0-9+/=]{40}`), and session tokens are URL-safe base64.

2. **Parameterized API**: Instead of constructing a SQL string, use DuckDB's C++ API directly to create secrets. This would require calling into DuckDB's SecretManager rather than going through the SQL layer.

3. **String escaping**: Escape single quotes by doubling them (`'` → `''`).

**Recommendation**: Use **option 1** (input validation) as the primary defense. AWS credential formats are well-defined and don't contain SQL metacharacters. Add a `check_s3_credential_value` function:

```cpp
static bool check_s3_credential_value(sys_var * /*self*/, THD *thd,
                                      set_var *var) {
  if (var->save_result.string_value.str == nullptr) return false;
  const char *val = var->save_result.string_value.str;
  for (size_t i = 0; val[i]; i++) {
    if (val[i] == '\'' || val[i] == ';' || val[i] == '\\') {
      my_error(ER_WRONG_VALUE_FOR_VAR, MYF(0), var->var->name.str, val);
      return true;
    }
  }
  return false;
}
```

---

### 3.4 DuckDB Initialization Changes

#### 3.4.1 Extension and Credential Setup

**File**: `sql/duckdb/duckdb_manager.cc`

The `Initialize()` method currently (lines 96–109):

```cpp
m_database = new duckdb::DuckDB(path, &config);
if (m_database == nullptr) return true;
TimeZoneOffsetHelper::init_timezone();
duckdb::Connection con(*m_database);
register_mysql_udf(&con);
LogErr(INFORMATION_LEVEL, ER_DUCKDB, "DuckdbManager::Initialize succeed.");
```

**Change**: After `register_mysql_udf(&con)`, add S3 configuration and optional Glue catalog attachment:

```cpp
m_database = new duckdb::DuckDB(path, &config);
if (m_database == nullptr) return true;
TimeZoneOffsetHelper::init_timezone();
duckdb::Connection con(*m_database);
register_mysql_udf(&con);

// Phase 1: S3/Iceberg integration
ConfigureS3Credentials(con);
if (global_glue_enabled) {
  AttachGlueCatalog(con);
}

LogErr(INFORMATION_LEVEL, ER_DUCKDB, "DuckdbManager::Initialize succeed.");
```

#### 3.4.2 ConfigureS3Credentials Implementation

**File**: `sql/duckdb/duckdb_manager.cc`

```cpp
void DuckdbManager::ConfigureS3Credentials(duckdb::Connection &con) {
  std::ostringstream oss;

  if (global_s3_use_credential_chain) {
    oss << "CREATE SECRET duckdb_s3 (TYPE s3, PROVIDER credential_chain";
  } else if (global_s3_access_key_id != nullptr &&
             global_s3_secret_access_key != nullptr) {
    oss << "CREATE SECRET duckdb_s3 (TYPE s3"
        << ", KEY_ID '" << global_s3_access_key_id << "'"
        << ", SECRET '" << global_s3_secret_access_key << "'";
    if (global_s3_session_token != nullptr) {
      oss << ", SESSION_TOKEN '" << global_s3_session_token << "'";
    }
  } else {
    LogErr(INFORMATION_LEVEL, ER_DUCKDB,
           "S3 credentials not configured. S3 access disabled.");
    return;
  }

  if (global_s3_region != nullptr) {
    oss << ", REGION '" << global_s3_region << "'";
  }
  if (global_s3_endpoint != nullptr) {
    oss << ", ENDPOINT '" << global_s3_endpoint << "'";
  }
  oss << ")";

  auto result = con.Query(oss.str());
  if (result->HasError()) {
    std::string msg =
        "Failed to configure S3 credentials: " + result->GetError();
    LogErr(WARNING_LEVEL, ER_DUCKDB, msg.c_str());
  } else {
    LogErr(INFORMATION_LEVEL, ER_DUCKDB,
           "S3 credentials configured successfully.");
  }
}
```

**Design notes**:
- Uses `CREATE SECRET` (not `CREATE OR REPLACE SECRET`) at init time since no secret exists yet.
- S3 configuration failure is a WARNING, not an error — it doesn't prevent DuckDB from starting. Local DuckDB operations continue to work.
- At runtime, `update_s3_credentials()` uses `CREATE OR REPLACE SECRET` to allow credential rotation.

#### 3.4.3 AttachGlueCatalog Implementation

**File**: `sql/duckdb/duckdb_manager.cc`

```cpp
void DuckdbManager::AttachGlueCatalog(duckdb::Connection &con) {
  if (global_glue_catalog_id == nullptr) {
    LogErr(WARNING_LEVEL, ER_DUCKDB,
           "duckdb_glue_enabled=ON but duckdb_glue_catalog_id is not set. "
           "Glue catalog will not be attached.");
    return;
  }

  // Determine region: prefer glue-specific region, fall back to S3 region
  const char *region =
      global_glue_region != nullptr ? global_glue_region : global_s3_region;
  if (region == nullptr) {
    LogErr(WARNING_LEVEL, ER_DUCKDB,
           "duckdb_glue_enabled=ON but no region configured. "
           "Set duckdb_glue_region or duckdb_s3_region.");
    return;
  }

  // Construct ATTACH statement for Glue Data Catalog
  // DuckDB's Iceberg extension supports Glue via ENDPOINT_TYPE 'GLUE'
  std::ostringstream oss;
  oss << "ATTACH '" << global_glue_catalog_id << "' AS glue_catalog ("
      << "TYPE iceberg, "
      << "ENDPOINT_TYPE 'GLUE', "
      << "AUTHORIZATION_TYPE 'SigV4', "
      << "DEFAULT_REGION '" << region << "'"
      << ")";

  auto result = con.Query(oss.str());
  if (result->HasError()) {
    std::string msg =
        "Failed to attach Glue Data Catalog: " + result->GetError();
    LogErr(WARNING_LEVEL, ER_DUCKDB, msg.c_str());
  } else {
    LogErr(INFORMATION_LEVEL, ER_DUCKDB,
           "AWS Glue Data Catalog attached as 'glue_catalog'.");
  }
}
```

**Design notes**:
- The catalog name `glue_catalog` is hardcoded for Phase 1. A future enhancement could make it configurable via `duckdb_glue_catalog_name`.
- `ENDPOINT_TYPE 'GLUE'` tells DuckDB's Iceberg extension to use the Glue REST API endpoint (`glue.REGION.amazonaws.com/iceberg`).
- `AUTHORIZATION_TYPE 'SigV4'` enables AWS Signature V4 signing for Glue API requests, using the same credentials configured in the S3 secret.
- Glue attachment failure is a WARNING — it doesn't prevent DuckDB from operating on local data.

#### 3.4.4 Header Changes

**File**: `sql/duckdb/duckdb_manager.h`

Add private static methods:

```cpp
class DuckdbManager {
 public:
  // ... existing public methods ...

 private:
  // ... existing private members ...

  /** Configure S3 credentials in DuckDB via CREATE SECRET. */
  static void ConfigureS3Credentials(duckdb::Connection &con);

  /** Attach AWS Glue Data Catalog as an Iceberg catalog. */
  static void AttachGlueCatalog(duckdb::Connection &con);
};
```

---

### 3.5 Credential Lifecycle

#### 3.5.1 Startup Flow

```
mysqld starts
    │
    ▼
sys_vars_ext.cc: System variables parsed from my.cnf / command line
    │  duckdb_s3_region = "us-east-1"
    │  duckdb_s3_use_credential_chain = ON
    │  duckdb_glue_enabled = ON
    │  duckdb_glue_catalog_id = "123456789012"
    │
    ▼
duckdb_manager.cc: DuckdbManager::Initialize()
    │
    ├── duckdb::DuckDB(path, &config)  // creates DuckDB instance
    │   └── Statically linked extensions auto-loaded:
    │       httpfs, aws, iceberg, parquet, icu, json, delta
    │
    ├── register_mysql_udf(&con)
    │
    ├── ConfigureS3Credentials(con)
    │   └── CREATE SECRET duckdb_s3 (TYPE s3, PROVIDER credential_chain, REGION 'us-east-1')
    │
    └── AttachGlueCatalog(con)
        └── ATTACH '123456789012' AS glue_catalog (TYPE iceberg, ENDPOINT_TYPE 'GLUE', ...)
```

#### 3.5.2 Runtime Credential Rotation

```
DBA runs: SET GLOBAL duckdb_s3_access_key_id = 'AKIANEWKEY';
    │
    ▼
sys_vars_ext.cc: ON_UPDATE callback fires
    │
    ▼
duckdb_config.cc: update_s3_credentials()
    │
    └── CREATE OR REPLACE SECRET duckdb_s3 (TYPE s3, KEY_ID 'AKIANEWKEY', SECRET '...', REGION '...')
        │
        └── DuckDB SecretManager replaces the existing secret
            └── Subsequent S3 requests use new credentials
```

#### 3.5.3 Credential Priority

When `duckdb_s3_use_credential_chain = ON`:
1. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
2. AWS config/credentials files (`~/.aws/credentials`)
3. EC2 instance metadata (IMDSv2)
4. ECS task IAM role
5. STS AssumeRole (via AWS SDK)

When `duckdb_s3_use_credential_chain = OFF`:
- Uses explicitly configured `duckdb_s3_access_key_id` + `duckdb_s3_secret_access_key`

---

## 4. Configuration Examples

### 4.1 Static Credentials (my.cnf)

```ini
[mysqld]
duckdb_mode = ON
duckdb_s3_region = us-east-1
duckdb_s3_access_key_id = AKIAIOSFODNN7EXAMPLE
duckdb_s3_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

### 4.2 IAM Role (EC2/ECS)

```ini
[mysqld]
duckdb_mode = ON
duckdb_s3_region = us-east-1
duckdb_s3_use_credential_chain = ON
```

### 4.3 Glue Data Catalog

```ini
[mysqld]
duckdb_mode = ON
duckdb_s3_region = us-east-1
duckdb_s3_use_credential_chain = ON
duckdb_glue_enabled = ON
duckdb_glue_catalog_id = 123456789012
# duckdb_glue_region defaults to duckdb_s3_region
```

### 4.4 S3-Compatible Service (MinIO / Alibaba Cloud OSS)

```ini
[mysqld]
duckdb_mode = ON
duckdb_s3_region = cn-hangzhou
duckdb_s3_endpoint = oss-cn-hangzhou.aliyuncs.com
duckdb_s3_access_key_id = <OSS_ACCESS_KEY>
duckdb_s3_secret_access_key = <OSS_SECRET_KEY>
```

### 4.5 Runtime Credential Rotation

```sql
-- Rotate credentials without restart
SET GLOBAL duckdb_s3_access_key_id = 'AKIANEWROTATEDKEY';
SET GLOBAL duckdb_s3_secret_access_key = 'newSecretKey123456789';

-- Verify new credentials work
CALL dbms_duckdb.query("SELECT count(*) FROM read_parquet('s3://my-bucket/test.parquet')");
```

---

## 5. File Change Summary

| File | Type | Lines Changed (est.) | Description |
|------|------|---------------------|-------------|
| `extra/duckdb/extension/extension_config.cmake` | Modify | +10 | Add httpfs, aws, iceberg extensions |
| `extra/duckdb/extension/delta/extension_config.cmake` | Modify | -1 | Remove duplicate httpfs load |
| `sql/duckdb/duckdb_config.h` | Modify | +15 | Declare S3/Glue config variables |
| `sql/duckdb/duckdb_config.cc` | Modify | +50 | Define variables + update_s3_credentials callback |
| `sql/sys_vars_ext.cc` | Modify | +80 | Register 9 new MySQL system variables |
| `sql/duckdb/duckdb_manager.h` | Modify | +5 | Declare ConfigureS3Credentials, AttachGlueCatalog |
| `sql/duckdb/duckdb_manager.cc` | Modify | +70 | Implement S3 config + Glue ATTACH at init |
| `mysql-test/suite/duckdb/t/duckdb_s3_config.test` | New | +40 | Test for S3/Glue config variables |
| `mysql-test/suite/duckdb/r/duckdb_s3_config.result` | New | +40 | Expected test results |
| **Total** |  | **~310** | |

---

## 6. Testing Strategy

### 6.1 Unit Test: System Variable Defaults

**File**: `mysql-test/suite/duckdb/t/duckdb_s3_config.test`

```sql
--echo #
--echo # Test 1: S3 variable defaults
--echo #
SELECT @@global.duckdb_s3_region;
SELECT @@global.duckdb_s3_access_key_id;
SELECT @@global.duckdb_s3_secret_access_key;
SELECT @@global.duckdb_s3_session_token;
SELECT @@global.duckdb_s3_endpoint;
SELECT @@global.duckdb_s3_use_credential_chain;

--echo #
--echo # Test 2: Glue variable defaults
--echo #
SELECT @@global.duckdb_glue_enabled;
SELECT @@global.duckdb_glue_catalog_id;
SELECT @@global.duckdb_glue_region;

--echo #
--echo # Test 3: S3 vars are session-denied (GLOBAL only)
--echo #
--error ER_INCORRECT_GLOBAL_LOCAL_VAR
SELECT @@session.duckdb_s3_region;

--echo #
--echo # Test 4: Glue vars are read-only
--echo #
--error ER_INCORRECT_GLOBAL_LOCAL_VAR
SET GLOBAL duckdb_glue_enabled = ON;

--echo #
--echo # Test 5: S3 credential chain can be toggled
--echo #
SET GLOBAL duckdb_s3_use_credential_chain = ON;
SELECT @@global.duckdb_s3_use_credential_chain;
SET GLOBAL duckdb_s3_use_credential_chain = OFF;

--echo #
--echo # Test 6: Extensions are loaded (httpfs provides s3_region setting)
--echo #
CALL dbms_duckdb.query("SELECT current_setting('s3_region')");
```

### 6.2 Integration Test: S3 Read (Manual)

Requires actual S3 bucket access:

```sql
-- Configure credentials
SET GLOBAL duckdb_s3_region = 'us-east-1';
SET GLOBAL duckdb_s3_access_key_id = 'AKIAXXXXXXXX';
SET GLOBAL duckdb_s3_secret_access_key = 'XXXXXXXX';

-- Read a public S3 dataset
CALL dbms_duckdb.query("
    SELECT count(*)
    FROM read_parquet('s3://us-prd-motherduck-open-datasets/tpch/0_01/parquet/lineitem.parquet')
");
```

### 6.3 Integration Test: Glue Catalog (Manual)

Requires AWS Glue access:

```bash
mysqld --duckdb_mode=ON \
       --duckdb_s3_region=us-east-1 \
       --duckdb_s3_use_credential_chain=ON \
       --duckdb_glue_enabled=ON \
       --duckdb_glue_catalog_id=123456789012
```

```sql
-- Browse Glue databases
CALL dbms_duckdb.query("SHOW SCHEMAS IN glue_catalog");

-- Browse tables in a database
CALL dbms_duckdb.query("SHOW TABLES IN glue_catalog.my_database");

-- Query a Glue Iceberg table
CALL dbms_duckdb.query("SELECT * FROM glue_catalog.my_database.my_table LIMIT 10");
```

### 6.4 Regression Testing

```bash
# Run all existing DuckDB tests to verify no regressions
./mysql-test/mysql-test-run.pl --suite=duckdb
./mysql-test/mysql-test-run.pl --suite=duckdb_rpl
```

---

## 7. Risks and Mitigations

### 7.1 Build Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| aws extension requires vcpkg or system AWS SDK | Medium | High (build failure) | Pre-install AWS SDK via vcpkg manifest or system package; may need to add vcpkg.json entries |
| iceberg extension has Rust dependency (avro) | Low | Medium | Iceberg extension's CMakeLists handles this; may need Rust toolchain for Avro support |
| GIT_TAG mismatch with DuckDB v1.3.x | Medium | High | Test with `main` branch first, then pin to specific release tag |
| Binary size increase > 20 MB | Low | Low | Acceptable; monitor with `ls -la libduckdb_bundle.a` |

### 7.2 Runtime Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Credential leakage in error messages/logs | Medium | High | DuckDB masks secrets in error output; verify with test. Don't log CREATE SECRET SQL. |
| Glue ATTACH fails on startup | Medium | Low | WARNING log only; DuckDB continues to work for local operations |
| S3 latency spikes degrade MySQL thread pool | Low | Medium | DuckDB queries run in separate threads; MySQL connection pool is unaffected |
| IAM credential_chain requires AWS SDK at runtime | Low | Medium | The `aws` extension provides this; ensure SDK is linked |

### 7.3 Security Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| SQL injection via credential system variables | Low | High | Input validation check callback (see Section 3.3.2) |
| Credentials visible via SHOW VARIABLES | Medium | Medium | S3 secret key visible to SUPER users; use mysql.global_variables privilege controls |
| S3 secret stored in DuckDB memory (not encrypted) | Low | Low | Consistent with all DuckDB deployments; use IAM roles where possible |

---

## 8. Future Considerations

### 8.1 Phase 2 Dependencies

Phase 2 (cold data offloading) depends on Phase 1's S3 credentials being configured. The background offloader thread will use `COPY ... TO 's3://...' (FORMAT PARQUET)`, which requires the httpfs extension and S3 secret established in Phase 1.

### 8.2 Alibaba Cloud OSS Compatibility

DuckDB's httpfs extension uses the S3 API protocol, which is supported by Alibaba Cloud OSS (and MinIO, Cloudflare R2, etc.) via the `duckdb_s3_endpoint` variable. For Alibaba Cloud customers, the typical configuration is:

```ini
duckdb_s3_endpoint = oss-cn-hangzhou-internal.aliyuncs.com
duckdb_s3_region = cn-hangzhou
```

### 8.3 Per-Session Credentials

Phase 1 uses GLOBAL credentials shared by all connections. A future enhancement could support per-session credentials (e.g., for multi-tenant scenarios), which would require creating per-connection DuckDB secrets in `DuckdbThdContext::compare_and_config()`.

### 8.4 Catalog Name Configurability

Phase 1 hardcodes the Glue catalog attachment as `glue_catalog`. A future `duckdb_glue_catalog_name` variable could make this configurable for environments that need multiple Glue catalog attachments or custom naming.

---

## 9. Appendix: DuckDB Extension Auto-Loading

DuckDB v1.3.x has two extension loading modes:

1. **Statically linked** (compile-time): Extensions are compiled into `libduckdb_bundle.a`. They are automatically available — no `LOAD` or `INSTALL` needed. This is what AliSQL uses.

2. **Dynamically loaded** (runtime): Extensions are `.duckdb_extension` files loaded via `INSTALL`/`LOAD`. Not used in AliSQL since the build produces a single static library.

When extensions are statically linked, the following are automatically available:
- All functions registered in `extension_entries.hpp` (e.g., `iceberg_scan`, `load_aws_credentials`, DuckLake functions)
- All secret types (S3, GCS, Azure, Huggingface)
- All catalog types (Iceberg, DuckLake)
- All settings (e.g., `s3_region`, `unsafe_enable_version_guessing`)

The entries in `extension_entries.hpp` (lines 221-224, 433, 993-995, 1040, 1044-1045, 1112-1137) confirm that iceberg, aws, and ducklake are fully registered in DuckDB's extension catalog system. Once built and statically linked, they will work immediately.
