---
name: cassandra-tools
description: Covers Apache Cassandra command-line tools collection including cqlsh (CQL shell), nodetool (cluster management), SSTable tools, cassandra-stress (performance testing). Use when working with any official Cassandra CLI utilities.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: cassandra
  tags: [cassandra, cqlsh, nodetool, sstable, cassandra-stress, cli-tools]
---

# Apache Cassandra Tools

## Overview

Cassandra provides various command-line tools for cluster management, data manipulation, and performance testing. This skill covers core usage of cqlsh, nodetool, SSTable tools, and cassandra-stress.

**Core Principle**: Understand each tool's specialty and apply in appropriate situations.

---

## SOP: Step-by-Step Procedures

### 1. cqlsh — CQL Shell

```bash
# Basic connection
cqlsh <host> <port> -u <username> -p <password>

# SSL connection
cqlsh --ssl <host>

# Execute script
cqlsh -f migration.sql

# CSV output format
cqlsh --format=csv -e "SELECT * FROM users LIMIT 10;"

# Automate cqlsh with Python scripts
echo "DESCRIBE KEYSPACES;" | cqlsh localhost
```

**cqlsh Configuration File**: `~/.cassandra/cqlshrc`

```ini
[connection]
hostname = localhost
port = 9042
factory = cqlshlib.cql.connection_factory

[auth]
username = admin
password = secure_password

[display]
paging = true
page_size = 100
```

**Step-by-Step**:
1. Verify connection settings (host, port, authentication)
2. Execute script files with `-f` option
3. Specify output format with `--format=csv/json`
4. Autocomplete enabled by default

### 2. nodetool — Cluster Management

```bash
# Check cluster status
nodetool status
nodetool status -p  # Include port information

# Node metrics
nodetool stats my_keyspace
nodetool tablestats my_table
nodetool tpstats    # Thread pool queue status

# Compaction management
nodetool compactionstats      # Check ongoing compactions
nodetool compact              # Trigger manual compaction
nodetool compactionhistory    # Compaction history

# Snapshot management
nodetool snapshot -t backup_20240101 my_keyspace
nodetool listsnapshots
nodetool clearsnapshot -t backup_20240101

# Repair
nodetool repair my_keyspace
nodetool repair -pr           # Use partitioner ranges
nodetool repair -full         # Full cluster repair

# Node control
nodetool drain                # Prepare node shutdown after data migration
nodetool stopdaemon           # Stop nodetool service
nodetool startdaemon          # Start nodetool service

# View/change settings (dynamic)
nodetool getcompactionthroughput
nodetool setcompactionthroughput 16
nodetool getlogginglevels
```

**Step-by-Step**:
1. Check cluster health with `nodetool status`
2. Monitor thread pool wait times via `nodetool tpstats`
3. Track compaction load with `nodetool compactionstats`
4. Tune performance with dynamic settings changes

### 3. SSTable Tools — SSTable Processing

```bash
# Convert SSTable to CSV
sstable2csv /path/to/sstables > data.csv

# Convert CSV to SSTable
csv2sstable -p <partition_key_index> data.csv my_table

# Analyze SSTable structure
sstabledump /path/to/sstable/my_table-abc123.db

# Debug SSTable
sstableverify keyspace table /path/to/sstables

# Check SSTable metadata
sstablemetadata /path/to/sstable/my_table-abc123.db

# Split/merge SSTables
sstablerepairedex  # Extract repaired data
sstablelevelize    # Force leveled compaction
```

**Step-by-Step**:
1. Export data with `sstable2csv`
2. Import data with `csv2sstable` (partition key index required)
3. Analyze SSTable structure with `sstabledump`
4. Verify integrity with `sstableverify`

### 4. cassandra-stress — Performance Testing

```bash
# Basic write test
cassandra-stress write n=100000 -mode cql3 user=admin password=password \
  -schema "keyspace=test table=users replicas=1" \
  -pop seq=1..100000 -rate threads=10

# Read test
cassandra-stress read n=100000 -mode cql3 user=admin password=password \
  -schema "keyspace=test table=users replicas=1" \
  -pop uniform=1..100000 -rate threads=10

# Mixed workload (70% read, 30% write)
cassandra-stress mixed n=1000000 -mode cql3 user=admin password=password \
  -schema "keyspace=test table=users replicas=1" \
  -pop seq=1..1000000 -rate threads=50 \
  -ops '{"write":1,"select":3}'

# Use custom queries
cassandra-stress write n=10000 -mode cql3 native user=admin password=password \
  -schema "keyspace=test table=users replicas=1" \
  -pop seq=1..10000 \
  -query "INSERT INTO users (id, name) VALUES (?, ?)" \
  -rate threads=20
```

**Step-by-Step**:
1. Define test purpose (write/read/mixed)
2. Set data size (n) and thread count (threads)
3. Select workload pattern (population): seq/uniform/gaussian
4. Analyze results: latency, TPS, error rate

### 5. sstableloader — Bulk Data Load

```bash
# Copy SSTables to another node
sstableloader -d <target_node_ip> /path/to/sstables

# With authentication
sstableloader -u admin -p password -d <target_node_ip> /path/to/sstables

# Using SSL
sstableloader --ssl -d <target_node_ip> /path/to/sstables

# Throttling (limit network bandwidth)
sstableloader -t 100 -d <target_node_ip> /path/to/sstables
```

**Step-by-Step**:
1. Prepare SSTable directory
2. Specify target node IP
3. Verify authentication/SSL settings
4. Apply throttling considering network bandwidth

---

## Tool Integration

| Tool | Purpose | Example |
|------|---------|---------|
| `run_command` | Execute all Cassandra CLI tools | `nodetool status`, `cqlsh -f script.sql` |
| `search_files` | Search patterns in logs/config files | `grep "ERROR" system.log` |
| `read_file` | Read scripts/configuration files | Analyze migration.sql, cqlshrc |
| `edit_file` | Modify configuration files | Record nodetool dynamic setting changes |

---

## Anti-Patterns & Guardrails

❌ **Frequent execution of `nodetool repair -full`** — Cluster overload, performance degradation  
❌ **Using sstabledump on large SSTables** — Risk of memory exhaustion  
❌ **Not cleaning up after cassandra-stress tests** — Test data leakage  
❌ **Running sstableloader without throttling** — Network saturation, service impact  
❌ **Removing nodes without `nodetool drain`** — Risk of data loss  

⚠️ **cqlsh bulk INSERT not recommended** — Batch size limit (recommended under 1000)  
⚠️ **Frequent execution of `nodetool compact`** — Increased compaction overhead  
⚠️ **Using sstable2csv on large datasets** — Consider disk space  

---

## Best Practices

1. **Always check `nodetool status`** — Monitor cluster health
2. **Version control cqlsh scripts** — Manage as migration files
3. **Regular cassandra-stress testing** — Maintain performance baselines
4. **Use sstabledump for debugging** — Analyze SSTable structure
5. **Mandatory throttling with sstableloader** — Consider network bandwidth
6. **Leverage nodetool dynamic settings** — Tune without restart
7. **Automate tool output** — Build monitoring pipelines with scripts

---

## References

- [cqlsh Documentation](https://cassandra.apache.org/doc/latest/cassandra/managing/tools/cqlsh.html)
- [nodetool Reference](https://cassandra.apache.org/doc/latest/cassandra/managing/tools/nodetool/nodetool.html)
- [SSTable Tools](https://cassandra.apache.org/doc/latest/cassandra/managing/tools/sstable/index.html)
- [cassandra-stress](https://cassandra.apache.org/doc/latest/cassandra/managing/tools/cassandra_stress.adoc)
