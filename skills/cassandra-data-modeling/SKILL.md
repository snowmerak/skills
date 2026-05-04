---
name: cassandra-data-modeling
description: Covers Cassandra-specific data modeling, including RDBMS vs NoSQL design differences, query-first approach, Conceptual/Logical/Physical design stages, Partition Key and Clustering Column strategies, denormalization patterns. Use when designing or migrating Cassandra schemas.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: cassandra
  tags: [cassandra, data-modeling, nosql, schema-design, partition-key]
---

# Apache Cassandra Data Modeling

## Overview

Cassandra data modeling is fundamentally different from RDBMS. **Define query patterns first**, then design tables accordingly. Denormalization takes priority over normalization; read performance is the top consideration.

**Core Philosophy**: "Query-first design" — think about queries first, design tables second.

---

## SOP: Step-by-Step Procedures

### 1. Conceptual Design

```
User Requirements → Query Pattern Analysis → Entity Identification
```

**Step-by-Step**:
1. List all read queries for the application
2. Identify filter conditions, sorting criteria, join requirements for each query
3. Estimate data access frequency and size
4. Verify consistency requirements (Strong vs Eventual)

### 2. Logical Design

```cql
-- Example: Order system
CREATE TABLE orders_by_customer (
    customer_id UUID,
    order_id UUID,
    order_date timestamp,
    total_amount decimal,
    status text,
    items list<text>,
    PRIMARY KEY (customer_id, order_date, order_id)
) WITH CLUSTERING ORDER BY (order_date DESC, order_id ASC);

-- Separate table: for order detail lookup
CREATE TABLE orders_by_id (
    order_id UUID PRIMARY KEY,
    customer_id UUID,
    order_date timestamp,
    total_amount decimal,
    status text,
    items list<text>
);
```

**Step-by-Step**:
1. Create separate table for each query pattern
2. Partition Key: columns with equality conditions in WHERE clause
3. Clustering Column: columns for ORDER BY and range conditions
4. Allow duplicate data (denormalization)

### 3. Physical Design

```cql
-- TTL setting (for temporary data)
CREATE TABLE sessions (
    session_id UUID PRIMARY KEY,
    user_id UUID,
    data text,
    created_at timestamp
) WITH ttl = 86400;  -- Auto-delete after 24 hours

-- Compaction strategy setting
CREATE TABLE events (
    event_id UUID PRIMARY KEY,
    event_type text,
    payload text,
    created_at timestamp
) WITH compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'window_size_in_seconds': 3600
};

-- Compression setting
CREATE TABLE logs (
    log_id UUID PRIMARY KEY,
    message text,
    level text,
    created_at timestamp
) WITH compression = {
    'sstable_compression': 'SnappyCompressor'
};
```

**Step-by-Step**:
1. Analyze data lifecycle → decide on TTL application
2. Select compaction strategy based on write/read ratio
3. Optimize data types (text vs blob, list vs set)
4. Consider storage space → configure compression

### 4. RDBMS Migration Patterns

```sql
-- RDBMS (using JOINs)
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id;
```

```cql
-- Cassandra (denormalized)
CREATE TABLE customer_orders (
    customer_id UUID,
    order_id UUID,
    customer_name text,  -- Stored redundantly
    order_date timestamp,
    total_amount decimal,
    PRIMARY KEY (customer_id, order_date, order_id)
);

-- Separate table: for customer info lookup
CREATE TABLE customers_by_id (
    customer_id UUID PRIMARY KEY,
    name text,
    email text,
    created_at timestamp
);
```

**Step-by-Step**:
1. Identify join patterns
2. Separate each entity into its own table
3. Store frequently accessed-together data redundantly
4. Establish update strategy for consistency maintenance

### 5. Model Evaluation and Improvement

```cql
-- Check hotspots (use nodetool stats)
nodetool stats my_keyspace

-- Check partition size
SELECT * FROM system_schema.tables 
WHERE keyspace_name = 'my_app';
```

**Step-by-Step**:
1. Monitor partition size (recommended: under 100MB)
2. Measure read/write latency
3. Check compaction load
4. Split or merge tables if needed

---

## Tool Integration

| Tool | Purpose | Example |
|------|---------|---------|
| `run_command` | Check metrics via nodetool | `nodetool stats keyspace_name` |
| `search_files` | Search query patterns | `grep -r "WHERE" *.sql` |
| `read_file` | Analyze schema files | Read schema migration files |
| `edit_file` | Modify table structure | Apply DDL changes |

---

## Anti-Patterns & Guardrails

❌ **Attempting to use JOINs** — Cassandra does not support JOINs (resolve via denormalization)  
❌ **Excessive data in single partition** — Performance degradation, unreadable when exceeding 100MB  
❌ **Writes concentrated on same Partition Key** — Hotspot occurs, replica overload  
❌ **Applying RDBMS normalization patterns** — Denormalization is mandatory for Cassandra  
❌ **Changing Clustering Column order** — Cannot change via ALTER TABLE, requires recreation  
❌ **Using large list/set** — Performance degradation when exceeding 100k items  

⚠️ **Low Partition Key cardinality** — Data concentration occurs, distribution fails  
⚠️ **TTL conflicts with Compaction strategy** — TWCS recommended when TTL is applied  
⚠️ **Overusing Materialized Views** — Write performance degradation + 2x storage space  

---

## Best Practices

1. **Analyze query patterns first** — List all read queries
2. **High-cardinality Partition Key** — Ensure data distribution
3. **Clustering Column for sorting purposes** — Match ORDER BY
4. **Actively use denormalization** — Allow duplicate data, prioritize read performance
5. **Monitor partition size** — Recommended to keep under 100MB
6. **Prevent hotspots** — Consider adding sequence/timestamp to Partition Key
7. **Table separation strategy** — Create separate table for each query pattern

---

## References

- [Introduction to Data Modeling](https://cassandra.apache.org/doc/latest/cassandra/developing/data-modeling/intro.html)
- [Data Modeling Best Practices](https://cassandra.apache.org/doc/latest/cassandra/developing/data-modeling/best_practices.html)
- [Query-Driven Design](https://cassandra.apache.org/doc/latest/cassandra/developing/data-modeling/query-driven.html)
