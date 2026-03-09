# Redis 성능 튜닝

## 적절한 Eviction 정책 설정

Redis는 인메모리 저장소이므로 메모리가 가득 찼을 때 어떤 키를 삭제할지 결정하는 정책이 필요하다.

### maxmemory 설정

```conf
# 사용 가능한 최대 메모리 (0 = 제한 없음, 운영 환경에서는 반드시 설정)
maxmemory 4gb

# 메모리 초과 시 동작 정책
maxmemory-policy allkeys-lru
```

> **주의:** `maxmemory`를 설정하지 않으면 Redis가 시스템 메모리를 모두 소진해 OOM으로 프로세스가 종료될 수 있다.

### Eviction 정책 종류

| 정책 | 대상 키 | 방식 | 설명 |
|------|---------|------|------|
| `noeviction` | - | - | 메모리 초과 시 쓰기 요청 에러 반환 (기본값) |
| `allkeys-lru` | 전체 | LRU | 가장 오래 사용되지 않은 키 삭제 |
| `volatile-lru` | TTL 있는 키 | LRU | TTL 설정 키 중 LRU 삭제 |
| `allkeys-lfu` | 전체 | LFU | 가장 적게 사용된 키 삭제 |
| `volatile-lfu` | TTL 있는 키 | LFU | TTL 설정 키 중 LFU 삭제 |
| `allkeys-random` | 전체 | 랜덤 | 무작위 삭제 |
| `volatile-random` | TTL 있는 키 | 랜덤 | TTL 설정 키 중 무작위 삭제 |
| `volatile-ttl` | TTL 있는 키 | TTL | TTL이 가장 짧은 키 삭제 |

### LRU vs LFU

| 구분 | LRU (Least Recently Used) | LFU (Least Frequently Used) |
|------|--------------------------|------------------------------|
| 기준 | 마지막 접근 시간 | 접근 빈도 |
| 특징 | 최근 사용 패턴 반영 | 장기 사용 빈도 반영 |
| 단점 | 한 번 접근한 키도 보호 | 초기 접근 빈도 낮으면 삭제될 수 있음 |
| 적합한 상황 | 최근 데이터 중심 캐시 | 인기 데이터 중심 캐시 |

Redis의 LRU/LFU는 정확한 구현이 아닌 **근사 알고리즘**이다. `maxmemory-samples`로 샘플 수를 조정한다.

```conf
# 샘플 수가 많을수록 정확하지만 CPU 사용 증가 (기본값 5, 권장 10)
maxmemory-samples 10
```

### 정책 선택 가이드

| 상황 | 권장 정책 |
|------|-----------|
| 순수 캐시 (모든 키 삭제 가능) | `allkeys-lru` 또는 `allkeys-lfu` |
| 일부 키는 영구 보존 | `volatile-lru` (보존할 키에 TTL 미설정) |
| TTL 기반 만료 정책 사용 중 | `volatile-ttl` |
| 쓰기 실패를 허용하지 않음 | `noeviction` (메모리 여유 충분히 확보) |

---

## 시스템 튜닝

### redis-benchmark

Redis 내장 성능 측정 도구. 실제 운영 전 현재 서버 환경의 처리량을 측정한다.

```bash
# 기본 벤치마크 (모든 명령어, 50개 클라이언트, 100,000 요청)
redis-benchmark

# 특정 명령어만 테스트
redis-benchmark -t set,get,lpush

# 옵션 설명
redis-benchmark \
  -h 127.0.0.1 \    # 호스트
  -p 6379 \         # 포트
  -c 100 \          # 동시 클라이언트 수
  -n 1000000 \      # 총 요청 수
  -d 128 \          # 데이터 크기 (bytes)
  -P 16 \           # 파이프라인 요청 수 (묶음 처리)
  --csv             # CSV 형식 출력
```

**출력 예시:**
```
====== SET ======
  100000 requests completed in 0.88 seconds
  100 parallel clients
  3 bytes payload
  keep alive: 1

99.50% <= 1 milliseconds
100.00% <= 2 milliseconds
114416.48 requests per second
```

**파이프라이닝 효과:**
```bash
# 파이프라인 없음 (RTT마다 1개 요청)
redis-benchmark -t set -n 100000
→ ~100,000 ops/sec

# 파이프라인 16 (RTT마다 16개 요청)
redis-benchmark -t set -n 100000 -P 16
→ ~700,000 ops/sec
```

### Redis 성능에 영향을 미치는 요소들

#### 1. 네트워크 지연 (RTT)

Redis 자체보다 **네트워크 왕복 시간(RTT)** 이 병목이 되는 경우가 많다.

```
Redis 처리 시간: ~1μs
네트워크 RTT (로컬): ~100μs
네트워크 RTT (원격): ~1ms+

→ Redis가 아무리 빨라도 RTT가 크면 전체 지연이 증가
```

**해결책:**
- Redis와 애플리케이션을 **같은 서버 또는 같은 네트워크 존**에 배치
- **파이프라이닝**: 여러 명령을 한 번에 묶어 RTT 횟수 감소
- **Connection Pool**: 연결 생성/해제 오버헤드 제거

```python
# 파이프라이닝 예시 (redis-py)
pipe = redis_client.pipeline()
for i in range(1000):
    pipe.set(f"key:{i}", f"value:{i}")
pipe.execute()  # 1000개 명령을 1번의 RTT로 처리
```

#### 2. 메모리 할당과 단편화

```conf
# 메모리 할당자 (jemalloc 기본 사용, 단편화 최소화에 유리)
# redis-server --allocator jemalloc

# 메모리 단편화 비율 확인
INFO memory
→ mem_fragmentation_ratio: 1.5  # 1.0~1.5 정상, 1.5 이상 단편화 심각

# 능동적 단편화 제거 (Redis 4.0+)
activedefrag yes
active-defrag-ignore-bytes 100mb  # 단편화 100mb 이상일 때 활성화
active-defrag-threshold-lower 10  # 단편화율 10% 이상일 때 활성화
```

#### 3. 지연 유발 명령어 (O(N) 명령)

Redis는 싱글 스레드이므로 하나의 느린 명령이 전체 요청을 블로킹한다.

| 위험 명령어 | 문제 | 대안 |
|------------|------|------|
| `KEYS *` | 전체 키 순회, 블로킹 | `SCAN` (커서 기반, 논블로킹) |
| `SMEMBERS` (대형 Set) | 전체 멤버 반환 | `SSCAN` |
| `HGETALL` (대형 Hash) | 전체 필드 반환 | `HSCAN` |
| `LRANGE 0 -1` (대형 List) | 전체 리스트 반환 | 페이징 처리 |
| `FLUSHALL` / `FLUSHDB` | 전체 삭제, 블로킹 | `FLUSHALL ASYNC` |
| `DEL` (대형 컬렉션) | 블로킹 삭제 | `UNLINK` (비동기 삭제) |

```bash
# KEYS 대신 SCAN 사용
SCAN 0 MATCH user:* COUNT 100
# → cursor, [keys] 반환. cursor가 0이 되면 순회 완료
```

#### 4. 지속성(Persistence) 설정의 영향

```conf
# RDB BGSAVE: fork() 시 메모리 2배 일시 사용, 저장 중 I/O 증가
# 성능 민감한 환경에서는 저장 주기를 늘리거나 Replica에서만 RDB 실행

# AOF fsync: always는 매 명령마다 디스크 I/O → 성능 저하
appendfsync everysec  # 균형점

# AOF rewrite 중 fsync 비활성화 (rewrite 시 I/O 집중 방지)
no-appendfsync-on-rewrite yes
```

#### 5. OS 수준 튜닝

```bash
# (1) Transparent Huge Pages (THP) 비활성화
# THP 활성화 시 fork() 시 latency spike 발생
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# (2) vm.overcommit_memory 설정
# 0(기본): 커널이 메모리 여유를 확인 후 할당 → fork() 실패 가능
# 1: 항상 할당 허용 → BGSAVE/AOF rewrite 안정적
echo 1 > /proc/sys/vm/overcommit_memory

# (3) somaxconn / tcp-backlog
# 동시 연결 대기 큐 크기
echo 65535 > /proc/sys/net/core/somaxconn
```

```conf
# redis.conf에서도 설정
tcp-backlog 511
```

#### 6. 연결 수 관리

```conf
# 최대 클라이언트 연결 수 (기본값 10000)
maxclients 10000

# 유휴 연결 타임아웃 (초, 0 = 무제한)
timeout 300

# TCP keepalive (초, 0 = 비활성화)
tcp-keepalive 300
```

---

## SLOWLOG를 이용한 쿼리 튜닝

SLOWLOG는 실행 시간이 기준을 초과한 명령을 자동으로 기록하는 Redis 내장 기능이다.

### 설정

```conf
# 기록 기준 시간 (마이크로초, 1000 = 1ms 이상인 명령 기록)
slowlog-log-slower-than 1000

# 보관할 최대 SLOWLOG 항목 수 (오래된 항목은 자동 제거)
slowlog-max-len 128
```

실행 중 동적으로 변경 가능:
```bash
CONFIG SET slowlog-log-slower-than 500   # 0.5ms 이상 기록
CONFIG SET slowlog-max-len 256
```

### SLOWLOG 조회

```bash
# 최근 10개 SLOWLOG 조회
SLOWLOG GET 10

# SLOWLOG 항목 수 확인
SLOWLOG LEN

# SLOWLOG 초기화
SLOWLOG RESET
```

**출력 예시:**
```
1) 1) (integer) 14          # SLOWLOG ID
   2) (integer) 1709890123  # 실행 시각 (Unix timestamp)
   3) (integer) 15000       # 실행 시간 (마이크로초, 15ms)
   4) 1) "KEYS"             # 실행된 명령
      2) "*"
   5) "127.0.0.1:52341"     # 클라이언트 주소
   6) ""                    # 클라이언트 이름
```

### SLOWLOG 분석 및 대응

```
실행 시간 15000μs (15ms) → KEYS * 명령 사용
```

| 발견 패턴 | 원인 | 조치 |
|----------|------|------|
| `KEYS *` | 전체 키 순회 | `SCAN`으로 교체 |
| `SMEMBERS` 느림 | Set 크기 과대 | `SSCAN` + 페이징 |
| `HGETALL` 느림 | Hash 필드 수 과대 | `HMGET`으로 필요한 필드만 조회 |
| `DEL` 느림 | 대형 컬렉션 삭제 | `UNLINK`로 비동기 삭제 |
| `LRANGE 0 -1` | 전체 리스트 반환 | 크기 제한 + `LTRIM` |
| 모든 명령이 느림 | 서버 과부하, I/O 병목 | `INFO stats`, `INFO memory` 종합 점검 |

### INFO 명령으로 종합 모니터링

```bash
INFO all          # 전체 정보
INFO stats        # 명령어 통계, 키 히트율
INFO memory       # 메모리 사용량, 단편화율
INFO clients      # 연결된 클라이언트 수, 블로킹 클라이언트
INFO replication  # 복제 상태
INFO latencystats # 지연 통계 (Redis 7.0+)
```

**주요 지표:**

| 지표 | 위치 | 설명 | 정상 기준 |
|------|------|------|-----------|
| `keyspace_hits` / `keyspace_misses` | INFO stats | 캐시 히트율 | 히트율 90% 이상 |
| `mem_fragmentation_ratio` | INFO memory | 메모리 단편화 | 1.0 ~ 1.5 |
| `blocked_clients` | INFO clients | 블로킹 명령 대기 클라이언트 | 0에 가까울수록 좋음 |
| `rdb_last_bgsave_status` | INFO persistence | 마지막 BGSAVE 결과 | ok |
| `instantaneous_ops_per_sec` | INFO stats | 현재 초당 처리 명령 수 | 서버 스펙 대비 적정 수준 |
| `latency_ms` | LATENCY LATEST | 최근 지연 이벤트 | 1ms 이하 |

### LATENCY 모니터링 (Redis 2.8.13+)

```conf
# latency 모니터링 활성화
latency-monitor-threshold 100  # 100ms 이상 이벤트 기록
```

```bash
LATENCY LATEST    # 최근 지연 이벤트 목록
LATENCY HISTORY event  # 특정 이벤트 이력
LATENCY RESET     # 초기화
```

### 성능 튜닝 체크리스트

| 항목 | 확인 방법 | 조치 |
|------|-----------|------|
| `maxmemory` 설정 | `CONFIG GET maxmemory` | 적절한 값 및 정책 설정 |
| O(N) 명령 사용 여부 | `SLOWLOG GET` | SCAN 계열로 교체 |
| 메모리 단편화 | `INFO memory` → `mem_fragmentation_ratio` | `activedefrag yes` |
| THP 비활성화 | `cat /sys/kernel/mm/transparent_hugepage/enabled` | `echo never` 설정 |
| `vm.overcommit_memory` | `cat /proc/sys/vm/overcommit_memory` | `echo 1` 설정 |
| Connection Pool 사용 | 애플리케이션 코드 확인 | Pool 적용 |
| 파이프라이닝 사용 | 애플리케이션 코드 확인 | 배치 명령 파이프라인 처리 |
| AOF fsync 정책 | `CONFIG GET appendfsync` | `everysec` 권장 |
