---
name: cassandra-troubleshooting
description: Apache Cassandra 클러스터 문제 해결 방법을 다룹니다. 이상 노드 찾기, 로그 분석, nodetool 활용 디버깅, 외부 도구 활용한 심층 분석 등 운영 중 발생하는 문제를 체계적으로 진단하고 해결하는 방법을 학습합니다.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: cassandra
  tags: [cassandra, troubleshooting, debugging, logs, nodetool, diagnostics]
---

# Apache Cassandra Troubleshooting

## Overview

분산 데이터베이스인 Cassandra는 다양한 이유로 문제가 발생할 수 있습니다. 체계적인 문제 해결 접근법과 도구 활용이 필수적입니다. 일반적으로 **이상 노드 식별 → 로그 분석 → 도구 기반 심층 진단**의 3단계로 진행합니다.

**핵심 원칙**: 시스템적 접근, 데이터 기반 진단, 단계적 좁히기.

---

## SOP: Step-by-Step Procedures

### 1. 이상 노드 찾기 (Finding Misbehaving Nodes)

```bash
# 클러스터 상태 확인
nodetool status
# DOWN 표시된 노드, UNBALANCED 파티션 확인

# 토큰 분포 확인
nodetool status -p

# 노드 응답 시간 확인
nodetool tpstats
# Blocked Operations 카운트 증가 시 문제 신호

# 커넥션 상태 확인
nodetool netstats
# Active connections 비정상 증가 시 진단 필요

# GC 상태 확인
nodetool gcstats
# GC 휴지 시간 500ms 초과 시 문제
```

**Step-by-Step**:
1. `nodetool status`로 DOWN/UNBALANCED 노드 식별
2. `nodetool tpstats`로 Blocked Operations 확인
3. `nodetool gcstats`로 GC 휴지 시간 측정
4. 이상 징후 노드를 우선 순위로 진단

### 2. 로그 분석 (Reading Cassandra Logs)

```bash
# 주요 로그 파일 위치
/var/log/cassandra/system.log      # 시스템 이벤트
/var/log/cassandra/debug.log       # 디버그 정보
/var/log/cassandra/gc.log          # GC 로그
/var/log/cassandra/audit.log       # 보안 감사 로그

# ERROR 레벨 필터링
grep "ERROR" /var/log/cassandra/system.log | tail -100

# 특정 패턴 검색 (예: Timeout)
grep "TimeoutException" /var/log/cassandra/system.log

# GC 문제 분석
grep "GC pause" /var/log/cassandra/gc.log

# 실시간 로그 모니터링
tail -f /var/log/cassandra/system.log
```

**주요 오류 패턴**:

| 패턴 | 원인 | 해결 방안 |
|------|------|-----------|
| `TimeoutException` | 네트워크/디스크 지연 | 노드 상태, 디스크 I/O 확인 |
| `OutOfMemoryError` | Heap 메모리 부족 | Heap 크기 조정, 메모리 누수 진단 |
| `CompactionException` | 컴팩션 실패 | 디스크 공간, 컴팩션 설정 확인 |
| `BootstrapException` | 노드 합류 실패 | 네트워크, 토큰 범위 확인 |
| `CorruptionException` | 데이터 손상 | SSTable 무결성 검사 |

**Step-by-Step**:
1. `system.log`에서 ERROR/WARN 레벨 필터링
2. 타임스탬프로 문제 발생 시간대 식별
3. 관련 패턴(Timeout, OOM 등) 검색
4. GC 로그로 메모리 문제 교차 검증

### 3. nodetool 활용 디버깅

```bash
# 스레드 풀 상태 심층 분석
nodetool tpstats
# Waiting on freeable slot: 대기 중인 작업 수
# Blocked operations: 차단된 작업 수 (0이어야 함)

# 테이블별 상세 메트릭
nodetool tablestats my_keyspace my_table
# Read latency, Write latency, Row cache hit rate 확인

# 컴팩션 상태 분석
nodetool compactionstats
# Active compactions, Estimated progress 확인

# 힌트 상태 확인
nodetool showhints
# 미처리 힌트 수 확인

# 캐시 통계
nodetool cachestats
# Key cache, Row cache hit rate 확인

# 디스크 사용량
nodetool info
# Disk usage, Space available 확인
```

**Step-by-Step**:
1. `tpstats`로 스레드 풀 상태 확인 (Blocked = 0이어야 함)
2. `tablestats`로 테이블별 레이턴시 분석
3. `compactionstats`로 컴팩션 부하 추적
4. `info`로 디스크 공간 확인

### 4. 외부 도구 활용한 심층 분석

```bash
# JVM 힙 덤프 (OutOfMemoryError 시)
jmap -dump:format=b,file=heap.hprof <pid>

# 스레드 덤프 (Hang 상태 시)
jstack <pid> > thread_dump.txt

# 네트워크 연결 확인
netstat -an | grep 9042
ss -tlnp | grep 7000

# 디스크 I/O 모니터링
iostat -x 1
iotop

# 메모리 사용량 분석
free -m
vmstat 1

# 프로세스 상태 확인
ps aux | grep cassandra
```

**JVM 힙 덤프 분석**:
```bash
# MAT (Memory Analyzer Tool) 또는 jhat 사용
jhat heap.hprof
# http://localhost:7000에서 브라우저로 접근
```

**스레드 덤프 분석**:
```bash
# 차단된 스레드 식별
grep "BLOCKED" thread_dump.txt
# 데드락 패턴 확인
grep "waiting to lock" thread_dump.txt
```

**Step-by-Step**:
1. `jstack`로 스레드 상태 덤프
2. `jmap`으로 힙 메모리 덤프 (OOM 시)
3. 외부 도구(MAT, jhat)로 분석
4. 차단 패턴/메모리 누수 식별

### 5. 일반적인 문제 시나리오 및 해결

**시나리오 1: 쓰기 레이턴시 급증**
```bash
# 진단
nodetool tpstats          # Blocked operations 확인
nodetool compactionstats  # 컴팩션 부하 확인
iostat -x 1               # 디스크 I/O 병목 확인

# 해결
nodetool setcompactionthroughput 32  # 컴팩션 스로틀 증가
```

**시나리오 2: 읽기 레이턴시 급증**
```bash
# 진단
nodetool tablestats keyspace table  # Read latency 확인
nodetool cachestats                  # Cache hit rate 확인
nodetool tpstats                     # Read thread 대기 확인

# 해결
# 캐시 효율성 개선, 인덱스 최적화 검토
```

**시나리오 3: 노드 다운**
```bash
# 진단
nodetool status              # DOWN 노드 식별
grep "ERROR" system.log      # 종료 원인 확인
dmesg | tail                 # 커널 메시지 확인 (OOM Killer 등)

# 해결
# OOM: Heap 크기 증가 또는 메모리 누수 수정
# 디스크 부족: 데이터 정리 또는 디스크 확장
```

**시나리오 4: 데이터 불일치**
```bash
# 진단
nodetool repair -pr keyspace table  # 파티션별 리페어
nodetool verify keyspace table       # 데이터 무결성 검사

# 해결
# 정기적 리페어 스케줄링
# auto_snapshot 활성화 확인
```

---

## Tool Integration

| 도구 | 사용 목적 | 예시 |
|------|-----------|------|
| `run_command` | nodetool, jstack, jmap 등 실행 | `nodetool tpstats`, `jstack <pid>` |
| `search_files` | 로그 파일에서 오류 패턴 검색 | `grep "ERROR" system.log` |
| `read_file` | 로그/설정 파일 분석 | system.log, cassandra.yaml 확인 |
| `edit_file` | 설정 변경으로 문제 해결 | compaction_throughput 조정 |

---

## Anti-Patterns & Guardrails

❌ **로그 없이 추측 진단** — 데이터 기반 접근 필수  
❌ **프로덕션에서 DEBUG 로깅 활성화** — 디스크 고갈, 성능 저하  
❌ **jmap 힙 덤프 빈번 실행** — 서비스 중단 시간 증가  
❌ **리페어 없이 데이터 불일치 방치** — 데이터 손실 누적  
❌ **Blocked Operations 무시** — 문제 악화, 서비스 중단  

⚠️ **GC 휴지 시간 500ms 초과** — 클라이언트 타임아웃 유발  
⚠️ **디스크 사용률 85% 초과** — 쓰기 실패 위험  
⚠️ **스레드 풀 고갈** — 모든 요청 대기, 서비스 불가  

---

## Best Practices

1. **모니터링 상시 유지** — 메트릭 수집 및 알림 설정
2. **로그 롤링 필수** — 디스크 고갈 방지
3. **정기적 리페어** — 데이터 불일치 예방
4. **GC 로그 모니터링** — 메모리 문제 조기 발견
5. **tpstats 상시 확인** — Blocked Operations = 0 유지
6. **디스크 공간 확보** — 15% 이상 여유 유지
7. **문제 시나리오 문서화** — 대응 절차 표준화

---

## References

- [Finding Misbehaving Nodes](https://cassandra.apache.org/doc/latest/cassandra/troubleshooting/finding_nodes.html)
- [Reading Cassandra Logs](https://cassandra.apache.org/doc/latest/cassandra/troubleshooting/reading_logs.html)
- [Using nodetool](https://cassandra.apache.org/doc/latest/cassandra/troubleshooting/use_nodetool.html)
- [Using External Tools](https://cassandra.apache.org/doc/latest/cassandra/troubleshooting/use_tools.html)
