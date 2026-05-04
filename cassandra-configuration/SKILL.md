---
name: cassandra-configuration
description: Apache Cassandra의 설정 파일 전반을 다룹니다. cassandra.yaml 핵심 파라미터, JVM 옵션(jvm-*), 로깅(logback.xml), 환경 변수(cassandra-env.sh) 등 모든 설정 파일을 이해하고 튜닝하는 방법을 학습합니다.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: cassandra
  tags: [cassandra, configuration, cassandra-yaml, jvm-options, logback]
---

# Apache Cassandra Configuration

## Overview

Cassandra는 여러 설정 파일을 통해 동작을 제어합니다. 각 파일의 역할과 파라미터를 정확히 이해해야 안정적이고 고성능의 클러스터를 운영할 수 있습니다.

**핵심 원칙**: 기본값은 신중하게 변경, 모니터링 기반 튜닝, 테스트 환경에서 검증 후 적용.

---

## SOP: Step-by-Step Procedures

### 1. cassandra.yaml — 핵심 설정 파일

```yaml
# cluster 설정
cluster_name: 'MyCassandraCluster'
num_tokens: 256  # vnode 수 (3.0+ 기본값)

# 디렉토리 설정
data_file_directories:
  - /var/lib/cassandra/data
commitlog_directory: /var/lib/cassandra/commitlog
saved_caches_directory: /var/lib/cassandra/saved_caches
hints_directory: /var/lib/cassandra/hints

# 메모리 설정 (jvm 옵션과 연동)
memtable_allocation_type: heap_buffers  # heap_buffers/direct_buffers/offheap_buffers
memtable_heap_space_in_mb: 2048
memtable_offheap_space_in_mb: 2048

# 네트워크 설정
native_transport_port: 9042
storage_port: 7000
ssl_storage_port: 7001
listen_address: localhost
rpc_address: localhost

# 레플리케이션 설정
endpoint_snitch: GossipingPropertyFileSnitch
num_cpus: 8  # 자동 감지 안 될 때 수동 설정

# 컴팩션 설정
compaction_throughput_mb_per_sec: 16
compaction_large_partition_warning_threshold_mb: 100
concurrent_compactors: 2

# 읽기/쓰기 설정
read_request_timeout_in_ms: 5000
write_request_timeout_in_ms: 2000
range_request_timeout_in_ms: 10000

# 캐시 설정
key_cache_size_in_mb:
key_cache_save_period: 14400
row_cache_size_in_mb: 0  # 비활성화 권장
row_cache_save_period: 0

# 로깅 설정
trickle_fsync: false
trickle_fsync_interval_in_kb: 10240

# 자동 스냅샷 (리페어 시)
auto_snapshot: true
```

**Step-by-Step**:
1. `cluster_name` 고유값으로 변경
2. 디렉토리 경로 확인 (별도 디스크 권장)
3. `num_tokens` 256 유지 (vnode 활성화)
4. 메모리 설정 확인 (jvm 옵션과 일치)
5. 네트워크 포트 확인 (방화벽 개방)

### 2. JVM 옵션 — jvm-server.options 등

```bash
# /etc/cassandra/jvm-server.options
# Heap 크기 (6-8GB 권장, 초과 금지)
-Xms6g
-Xmx6g

# GC 설정 (G1GC 권장)
-XX:+UseG1GC
-XX:G1ReservePercent=25
-XX:InitiatingHeapOccupancyPercent=30

# 디버깅 옵션 (프로덕션 비권장)
# -Xloggc:/var/log/cassandra/gc.log
# -XX:+PrintGCDetails
# -XX:+PrintGCDateStamps

# 직접 메모리 할당 (offheap memtable용)
-XX:MaxDirectMemorySize=<value_in_mb>
```

**Step-by-Step**:
1. Heap 크기 6-8GB로 설정 (초과 시 GC 지연 위험)
2. G1GC 활성화 (Cassandra 3.0+ 기본)
3. `MaxDirectMemorySize` offheap memtable 크기와 일치
4. GC 로그는 문제 해결 시에만 활성화

### 3. cassandra-env.sh — 환경 변수

```bash
#!/bin/bash
# JVM 옵션 파일 경로 설정
JVM_OPTS="$JVM_OPTS -Dcassandra.logdir=/var/log/cassandra"
JVM_OPTS="$JVM_OPTS -Dcassandra.storagedir=/var/lib/cassandra"

# 추가 JVM 옵션 (선택적)
# JVM_OPTS="$JVM_OPTS -XX:+AlwaysPreTouch"  # 메모리 미리 할당

# 시스템 제한 확인
ulimit -n 100000  # 파일 디스크립터 제한
```

**Step-by-Step**:
1. 로그/데이터 디렉토리 경로 설정
2. 추가 JVM 옵션이 필요할 때만 사용
3. 시스템 제한(ulimit) 확인 및 조정

### 4. logback.xml — 로깅 설정

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- 시스템 로그 -->
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

    <!-- GC 로그 -->
    <appender name="GC_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/cassandra/gc.log</file>
        ...
    </appender>

    <!-- 로깅 레벨 설정 -->
    <logger name="org.apache.cassandra" level="INFO"/>
    <logger name="com.datastax" level="WARN"/>
</configuration>
```

**Step-by-Step**:
1. 로그 디렉토리 확인 (`/var/log/cassandra`)
2. 로그 롤링 설정 (크기/시간 기반)
3. 로깅 레벨 조정 (DEBUG는 문제 해결 시만)
4. GC 로그 활성화 (필요시)

### 5. cassandra-rackdc.properties — 라크/데이터센터 설정

```properties
# 데이터센터 이름
dc=dc1

# 라크 이름
rack=rack1
```

**Step-by-Step**:
1. `endpoint_snitch`에 맞는 Snitch 유형 확인
2. PropertyFileSnitch/GossipingPropertyFileSnitch 사용 시 필요
3. 각 노드에 고유한 dc/rack 설정
4. 다중 DC 환경에서 NetworkTopologyStrategy와 함께 사용

### 6. cassandra-topologies.properties — 네트워크 토폴로지 (EC2)

```properties
# AWS EC2 토폴로지 매핑
10.0.1.1 = us-east-1a
10.0.1.2 = us-east-1b
10.0.2.1 = us-east-1c
```

### 7. commitlog-archiving.properties — 커밋로그 아카이빙

```properties
# 커밋로그 아카이빙 설정
strategy=org.apache.cassandra.archiving.PassiveArchiver
params:{target:/backup/commitlogs, keep:5}
```

---

## Tool Integration

| 도구 | 사용 목적 | 예시 |
|------|-----------|------|
| `read_file` | 설정 파일 읽기/분석 | cassandra.yaml, jvm 옵션 확인 |
| `edit_file` | 설정 파일 수정 | 파라미터 변경 적용 |
| `search_files` | 설정 패턴 검색 | `grep "compaction" *.yaml` |
| `run_command` | 설정 검증 및 적용 | `nodetool getcompactionthroughput` |

---

## Anti-Patterns & Guardrails

❌ **Heap 8GB 초과** — GC 휴지 시간 증가, 서비스 중단 위험  
❌ **row_cache 활성화** — 메모리 낭비, 비권장  
❌ **DEBUG 로깅 프로덕션** — 디스크 고갈, 성능 저하  
❌ **num_cpus 수동 설정 (자동 감지 가능 시)** — 잘못된 값으로 오동작  
❌ **단일 디렉토리 모든 데이터 저장** — I/O 경쟁, 성능 저하  

⚠️ **설정 변경 후 재시작 필요** — 일부 파라미터는 동적 변경 불가  
⚠️ **jvm 옵션과 cassandra.yaml 메모리 설정 불일치** — 메모리 과할당 위험  
⚠️ **log 롤링 미설정** — 디스크 고갈  

---

## Best Practices

1. **Heap 6-8GB로 제한** — 초과 시 GC 성능 저하
2. **G1GC 활성화** — Cassandra 3.0+ 기본, 최적화됨
3. **디렉토리 분리** — 데이터/커밋로그/캐시 별도 디스크
4. **log 롤링 필수 설정** — 디스크 고갈 방지
5. **row_cache 비활성화** — 메모리 효율적 사용
6. **변경 전 테스트 환경 검증** — 프로덕션 직접 변경 금지
7. **설정 파일 버전 관리** — Git으로 변경 이력 추적

---

## References

- [cassandra.yaml Reference](https://cassandra.apache.org/doc/latest/cassandra/managing/configuration/cass_yaml_file.html)
- [cassandra-env.sh](https://cassandra.apache.org/doc/latest/cassandra/managing/configuration/cass_env_sh_file.html)
- [jvm-* Options](https://cassandra.apache.org/doc/latest/cassandra/managing/configuration/cass_jvm_options_file.html)
- [logback.xml](https://cassandra.apache.org/doc/latest/cassandra/managing/configuration/cass_logback_xml_file.html)
- [Configuration Parameters](https://cassandra.apache.org/doc/latest/cassandra/managing/configuration/configuration.html)
