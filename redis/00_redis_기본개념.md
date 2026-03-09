# Redis 기본 개념

## Redis 의 정의

Redis(Remote Dictionary Server)는 오픈소스 인메모리 데이터 저장소다.
단순한 캐시를 넘어 데이터베이스, 캐시, 메시지 브로커, 스트리밍 엔진으로 활용된다.

주요 특징:
- **싱글 스레드** 기반으로 동작 → 원자적(atomic) 연산 보장
- **비동기 I/O** + 이벤트 루프 구조로 높은 처리량 달성
- 기본적으로 **TCP 6379 포트** 사용
- 다양한 클라이언트 라이브러리 지원 (Java, Python, Node.js 등)

---

## In-Memory DB 로서의 Redis

Redis는 모든 데이터를 **RAM에 저장**하기 때문에 디스크 기반 DB보다 수십~수백 배 빠른 읽기/쓰기 속도를 제공한다.

| 구분 | Redis (in-memory) | MySQL (disk 기반) |
|------|-------------------|-------------------|
| 읽기 속도 | ~100,000 ops/sec | ~수천 ops/sec |
| 쓰기 속도 | ~100,000 ops/sec | ~수천 ops/sec |
| 데이터 영속성 | 선택적 (RDB/AOF) | 기본 보장 |
| 용량 한계 | RAM 크기에 종속 | 디스크 크기에 종속 |

**영속성 옵션:**
- **RDB (Redis Database Backup):** 특정 시점의 스냅샷을 디스크에 저장. 복구 빠르지만 마지막 스냅샷 이후 데이터 유실 가능.
- **AOF (Append Only File):** 모든 쓰기 명령을 로그로 기록. 데이터 유실 최소화, 하지만 파일 크기가 커짐.
- **RDB + AOF 혼합:** Redis 4.0부터 지원. 두 방식의 장점을 결합.

---

## Key-Value 구조의 장단점

Redis는 데이터를 **Key-Value 쌍**으로 저장한다.

```
SET user:1001:name "Alice"
GET user:1001:name  →  "Alice"
```

**장점:**
- **O(1) 조회 속도:** 키를 알면 해시 테이블로 즉시 접근
- **단순한 모델:** 구현 및 운영이 직관적
- **수평 확장 용이:** 키 기반으로 데이터를 샤딩하기 쉬움
- **다양한 Value 타입 지원:** 단순 문자열 외에도 복잡한 자료구조 저장 가능

**단점:**
- **복잡한 쿼리 불가:** JOIN, 복잡한 WHERE 조건 등 관계형 쿼리를 지원하지 않음
- **키 설계가 중요:** 키 네이밍 전략이 잘못되면 충돌·관리 어려움 발생
- **부분 검색의 한계:** 키의 전체 이름을 알아야 조회 가능 (패턴 검색은 `KEYS`, `SCAN`으로 가능하나 성능 이슈 존재)

**키 네이밍 컨벤션 예시:**
```
{서비스}:{엔티티}:{ID}:{필드}
user:session:1001
product:cache:item:5002
```

---

## NoSQL 로서의 Redis

Redis는 전통적인 RDBMS와 달리 **스키마가 없는 NoSQL** 저장소다.

| 구분 | RDBMS | Redis (NoSQL) |
|------|-------|---------------|
| 스키마 | 고정 스키마 필수 | 스키마 없음 |
| 관계 표현 | JOIN, 외래키 | 직접 키 참조로 표현 |
| 트랜잭션 | ACID 완전 지원 | 제한적 지원 (MULTI/EXEC) |
| 확장 방식 | 수직 확장 주 | 수평 확장 (클러스터) |
| 적합한 데이터 | 정형 데이터 | 캐시, 세션, 실시간 데이터 |

**Redis의 NoSQL 특성:**
- **BASE 모델** 지향 (Basically Available, Soft state, Eventually consistent)
- 클러스터 구성 시 분산 환경에서 고가용성 제공
- Pub/Sub, Stream 등으로 메시지 브로커 역할도 수행

---

## Redis Data Type

Redis는 단순 문자열 외에 다양한 자료구조를 Value로 지원한다.

### String
가장 기본적인 타입. 텍스트, 숫자, 직렬화된 JSON 등 저장 가능.
```
SET counter 100
INCR counter        # 101 (원자적 증가)
APPEND key " world" # 문자열 추가
```
- 최대 크기: 512MB
- 활용: 캐시, 카운터, 분산 락

**주요 명령어:**

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `SET key value` | 값 저장 | `SET name "Alice"` |
| `GET key` | 값 조회 | `GET name` |
| `DEL key` | 키 삭제 | `DEL name` |
| `EXISTS key` | 키 존재 여부 | `EXISTS name` |
| `EXPIRE key seconds` | TTL 설정 | `EXPIRE name 3600` |
| `TTL key` | 남은 TTL 조회 | `TTL name` |
| `INCR key` | 정수 값 1 증가 (원자적) | `INCR counter` |
| `INCRBY key n` | 정수 값 n 증가 | `INCRBY counter 5` |
| `DECR key` | 정수 값 1 감소 | `DECR counter` |
| `APPEND key value` | 문자열 뒤에 추가 | `APPEND greeting " world"` |
| `STRLEN key` | 문자열 길이 | `STRLEN name` |
| `MSET k1 v1 k2 v2` | 여러 키 한번에 저장 | `MSET a 1 b 2` |
| `MGET k1 k2` | 여러 키 한번에 조회 | `MGET a b` |
| `SETNX key value` | 키가 없을 때만 저장 (분산 락) | `SETNX lock "1"` |
| `SETEX key seconds value` | TTL 포함 저장 | `SETEX token 3600 "abc"` |

---

### List
순서가 있는 문자열의 연결 리스트. 양 끝에서 O(1) 삽입/삭제.
```
LPUSH queue "job1"   # 왼쪽 삽입
RPUSH queue "job2"   # 오른쪽 삽입
LPOP queue           # 왼쪽에서 꺼내기
LRANGE queue 0 -1    # 전체 조회
```
- 활용: 작업 큐, 최근 활동 피드, 로그 버퍼

**주요 명령어:**

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `LPUSH key v1 v2` | 왼쪽(head)에 삽입 | `LPUSH queue "job1"` |
| `RPUSH key v1 v2` | 오른쪽(tail)에 삽입 | `RPUSH queue "job2"` |
| `LPOP key` | 왼쪽에서 꺼내기 | `LPOP queue` |
| `RPOP key` | 오른쪽에서 꺼내기 | `RPOP queue` |
| `LRANGE key start stop` | 범위 조회 (0 -1 = 전체) | `LRANGE queue 0 -1` |
| `LLEN key` | 리스트 길이 | `LLEN queue` |
| `LINDEX key index` | 인덱스로 단일 조회 | `LINDEX queue 0` |
| `LSET key index value` | 인덱스 위치 값 수정 | `LSET queue 0 "newjob"` |
| `LREM key count value` | 특정 값 삭제 | `LREM queue 1 "job1"` |
| `LTRIM key start stop` | 범위 외 요소 제거 | `LTRIM queue 0 99` |
| `BLPOP key timeout` | 왼쪽 블로킹 팝 (큐 대기) | `BLPOP queue 30` |
| `BRPOP key timeout` | 오른쪽 블로킹 팝 | `BRPOP queue 30` |

---

### Hash
필드-값 쌍의 집합. 객체를 표현하기에 적합.
```
HSET user:1001 name "Alice" age 30 city "Seoul"
HGET user:1001 name     # "Alice"
HGETALL user:1001       # 전체 필드 조회
HINCRBY user:1001 age 1 # 나이 1 증가
```
- 활용: 사용자 프로필, 상품 정보 등 객체 저장

**주요 명령어:**

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `HSET key field value` | 필드 저장 (복수 가능) | `HSET user:1 name "Alice" age 30` |
| `HGET key field` | 단일 필드 조회 | `HGET user:1 name` |
| `HMGET key f1 f2` | 여러 필드 조회 | `HMGET user:1 name age` |
| `HGETALL key` | 전체 필드+값 조회 | `HGETALL user:1` |
| `HDEL key field` | 필드 삭제 | `HDEL user:1 age` |
| `HEXISTS key field` | 필드 존재 여부 | `HEXISTS user:1 name` |
| `HKEYS key` | 전체 필드명 조회 | `HKEYS user:1` |
| `HVALS key` | 전체 값 조회 | `HVALS user:1` |
| `HLEN key` | 필드 개수 | `HLEN user:1` |
| `HINCRBY key field n` | 정수 필드 n 증가 | `HINCRBY user:1 age 1` |
| `HINCRBYFLOAT key field n` | 실수 필드 n 증가 | `HINCRBYFLOAT user:1 score 1.5` |

---

### Set
중복 없는 문자열의 집합. 집합 연산(합집합, 교집합, 차집합) 지원.
```
SADD tags "redis" "db" "cache"
SISMEMBER tags "redis"   # 포함 여부 확인
SUNION tags1 tags2       # 합집합
SINTER tags1 tags2       # 교집합
```
- 활용: 팔로워/팔로잉, 태그, 좋아요 중복 방지

**주요 명령어:**

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `SADD key v1 v2` | 멤버 추가 | `SADD tags "redis" "db"` |
| `SREM key value` | 멤버 삭제 | `SREM tags "db"` |
| `SISMEMBER key value` | 멤버 존재 여부 | `SISMEMBER tags "redis"` |
| `SMISMEMBER key v1 v2` | 여러 멤버 존재 여부 | `SMISMEMBER tags "redis" "db"` |
| `SMEMBERS key` | 전체 멤버 조회 | `SMEMBERS tags` |
| `SCARD key` | 멤버 수 | `SCARD tags` |
| `SPOP key` | 임의 멤버 꺼내기 (삭제) | `SPOP tags` |
| `SRANDMEMBER key n` | 임의 멤버 n개 조회 (삭제 안함) | `SRANDMEMBER tags 2` |
| `SUNION k1 k2` | 합집합 | `SUNION tags1 tags2` |
| `SINTER k1 k2` | 교집합 | `SINTER tags1 tags2` |
| `SDIFF k1 k2` | 차집합 (k1 - k2) | `SDIFF tags1 tags2` |
| `SUNIONSTORE dst k1 k2` | 합집합 결과를 dst에 저장 | `SUNIONSTORE result t1 t2` |
| `SINTERSTORE dst k1 k2` | 교집합 결과를 dst에 저장 | `SINTERSTORE result t1 t2` |

---

### Sorted Set (ZSet)
각 멤버에 **score**가 부여된 정렬된 집합. score 기준 자동 정렬.
```
ZADD leaderboard 1500 "Alice"
ZADD leaderboard 2000 "Bob"
ZRANGE leaderboard 0 -1 WITHSCORES  # 오름차순 조회
ZREVRANK leaderboard "Alice"        # 역순 순위 조회
```
- 활용: 실시간 랭킹, 우선순위 큐, 범위 검색

**주요 명령어:**

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `ZADD key score member` | 멤버 추가 | `ZADD rank 1500 "Alice"` |
| `ZREM key member` | 멤버 삭제 | `ZREM rank "Alice"` |
| `ZSCORE key member` | 멤버의 score 조회 | `ZSCORE rank "Alice"` |
| `ZINCRBY key n member` | score n 증가 | `ZINCRBY rank 100 "Alice"` |
| `ZRANK key member` | 오름차순 순위 (0-based) | `ZRANK rank "Alice"` |
| `ZREVRANK key member` | 내림차순 순위 | `ZREVRANK rank "Alice"` |
| `ZRANGE key start stop` | 오름차순 범위 조회 | `ZRANGE rank 0 -1 WITHSCORES` |
| `ZREVRANGE key start stop` | 내림차순 범위 조회 | `ZREVRANGE rank 0 2 WITHSCORES` |
| `ZRANGEBYSCORE key min max` | score 범위로 조회 | `ZRANGEBYSCORE rank 1000 2000` |
| `ZCOUNT key min max` | score 범위 내 멤버 수 | `ZCOUNT rank 1000 2000` |
| `ZCARD key` | 전체 멤버 수 | `ZCARD rank` |
| `ZREMRANGEBYSCORE key min max` | score 범위 멤버 삭제 | `ZREMRANGEBYSCORE rank 0 999` |
| `ZUNIONSTORE dst k1 k2` | 합집합 결과 저장 | `ZUNIONSTORE result r1 r2` |
| `ZINTERSTORE dst k1 k2` | 교집합 결과 저장 | `ZINTERSTORE result r1 r2` |

---

### Bitmap
비트 단위로 데이터를 저장. 공간 효율적인 불리언 플래그 관리.
```
SETBIT login:2024-03-08 1001 1  # 유저 1001 오늘 로그인
GETBIT login:2024-03-08 1001    # 로그인 여부 확인
BITCOUNT login:2024-03-08       # 오늘 로그인한 유저 수
```
- 활용: DAU 집계, 출석 체크, 기능 플래그

**주요 명령어:**

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `SETBIT key offset value` | 특정 비트 설정 (0 또는 1) | `SETBIT login:0308 1001 1` |
| `GETBIT key offset` | 특정 비트 조회 | `GETBIT login:0308 1001` |
| `BITCOUNT key [start end]` | 1인 비트 수 카운트 | `BITCOUNT login:0308` |
| `BITPOS key bit [start end]` | 첫 번째 0 또는 1 위치 | `BITPOS login:0308 1` |
| `BITOP op dst k1 k2` | 비트 연산 (AND/OR/XOR/NOT) | `BITOP AND result d1 d2` |

---

### HyperLogLog
대용량 집합의 **카디널리티(고유 원소 수)를 근사치**로 계산. 메모리를 거의 사용하지 않음 (최대 12KB).
```
PFADD visitors "user1" "user2" "user3"
PFCOUNT visitors   # 고유 방문자 수 근사값 (오차율 ~0.81%)
```
- 활용: UV(Unique Visitor) 집계, 대규모 중복 제거

**주요 명령어:**

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `PFADD key v1 v2 ...` | 원소 추가 | `PFADD visitors "u1" "u2"` |
| `PFCOUNT k1 k2 ...` | 고유 원소 수 근사값 | `PFCOUNT visitors` |
| `PFMERGE dst k1 k2` | 여러 HLL 병합 | `PFMERGE total week1 week2` |

---

### Stream
시간 순서로 정렬된 이벤트 로그. Kafka와 유사한 메시지 스트리밍.
```
XADD events * action "click" page "home"  # 이벤트 추가
XREAD COUNT 10 STREAMS events 0           # 이벤트 읽기
XGROUP CREATE events mygroup $ MKSTREAM   # 컨슈머 그룹 생성
```
- 활용: 이벤트 로그, 실시간 스트리밍, Kafka 대체

**주요 명령어:**

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `XADD key * field value` | 메시지 추가 (`*` = 자동 ID) | `XADD events * action "click"` |
| `XLEN key` | 메시지 수 | `XLEN events` |
| `XRANGE key start end` | ID 범위로 조회 (`-` `+` = 전체) | `XRANGE events - +` |
| `XREVRANGE key end start` | 역순 범위 조회 | `XREVRANGE events + -` |
| `XREAD COUNT n STREAMS key id` | id 이후 메시지 읽기 | `XREAD COUNT 10 STREAMS events 0` |
| `XGROUP CREATE key group id` | 컨슈머 그룹 생성 | `XGROUP CREATE events grp $` |
| `XREADGROUP GROUP g consumer STREAMS key >` | 그룹으로 메시지 읽기 | `XREADGROUP GROUP grp c1 STREAMS events >` |
| `XACK key group id` | 메시지 처리 완료 확인 | `XACK events grp 1234-0` |
| `XDEL key id` | 특정 메시지 삭제 | `XDEL events 1234-0` |
| `XTRIM key MAXLEN n` | 최대 n개만 유지 | `XTRIM events MAXLEN 1000` |

---

### 자료구조 선택 가이드

| 요구사항 | 추천 타입 |
|----------|-----------|
| 단순 캐싱 | String |
| 객체/레코드 저장 | Hash |
| 순서 있는 목록, 큐 | List |
| 중복 제거, 집합 연산 | Set |
| 점수 기반 정렬/랭킹 | Sorted Set |
| 대규모 UV 집계 | HyperLogLog |
| 불리언 플래그 대량 관리 | Bitmap |
| 이벤트 스트리밍 | Stream |
