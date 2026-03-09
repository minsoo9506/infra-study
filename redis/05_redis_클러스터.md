# Redis 클러스터

## 확장성과 분산

단일 Redis 노드는 RAM 크기와 단일 CPU 코어의 한계에 묶인다.
트래픽과 데이터가 증가할 때 두 가지 확장 방법이 있다.

| 구분 | 수직 확장 (Scale Up) | 수평 확장 (Scale Out) |
|------|---------------------|----------------------|
| 방법 | 더 큰 서버로 교체 | 노드 추가 |
| 한계 | 물리적 상한 존재 | 사실상 무제한 |
| 비용 | 고가 | 비교적 저렴 |
| 단일 장애점 | 있음 | 없음 |
| Redis 지원 | 단순 (재시작만) | 클러스터 구성 필요 |

Redis Cluster는 **수평 확장**을 공식 지원하는 분산 모드다.
- 데이터를 여러 노드에 자동으로 **샤딩(Sharding)**
- 각 노드에 **Replica를 붙여 고가용성** 확보
- 외부 Sentinel 없이 클러스터 자체적으로 장애조치

```
[Node 1: Primary]  [Node 2: Primary]  [Node 3: Primary]
 슬롯 0~5460         슬롯 5461~10922     슬롯 10923~16383
      │                    │                    │
[Node 4: Replica]  [Node 5: Replica]  [Node 6: Replica]
```

---

## Redis Cluster 특징

### 기본 구성 요건

- 최소 **3개의 Primary 노드** 필요 (quorum 확보)
- 고가용성을 위해 각 Primary마다 **Replica 1개 이상** 권장 → 최소 6노드
- 노드 간 통신: 클라이언트 포트 + **10000** (gossip 프로토콜, 예: 6379 → 16379)

### Gossip 프로토콜

Sentinel 없이 노드들이 서로 상태를 주고받는 **P2P 방식**의 클러스터 관리 프로토콜.

```
Node A ──▶ Node B: "Node C 살아있어?"
Node B ──▶ Node A: "나는 살아있어, Node C는 의심스러워"
→ 과반수가 동의하면 PFAIL → FAIL 선언 후 장애조치
```

- 주기적으로 PING/PONG 메시지 교환
- 클러스터 토폴로지(슬롯 배치, 노드 상태) 전파

### 설정 (redis.conf)

```conf
# 클러스터 모드 활성화
cluster-enabled yes

# 클러스터 상태 파일 (자동 생성/관리)
cluster-config-file nodes.conf

# 노드 응답 없으면 n밀리초 후 장애로 판단
cluster-node-timeout 5000

# Replica가 없는 Primary 장애 시 클러스터 중단 여부
# yes: 데이터 무결성 우선 / no: 가용성 우선
cluster-require-full-coverage yes
```

### 클러스터 생성

```bash
# redis-cli로 클러스터 구성 (--cluster-replicas 1 = Primary당 Replica 1개)
redis-cli --cluster create \
  192.168.1.1:6379 192.168.1.2:6379 192.168.1.3:6379 \
  192.168.1.4:6379 192.168.1.5:6379 192.168.1.6:6379 \
  --cluster-replicas 1
```

### 주요 클러스터 명령어

| 명령어 | 설명 |
|--------|------|
| `CLUSTER INFO` | 클러스터 전체 상태 조회 |
| `CLUSTER NODES` | 노드 목록 및 슬롯 배치 조회 |
| `CLUSTER MEET <ip> <port>` | 새 노드를 클러스터에 참가시킴 |
| `CLUSTER FORGET <node-id>` | 노드 제거 |
| `CLUSTER FAILOVER` | 수동 장애조치 (Replica에서 실행) |
| `CLUSTER RESET` | 노드 초기화 |

---

## 데이터 분산과 Key 관리

### 해시 슬롯 (Hash Slot)

Redis Cluster는 전체 키 공간을 **16384개의 슬롯**으로 나누고, 각 Primary 노드가 슬롯의 일부를 담당한다.

```
슬롯 번호 = CRC16(key) % 16384
```

3노드 기본 배분 예시:
```
Node 1: 슬롯 0     ~ 5460
Node 2: 슬롯 5461  ~ 10922
Node 3: 슬롯 10923 ~ 16383
```

키가 어느 노드에 저장될지는 슬롯 번호가 결정한다.

```
SET user:1001 "Alice"
→ CRC16("user:1001") % 16384 = 4231
→ 슬롯 4231은 Node 1 담당
→ Node 1에 저장
```

### MOVED 리다이렉트

클라이언트가 잘못된 노드에 요청을 보내면, 해당 노드가 올바른 노드 주소를 반환한다.

```
클라이언트 → Node 1: GET user:2000
Node 1 → 클라이언트: MOVED 7823 192.168.1.2:6379
클라이언트 → Node 2: GET user:2000 (재요청)
```

- Smart Client(redis-py, Jedis 등)는 슬롯 맵을 캐싱해 자동으로 올바른 노드에 바로 접근

### ASK 리다이렉트

슬롯 **마이그레이션 중**에만 발생하는 임시 리다이렉트. 슬롯 소유권은 아직 이전되지 않은 상태.

```
MOVED: 슬롯 소유권이 완전히 이전됨 → 슬롯 맵 갱신
ASK:   이전 진행 중 → 슬롯 맵 갱신하지 않고 해당 요청만 리다이렉트
```

### Hash Tag로 같은 노드에 배치

중괄호 `{}` 안의 문자열만 해싱에 사용된다. 같은 태그를 가진 키는 **같은 슬롯에 저장**된다.

```
user:{1001}:name   ──▶ CRC16("1001") % 16384 = 슬롯 X
user:{1001}:email  ──▶ CRC16("1001") % 16384 = 슬롯 X (동일)
order:{1001}:list  ──▶ CRC16("1001") % 16384 = 슬롯 X (동일)
```

→ 같은 노드에 배치되므로 **MGET, MSET, 트랜잭션**을 같은 노드 내에서 실행 가능.

### 슬롯 리밸런싱 (노드 추가/제거)

```bash
# 슬롯 마이그레이션 (Node1의 슬롯 100개를 Node4로 이동)
redis-cli --cluster reshard 192.168.1.1:6379

# 클러스터 균형 자동 조정
redis-cli --cluster rebalance 192.168.1.1:6379
```

- 마이그레이션 중에도 서비스 중단 없이 실시간으로 데이터 이동
- ASK 리다이렉트로 이전 중인 슬롯 요청 처리

---

## 성능과 가용성

### 성능

**읽기 처리량 확장:**
```
단일 노드: ~100,000 ops/sec
3 Primary 클러스터: ~300,000 ops/sec (쓰기)
Replica 읽기까지 활용: 더 높은 읽기 처리량
```

Replica에서 읽기를 허용하려면 클라이언트에서 `READONLY` 명령 실행:
```
READONLY        # 해당 연결에서 Replica 읽기 허용
GET user:1001   # Replica에서 처리
```

> 기본적으로 Replica는 읽기 요청도 Primary로 리다이렉트. `READONLY`를 명시해야 Replica에서 직접 읽기 가능.

### 가용성 — 장애조치

Primary 노드 장애 시 해당 Primary의 Replica가 자동으로 승격된다.

```
장애조치 절차:
1. 노드 A(Primary) 응답 없음
   → 다른 노드들이 PFAIL 표시

2. cluster-node-timeout 초과 + 과반수 노드 동의
   → FAIL 선언

3. 노드 A의 Replica가 선거(Election) 시작
   → 다른 Primary들의 투표를 받아 새 Primary로 승격

4. 노드 A 슬롯들을 새 Primary가 인수
5. 노드 A 복구 시 Replica로 재편입
```

장애조치 시간: 일반적으로 `cluster-node-timeout` 기준 수 초 이내.

### cluster-require-full-coverage

```conf
# yes (기본): 커버되지 않는 슬롯이 있으면 클러스터 전체 쓰기 중단
# no: 일부 슬롯 손실되어도 나머지 슬롯은 계속 서비스
cluster-require-full-coverage yes
```

| 설정 | 상황 | 동작 |
|------|------|------|
| `yes` | Replica 없는 Primary 장애 | 클러스터 전체 중단 |
| `no` | Replica 없는 Primary 장애 | 해당 슬롯만 불가, 나머지 정상 |

---

## 클러스터의 제약 사항

### 1. 멀티 키 연산 제한

서로 다른 슬롯에 걸친 키는 단일 명령으로 처리 불가.

```
# 에러 발생 (key1과 key2가 다른 슬롯에 있을 경우)
MGET key1 key2
→ CROSSSLOT Keys in request don't hash to the same slot

# 해결책: Hash Tag로 같은 슬롯에 배치
MGET {tag}:key1 {tag}:key2  # OK
```

### 2. 트랜잭션 (MULTI/EXEC) 제한

```
# 다른 슬롯에 있는 키를 하나의 트랜잭션으로 처리 불가
MULTI
SET user:1 "Alice"   # 슬롯 A
SET order:1 "item"   # 슬롯 B  → CROSSSLOT 에러
EXEC

# 해결책: Hash Tag 사용
MULTI
SET {1}:user "Alice"
SET {1}:order "item"
EXEC  # 같은 슬롯이므로 OK
```

### 3. Lua 스크립트 제한

Lua 스크립트 내에서 접근하는 모든 키는 같은 슬롯에 있어야 한다.

```lua
-- 에러: 키가 다른 슬롯에 있을 경우
redis.call('GET', KEYS[1])  -- 슬롯 A
redis.call('GET', KEYS[2])  -- 슬롯 B  → 에러

-- 해결책: Hash Tag로 같은 슬롯에 배치 후 실행
```

### 4. DB 인덱스 제한

클러스터 모드에서는 **DB 0만 사용 가능**. `SELECT` 명령으로 DB 전환 불가.

```
SELECT 1  → ERR SELECT is not allowed in cluster mode
```

### 5. Pub/Sub 제한

클러스터에서 Pub/Sub 메시지는 **연결된 노드에만 전달**된다. 전체 노드 브로드캐스트가 필요하면 모든 노드에 각각 구독해야 한다.

> Redis 7.0+ : `SSUBSCRIBE` (Sharded Pub/Sub)로 슬롯 기반 Pub/Sub 지원.

### 6. 복잡한 운영

| 작업 | 단일 인스턴스 | 클러스터 |
|------|-------------|----------|
| 백업 | `BGSAVE` 1회 | 노드별 각각 실행 |
| 모니터링 | 단일 엔드포인트 | 노드 수만큼 수집 |
| 키 스캔 | `SCAN` 1회 | 노드별 `SCAN` 후 합산 |
| 클라이언트 | 단순 연결 | Smart Client 필요 |

### 제약 사항 정리

| 제약 | 원인 | 해결책 |
|------|------|--------|
| 멀티 키 연산 불가 | 키가 다른 노드에 분산 | Hash Tag |
| 트랜잭션 단일 슬롯 | 원자성 보장 범위 한계 | Hash Tag |
| DB 0만 사용 | 멀티 DB = 복잡한 슬롯 관리 | 키 네이밍으로 구분 |
| Pub/Sub 범위 제한 | 노드 간 메시지 미전파 | Sharded Pub/Sub (7.0+) |
| Lua 스크립트 제한 | 크로스 슬롯 접근 불가 | Hash Tag |
