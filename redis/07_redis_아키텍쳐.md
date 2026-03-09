# Redis 아키텍처

## Redis Streams를 이용한 Event-Driven 아키텍처

### Redis Streams란

Redis Streams는 Redis 5.0에서 추가된 자료구조로, **시간 순서로 정렬된 이벤트 로그**를 저장하고 소비하는 기능이다.
Kafka와 유사한 패턴을 Redis 안에서 구현할 수 있다.

**기존 Pub/Sub과의 차이:**

| 항목 | Pub/Sub | Streams |
|------|---------|---------|
| 메시지 저장 | 저장 안 함 (휘발성) | 영구 저장 (로그) |
| 재처리 | 불가 | 가능 (ID로 재읽기) |
| 소비자 그룹 | 없음 | 있음 (Consumer Group) |
| 오프라인 소비자 | 메시지 유실 | 재연결 후 미처리 메시지 수신 |
| 처리 확인 | 없음 | ACK 기반 처리 확인 |

### 메시지 구조

각 메시지는 `<millisecondsTime>-<sequenceNumber>` 형태의 ID를 가진다.

```
Stream Key: orders
│
├─ 1709890100000-0  action=created  order_id=1001  amount=50000
├─ 1709890101000-0  action=paid     order_id=1001
├─ 1709890102000-0  action=shipped  order_id=1001
└─ 1709890103000-0  action=created  order_id=1002  amount=30000
```

### Event-Driven 아키텍처 패턴

```
[Producer 서비스]
  주문 생성, 결제 완료 등
  이벤트 발생 시 XADD로 Stream에 기록
         │
         ▼
    [Redis Stream]
    orders, payments, notifications ...
         │
         ├──▶ [Consumer Group A: 재고 서비스]
         │       재고 차감 처리
         │
         ├──▶ [Consumer Group B: 알림 서비스]
         │       푸시/이메일 발송
         │
         └──▶ [Consumer Group C: 통계 서비스]
                 매출 집계 업데이트
```

- 각 Consumer Group은 **독립적으로 오프셋을 관리**하므로 같은 이벤트를 각자 처리
- 같은 Group 내 여러 Consumer는 **경쟁 소비(load balancing)** — 이벤트를 나눠서 처리

### 주요 명령어 흐름

```bash
# Producer: 이벤트 발행
XADD orders * action created order_id 1001 amount 50000
# → "1709890100000-0" (자동 생성된 메시지 ID 반환)

# Consumer Group 생성 ($ = 이후 새 메시지부터 / 0 = 처음부터)
XGROUP CREATE orders inventory-group $ MKSTREAM

# Consumer: 메시지 읽기 (> = 미처리 메시지)
XREADGROUP GROUP inventory-group consumer-1 COUNT 10 STREAMS orders >

# Consumer: 처리 완료 ACK
XACK orders inventory-group 1709890100000-0

# 미처리(ACK 안 된) 메시지 조회 (장애 복구 시 활용)
XPENDING orders inventory-group - + 10
```

### 실패 처리 — PEL (Pending Entry List)

Consumer가 읽었지만 ACK하지 않은 메시지는 **PEL**에 남는다.
Consumer 장애 시 다른 Consumer가 PEL의 메시지를 재처리할 수 있다.

```bash
# 60초 이상 미처리 메시지를 consumer-2에게 재할당
XCLAIM orders inventory-group consumer-2 60000 1709890100000-0

# Redis 6.2+: 자동 재할당
XAUTOCLAIM orders inventory-group consumer-2 60000 0-0
```

### 스트림 크기 관리

이벤트가 계속 쌓이므로 오래된 메시지는 주기적으로 정리한다.

```bash
# 최대 10000개만 유지 (초과분 자동 삭제)
XADD orders MAXLEN 10000 * action created order_id 1002

# 정확한 트리밍 (= 정확히 1000개 이하)
XTRIM orders MAXLEN = 1000

# 근사 트리밍 (~ 효율적, 약 1000개 수준 유지)
XTRIM orders MAXLEN ~ 1000
```

### 구체적인 설계 예시 — 주문 처리 시스템

```
[Order API]
    │ XADD order-events * type order_created order_id 1001
    ▼
[Redis Stream: order-events]
    │
    ├──▶ Consumer Group: stock-service
    │         XREADGROUP → 재고 확인/차감
    │         성공: XACK
    │         실패: 재처리 큐로 이동
    │
    ├──▶ Consumer Group: payment-service
    │         XREADGROUP → 결제 처리
    │         XADD payment-events * result success
    │
    └──▶ Consumer Group: notification-service
              XREADGROUP → 알림 발송
              XACK
```

**장점:**
- 서비스 간 **느슨한 결합(Loose Coupling)**
- 이벤트 **재처리 및 감사(Audit) 가능**
- Consumer 추가/제거가 Producer에 영향 없음
- 이미 운영 중인 Redis 인프라 재활용 가능

**단점:**
- Kafka 대비 **메시지 보존 한계** (메모리 기반, 장기 보관 비용 큼)
- **파티셔닝 없음** — 단일 Stream은 단일 노드에 위치 (클러스터에서 Hash Tag로 분산 가능)
- 복잡한 스트림 처리(집계, 윈도우 연산)는 별도 로직 필요

---

## Active-Active 아키텍처

### 개념

Active-Active(다중 마스터)는 **지리적으로 분산된 여러 데이터센터(또는 리전)에서 동시에 쓰기를 허용**하는 고가용성 아키텍처다.

**기존 Active-Passive와 비교:**

```
Active-Passive (Sentinel/단순 복제):
  [DC 서울: Primary] ──▶ [DC 부산: Replica (읽기 전용)]
  서울 장애 시 → 부산 수동/자동 승격 (Failover 시간 존재)

Active-Active:
  [DC 서울: Primary] ◀──▶ [DC 부산: Primary]
  양쪽에서 동시에 읽기/쓰기 가능
  한쪽 장애 시 → 즉시 다른 쪽으로 전환 (Failover 없음)
```

| 항목 | Active-Passive | Active-Active |
|------|---------------|---------------|
| 쓰기 가능 노드 | 1개 (Primary) | 모든 노드 |
| 읽기 지연 | Replica 리전에서 낮음 | 모든 리전에서 낮음 |
| 쓰기 지연 | Replica 리전에서 높음 | 모든 리전에서 낮음 |
| 장애 복구 | Failover 시간 필요 | 즉시 전환 |
| 충돌 처리 | 없음 | 필요 (Conflict Resolution) |
| 복잡도 | 낮음 | 높음 |

### Redis에서의 구현 방법

**1. Redis Enterprise (상용) — CRDT 기반 Active-Active**

Redis Enterprise는 **CRDT(Conflict-free Replicated Data Types)** 를 사용해 충돌 없는 멀티 마스터 복제를 공식 지원한다.

```
[Seoul Region]          [US-East Region]
Redis Enterprise   ◀──▶  Redis Enterprise
  (Primary)               (Primary)
      │                       │
  로컬 쓰기 즉시 적용       로컬 쓰기 즉시 적용
      └──── 비동기 양방향 복제 ────┘
```

CRDT 충돌 해결 규칙:
- **LWW (Last Write Wins):** String 타입 — 최신 타임스탬프 기준 승리
- **Counter:** Counter 타입 — 양쪽 증감을 합산
- **Set:** 추가는 병합, 삭제 vs 추가 충돌 시 추가 우선

**2. 오픈소스 Redis — 애플리케이션 레벨 설계**

오픈소스 Redis는 공식 Active-Active를 지원하지 않으므로 **애플리케이션 레벨에서 설계**가 필요하다.

```
패턴 1: 리전별 쓰기 분리 + 단방향 복제
─────────────────────────────────────────
user-1 → Seoul Redis (쓰기)
user-2 → US Redis (쓰기)
→ 키 네이밍으로 충돌 방지 ({seoul}:key, {us}:key)
→ 단방향 복제로 전체 데이터 동기화

패턴 2: Global Redis + Local Cache
────────────────────────────────────
각 리전에 Local Redis Cache
→ Miss 시 Global DB 조회
→ 쓰기는 Global로, 읽기는 Local Cache 우선
```

### 충돌(Conflict) 해결 전략

Active-Active에서 두 노드가 동시에 같은 키를 다르게 쓰면 충돌이 발생한다.

```
t=1: Seoul SET user:1:balance 1000
t=1: US-East SET user:1:balance 2000 (동시에 다른 값)
→ 어떤 값이 최종값인가?
```

| 전략 | 방식 | 적합한 경우 |
|------|------|------------|
| LWW (Last Write Wins) | 타임스탬프가 최신인 쪽 승리 | 단순 상태값 (프로필, 설정) |
| CRDT Counter | 양쪽 증감량 합산 | 카운터, 재고 |
| CRDT Set | 합집합 (추가 우선) | 태그, 팔로워 목록 |
| 애플리케이션 병합 | 비즈니스 로직으로 직접 처리 | 복잡한 도메인 |
| 충돌 회피 설계 | 리전별 키 소유권 분리 | 쓰기 충돌 자체를 없앰 |

### 실전 아키텍처 패턴

#### 패턴 1: 지역 캐시 + 글로벌 원본

```
[사용자, Seoul]          [사용자, US]
      │                       │
[Seoul Redis Cache]    [US Redis Cache]
 TTL 기반 캐시           TTL 기반 캐시
      │     캐시 미스          │
      └──────────┬────────────┘
                 ▼
         [Primary DB (RDS, etc.)]
         쓰기는 항상 Primary DB
```

- Redis는 캐시 레이어만 담당, 충돌 없음
- 쓰기는 DB가 처리, 캐시는 무효화(Invalidation) 전략으로 갱신

#### 패턴 2: 세션 공유 (Sentinel + 글로벌 Redis)

```
[Seoul App Server]
[US App Server]
[EU App Server]
      │
      └──▶ [Redis Sentinel Cluster (중앙)]
            세션 데이터 공유
            → 어느 리전에서 로그인해도 세션 유효
```

- 세션처럼 **정합성이 중요한 데이터**는 단일 중앙 Redis로 집중
- 지연은 다소 발생하지만 충돌 없음

#### 패턴 3: Redis Cluster + 리전별 샤딩

```
[Seoul Region]                [US Region]
Redis Cluster                 Redis Cluster
  Node1 (슬롯 0~5460)           Node4 (슬롯 0~5460)
  Node2 (슬롯 5461~10922)       Node5 (슬롯 5461~10922)
  Node3 (슬롯 10923~16383)      Node6 (슬롯 10923~16383)
       │                              │
       └──── 리전 간 읽기 전용 복제 ────┘
             (쓰기는 각 리전의 로컬 Cluster)
```

- 각 리전이 독립적으로 동작
- 재해 복구(DR) 목적으로 다른 리전에 복제본 유지

### Active-Active 도입 체크리스트

| 확인 항목 | 설명 |
|----------|------|
| 충돌 가능성 분석 | 같은 키를 여러 리전에서 동시에 쓰는 케이스가 있는가 |
| 충돌 허용 여부 | 최종 일관성(Eventual Consistency)으로 충분한가 |
| 데이터 특성 | 카운터, 세션, 캐시 등 타입에 따라 전략이 다름 |
| 네트워크 지연 | 리전 간 복제 지연이 비즈니스 로직에 영향을 주는가 |
| 운영 복잡도 | 팀이 멀티 리전 Redis를 운영할 역량이 있는가 |
| 비용 | Redis Enterprise vs 오픈소스 + 자체 설계 트레이드오프 |

### 아키텍처 선택 가이드

```
단일 리전, 가용성 필요
  → Redis Sentinel (Primary 1 + Replica N)

단일 리전, 대규모 데이터/트래픽
  → Redis Cluster

멀티 리전, 읽기 지연 최소화 (쓰기는 중앙)
  → Sentinel/Cluster + 리전별 읽기 Replica

멀티 리전, 쓰기도 로컬에서 처리 필요
  → Redis Enterprise Active-Active (CRDT)
     또는 충돌 회피 설계 (리전별 키 소유권 분리)

이벤트 기반 서비스 통합
  → Redis Streams (Kafka 없이 경량 이벤트 버스)
```
