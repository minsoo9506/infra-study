# Redis 백업과 장애 복구

## RDB(Redis Database)를 사용한 백업

RDB는 특정 시점의 메모리 스냅샷을 `.rdb` 파일로 디스크에 저장하는 방식이다.

### 동작 방식

Redis가 `BGSAVE` 명령을 실행하면 **fork()** 로 자식 프로세스를 생성하고, 자식 프로세스가 현재 메모리 상태를 `.rdb` 파일로 직렬화해 저장한다. 부모 프로세스는 계속 클라이언트 요청을 처리한다.

```
[Redis 프로세스]
      │
      ├─ fork() ──▶ [자식 프로세스] ──▶ dump.rdb 저장
      │                                   (백그라운드)
      └─ 계속 클라이언트 요청 처리
```

**Copy-on-Write(CoW):** fork 이후 메모리를 즉시 복사하지 않고, 수정이 발생한 페이지만 복사. 메모리 오버헤드를 최소화.

### 설정 (redis.conf)

```conf
# save <초> <변경횟수> — 조건을 만족하면 자동 BGSAVE
save 900 1      # 900초 동안 1개 이상 변경 시 저장
save 300 10     # 300초 동안 10개 이상 변경 시 저장
save 60 10000   # 60초 동안 10000개 이상 변경 시 저장

# RDB 파일 이름 및 경로
dbfilename dump.rdb
dir /var/lib/redis

# 저장 실패 시 쓰기 요청 차단 여부
stop-writes-on-bgsave-error yes

# RDB 파일 압축 여부 (LZF 압축, 기본값 yes)
rdbcompression yes
```

### 주요 명령어

| 명령어 | 설명 |
|--------|------|
| `SAVE` | 동기 저장 (블로킹, 운영 환경에서 사용 금지) |
| `BGSAVE` | 비동기 저장 (백그라운드 fork) |
| `LASTSAVE` | 마지막 저장 시각 (Unix timestamp) |
| `DEBUG RELOAD` | RDB 저장 후 즉시 재로드 (테스트용) |

### 장단점

| 구분 | 내용 |
|------|------|
| 장점 | 재시작 속도 빠름, 파일 크기 작음, 특정 시점 복원 용이 |
| 단점 | 마지막 스냅샷 이후 데이터 유실 가능, fork 시 메모리 2배 일시 사용 |

### 복구

```bash
# Redis 종료 후 dump.rdb 파일을 dir 경로에 복사하고 재시작하면 자동 로드
cp dump.rdb /var/lib/redis/dump.rdb
systemctl start redis
```

---

## AOF(Append Only File)를 사용한 백업

AOF는 Redis에 실행된 **모든 쓰기 명령을 순서대로 파일에 기록**하는 방식이다. 서버 재시작 시 AOF 파일을 재실행해 데이터를 복원한다.

### 동작 방식

```
클라이언트 명령 (SET, LPUSH ...)
      │
      ▼
Redis 메모리 적용
      │
      ▼
AOF 버퍼에 명령 기록
      │
      ▼
fsync 정책에 따라 디스크에 flush → appendonly.aof
```

### fsync 정책

| 옵션 | 동작 | 안정성 | 성능 |
|------|------|--------|------|
| `always` | 명령마다 즉시 fsync | 최대 (유실 0) | 느림 |
| `everysec` | 1초마다 fsync (기본값) | 최대 1초 유실 | 균형 |
| `no` | OS에 맡김 | 수십 초 유실 가능 | 빠름 |

### 설정 (redis.conf)

```conf
# AOF 활성화
appendonly yes

# AOF 파일 이름
appendfilename "appendonly.aof"

# fsync 정책
appendfsync everysec

# AOF Rewrite 트리거 조건
auto-aof-rewrite-percentage 100  # 파일 크기가 2배가 되면 rewrite
auto-aof-rewrite-min-size 64mb   # 최소 64MB 이상일 때만 트리거
```

### AOF Rewrite

시간이 지날수록 AOF 파일은 커진다. **BGREWRITEAOF** 명령으로 현재 메모리 상태를 기준으로 최소한의 명령만 남겨 파일을 압축한다.

```
기존 AOF:
SET counter 1
INCR counter   (x 999번)
→ counter = 1000

Rewrite 후:
SET counter 1000
```

```bash
# 수동 AOF rewrite 실행
redis-cli BGREWRITEAOF
```

### RDB + AOF 혼합 모드 (Redis 4.0+)

```conf
# 혼합 모드 활성화
aof-use-rdb-preamble yes
```

Rewrite 시 RDB 스냅샷 + 이후 명령을 AOF에 혼합 저장.
→ 재시작 속도(RDB의 장점) + 데이터 유실 최소화(AOF의 장점)를 동시에 달성.

### 장단점

| 구분 | 내용 |
|------|------|
| 장점 | 데이터 유실 최소화 (최대 1초), 명령 단위 복원 가능 |
| 단점 | RDB보다 파일 크기 크고 재시작 느림, I/O 부하 증가 |

### RDB vs AOF 비교

| 항목 | RDB | AOF |
|------|-----|-----|
| 저장 방식 | 스냅샷 | 명령 로그 |
| 데이터 유실 | 마지막 스냅샷 이후 전체 | 최대 1초 (everysec 기준) |
| 재시작 속도 | 빠름 | 느림 |
| 파일 크기 | 작음 | 큼 |
| 복구 정밀도 | 낮음 | 높음 |
| 권장 사용 | 복구 속도 중시 | 데이터 안정성 중시 |

---

## Redis의 복제

Redis 복제는 **Primary(마스터)** 의 데이터를 하나 이상의 **Replica(슬레이브)** 에 비동기로 복사하는 기능이다.

```
[Primary]
   │  쓰기(Write)
   │  비동기 복제
   ├──────▶ [Replica 1]  읽기(Read)
   └──────▶ [Replica 2]  읽기(Read)
```

### 복제 설정

**Replica 측 redis.conf:**
```conf
# Redis 5.0+
replicaof <primary-ip> <primary-port>

# 또는 실행 중 명령으로
REPLICAOF 192.168.1.10 6379
```

**Primary 측 설정:**
```conf
# 복제 백로그 크기 (네트워크 끊김 후 부분 재동기화에 사용)
repl-backlog-size 1mb

# Replica가 응답 없으면 n초 후 연결 해제
repl-timeout 60

# 복제 지연 허용 범위 (0 = 비동기, 동기화 보장 없음)
min-replicas-to-write 1
min-replicas-max-lag 10
```

### 초기 동기화 과정 (Full Sync)

```
Replica → Primary: PSYNC ? -1 (최초 연결)
Primary: BGSAVE 실행 → RDB 생성
Primary → Replica: RDB 파일 전송
Primary → Replica: RDB 전송 중 누적된 버퍼(write 명령) 전송
Replica: RDB 로드 후 이후 명령 지속 적용
```

### 부분 재동기화 (Partial Sync)

네트워크 단절 후 재연결 시, 백로그 버퍼에 남아있는 명령만 전송해 전체 동기화를 피한다.

```
Replica → Primary: PSYNC <replication_id> <offset>
Primary: 오프셋 이후 명령만 전송 (backlog 범위 내)
```

### 주요 명령어

| 명령어 | 설명 |
|--------|------|
| `INFO replication` | 복제 상태 조회 (역할, 오프셋, 지연 등) |
| `REPLICAOF NO ONE` | Replica를 독립 Primary로 전환 |
| `WAIT numreplicas timeout` | n개 Replica 동기화 완료까지 대기 |

### INFO replication 예시

```
role:master
connected_slaves:2
slave0:ip=192.168.1.11,port=6379,state=online,offset=12345,lag=0
slave1:ip=192.168.1.12,port=6379,state=online,offset=12340,lag=1
master_repl_offset:12345
repl_backlog_size:1048576
```

### 복제 특성

- **비동기 복제:** Primary는 Replica 동기화를 기다리지 않음 → 소량의 데이터 유실 가능
- **읽기 부하 분산:** Replica에서 읽기 요청 처리 가능
- **Replica의 Replica:** 체인(cascade) 복제 구성 가능
- **읽기 전용:** 기본적으로 Replica는 쓰기 불가 (`replica-read-only yes`)

---

## Redis Sentinel을 이용한 자동 장애조치

Redis Sentinel은 Redis의 **고가용성(HA)** 솔루션이다. Primary 장애를 자동 감지하고 Replica를 새 Primary로 승격시킨다.

### 구성

```
[Sentinel 1]  [Sentinel 2]  [Sentinel 3]
      │              │              │
      └──────────────┴──────────────┘
                     │ 모니터링
            ┌────────┴────────┐
       [Primary]         [Replica]
```

- **Sentinel 최소 3개** 권장 (과반수 quorum으로 장애 판단, 홀수 구성)
- Sentinel 자체는 별도 프로세스로 실행 (`redis-sentinel` 또는 `redis-server --sentinel`)

### 설정 (sentinel.conf)

```conf
# 모니터링할 Primary 등록
# sentinel monitor <이름> <ip> <port> <quorum>
sentinel monitor mymaster 192.168.1.10 6379 2

# Primary가 n밀리초 응답 없으면 주관적 다운(SDOWN)으로 판단
sentinel down-after-milliseconds mymaster 5000

# 장애조치 제한 시간
sentinel failover-timeout mymaster 60000

# 장애조치 후 동시에 동기화할 Replica 수
sentinel parallel-syncs mymaster 1

# 인증 (Primary에 requirepass 설정 시)
sentinel auth-pass mymaster <password>
```

### 장애조치 과정

```
1. Primary 응답 없음 감지
   → Sentinel 1: SDOWN (주관적 다운) 선언

2. quorum 이상 Sentinel이 동의
   → ODOWN (객관적 다운) 선언

3. Sentinel들 중 리더(Leader) 선출 (Raft 알고리즘)

4. 리더 Sentinel이 장애조치 실행:
   - 가장 최신 오프셋의 Replica를 새 Primary로 선택
   - REPLICAOF NO ONE 실행 → 새 Primary 승격
   - 나머지 Replica에 새 Primary 주소 통보
   - 구 Primary 복구 시 Replica로 재편입

5. 클라이언트에 새 Primary 주소 통보
```

### SDOWN vs ODOWN

| 구분 | 의미 | 조건 |
|------|------|------|
| SDOWN (Subjectively Down) | 단일 Sentinel이 판단한 다운 | down-after-milliseconds 초과 |
| ODOWN (Objectively Down) | 과반수(quorum) 이상이 동의한 다운 | quorum 이상의 Sentinel이 SDOWN 동의 |

장애조치는 ODOWN 상태일 때만 실행된다.

### 클라이언트 연동

클라이언트는 Sentinel을 통해 현재 Primary 주소를 조회해야 한다.

```python
# Python (redis-py 예시)
from redis.sentinel import Sentinel

sentinel = Sentinel(
    [('sentinel1', 26379), ('sentinel2', 26379), ('sentinel3', 26379)],
    socket_timeout=0.5
)

# Sentinel이 현재 Primary 주소를 반환
master = sentinel.master_for('mymaster', password='secret')
slave = sentinel.slave_for('mymaster', password='secret')

master.set('key', 'value')
slave.get('key')
```

### 주요 Sentinel 명령어

| 명령어 | 설명 |
|--------|------|
| `SENTINEL masters` | 모니터링 중인 Primary 목록 |
| `SENTINEL slaves <name>` | 해당 Primary의 Replica 목록 |
| `SENTINEL sentinels <name>` | 다른 Sentinel 목록 |
| `SENTINEL get-master-addr-by-name <name>` | 현재 Primary IP/Port 조회 |
| `SENTINEL failover <name>` | 수동 장애조치 실행 |
| `SENTINEL reset <name>` | Sentinel 상태 초기화 |

### Sentinel 한계

- **클라이언트가 Sentinel을 지원해야 함:** Sentinel 인지 클라이언트 라이브러리 필요
- **쓰기는 단일 Primary:** 수평 쓰기 확장 불가 → 클러스터 구성 필요
- **장애조치 시 일시적 쓰기 불가:** Failover 수행 중(수 초) 쓰기 요청 실패 가능
- **데이터 유실 가능성:** 비동기 복제이므로 장애조치 시 복제 지연만큼 유실 가능

### Sentinel vs Cluster 선택 기준

| 항목 | Sentinel | Cluster |
|------|----------|---------|
| 목적 | 고가용성 (HA) | HA + 수평 확장 |
| 데이터 분산 | 전체 복제 | 샤딩 (16384 슬롯) |
| 구성 복잡도 | 낮음 | 높음 |
| 적합한 규모 | 중소규모 | 대규모 |
| 쓰기 확장 | 불가 | 가능 |
