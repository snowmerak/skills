---
name: cassandra-architecture
description: Covers internal architecture and operational principles of Apache Cassandra, including Dynamo-based design, Storage Engine structure (SSTable/Memtable/CommitLog), consistency guarantees (Consistency Levels), internode messaging and streaming, Snitches, and more. Use when working with Cassandra's distributed system internals.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: cassandra
  tags: [cassandra, architecture, storage-engine, consistency, dynamo, snitch]
---

# Apache Cassandra Architecture

## Overview

Cassandra is a distributed NoSQL database combining Amazon's Dynamo paper and Google's Bigtable paper. Its core design goals are high availability, linear scalability, and multi-datacenter support.

**Core Philosophy**: Eliminate single points of failure, partition-based distribution, accept eventual consistency.

---

## SOP: Step-by-Step Procedures

### 1. Understand Dynamo-Based Design

```
Ring Structure (Virtual Nodes):
┌─────────────────────────────────────────────┐
│  vnode  │  vnode  │  vnode  │  vnode       │
│  N3     │  N1     │  N4     │  N2          │
└─────────────────────────────────────────────┘
         ↑           ↑           ↑
      Token Range  Token Range  Token Range
```

**Step-by-Step**:
1. Understand Ring structure: nodes arranged in a circle
2. Optimize data distribution with Virtual Nodes (vnode, default: 256)
3. Partition data using Token Ranges
4. Balance load with Consistent Hashing

### 2. Storage Engine Structure

```
Write Path:
Client → CommitLog → Memtable → SSTable

Read Path:
Client → Query Cache → Partition Cache → Memtable + SSTables
```

**Component Breakdown**:
- **CommitLog**: Persistent log for crash recovery (append-only)
- **Memtable**: In-memory write buffer (sorted structure)
- **SSTable**: Read-only static files stored on disk
- **Query Cache**: Caching of read results (Row cache)
- **Partition Cache**: Caching of partition metadata

**Step-by-Step**:
1. Write: Record to CommitLog → Memtable in order
2. When Memtable reaches threshold, flush → create SSTable
3. Read: Sequential lookup through Cache → Memtable → SSTables
4. Merge multiple SSTables via Compaction

### 3. Consistency Guarantees (Consistency Levels)

```cql
-- Examples of using various Consistency Levels

-- Strongest consistency (verify all replicas)
INSERT INTO users (id, name) VALUES (1, 'John') USING CONSISTENCY ALL;

-- Balanced consistency (QUORUM recommended)
SELECT * FROM users WHERE id = 1 USING CONSISTENCY QUORUM;

-- Fast read (ONE: verify minimum one)
SELECT * FROM users WHERE id = 1 USING CONSISTENCY ONE;
```

**Consistency Level Comparison**:

| CL | Write Verify | Read Verify | Description |
|----|--------------|-------------|-------------|
| ANY | X | X | Allows Hinted Handoff, fastest |
| ONE | 1 | 1 | Verify minimum one node |
| QUORUM | N/2+1 | N/2+1 | Verify majority (recommended) |
| LOCAL_QUORUM | DC: N/2+1 | DC: N/2+1 | Verify only same DC |
| ALL | N | N | Verify all replicas |
| SERIAL | N/2+1 | N/2+1 | For conditional updates |

**Step-by-Step**:
1. Select CL based on workload
   - Read-heavy: QUORUM or LOCAL_QUORUM
   - Write-heavy: ONE or QUORUM
   - Financial/critical data: ALL or SERIAL
2. Understand relationship between Replication Factor and CL (RF > CL required)
3. Consider availability vs consistency tradeoffs

### 4. Internode Messaging

```yaml
# cassandra.yaml messaging configuration
native_transport_port: 9042  # CQL client port
ssl_storage_port: 7001       # Node-to-node communication with SSL
storage_port: 7000           # Node-to-node communication (non-SSL)
```

**Step-by-Step**:
1. Verify node-to-node communication ports (7000/7001)
2. Verify CQL client port (9042)
3. Separate ports when using SSL
4. Open ports with firewall rules

### 5. Streaming Mechanism

```bash
# Check streaming status
nodetool netstats

# Trigger manual streaming (runs automatically on node add/remove)
nodetool move <token_value>
```

**Step-by-Step**:
1. Data relocation starts automatically when adding/removing nodes
2. Check streaming progress with `nodetool netstats`
3. Set `stream_throughput_outbound_megabits_per_sec` if bandwidth limiting needed
4. Verify cluster status after streaming completes

### 6. Understand and Configure Snitches

```yaml
# cassandra.yaml Snitch configuration
endpoint_snitch: GossipingPropertyFileSnitch
```

**Snitch Types**:

| Snitch | Use Case | Description |
|--------|----------|-------------|
| SimpleSnitch | Single DC | Places replicas in round-robin fashion |
| PropertyFileSnitch | Testing/Small-scale | Direct definition via config file |
| GossipingPropertyFileSnitch | Production (single DC) | Default, uses gossip protocol |
| EC2Snitch | AWS | Detects region/availability zone via AWS API |
| EC2MultiRegionSnitch | AWS Multi-region | Supports cross-region replication |
| GCESnitch | Google Cloud | Detects GCP project/zone |

**Step-by-Step**:
1. Select Snitch appropriate for cloud environment
2. Configure rack/datacenter with `cassandra-rackdc.properties`
3. Use with NetworkTopologyStrategy in multi-DC environments
4. Cluster restart required when changing Snitch

---

## Tool Integration

| Tool | Purpose | Example |
|------|---------|---------|
| `run_command` | Check architecture status via nodetool | `nodetool status -p` |
| `search_files` | Search for Snitch/CL patterns in config files | `grep "endpoint_snitch" *.yaml` |
| `read_file` | Analyze cassandra.yaml | Verify architecture-related settings |
| `edit_file` | Change Snitch/CL configuration | Modify endpoint_snitch |

---

## Anti-Patterns & Guardrails

❌ **Using SimpleSnitch in multi-DC environment** — Fails to distribute replicas across datacenters  
❌ **Overusing ALL Consistency Level** — Spikes write latency, reduces availability  
❌ **Replication Factor < Consistency Level** — Write failures occur  
❌ **Disabling vnode** — Causes data imbalance, difficult relocation  
❌ **Insufficient CommitLog disk space** — Writes fail, service interruption  

⚠️ **Cluster restart required when changing Snitch** — Use planned maintenance windows  
⚠️ **Verify replica count when changing CL** — Errors occur if RF < CL  
⚠️ **Never remove nodes during streaming** — Risk of data loss  

---

## Best Practices

1. **Enable vnode** (default) — Optimize data distribution
2. **Recommend QUORUM Consistency Level** — Balance consistency/availability
3. **RF ≥ 3** — Ensure fault tolerance
4. **Use cloud-specific Snitches** — Select AWS/GCP/Azure dedicated Snitch
5. **Continuous monitoring** — Track streaming, compaction, GC status
6. **Separate CommitLog disk** — I/O separation recommended
7. **Set appropriate memory pool size** — Heap 6-8GB, ensure headroom for metadata

---

## References

- [Architecture Overview](https://cassandra.apache.org/doc/latest/cassandra/architecture/overview.html)
- [Dynamo Design](https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html)
- [Storage Engine](https://cassandra.apache.org/doc/latest/cassandra/architecture/storage-engine.html)
- [Guarantees (Consistency)](https://cassandra.apache.org/doc/latest/cassandra/architecture/guarantees.html)
- [Improved Internode Messaging](https://cassandra.apache.org/doc/latest/cassandra/architecture/messaging.html)
- [Improved Streaming](https://cassandra.apache.org/doc/latest/cassandra/architecture/streaming.html)
