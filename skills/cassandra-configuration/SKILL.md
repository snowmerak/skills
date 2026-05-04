---
name: cassandra-configuration
description: Covers all configuration files of Apache Cassandra, including core parameters in cassandra.yaml, JVM options (jvm-*), logging (logback.xml), environment variables (cassandra-env.sh). Use when understanding and tuning any Cassandra configuration file.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: cassandra
  tags: [cassandra, configuration, cassandra-yaml, jvm-options, logback]
---

# Apache Cassandra Configuration

## Overview

Cassandra controls its behavior through multiple configuration files. Understanding the role and parameters of each file is essential for operating a stable, high-performance cluster.

**Core Principle**: Change defaults cautiously, tune based on monitoring, verify in test environment before applying.

---

## SOP: Step-by-Step Procedures

### 1. cassandra.yaml — Core Configuration File

```yaml
# Cluster configuration
cluster_name: 'MyCassandraCluster'
num_tokens: 256  # Number of vnodes (default for 3.0+)

# Directory settings
data_file_directories:
  - /var/lib/cassandra/data
commitlog_directory: /var/lib/cassandra/commitlog
saved_caches_directory: /var/lib/cassandra/saved_caches
hints_directory: /var/lib/cassandra/hints

# Memory settings (integrated with JVM options)
memtable_allocation_type: heap_buffers  # heap_buffers/direct_buffers/offheap_buffers
memtable_heap_space_in_mb: 2048
memtable_offheap_space_in_mb: 2048

# Network configuration
native_transport_port: 9042
storage_port: 7000
ssl_storage_port: 7001
listen_address: localhost
rpc_address: localhost

# Replication settings
endpoint_snitch: GossipingPropertyFileSnitch
num_cpus: 8  # Manual setting if auto-detection fails

# Compaction settings
compaction_throughput_mb_per_sec: 16
compaction_large_partition_warning_threshold_mb: 100
concurrent_compactors: 2

# Read/Write timeout settings
read_request_timeout_in_ms: 5000
write_request_timeout_in_ms: 2000
range_request_timeout_in_ms: 10000

# Cache settings
key_cache_size_in_mb:
key_cache_save_period: 14400
row_cache_size_in_mb: 0  # Disabled recommended
row_cache_save_period: 0

# Logging settings
trickle_fsync: false
trickle_fsync_interval_in_kb: 10240

# Auto-snapshot (during repair)
auto_snapshot: true
```

**Step-by-Step**:
1. Change `cluster_name` to a unique value
2. Verify directory paths (separate disks recommended)
3. Keep `num_tokens` at 256 (vnode enabled)
4. Verify memory settings (match JVM options)
5. Verify network ports (open firewall)

### 2. JVM Options — jvm-server.options, etc.

```bash
# /etc/cassandra/jvm-server.options
# Heap size (recommended 6-8GB, do not exceed)
-Xms6g
-Xmx6g

# GC settings (G1GC recommended)
-XX:+UseG1GC
-XX:G1ReservePercent=25
-XX:InitiatingHeapOccupancyPercent=30

# Debug options (not recommended for production)
# -Xloggc:/var/log/cassandra/gc.log
# -XX:+PrintGCDetails
# -XX:+PrintGCDateStamps

# Direct memory allocation (for offheap memtable)
-XX:MaxDirectMemorySize=<value_in_mb>
```

**Step-by-Step**:
1. Set Heap size to 6-8GB (exceeding risks GC pauses)
2. Enable G1GC (default for Cassandra 3.0+)
3. Match `MaxDirectMemorySize` with offheap memtable size
4. Enable GC logs only for troubleshooting

### 3. cassandra-env.sh — Environment Variables

```bash
#!/bin/bash
# Set JVM options file path
JVM_OPTS="$JVM_OPTS -Dcassandra.logdir=/var/log/cassandra"
JVM_OPTS="$JVM_OPTS -Dcassandra.storagedir=/var/lib/cassandra"

# Additional JVM options (optional)
# JVM_OPTS="$JVM_OPTS -XX:+AlwaysPreTouch"  # Pre-allocate memory

# Check system limits
ulimit -n 100000  # File descriptor limit
```

**Step-by-Step**:
1. Set log/data directory paths
2. Use only when additional JVM options are needed
3. Verify and adjust system limits (ulimit)

### 4. logback.xml — Logging Configuration

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- System log -->
    <appender name="SYSTEM_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/cassandra/system.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>/var/log/cassandra/system.log.%d{yyyy-MM-dd}.%i.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%date %-5level [%thread] %logger{40}: %msg%n</pattern>
        </encoder>
    </appender>

    <!-- GC log -->
    <appender name="GC_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/cassandra/gc.log</file>
        ...
    </appender>

    <!-- Logging level settings -->
    <logger name="org.apache.cassandra" level="INFO"/>
    <logger name="com.datastax" level="WARN"/>
</configuration>
```

**Step-by-Step**:
1. Verify log directory (`/var/log/cassandra`)
2. Configure log rolling (size/time-based)
3. Adjust logging levels (DEBUG only for troubleshooting)
4. Enable GC logs if needed

### 5. cassandra-rackdc.properties — Rack/Datacenter Configuration

```properties
# Datacenter name
dc=dc1

# Rack name
rack=rack1
```

**Step-by-Step**:
1. Verify Snitch type matching `endpoint_snitch`
2. Required when using PropertyFileSnitch/GossipingPropertyFileSnitch
3. Unique dc/rack settings per node
4. Use with NetworkTopologyStrategy in multi-DC environments

### 6. cassandra-topologies.properties — Network Topology (EC2)

```properties
# AWS EC2 topology mapping
10.0.1.1 = us-east-1a
10.0.1.2 = us-east-1b
10.0.2.1 = us-east-1c
```

### 7. commitlog-archiving.properties — CommitLog Archiving

```properties
# CommitLog archiving configuration
strategy=org.apache.cassandra.archiving.PassiveArchiver
params:{target:/backup/commitlogs, keep:5}
```

---

## Tool Integration

| Tool | Purpose | Example |
|------|---------|---------|
| `read_file` | Read/analyze config files | Check cassandra.yaml, JVM options |
| `edit_file` | Modify configuration files | Apply parameter changes |
| `search_files` | Search for config patterns | `grep "compaction" *.yaml` |
| `run_command` | Verify and apply settings | `nodetool getcompactionthroughput` |

---

## Anti-Patterns & Guardrails

❌ **Heap exceeding 8GB** — Increases GC pause time, risk of service interruption  
❌ **Enabling row_cache** — Wastes memory, not recommended  
❌ **DEBUG logging in production** — Disk exhaustion, performance degradation  
❌ **Manually setting num_cpus (when auto-detection works)** — Misconfiguration from wrong values  
❌ **Storing all data in single directory** — I/O contention, performance degradation  

⚠️ **Restart required after configuration changes** — Some parameters cannot be changed dynamically  
⚠️ **Mismatch between JVM options and cassandra.yaml memory settings** — Risk of memory over-allocation  
⚠️ **No log rolling configured** — Disk exhaustion  

---

## Best Practices

1. **Limit Heap to 6-8GB** — GC performance degrades beyond this
2. **Enable G1GC** — Default for Cassandra 3.0+, optimized
3. **Separate directories** — Data/commitlog/cache on separate disks
4. **Configure log rolling mandatory** — Prevent disk exhaustion
5. **Disable row_cache** — Efficient memory usage
6. **Verify in test environment before changes** — Never change production directly
7. **Version control configuration files** — Track changes with Git

---

## References

- [cassandra.yaml Reference](https://cassandra.apache.org/doc/latest/cassandra/managing/configuration/cass_yaml_file.html)
- [cassandra-env.sh](https://cassandra.apache.org/doc/latest/cassandra/managing/configuration/cass_env_sh_file.html)
- [jvm-* Options](https://cassandra.apache.org/doc/latest/cassandra/managing/configuration/cass_jvm_options_file.html)
- [logback.xml](https://cassandra.apache.org/doc/latest/cassandra/managing/configuration/cass_logback_xml_file.html)
- [Configuration Parameters](https://cassandra.apache.org/doc/latest/cassandra/managing/configuration/configuration.html)
