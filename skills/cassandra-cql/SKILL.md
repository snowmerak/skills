---
name: cassandra-cql
description: Covers the full range of CQL (Cassandra Query Language), Cassandra's query language. Handles DDL, DML, data types, indexing, Materialized Views, JSON support, security, and all CQL-related operations. Follows SQL-like syntax but with Cassandra-specific design philosophy.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: cassandra
  tags: [cassandra, cql, database, query, ddl, dml]
---

# Apache Cassandra CQL (Cassandra Query Language)

## Overview

CQL is the primary query language for interacting with Cassandra databases. It uses SQL-like syntax but requires understanding of **distributed database constraints**. CQL 3 is the current standard; table/row/column concepts are defined identically to SQL.

**Core Principle**: Cassandra is not an RDBMS. Query-driven design is essential, and denormalization takes priority over normalization.

---

## SOP: Step-by-Step Procedures

### 1. Keyspace Creation and Management

```cql
-- Create keyspace (replication setting required)
CREATE KEYSPACE IF NOT EXISTS my_app
WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'dc1': 3,
    'dc2': 3
};

-- Check available replication classes
DESCRIBE KEYSPACES;
```

**Step-by-Step**:
1. Choose `NetworkTopologyStrategy` or `SimpleStrategy` (former required for multi-DC)
2. Set replica count per datacenter (typically 3)
3. Ensure safety with `IF NOT EXISTS`
4. Subsequent modifications possible via `ALTER KEYSPACE`

### 2. Table Creation (DDL)

```cql
CREATE TABLE IF NOT EXISTS users (
    user_id UUID,
    email text,
    name text,
    created_at timestamp,
    PRIMARY KEY (user_id)
) WITH comment = 'User profile table';
```

**Step-by-Step**:
1. Define Partition Key (`PRIMARY KEY`)
2. Design sorting/range queries with Clustering Column
3. Ensure re-execution safety with `IF NOT EXISTS`
4. Set `WITH` options as needed (compaction, compression, etc.)

### 3. Data Insertion and Retrieval (DML)

```cql
-- INSERT
INSERT INTO users (user_id, email, name, created_at)
VALUES (uuid(), 'john@example.com', 'John Doe', now());

-- SELECT (Partition Key required in WHERE clause)
SELECT * FROM users WHERE user_id = <uuid_value>;

-- Range query (using Clustering Column)
SELECT * FROM orders
WHERE customer_id = <uuid>
  AND order_date >= '2024-01-01'
  AND order_date < '2024-02-01';
```

**Step-by-Step**:
1. Partition Key must be included in WHERE clause
2. Use Clustering Column for range conditions
3. `ALLOW FILTERING` causes performance degradation → prohibited (Anti-Pattern)
4. Recommend limiting result set size with `LIMIT`

### 4. Indexing Strategy

```cql
-- Standard Index (single column, suitable for small datasets)
CREATE INDEX ON users (email);

-- SASI Index (Advanced: supports prefix, order, range)
CREATE CUSTOM INDEX user_email_idx ON users (email)
USING 'org.apache.cassandra.index.sasi.SASIIndex'
WITH OPTIONS = {
    'mode': 'CONTAINS',
    'analyzer': {
        'class': 'org.apache.cassandra.index.sasi.analyzer.StandardAnalyzer'
    }
};

-- SAI Index (5.0+, Storage Attached Index - recommended)
CREATE INDEX ON users (email);  -- Auto-created as SAI by default
```

**Step-by-Step**:
1. Standard Index: use for small datasets
2. SASI: when advanced analysis features needed
3. SAI (5.0+): always recommend SAI for new projects
4. Check cardinality of indexed column before deciding

### 5. Materialized Views

```cql
-- Base table
CREATE TABLE users_by_id (
    user_id UUID PRIMARY KEY,
    email text,
    name text
);

-- Create Materialized View (auto-maintained as separate table)
CREATE MATERIALIZED VIEW users_by_email AS
SELECT * FROM users_by_id
WHERE email IS NOT NULL AND user_id IS NOT NULL
PRIMARY KEY (email, user_id);
```

**Step-by-Step**:
1. MV uses separate storage space → consider capacity
2. `WHERE` clause must include PK
3. Write performance degradation occurs (auto-sync overhead)
4. Suitable for read-only workloads

### 6. JSON Support

```cql
-- INSERT with JSON
INSERT INTO users (user_id, data_json)
VALUES (uuid(), '{"name": "John", "age": 30}');

-- JSON query
SELECT * FROM users WHERE data_json->>'name' = 'John';
```

### 7. Security (RBAC)

```cql
-- Create user
CREATE USER IF NOT EXISTS app_user WITH PASSWORD 'secure_password';

-- Grant permissions
GRANT SELECT, INSERT ON KEYSPACE my_app TO app_user;
GRANT ALL ON KEYSPACE my_app TO admin_user;

-- Check permissions
LIST PERMISSIONS OF app_user;
```

---

## Tool Integration

| Tool | Purpose | Example |
|------|---------|---------|
| `run_command` | Execute cqlsh, process scripts | `cqlsh -e "DESCRIBE KEYSPACES;"` |
| `search_files` | Search CQL query patterns | `grep -r "CREATE TABLE" *.sql` |
| `read_file` | Read/analyze CQL scripts | Verify migration files |
| `edit_file` | Modify DDL/DML | Change table structure |

**cqlsh Usage**:
```bash
# Connect
cqlsh <host> <port> -u <username> -p <password>

# Execute script
cqlsh -f migration.sql

# Specify output format
cqlsh --format=csv -e "SELECT * FROM users LIMIT 10;"
```

---

## Anti-Patterns & Guardrails

❌ **Using WHERE clause without Partition Key** — Full scan occurs, fatal performance impact  
❌ **Overusing `ALLOW FILTERING`** — Triggers full node scans in distributed environment  
❌ **Applying RDBMS normalization patterns** — Denormalization is mandatory for Cassandra  
❌ **Excessive writes to same Partition** — Hotspot occurs, replica overload  
❌ **Using Standard Index on large columns** — Memory overhead spikes  
❌ **Overusing Materialized Views** — Write performance degradation + 2x storage space  

⚠️ **Cannot change Clustering Column order** — Cannot alter via ALTER TABLE, requires table recreation  
⚠️ **`now()` function uses server time** — May differ from client time  
⚠️ **TTL supported only at row/column level** — Query-level TTL not available  

---

## Best Practices

1. **Design queries first, tables second** — Core Cassandra philosophy
2. **High-cardinality Partition Key** — Essential for data distribution
3. **Clustering Column for sorting/range purposes** — Must match ORDER BY
4. **Minimize Batch usage** — Only `UNLOGGED BATCH` allowed, `LOGGED BATCH` degrades performance
5. **Prioritize SAI (Storage Attached Index)** — Always use SAI for 5.0+ projects
6. **NetworkTopologyStrategy mandatory in multi-DC environments**
7. **Version control CQL scripts** — Can integrate with Flyway/Liquibase

---

## References

- [CQL Definitions](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/definitions.html)
- [Data Types](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/types.html)
- [DDL Reference](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/ddl.html)
- [DML Reference](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/dml.html)
- [Indexing Concepts](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/indexing/indexing-concepts.html)
- [Materialized Views](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/mvs.html)
- [CQL Security](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/security.html)
- [JSON Support](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/json.html)
