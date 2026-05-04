---
name: cassandra-operating
description: Covers overall Apache Cassandra cluster operations, including backup/restore, repair, compaction, monitoring, security configuration, hardware guidelines, transient replication. Use when performing daily operational tasks on Cassandra clusters.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: cassandra
  tags: [cassandra, operations, backup, repair, compaction, monitoring, security]
---

# Apache Cassandra Operating

## Overview

Cassandra cluster operations require special attention due to distributed system characteristics. Core operational tasks — backups, repairs, compaction, monitoring — must be performed systematically to ensure stability.

**Core Principle**: Prioritize automation, continuous monitoring, preventive maintenance.

---

## SOP: Step-by-Step Procedures

### 1. Backups & Restore

```bash
# Create snapshot
nodetool snapshot -t backup_20240101 my_keyspace

# Check snapshot list
nodetool listsnapshots

# Restore data to another node using sstableloader
sstableloader -d <target_node_ip> /path/to/sstables

# Delete snapshot (after restore)
nodetool clearsnapshot -t backup_20240101
```

**Step-by-Step**:
1. Create current state snapshot with `nodetool snapshot`
2. Backup snapshot directory to external storage (S3/GCS, etc.)
3. Use `sstableloader` or `sstablerepairedex` for restore
4. Clean up snapshots after complete restore

### 2. Repair

```bash
# Full cluster repair
nodetool repair

# Specific keyspace repair
nodetool repair my_keyspace

# Archive repair (Auto Repair recommended)
# Configure in cassandra.yaml
auto_snapshot: false  # Disable snapshot during repair
```

**Step-by-Step**:
1. Schedule regular repairs (daily/weekly)
2. Recommend using `nodetool repair -pr` with partitioner ranges
3. Consider automation via Auto Repair configuration
4. Monitor during repair (check CPU, network load)

### 3. Compaction Management

```yaml
# Compaction strategy settings in cassandra.yaml
compaction_throughput_mb_per_sec: 16  # MB per second
compaction_large_partition_warning_threshold_mb: 100

# SizeTiered Compaction (write-heavy)
STCS:
  class: org.apache.cassandra.db.compaction.SizeTieredCompactionStrategy
  max_threshold: 32
  min_threshold: 4

# Leveled Compaction (read-heavy)
LCS:
  class: org.apache.cassandra.db.compaction.LeveledCompactionStrategy
```

**Step-by-Step**:
1. Select compaction strategy based on workload type
   - STCS: High write frequency
   - LCS: High read frequency
   - TWCS: Timestamp-based data (logs, events)
2. Tune `compaction_throughput_mb_per_sec`
3. Monitor compaction: `nodetool compactionstats`

### 4. Monitoring Metrics

```bash
# Check node status
nodetool status

# Check metrics
nodetool stats my_keyspace
nodetool tpstats  # Thread pool queue status
nodetool tablestats my_keyspace my_table

# GC information
nodetool gcstats

# Connection information
nodetool netstats
```

**Step-by-Step**:
1. Check cluster health with `nodetool status`
2. Monitor thread pool wait times via `nodetool tpstats`
3. Track per-table read/write latency with `nodetool tablestats`
4. Diagnose memory issues by analyzing GC logs

### 5. Security Configuration

```cql
-- Enable authentication (in cassandra.yaml)
authenticator: PasswordAuthenticator

-- Enable authorization
authorizer: CassandraAuthorizer

-- SSL/TLS configuration
client_encryption_options:
  enabled: true
  optional: false
  keystore: /path/to/keystore.jks
  keystore_password: <password>

-- RBAC permission grant
CREATE USER admin WITH PASSWORD 'secure_password' SUPERUSER;
GRANT SELECT ON KEYSPACE my_app TO app_user;
```

**Step-by-Step**:
1. Enable `PasswordAuthenticator`
2. Configure RBAC with `CassandraAuthorizer`
3. Encrypt node-to-node/client communication with SSL/TLS
4. Apply principle of least privilege (GRANT/REVOKE)

### 6. Hardware Guidelines

```yaml
# Recommended specifications
CPU: 8+ cores (compaction requires significant CPU)
RAM: 32GB+ (consider memory pool allocation)
Disk: SSD mandatory (SSTable I/O performance critical)
Network: 1Gbps or higher (node-to-node communication)

# Memory settings (jvm-server.options)
# Heap size recommended at 6-8GB (exceeding degrades performance)
-Xms6g
-Xmx6g
```

**Step-by-Step**:
1. Use SSD disks mandatory (HDD not recommended)
2. RAM for metadata storage, limit Heap to 6-8GB
3. Ensure network bandwidth (consider node-to-node streaming)
4. Verify CPU core count (compaction parallel processing)

### 7. Transient Replication

```bash
# Automatic replication when adding nodes
# Enable in cassandra.yaml
auto_bootstrap: true

# Manual transient repair
nodetool repair -pr -t <new_node_ip>
```

**Step-by-Step**:
1. Verify `auto_bootstrap: true` when adding new node
2. Run automatic repair after node joins
3. Confirm data synchronization complete (`nodetool status`)
4. Clean up transient data

---

## Tool Integration

| Tool | Purpose | Example |
|------|---------|---------|
| `run_command` | Execute nodetool, sstableloader | `nodetool repair my_keyspace` |
| `search_files` | Search log file patterns | `grep -r "ERROR" system.log` |
| `read_file` | Analyze configuration files | Check cassandra.yaml |
| `edit_file` | Apply configuration changes | Modify JVM options |

---

## Anti-Patterns & Guardrails

❌ **Skipping repairs** — Data inconsistencies accumulate, risk of data loss  
❌ **Using HDD** — Fatal SSTable I/O performance degradation  
❌ **Heap exceeding 8GB** — Increased GC pause time, risk of service interruption  
❌ **Not cleaning up snapshots** — Disk space exhaustion  
❌ **No monitoring** — Cannot detect issues early  
❌ **Disabling SSL/TLS** — Data breach risk (production)  

⚠️ **Network saturation during large repairs** — Separate by time or distribute execution  
⚠️ **Write delays during compaction** — Throttling settings recommended  
⚠️ **GC pause time exceeded** — Adjust Heap size  

---

## Best Practices

1. **Schedule regular repairs** — Daily or weekly automation
2. **Continuous monitoring** — Collect metrics and set alerts
3. **SSD disks mandatory** — Never use HDD
4. **Limit Heap to 6-8GB** — Performance degrades beyond this
5. **Enable SSL/TLS** — Mandatory for production environments
6. **Auto-clean snapshots** — Manage disk space
7. **Consider Auto Repair** — Reduce manual repair overhead

---

## References

- [Backups](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/backups.html)
- [Repair](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/repair.html)
- [Compaction](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/compaction/index.html)
- [Monitoring Metrics](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/metrics.html)
- [Security](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/security.html)
- [Hardware Guidelines](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/hardware.html)
- [Auto Repair](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/auto_repair.html)
