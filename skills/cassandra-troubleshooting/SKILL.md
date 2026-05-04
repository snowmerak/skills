---
name: cassandra-troubleshooting
description: Covers Apache Cassandra cluster troubleshooting methods, including finding misbehaving nodes, log analysis, nodetool-based debugging, in-depth analysis using external tools. Use when diagnosing and resolving operational issues in Cassandra clusters.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: cassandra
  tags: [cassandra, troubleshooting, debugging, logs, nodetool, diagnostics]
---

# Apache Cassandra Troubleshooting

## Overview

Cassandra, as a distributed database, can encounter issues for various reasons. Systematic troubleshooting approaches and tool usage are essential. Typically proceeds in 3 stages: **identify misbehaving nodes → analyze logs → in-depth diagnosis with tools**.

**Core Principle**: Systematic approach, data-driven diagnostics, step-by-step narrowing.

---

## SOP: Step-by-Step Procedures

### 1. Finding Misbehaving Nodes

```bash
# Check cluster status
nodetool status
# Look for DOWN nodes, UNBALANCED partitions

# Check token distribution
nodetool status -p

# Check node response times
nodetool tpstats
# Increasing Blocked Operations count is a warning sign

# Check connection status
nodetool netstats
# Abnormal increase in Active connections requires diagnosis

# Check GC status
nodetool gcstats
# GC pause time exceeding 500ms indicates problems
```

**Step-by-Step**:
1. Identify DOWN/UNBALANCED nodes with `nodetool status`
2. Check Blocked Operations via `nodetool tpstats`
3. Measure GC pause times with `nodetool gcstats`
4. Prioritize diagnosis of anomalous nodes

### 2. Reading Cassandra Logs

```bash
# Main log file locations
/var/log/cassandra/system.log      # System events
/var/log/cassandra/debug.log       # Debug information
/var/log/cassandra/gc.log          # GC logs
/var/log/cassandra/audit.log       # Security audit logs

# Filter ERROR level
grep "ERROR" /var/log/cassandra/system.log | tail -100

# Search for specific patterns (e.g., Timeout)
grep "TimeoutException" /var/log/cassandra/system.log

# Analyze GC issues
grep "GC pause" /var/log/cassandra/gc.log

# Real-time log monitoring
tail -f /var/log/cassandra/system.log
```

**Common Error Patterns**:

| Pattern | Cause | Resolution |
|---------|-------|------------|
| `TimeoutException` | Network/disk latency | Check node status, disk I/O |
| `OutOfMemoryError` | Insufficient Heap memory | Adjust Heap size, diagnose memory leaks |
| `CompactionException` | Compaction failure | Check disk space, compaction settings |
| `BootstrapException` | Node join failure | Check network, token ranges |
| `CorruptionException` | Data corruption | Verify SSTable integrity |

**Step-by-Step**:
1. Filter ERROR/WARN levels in `system.log`
2. Identify problem timeframes by timestamp
3. Search for related patterns (Timeout, OOM, etc.)
4. Cross-validate memory issues with GC logs

### 3. Debugging with nodetool

```bash
# In-depth thread pool analysis
nodetool tpstats
# Waiting on freeable slot: number of queued operations
# Blocked operations: number of blocked operations (should be 0)

# Detailed per-table metrics
nodetool tablestats my_keyspace my_table
# Check Read latency, Write latency, Row cache hit rate

# Compaction status analysis
nodetool compactionstats
# Check Active compactions, Estimated progress

# Check hint status
nodetool showhints
# Verify unprocessed hints count

# Cache statistics
nodetool cachestats
# Check Key cache, Row cache hit rate

# Disk usage
nodetool info
# Check Disk usage, Space available
```

**Step-by-Step**:
1. Check thread pool status via `tpstats` (Blocked should = 0)
2. Analyze per-table latency with `tablestats`
3. Track compaction load with `compactionstats`
4. Verify disk space with `info`

### 4. In-Depth Analysis Using External Tools

```bash
# JVM heap dump (on OutOfMemoryError)
jmap -dump:format=b,file=heap.hprof <pid>

# Thread dump (on Hang state)
jstack <pid> > thread_dump.txt

# Check network connections
netstat -an | grep 9042
ss -tlnp | grep 7000

# Monitor disk I/O
iostat -x 1
iotop

# Analyze memory usage
free -m
vmstat 1

# Check process status
ps aux | grep cassandra
```

**JVM Heap Dump Analysis**:
```bash
# Use MAT (Memory Analyzer Tool) or jhat
jhat heap.hprof
# Access via browser at http://localhost:7000
```

**Thread Dump Analysis**:
```bash
# Identify blocked threads
grep "BLOCKED" thread_dump.txt
# Check for deadlock patterns
grep "waiting to lock" thread_dump.txt
```

**Step-by-Step**:
1. Dump thread status with `jstack`
2. Dump heap memory with `jmap` (on OOM)
3. Analyze with external tools (MAT, jhat)
4. Identify blocking patterns/memory leaks

### 5. Common Problem Scenarios and Solutions

**Scenario 1: Write Latency Spike**
```bash
# Diagnosis
nodetool tpstats          # Check Blocked operations
nodetool compactionstats  # Check compaction load
iostat -x 1               # Check disk I/O bottleneck

# Resolution
nodetool setcompactionthroughput 32  # Increase compaction throttle
```

**Scenario 2: Read Latency Spike**
```bash
# Diagnosis
nodetool tablestats keyspace table  # Check Read latency
nodetool cachestats                  # Check Cache hit rate
nodetool tpstats                     # Check Read thread wait

# Resolution
# Improve cache efficiency, review index optimization
```

**Scenario 3: Node Down**
```bash
# Diagnosis
nodetool status              # Identify DOWN node
grep "ERROR" system.log      # Check termination cause
dmesg | tail                 # Check kernel messages (OOM Killer, etc.)

# Resolution
# OOM: Increase Heap size or fix memory leak
# Disk full: Clean data or expand disk
```

**Scenario 4: Data Inconsistency**
```bash
# Diagnosis
nodetool repair -pr keyspace table  # Per-partition repair
nodetool verify keyspace table       # Data integrity check

# Resolution
# Schedule regular repairs
# Verify auto_snapshot is enabled
```

---

## Tool Integration

| Tool | Purpose | Example |
|------|---------|---------|
| `run_command` | Execute nodetool, jstack, jmap | `nodetool tpstats`, `jstack <pid>` |
| `search_files` | Search error patterns in log files | `grep "ERROR" system.log` |
| `read_file` | Analyze logs/configuration files | Check system.log, cassandra.yaml |
| `edit_file` | Resolve issues via configuration changes | Adjust compaction_throughput |

---

## Anti-Patterns & Guardrails

❌ **Diagnosing without logs** — Data-driven approach is mandatory  
❌ **Enabling DEBUG logging in production** — Disk exhaustion, performance degradation  
❌ **Frequent execution of jmap heap dumps** — Increased service interruption time  
❌ **Ignoring data inconsistencies without repair** — Accumulating data loss  
❌ **Ignoring Blocked Operations** — Problem escalation, service interruption  

⚠️ **GC pause time exceeding 500ms** — Triggers client timeouts  
⚠️ **Disk usage exceeding 85%** — Risk of write failures  
⚠️ **Thread pool exhaustion** — All requests waiting, service unavailable  

---

## Best Practices

1. **Continuous monitoring** — Collect metrics and set alerts
2. **Configure log rolling mandatory** — Prevent disk exhaustion
3. **Regular repairs** — Prevent data inconsistencies
4. **Monitor GC logs** — Early detection of memory issues
5. **Always check tpstats** — Maintain Blocked Operations = 0
6. **Ensure disk space** — Keep at least 15% free
7. **Document problem scenarios** — Standardize response procedures

---

## References

- [Finding Misbehaving Nodes](https://cassandra.apache.org/doc/latest/cassandra/troubleshooting/finding_nodes.html)
- [Reading Cassandra Logs](https://cassandra.apache.org/doc/latest/cassandra/troubleshooting/reading_logs.html)
- [Using nodetool](https://cassandra.apache.org/doc/latest/cassandra/troubleshooting/use_nodetool.html)
- [Using External Tools](https://cassandra.apache.org/doc/latest/cassandra/troubleshooting/use_tools.html)
