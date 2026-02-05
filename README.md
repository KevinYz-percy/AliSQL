<p align="center">
  <img src="./alisql-logo.png" width="180" alt="AliSQL Logo"/>
</p>

<h1 align="center">AliSQL</h1>

<p align="center">
  <strong>Alibaba's Enterprise MySQL Branch with DuckDB OLAP & Native Vector Search</strong>
</p>

<p align="center">
  <em>Battle-tested in Alibaba's production environment, powering millions of databases</em>
</p>

<p align="center">
  <a href="https://github.com/alibaba/AliSQL/stargazers"><img src="https://img.shields.io/github/stars/alibaba/AliSQL?style=for-the-badge&logo=github&color=ffca28" alt="GitHub Stars"></a>
  <a href="https://github.com/alibaba/AliSQL/network/members"><img src="https://img.shields.io/github/forks/alibaba/AliSQL?style=for-the-badge&logo=github&color=8bc34a" alt="GitHub Forks"></a>
  <a href="https://github.com/alibaba/AliSQL/blob/master/LICENSE"><img src="https://img.shields.io/badge/License-GPL%202.0-blue?style=for-the-badge" alt="License"></a>
  <a href="https://github.com/alibaba/AliSQL/releases"><img src="https://img.shields.io/badge/MySQL-8.0.44%20LTS-orange?style=for-the-badge&logo=mysql&logoColor=white" alt="MySQL Version"></a>
</p>

<p align="center">
  <a href="#key-features">Features</a> •
  <a href="#quick-start">Quick Start</a> •
  <a href="#documentation">Docs</a> •
  <a href="#roadmap">Roadmap</a> •
  <a href="#contributing">Contributing</a>
</p>

<p align="center">
  <a href="./README_zh.md">简体中文</a> | <a href="./README.md">English</a>
</p>

## Why AliSQL?

AliSQL brings enterprise-grade capabilities to MySQL, combining the reliability of InnoDB OLTP with DuckDB's blazing-fast analytics and native vector search — all through familiar MySQL interfaces.

<table>
<tr>
<td width="33%" align="center">

### 200x Faster Analytics

DuckDB columnar engine delivers **200x speedup** on analytical queries compared to InnoDB

</td>
<td width="33%" align="center">

### Native Vector Search

Built-in HNSW algorithm supporting up to **16,383 dimensions** for AI/ML workloads

</td>
<td width="33%" align="center">

### 100% MySQL Compatible

Use your existing MySQL tools, drivers, and SQL — zero learning curve

</td>
</tr>
</table>

## Key Features

| Feature | Description | Status |
|---------|-------------|--------|
| **DuckDB Storage Engine** | Columnar OLAP engine with automatic compression, perfect for analytics workloads | Available |
| **Vector Index (VIDX)** | Native vector storage & ANN search with HNSW, supports COSINE & EUCLIDEAN distance | Available |
| **DDL Optimization** | Instant DDL, parallel B+tree construction, non-blocking locks | Planned |
| **RTO Optimization** | Accelerated crash recovery for faster instance startup | Planned |
| **Replication Boost** | Binlog Parallel Flush, Binlog in Redo, large transaction optimization | Planned |

## Quick Start

### Option 1: Build from Source

```bash
# Clone the repository
git clone https://github.com/alibaba/AliSQL.git
cd AliSQL

# Build (release mode)
sh build.sh -t release -d ~/alisql

# Install
make install
```

### Option 2: Set Up a DuckDB Analytical Node

> **Step-by-step guide:** [How to set up a DuckDB node](./wiki/duckdb/how-to-setup-duckdb-node-en.md)

### Initialize & Start Server

```bash
# Initialize data directory
~/alisql/bin/mysqld --initialize-insecure --datadir=~/alisql/data

# Start the server
~/alisql/bin/mysqld --datadir=~/alisql/data
```

## Usage Examples

### DuckDB for Analytics

```sql
-- Create an analytical table with DuckDB engine
CREATE TABLE sales_analytics (
    sale_date DATE,
    product_id INT,
    revenue DECIMAL(10,2),
    quantity INT
) ENGINE=DuckDB;

-- Run complex analytics (200x faster than InnoDB!)
SELECT
    DATE_FORMAT(sale_date, '%Y-%m') as month,
    SUM(revenue) as total_revenue,
    COUNT(*) as transactions
FROM sales_analytics
GROUP BY month
ORDER BY total_revenue DESC;
```

### Vector Search for AI Applications

```sql
-- Create a table with vector column
CREATE TABLE embeddings (
    id INT PRIMARY KEY,
    content TEXT,
    embedding VECTOR(768)  -- 768-dimensional vectors
) ENGINE=InnoDB;

-- Create HNSW index for fast ANN search
CREATE VECTOR INDEX idx_embedding ON embeddings(embedding);

-- Find similar items using cosine distance
SELECT id, content,
       COSINE_DISTANCE(embedding, '[0.1, 0.2, ...]') as distance
FROM embeddings
ORDER BY distance
LIMIT 10;
```

## Build Options

| Option | Description | Default |
|--------|-------------|---------|
| `-t release\|debug` | Build type | `debug` |
| `-d <dir>` | Installation directory | `/usr/local/alisql` |
| `-g asan\|tsan` | Enable sanitizer (memory/thread) | disabled |
| `-c` | Enable code coverage (gcov) | disabled |

**Prerequisites:** CMake 3.x+, Python 3, GCC 7+ or Clang 5+

## Roadmap

```
Q4 2025  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
         [x] DuckDB Storage Engine  [x] Vector Index (VIDX)   [x] Open Source

2026     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
         [ ] DDL Optimization       [ ] RTO Optimization      [ ] Replication Boost
             - Instant DDL              - Fast Crash Recovery     - Binlog Parallel Flush
             - Parallel B+tree          - Minimize RTO            - Binlog in Redo
             - Non-blocking Locks                                 - Large TX Optimization
```

## Documentation

| Document | Description |
|----------|-------------|
| [DuckDB Integration Guide](./wiki/duckdb/duckdb-en.md) | Complete guide for DuckDB storage engine |
| [Vector Index Guide](./wiki/vidx/vidx_readme.md) | Native vector storage and ANN search |
| [Release Notes](./wiki/changes-in-alisql-8.0.44.md) | What's new in AliSQL 8.0.44 |
| [Setup DuckDB Node](./wiki/duckdb/how-to-setup-duckdb-node-en.md) | Quick setup guide for analytics |

**External Resources:**
- [MySQL 8.0 Documentation](https://dev.mysql.com/doc/refman/8.0/en/)
- [DuckDB Official Docs](https://duckdb.org/docs/stable/)
- [Detailed Article (Chinese)](https://mp.weixin.qq.com/s/_YmlV3vPc9CksumXvXWBEw)

## Contributing

AliSQL became open source in December 2025 and is actively maintained by Alibaba Cloud Database Team.

We welcome contributions of all kinds!

1. **Fork** the repository
2. **Create** a feature branch (`git checkout -b feature/amazing-feature`)
3. **Commit** your changes (`git commit -m 'Add amazing feature'`)
4. **Push** to the branch (`git push origin feature/amazing-feature`)
5. **Open** a Pull Request

For bugs and feature requests, please use [GitHub Issues](https://github.com/alibaba/AliSQL/issues).

## Related Tools

### RDSAI CLI — AI-Powered Database Assistant

<p>
  <a href="https://github.com/aliyun/rdsai-cli"><img src="https://img.shields.io/badge/GitHub-rdsai--cli-blue?style=flat-square&logo=github" alt="RDSAI CLI"></a>
  <a href="https://www.python.org/downloads/"><img src="https://img.shields.io/badge/python-3.13+-blue.svg?style=flat-square" alt="Python 3.13+"></a>
</p>

[RDSAI CLI](https://github.com/aliyun/rdsai-cli) is a next-generation, AI-powered CLI that transforms how you interact with AliSQL and MySQL databases. Describe your intent in **natural language**, and the AI agent handles the rest.

```bash
# Install
curl -LsSf https://raw.githubusercontent.com/aliyun/rdsai-cli/main/install.sh | sh

# Connect and ask in natural language
rdsai --host localhost -u root -p secret -D mydb
mysql> analyze index usage on users table
mysql> show me slow queries from the last hour
mysql> why this query is slow: SELECT * FROM users WHERE name LIKE '%john%'
```

**Key Features:**
- Natural language to SQL conversion (English/中文)
- AI-powered query optimization and diagnostics
- Execution plan analysis with `Ctrl+E`
- Multi-model LLM support (Qwen, OpenAI, DeepSeek, Anthropic, etc.)
- Performance benchmarking with automated analysis

👉 **[Get Started with RDSAI CLI](https://github.com/aliyun/rdsai-cli)**

## Community & Support

<table>
<tr>
<td align="center" width="50%">

**GitHub Issues**

For bug reports & feature requests

[Open an Issue](https://github.com/alibaba/AliSQL/issues)

</td>
<td align="center" width="50%">

**Alibaba Cloud RDS**

Managed DuckDB analytical instances

[Learn More](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/duckdb-based-analytical-instance/)

</td>
</tr>
</table>

## License

AliSQL is licensed under **GPL-2.0**, the same license as MySQL.

See the [LICENSE](LICENSE) file for details.

## Star History

<p align="center">
  <a href="https://star-history.com/#alibaba/AliSQL&Date">
    <img src="https://api.star-history.com/svg?repos=alibaba/AliSQL&type=Date" alt="Star History Chart" width="600">
  </a>
</p>

<p align="center">
  Made with care by <a href="https://www.alibabacloud.com/product/apsaradb-for-rds-mysql">Alibaba Cloud Database Team</a>
</p>

<p align="center">
  <a href="https://github.com/alibaba/AliSQL">GitHub</a> •
  <a href="https://github.com/mysql/mysql-server">MySQL</a> •
  <a href="https://github.com/duckdb/duckdb">DuckDB</a>
</p>
