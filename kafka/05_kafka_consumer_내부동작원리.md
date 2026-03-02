- kafka v4 기준 설명

# 카프카 컨슈머 내부 동작 원리

## 목차

- [카프카 컨슈머 내부 동작 원리](#카프카-컨슈머-내부-동작-원리)
  - [목차](#목차)
- [Topic 구독 방법 - subscribe()](#topic-구독-방법---subscribe)
  - [subscribe() 방식에서 poll() 내부 흐름](#subscribe-방식에서-poll-내부-흐름)
  - [poll() 단계별 설명](#poll-단계별-설명)
    - [1단계 - Consumer Coordinator 처리 (백그라운드 작업)](#1단계---consumer-coordinator-처리-백그라운드-작업)
    - [2단계 - Fetcher: 데이터 수신](#2단계---fetcher-데이터-수신)
    - [3단계 - Interceptors → Application](#3단계---interceptors--application)
    - [단계 요약표](#단계-요약표)
- [컨슈머 오프셋 관리](#컨슈머-오프셋-관리)
  - [오프셋이란](#오프셋이란)
    - [커밋 오프셋 vs 현재 오프셋](#커밋-오프셋-vs-현재-오프셋)
  - [\_\_consumer\_offsets 토픽](#__consumer_offsets-토픽)
    - [오프셋 저장 경로](#오프셋-저장-경로)
  - [오프셋 커밋의 내부 동작](#오프셋-커밋의-내부-동작)
    - [자동 커밋 (enable.auto.commit=true)](#자동-커밋-enableautocommittrue)
    - [수동 커밋 - commitSync()](#수동-커밋---commitsync)
    - [수동 커밋 - commitAsync()](#수동-커밋---commitasync)
    - [특정 오프셋 커밋](#특정-오프셋-커밋)
    - [커밋 오프셋과 next fetch offset의 관계](#커밋-오프셋과-next-fetch-offset의-관계)
  - [auto.offset.reset - 오프셋이 없을 때](#autooffsetreset---오프셋이-없을-때)
  - [오프셋 재조정 (Seek)](#오프셋-재조정-seek)
- [그룹 코디네이터](#그룹-코디네이터)
  - [그룹 코디네이터의 역할](#그룹-코디네이터의-역할)
    - [코디네이터의 주요 기능](#코디네이터의-주요-기능)
  - [컨슈머 그룹 라이프사이클](#컨슈머-그룹-라이프사이클)
  - [JoinGroup / SyncGroup 프로토콜](#joingroup--syncgroup-프로토콜)
    - [1단계: JoinGroup (멤버 모집)](#1단계-joingroup-멤버-모집)
    - [2단계: SyncGroup (할당 동기화)](#2단계-syncgroup-할당-동기화)
    - [기존 리밸런스의 문제: Stop-the-World](#기존-리밸런스의-문제-stop-the-world)
  - [하트비트 메커니즘](#하트비트-메커니즘)
    - [하트비트 vs poll() 간격의 관계](#하트비트-vs-poll-간격의-관계)
  - [리밸런스 트리거 조건](#리밸런스-트리거-조건)
  - [Kafka 4.0 - 새로운 리밸런스 프로토콜 (KIP-848)](#kafka-40---새로운-리밸런스-프로토콜-kip-848)
- [스태틱 멤버십](#스태틱-멤버십)
  - [기존 동적 멤버십의 문제](#기존-동적-멤버십의-문제)
  - [스태틱 멤버십 동작 원리](#스태틱-멤버십-동작-원리)
    - [세션 타임아웃 초과 시](#세션-타임아웃-초과-시)
  - [스태틱 멤버십 설정](#스태틱-멤버십-설정)
    - [스태틱 멤버십 관련 설정](#스태틱-멤버십-관련-설정)
- [컨슈머 파티션 할당 전략](#컨슈머-파티션-할당-전략)
  - [RangeAssignor](#rangeassignor)
    - [RangeAssignor의 불균등 문제](#rangeassignor의-불균등-문제)
  - [RoundRobinAssignor](#roundrobinassignor)
    - [조건: 모든 컨슈머가 같은 토픽을 구독해야 함](#조건-모든-컨슈머가-같은-토픽을-구독해야-함)
  - [StickyAssignor](#stickyassignor)
    - [StickyAssignor는 Eager 리밸런스 사용](#stickyassignor는-eager-리밸런스-사용)
  - [CooperativeStickyAssignor](#cooperativestickyassignor)
  - [할당 전략 비교](#할당-전략-비교)
    - [설정 예시](#설정-예시)

---

# Topic 구독 방법 - subscribe()

토픽에서 데이터를 consume 하는 방법은 두 가지가 있다.

| 방식 | 메서드 | 파티션 할당 | 리밸런스 참여 | 주요 사용 사례 |
|------|--------|-----------|------------|------------|
| **subscribe** | `consumer.subscribe(topics)` | Group Coordinator가 자동 할당 | O | 일반적인 컨슈머 그룹 구성 |
| **assign** | `consumer.assign(partitions)` | 직접 지정 | X | 특정 파티션 리플레이, 관리 도구 |

`subscribe()`는 **컨슈머 그룹에 참여**하는 방식이다. Group Coordinator(브로커)가 파티션을 자동으로 할당하고, 그룹 멤버가 변경될 때마다 리밸런스를 통해 파티션을 재배분한다.

```python
consumer.subscribe(['orders', 'payments'])   # 토픽 목록 구독
consumer.subscribe(re.compile('^order-.*'))  # 정규식으로 구독 (새 토픽 자동 감지)
```

`subscribe()` 방식에서는 `poll()`을 호출할 때마다 Group Coordinator와 통신하며 하트비트 전송, 리밸런스 참여, 오프셋 커밋 등의 작업이 함께 수행된다. `assign()`은 이 과정이 없다.

## subscribe() 방식에서 poll() 내부 흐름

애플리케이션이 `poll()`을 호출하면 단순히 메시지를 가져오는 것이 아니라, 컨슈머 내부에서 여러 컴포넌트가 순서대로 동작한다.

```
  Application    Kafka Consumer   Consumer Coordinator         Fetcher       Key/Value        Interceptors    Kafka Broker
      │                │                  │                      │          Deserializer           │               │
      │   poll()       │                  │                      │               │                 │               │
      │───────────────▶│                  │                      │               │                 │               │
      │                │                  │                      │               │                 │               │
      │                │─────────────────▶│                      │               │                 │               │
      │                │                  │                      │               │                 │               │
      │                │         ┌────────┴────────┐             │               │                 │               │
      │                │         │ ① offset        │             │               │                 │               │
      │                │         │   callbacks     │             │               │                 │               │
      │                │         │   onComplete()  │             │               │                 │               │
      │                │         └────────┬────────┘             │               │                 │               │
      │                │                  │                      │               │                 │               │
      │                │                  │  ② send heartbeat    │               │                 │               │
      │                │                  │─────────────────────────────────────────────────────────────────────▶│
      │                │                  │◀─────────────────────────────────────────────────────────────────────│
      │                │                  │                      │               │                 │               │
      │                │                  │  ③ fetch metadata    │               │                 │               │
      │                │                  │─────────────────────────────────────────────────────────────────────▶│
      │                │                  │◀─────────────────────────────────────────────────────────────────────│
      │                │                  │                      │               │                 │               │
      │                │                  │  ④ join/re-join      │               │                 │               │
      │                │                  │    consumer group    │               │                 │               │
      │                │                  │─────────────────────────────────────────────────────────────────────▶│
      │                │                  │◀─────────────────────────────────────────────────────────────────────│
      │                │                  │                      │               │                 │               │
      │                │                  │  ⑤ auto commit       │               │                 │               │
      │                │                  │    offsets           │               │                 │               │
      │                │                  │─────────────────────────────────────────────────────────────────────▶│
      │                │                  │◀─────────────────────────────────────────────────────────────────────│
      │                │◀─────────────────│                      │               │                 │               │
      │                │                  │                      │               │                 │               │
      │                │  ⑥ fetch records │                      │               │                 │               │
      │                │─────────────────────────────────────────▶               │                 │               │
      │                │                  │                      │               │                 │               │
      │                │                  │                      │ ⑦ send fetch  │                 │               │
      │                │                  │                      │   requests    │                 │               │
      │                │                  │                      │───────────────────────────────────────────────▶│
      │                │                  │                      │               │                 │               │
      │                │                  │                      │  ⑧ await for records            │               │
      │                │                  │                      │◀──────────────────────────────────────────────│
      │                │                  │                      │  compressed batch records       │               │
      │                │                  │                      │               │                 │               │
      │                │                  │                      │ ⑨ deserialize │                 │               │
      │                │                  │                      │   key & value │                 │               │
      │                │                  │                      │──────────────▶│                 │               │
      │                │                  │                      │◀──────────────│                 │               │
      │                │                  │                      │               │                 │               │
      │                │◀─────────────────────────────────────────  records      │                 │               │
      │                │                  │                      │               │                 │               │
      │                │  ⑩ intercept     │                      │               │                 │               │
      │                │────────────────────────────────────────────────────────────────────────▶│               │
      │                │                  │                      │               │                 │               │
      │  updated       │                  │                      │               │                 │               │
      │  records       │                  │                      │               │                 │               │
      │◀───────────────────────────────────────────────────────────────────────────────────────────               │
      │                │                  │                      │               │                 │               │
```

## poll() 단계별 설명

### 1단계 - Consumer Coordinator 처리 (백그라운드 작업)

`poll()`이 호출되면 가장 먼저 **Consumer Coordinator**가 이전 주기에서 밀린 작업들을 처리한다.

```
① offset callbacks onComplete()
   └── 이전 비동기 커밋의 응답 콜백 실행
       (commitAsync()의 결과 처리)

② send heartbeat
   └── 그룹 코디네이터(브로커)에 하트비트 전송
       → 내가 살아있음을 알림
       → 리밸런스 여부를 응답으로 수신

③ fetch metadata
   └── 브로커로부터 클러스터 메타데이터 갱신
       → 파티션 리더 변경 여부 확인
       → 필요한 경우에만 수행 (메타데이터 만료 또는 에러 시)

④ join/re-join consumer group
   └── 리밸런스가 필요한 경우 그룹 재참여
       → JoinGroup + SyncGroup 프로토콜 수행
       → 필요한 경우에만 수행

⑤ auto commit offsets
   └── enable.auto.commit=true이고 auto.commit.interval.ms 경과 시
       → 직전 poll()에서 반환된 오프셋 자동 커밋
```

### 2단계 - Fetcher: 데이터 수신

```
⑥ fetch records
   └── Kafka Consumer가 Fetcher에게 데이터 요청
       → Fetcher는 할당된 파티션의 리더 브로커 주소를 알고 있음

⑦ send fetch requests
   └── Fetcher → Kafka Broker로 FetchRequest 전송
       → 파티션별 현재 오프셋(fetch.offset) 포함

⑧ await for records
   └── 브로커로부터 응답 대기
       → fetch.min.bytes 이상의 데이터가 모이거나
       → fetch.max.wait.ms 시간이 경과하면 응답
       → 응답에는 압축된 배치(compressed batch) 포함

⑨ deserialize key & value
   └── 압축 해제 후 Key/Value Deserializer로 역직렬화
       → byte[] → 지정된 타입으로 변환
       → 실패 시 SerializationException 발생
```

### 3단계 - Interceptors → Application

```
⑩ intercept
   └── ConsumerInterceptor.onConsume() 호출
       → 레코드를 애플리케이션에 전달하기 전 가로채기
       → 로깅, 모니터링, 필터링, 값 수정 등에 활용
       → 변환된 records를 Application에 반환

최종 반환:
   └── ConsumerRecords<K, V> 반환
       → 토픽, 파티션, 오프셋, 키, 값, 타임스탬프 포함
```

### 단계 요약표

| 단계 | 컴포넌트 | 동작 | 조건 |
|------|---------|------|------|
| ① | Consumer Coordinator | 이전 비동기 커밋 콜백 처리 | 대기 중인 콜백이 있을 때 |
| ② | Consumer Coordinator → Broker | 하트비트 전송 | `heartbeat.interval.ms` 주기 |
| ③ | Consumer Coordinator → Broker | 메타데이터 갱신 | 만료 또는 에러 발생 시 |
| ④ | Consumer Coordinator → Broker | 그룹 재참여(리밸런스) | 리밸런스가 필요한 경우에만 |
| ⑤ | Consumer Coordinator → Broker | 오프셋 자동 커밋 | `auto.commit` 활성화 + 주기 경과 |
| ⑥ | Kafka Consumer → Fetcher | fetch records 요청 | 매 poll() |
| ⑦ | Fetcher → Broker | FetchRequest 전송 | 매 fetch |
| ⑧ | Broker → Fetcher | 압축 배치 응답 | `fetch.min.bytes` or `fetch.max.wait.ms` |
| ⑨ | Fetcher → Deserializer | key/value 역직렬화 | 매 fetch |
| ⑩ | Kafka Consumer → Interceptors | 레코드 가로채기 | Interceptor 설정 시 |

> `poll()`은 단순한 데이터 읽기 호출이 아니다. 하트비트, 오프셋 커밋, 리밸런스 참여, 메타데이터 갱신까지 컨슈머의 모든 백그라운드 동작이 `poll()` 호출 시 함께 처리된다. 따라서 `poll()`을 오래 호출하지 않으면 그룹에서 제외(`max.poll.interval.ms`)될 수 있다.

---

# 컨슈머 오프셋 관리

## 오프셋이란

오프셋(Offset)은 **파티션 내에서 각 메시지의 고유 위치를 나타내는 순차적인 정수**이다. 컨슈머는 오프셋을 기록(커밋)함으로써 어디까지 읽었는지 추적하고, 재시작 시 그 지점부터 이어서 소비할 수 있다.

```
파티션 0 로그:
  offset 0: msg-A
  offset 1: msg-B
  offset 2: msg-C  ← 컨슈머가 여기까지 처리
  offset 3: msg-D  ← 아직 미처리
  offset 4: msg-E
         ...

커밋된 오프셋 = 2 (offset 2까지 처리 완료)
→ 재시작 시 offset 3부터 소비 시작
```

### 커밋 오프셋 vs 현재 오프셋

```
용어 구분:

Committed Offset (커밋된 오프셋):
  → 컨슈머가 브로커에 저장한 "처리 완료 위치"
  → __consumer_offsets 토픽에 저장

Current Position (현재 위치):
  → 컨슈머가 다음에 읽을 오프셋
  → 메모리에서만 관리 (브로커에 저장 안 됨)

예시:
  처리 완료: offset 0, 1, 2
  커밋 완료: offset 2
  Current Position: 3 (다음 poll 시 offset 3부터 가져옴)

  → 재시작 시 Committed Offset(2) + 1 = offset 3부터 시작
```

## __consumer_offsets 토픽

컨슈머 그룹의 오프셋은 Kafka 내부 토픽인 `__consumer_offsets`에 저장된다. 이 토픽은 기본적으로 **컴팩션 정책(compact)**이 적용되어 각 컨슈머 그룹 + 파티션 조합의 최신 오프셋만 유지된다.

```
__consumer_offsets 토픽 구조:

파티션 수: 기본 50개 (offsets.topic.num.partitions)
복제 팩터: 기본 3개 (offsets.topic.replication.factor)

저장 형식 (key-value):
  Key:   <group.id> + <topic> + <partition>
  Value: <offset> + <metadata> + <timestamp>

예시:
  Key:   "order-processor" + "orders" + 0
  Value: offset=1500, timestamp=2025-03-01T10:00:00

→ 컴팩션 정책으로 각 (group, topic, partition) 조합의 최신 오프셋만 유지
→ 컨슈머 그룹이 어느 브로커의 __consumer_offsets 파티션에 저장할지는
  hash(group.id) % 50 으로 결정 (이 파티션의 리더가 해당 그룹의 코디네이터)
```

### 오프셋 저장 경로

```
컨슈머 → 그룹 코디네이터 브로커 → __consumer_offsets 토픽
                                  (해당 파티션 리더가 코디네이터)
```

## 오프셋 커밋의 내부 동작

### 자동 커밋 (enable.auto.commit=true)

```
자동 커밋 동작 흐름:

poll() 호출 시점마다 내부적으로 체크:
  현재 시각 - 마지막 커밋 시각 > auto.commit.interval.ms(기본 5초)?
    YES → 이전 poll()에서 반환한 오프셋을 자동으로 커밋
    NO  → 커밋 없이 데이터만 반환

주의: 커밋 시점은 poll() 호출 시점이지 처리 완료 시점이 아님!

타임라인:
  poll() → 레코드 A, B, C 반환
  [처리 중... 2초]
  poll() → (5초 미경과, 커밋 안 함) → 레코드 D, E 반환
  [처리 중... 4초]
  poll() → (5초 경과!) → A, B, C, D, E 오프셋 커밋 → 레코드 F, G 반환
                                   ↑
              C, D, E 처리 완료 여부와 무관하게 커밋됨!
              → 처리 중 장애 시 C, D, E 유실 가능
```

### 수동 커밋 - commitSync()

```python
# commitSync(): 동기 커밋
# 커밋 요청 → 브로커 응답 대기 → 반환 (블로킹)

while True:
    records = consumer.poll(1.0)
    for record in records:
        process(record)
    consumer.commit()  # 모든 처리 완료 후 동기 커밋

# 장점: 커밋 성공 보장 (실패 시 재시도)
# 단점: 커밋마다 브로커 왕복 → 처리량 감소
```

### 수동 커밋 - commitAsync()

```python
# commitAsync(): 비동기 커밋
# 커밋 요청 전송 → 즉시 반환 (논블로킹)

def commit_callback(err, offsets):
    if err:
        print(f"커밋 실패: {err}")

while True:
    records = consumer.poll(1.0)
    for record in records:
        process(record)
    consumer.commit(asynchronous=True, callback=commit_callback)

# 장점: 블로킹 없이 처리량 유지
# 단점: 커밋 실패 시 자동 재시도 안 함
#       재시도하면 늦게 온 커밋이 이전 오프셋으로 덮어쓸 위험
#       → 종료 시 commitSync()로 마무리하는 패턴 권장
```

### 특정 오프셋 커밋

```python
from confluent_kafka import TopicPartition

# 파티션별로 특정 오프셋을 명시적으로 커밋
# 주로 배치 처리 후 파티션별 마지막 처리 오프셋을 커밋할 때 사용
offsets = [
    TopicPartition('orders', 0, 1500),  # 파티션 0의 오프셋 1500
    TopicPartition('orders', 1, 800),   # 파티션 1의 오프셋 800
]
consumer.commit(offsets=offsets)
```

### 커밋 오프셋과 next fetch offset의 관계

```
커밋하는 오프셋 = "다음에 읽을 오프셋" (Next Offset to Fetch)

offset 5까지 처리 완료 → 커밋값은 6 (5+1)
→ 재시작 시 offset 6부터 fetch

(confluent-kafka 등 클라이언트마다 내부적으로 +1 처리)
```

## auto.offset.reset - 오프셋이 없을 때

커밋된 오프셋이 존재하지 않거나 유효하지 않을 때, 컨슈머가 어디서 읽기를 시작할지 결정한다.

```
발생 상황:
  1. 새로운 컨슈머 그룹이 토픽을 처음 구독할 때
  2. 커밋된 오프셋이 retention 기간 초과로 삭제된 경우
  3. 커밋된 오프셋이 현재 파티션 범위를 벗어난 경우

auto.offset.reset 옵션:

earliest (가장 많이 사용):
  → 파티션의 가장 처음 메시지(가장 오래된 오프셋)부터 읽기
  → 데이터를 처음부터 전부 처리하고 싶을 때

latest (기본값):
  → 파티션의 가장 최신 오프셋(끝)부터 읽기
  → 컨슈머가 기동된 시점 이후의 메시지만 처리하고 싶을 때

none:
  → 커밋된 오프셋이 없으면 예외 발생 (NoOffsetForPartitionException)
  → 실수로 새 그룹을 만들거나 오프셋 손실을 방지하려는 보수적 설정

시각화:
  파티션: [offset 100] [101] [102] ... [999] [1000] ← LEO
                  ↑                              ↑
              earliest                         latest
```

## 오프셋 재조정 (Seek)

`seek()` API를 사용하면 컨슈머가 읽을 위치를 임의로 조정할 수 있다. 특정 오프셋 재처리, 장애 복구, 리플레이 등에 활용된다.

```python
from confluent_kafka import Consumer, TopicPartition

consumer = Consumer(conf)
consumer.subscribe(['orders'])

# subscribe 후 파티션이 실제 할당될 때까지 기다림
msg = consumer.poll(timeout=5.0)  # 첫 poll에서 할당 발생

# 특정 오프셋으로 이동
consumer.seek(TopicPartition('orders', 0, 500))  # 파티션 0, offset 500으로

# 파티션의 처음으로 이동
consumer.seek_to_beginning([TopicPartition('orders', 0)])

# 파티션의 끝으로 이동
consumer.seek_to_end([TopicPartition('orders', 0)])
```

```
seek()은 커밋된 오프셋을 변경하지 않는다!
→ 현재 세션의 읽기 위치(Current Position)만 변경
→ seek 후 재시작하면 커밋된 오프셋부터 다시 시작

seek 후 커밋된 오프셋도 변경하려면:
  seek() 이후 해당 위치부터 읽고 commit() 호출
```

---

# 그룹 코디네이터

## 그룹 코디네이터의 역할

**그룹 코디네이터(Group Coordinator)**는 Kafka 브로커 중 하나가 담당하는 특수 역할로, 특정 컨슈머 그룹의 멤버십과 오프셋을 관리한다.

```
그룹 코디네이터 결정 방식:

hash(group.id) % __consumer_offsets 파티션 수(기본 50)
= __consumer_offsets의 N번 파티션
→ 이 파티션의 리더 브로커 = 해당 그룹의 코디네이터

예시:
  group.id = "order-processor"
  hash("order-processor") % 50 = 17
  __consumer_offsets 파티션 17의 리더 = 브로커 2
  → 브로커 2가 "order-processor" 그룹의 코디네이터
```

### 코디네이터의 주요 기능

| 기능 | 설명 |
|------|------|
| 멤버십 관리 | 컨슈머 그룹 참여(JoinGroup) / 탈퇴(LeaveGroup) 처리 |
| 파티션 할당 | 그룹 내 컨슈머들에게 파티션 배분 (SyncGroup) |
| 하트비트 추적 | 컨슈머의 생존 여부 모니터링 |
| 오프셋 관리 | 커밋 오프셋 저장 및 조회 |
| 리밸런스 조율 | 멤버 변경 시 파티션 재할당 진행 |

## 컨슈머 그룹 라이프사이클

```
컨슈머 그룹 상태 전이:

Empty ──────────────────────────────────────────▶ Dead
  │                                               ↑
  │ 첫 번째 컨슈머가 JoinGroup 요청               │ 모든 멤버 탈퇴
  ▼                                              │
PreparingRebalance ◀──────────────── Stable ─────┘
  │         ↑ 리밸런스 필요한              │
  │         │  이벤트 발생                 │
  │         └── CompletingRebalance ◀─────┘
  │                        │
  └── 모든 멤버 JoinGroup ──┘
      응답 대기 후 SyncGroup
```

| 상태 | 설명 |
|------|------|
| `Empty` | 컨슈머 없음. 오프셋 커밋만 있을 수 있음 |
| `PreparingRebalance` | 리밸런스 준비 중. 모든 멤버의 JoinGroup 응답 수집 |
| `CompletingRebalance` | 모든 멤버가 Join 완료. 리더 컨슈머가 파티션 할당 계산 후 SyncGroup 전송 |
| `Stable` | 파티션 할당 완료. 정상 소비 중 |
| `Dead` | 그룹 삭제됨 |

## JoinGroup / SyncGroup 프로토콜

리밸런스는 **JoinGroup → SyncGroup** 두 단계로 이루어진다.

### 1단계: JoinGroup (멤버 모집)

```
리밸런스 발생 트리거:
  - 새 컨슈머가 그룹 합류
  - 컨슈머 탈퇴 or 장애
  - 구독 토픽 파티션 수 변경

JoinGroup 흐름:

  컨슈머 A  ─── JoinGroup 요청 ──────────▶ 코디네이터
  컨슈머 B  ─── JoinGroup 요청 ──────────▶ 코디네이터
  컨슈머 C  ─── JoinGroup 요청 ──────────▶ 코디네이터

  코디네이터:
    → 모든 멤버의 JoinGroup 수집 대기 (rebalance.timeout.ms)
    → 멤버 목록 정리 + 리더 컨슈머 선정 (보통 첫 번째 참여자)
    → JoinGroup 응답 전송

  컨슈머 A (리더) ← 응답: 멤버 목록 + Assignor 정보 포함
  컨슈머 B        ← 응답: 자신의 member.id만 포함
  컨슈머 C        ← 응답: 자신의 member.id만 포함

  → 파티션 할당 계산은 리더 컨슈머가 수행! (브로커 아님)
  → Kafka 4.0 이전 기준
```

### 2단계: SyncGroup (할당 동기화)

```
SyncGroup 흐름:

  컨슈머 A (리더):
    - 파티션 할당 계산 (Assignor 전략에 따라)
    - SyncGroup 요청 전송 (할당 결과 포함)

  컨슈머 B:
    - SyncGroup 요청 전송 (빈 할당)

  컨슈머 C:
    - SyncGroup 요청 전송 (빈 할당)

  코디네이터:
    - 리더의 할당 결과를 받아 각 멤버에게 분배
    - SyncGroup 응답 전송 (각자의 파티션 할당 정보 포함)

  컨슈머 A ← 파티션 0, 1 할당
  컨슈머 B ← 파티션 2 할당
  컨슈머 C ← 파티션 3 할당

  → Stable 상태로 전환, 소비 시작
```

### 기존 리밸런스의 문제: Stop-the-World

```
Eager 리밸런스 (Kafka 4.0 이전 기본 방식):

  컨슈머 A: 파티션 0, 1 소비 중 ──▶ 리밸런스 시작
  컨슈머 B: 파티션 2 소비 중     ──▶ 즉시 모든 파티션 반납
  컨슈머 C: 파티션 3 소비 중     ──▶ 즉시 모든 파티션 반납

  [멈춤 구간: 모든 컨슈머가 아무 파티션도 처리 안 함]

  JoinGroup ──▶ SyncGroup ──▶ 새 파티션 할당

  → 리밸런스 동안 전체 그룹의 소비가 중단됨
  → 컨슈머 수가 많을수록, 처리 지연이 클수록 영향 큼
```

## 하트비트 메커니즘

컨슈머는 별도의 **하트비트 스레드**를 통해 코디네이터에 생존 신호를 보낸다.

```
하트비트 흐름:

컨슈머 (Heartbeat Thread)
  │
  ├─ heartbeat.interval.ms(기본 3초)마다 HeartbeatRequest 전송
  │      │
  │      ▼
  │   그룹 코디네이터
  │      │
  │      └─ HeartbeatResponse 반환 (리밸런스 필요 여부 포함)
  │
  └─ HeartbeatResponse에 REBALANCE_IN_PROGRESS 포함 시
       → 컨슈머에게 리밸런스 시작 알림
       → 컨슈머가 다음 poll() 시 리밸런스 참여

session.timeout.ms(기본 45초) 내에 하트비트 없으면:
  → 코디네이터가 해당 컨슈머를 죽은 것으로 판단
  → 리밸런스 시작
```

### 하트비트 vs poll() 간격의 관계

```
두 가지 독립적인 타임아웃:

1. session.timeout.ms (하트비트 기반)
   → 하트비트 스레드가 별도로 동작하므로
   → poll() 없이도 하트비트가 전송됨
   → 네트워크 완전 단절, 프로세스 죽음을 감지

2. max.poll.interval.ms (poll() 기반)
   → poll() 호출 간격이 이 값을 초과하면
   → 컨슈머 로직이 멈춘 것으로 판단 → 리밸런스
   → 무거운 처리 로직으로 poll()을 오래 못 부를 때 발생

설정 권장:
  heartbeat.interval.ms < session.timeout.ms / 3
  예: heartbeat=3s, session=45s → 하트비트 15회 실패해야 탈퇴
```

## 리밸런스 트리거 조건

| 조건 | 설명 |
|------|------|
| 컨슈머 그룹 참여 | `subscribe()` 후 `poll()` 호출 시 JoinGroup 요청 |
| 컨슈머 정상 탈퇴 | `close()` 호출 → LeaveGroup 요청 → 즉시 리밸런스 |
| 컨슈머 장애 | `session.timeout.ms` 내 하트비트 없음 → 코디네이터가 제거 |
| poll() 지연 | `max.poll.interval.ms` 초과 → 코디네이터가 제거 |
| 파티션 수 변경 | 토픽의 파티션 수 증가 시 |
| 구독 토픽 변경 | 구독 패턴이 변경된 경우 (예: 정규식 구독 시 새 토픽 추가) |

## Kafka 4.0 - 새로운 리밸런스 프로토콜 (KIP-848)

Kafka 4.0에서는 리밸런스 로직이 **브로커(코디네이터) 측으로 이동**하여 Stop-the-World 없이 점진적 리밸런스가 가능해졌다.

```
KIP-848 (Consumer Group Protocol) 특징:

기존 방식 (Eager/Cooperative):
  컨슈머 → JoinGroup/SyncGroup → 코디네이터
  → 파티션 할당을 리더 컨슈머가 계산 (클라이언트 부담)
  → 모든 멤버가 응답해야 리밸런스 완료

새 방식 (KIP-848):
  컨슈머 → Heartbeat (목표 상태 포함) → 코디네이터
  → 파티션 할당을 코디네이터(브로커)가 계산
  → HeartbeatResponse에 새 할당 정보 포함
  → 개별 컨슈머가 독립적으로 파티션 이동

결과:
  → Stop-the-World 리밸런스 없음
  → 파티션이 하나씩 점진적으로 이동
  → 대규모 컨슈머 그룹에서 리밸런스 시간 대폭 감소
  → 클라이언트 측 Assignor 설정 불필요
```

---

# 스태틱 멤버십

## 기존 동적 멤버십의 문제

기존 Kafka는 컨슈머가 재시작될 때마다 새로운 `member.id`를 부여받고, 이를 그룹에서 탈퇴 후 재가입으로 처리한다. 이 과정이 불필요한 리밸런스를 유발한다.

```
동적 멤버십의 문제:

시나리오: 컨슈머 A가 잠시 재시작 (배포, 설정 변경 등)

컨슈머 A: 재시작
  ① LeaveGroup 전송 (또는 session.timeout.ms 대기)
  ② 코디네이터: 리밸런스 시작
  ③ 파티션 A의 파티션들이 다른 컨슈머로 재할당
  ④ 컨슈머 A 재기동 → JoinGroup
  ⑤ 코디네이터: 다시 리밸런스
  ⑥ 파티션 A의 파티션들이 다시 재할당

→ 재시작 1번에 리밸런스 2번 발생
→ 리밸런스 동안 처리 중단
→ 재할당으로 인한 State Store 마이그레이션 비용 (Kafka Streams)
→ 컨슈머가 많을수록 피해 큼
```

## 스태틱 멤버십 동작 원리

`group.instance.id`를 설정하면 컨슈머가 재시작되어도 같은 멤버로 인식되어 불필요한 리밸런스를 방지한다.

```
스태틱 멤버십 (group.instance.id 설정 시):

컨슈머 A: group.instance.id = "consumer-instance-0"

① 컨슈머 A가 재시작
② 코디네이터에 JoinGroup 요청 (instance.id 포함)
③ 코디네이터: "consumer-instance-0"을 이미 알고 있음
   → LeaveGroup + 리밸런스 없이 기존 할당 유지
   → (단, session.timeout.ms 이내에 재기동한 경우)

④ 컨슈머 A: 이전에 할당받았던 파티션 그대로 유지하여 소비 재개

비교:

동적 멤버십:              스태틱 멤버십:
재시작 → 리밸런스 2회   재시작 → 리밸런스 0회
파티션 재배치            파티션 유지
```

### 세션 타임아웃 초과 시

```
session.timeout.ms를 초과하여 재시작한 경우:
  → 코디네이터가 해당 instance.id를 제거 → 리밸런스 발생
  → 재기동 후 다시 기존 instance.id로 참여하면
    → 코디네이터가 이 instance.id를 새 멤버로 등록
    → 리밸런스 1회 (합류 시)만 발생

→ 스태틱 멤버십은 session.timeout.ms 내 재기동에 효과적
→ 배포 시간이 session.timeout.ms(기본 45초)보다 길면 효과 없음
→ 배포 환경에 맞게 session.timeout.ms를 늘려 사용
```

## 스태틱 멤버십 설정

```python
conf = {
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'order-processor',
    'group.instance.id': 'consumer-pod-0',   # 고유한 정적 ID
    'session.timeout.ms': '120000',           # 배포 시간 고려하여 늘림
    'heartbeat.interval.ms': '3000',
}

consumer = Consumer(conf)
consumer.subscribe(['orders'])
```

```
group.instance.id 명명 전략:

Kubernetes 환경:
  group.instance.id = f"{deployment_name}-{pod_ordinal}"
  예: "order-consumer-0", "order-consumer-1", "order-consumer-2"

→ StatefulSet 파드는 재시작 시 같은 이름으로 복구됨
→ 각 파드가 항상 같은 파티션을 담당
→ Kafka Streams의 State Store가 파드에 남아 재구축 비용 없음
```

### 스태틱 멤버십 관련 설정

| 설정 | 기본값 | 설명 |
|------|--------|------|
| `group.instance.id` | 없음 | 설정 시 스태틱 멤버십 활성화. 그룹 내 고유값이어야 함 |
| `session.timeout.ms` | 45000 (45초) | 스태틱 멤버십 사용 시 배포 시간을 고려하여 늘리는 것 권장 |

> **실무 요약**: 배포가 잦거나, Kafka Streams State Store를 사용하거나, 컨슈머 수가 많아 리밸런스 비용이 큰 경우 스태틱 멤버십을 적극 활용한다. Kubernetes StatefulSet + 스태틱 멤버십 조합이 대표적인 패턴이다.

---

# 컨슈머 파티션 할당 전략

파티션 할당 전략(Assignor)은 **컨슈머 그룹 내에서 파티션을 어떻게 나눌지** 결정한다. `partition.assignment.strategy` 설정으로 지정한다.

```
예시 환경: 토픽 2개 (T1: 파티션 3개, T2: 파티션 3개)
          컨슈머 3개 (C1, C2, C3)

T1: P0, P1, P2
T2: P0, P1, P2
```

## RangeAssignor

**토픽별로** 파티션을 연속된 범위로 나누어 할당한다.

```
RangeAssignor 동작:

T1(파티션 3개, 컨슈머 3개):
  C1 ← T1-P0
  C2 ← T1-P1
  C3 ← T1-P2

T2(파티션 3개, 컨슈머 3개):
  C1 ← T2-P0
  C2 ← T2-P1
  C3 ← T2-P2

최종 할당:
  C1: T1-P0, T2-P0
  C2: T1-P1, T2-P1
  C3: T1-P2, T2-P2

→ 파티션 수가 컨슈머 수의 배수일 때 균등 분배
```

### RangeAssignor의 불균등 문제

```
토픽 3개 (T1, T2, T3 각 3개 파티션), 컨슈머 2개 (C1, C2):

T1(3개 파티션, 컨슈머 2개):
  C1 ← T1-P0, T1-P1  (2개)
  C2 ← T1-P2          (1개)

T2:
  C1 ← T2-P0, T2-P1  (2개)
  C2 ← T2-P2          (1개)

T3:
  C1 ← T3-P0, T3-P1  (2개)
  C2 ← T3-P2          (1개)

최종:
  C1: 6개 파티션  ← 과부하!
  C2: 3개 파티션

→ 토픽이 많아질수록 앞쪽 컨슈머에 파티션이 집중됨
```

## RoundRobinAssignor

**모든 파티션을 하나의 목록**으로 만들어 컨슈머에게 순서대로 배분한다.

```
RoundRobinAssignor 동작:

전체 파티션 목록 (정렬됨):
  T1-P0, T1-P1, T1-P2, T2-P0, T2-P1, T2-P2

라운드 로빈 배분:
  C1 ← T1-P0
  C2 ← T1-P1
  C3 ← T1-P2
  C1 ← T2-P0
  C2 ← T2-P1
  C3 ← T2-P2

최종 할당:
  C1: T1-P0, T2-P0  (2개)
  C2: T1-P1, T2-P1  (2개)
  C3: T1-P2, T2-P2  (2개)

→ RangeAssignor보다 균등하게 분배
```

### 조건: 모든 컨슈머가 같은 토픽을 구독해야 함

```
컨슈머가 다른 토픽을 구독하는 경우 불균등 발생:
  C1: T1, T2 구독
  C2: T1만 구독

→ RoundRobinAssignor가 예상치 못한 할당을 할 수 있음
→ 모든 컨슈머가 동일 토픽을 구독할 때 효과적
```

## StickyAssignor

**기존 할당을 최대한 유지**하면서 균등하게 배분한다. 리밸런스 시 파티션 이동을 최소화한다.

```
StickyAssignor의 두 가지 목표:
  1. 파티션을 가능한 균등하게 배분
  2. 리밸런스 시 기존 할당을 최대한 유지 (이동 최소화)

리밸런스 전 (C1, C2, C3 각 2개):
  C1: T1-P0, T2-P0
  C2: T1-P1, T2-P1
  C3: T1-P2, T2-P2

C3가 탈퇴한 후 리밸런스:

  RoundRobin 방식:    Sticky 방식:
  C1: T1-P0, T1-P2    C1: T1-P0, T2-P0, T1-P2  ← 기존 유지 + 이동 최소
  C2: T2-P0, T2-P2    C2: T1-P1, T2-P1, T2-P2
  C1: T1-P1, T2-P1

  → RoundRobin은 기존 할당 무시하고 재배치
  → Sticky는 C1, C2가 기존 파티션을 유지하고
    C3의 파티션(T1-P2, T2-P2)만 이동
```

### StickyAssignor는 Eager 리밸런스 사용

```
StickyAssignor도 Eager(Stop-the-World) 리밸런스를 사용한다.
→ 리밸런스 시작 시 모든 파티션을 잠시 반납 후 재할당
→ 단, 재할당 결과가 이전과 유사하여 처리 재개가 빠름

완전한 비중단 리밸런스를 원한다면 CooperativeStickyAssignor 사용
```

## CooperativeStickyAssignor

**점진적(Incremental) 리밸런스**를 지원하여 Stop-the-World를 방지한다. Kafka 2.4+에서 도입되었으며, Kafka 4.0 이전 기준으로 최신 권장 전략이다.

```
CooperativeStickyAssignor 동작:

리밸런스 1라운드:
  → 코디네이터가 "이동 필요한 파티션 목록" 전달
  → 각 컨슈머는 이동 대상 파티션만 반납 (나머지는 유지)
  → 소비 계속!

리밸런스 2라운드:
  → 반납된 파티션을 새 컨슈머에게 할당
  → 새 할당 완료

예시 (C4가 새로 합류):
  C1: T1-P0, T2-P0  (유지)
  C2: T1-P1, T2-P1  (유지)
  C3: T1-P2, T2-P2  (유지)
  C4: 합류

  1라운드: C3가 T2-P2만 반납 (T1-P2 유지하며 소비 중)
  2라운드: T2-P2를 C4에게 할당

  최종:
    C1: T1-P0, T2-P0  (소비 중단 없음)
    C2: T1-P1, T2-P1  (소비 중단 없음)
    C3: T1-P2          (T2-P2 반납, 나머지 소비 계속)
    C4: T2-P2          (새로 할당)

  → 이동이 필요한 파티션만 잠시 중단
  → 전체 그룹 중단 없음!
```

```python
conf = {
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'order-processor',
    'partition.assignment.strategy': 'cooperative-sticky',  # 권장
}
```

## 할당 전략 비교

| 전략 | 리밸런스 방식 | 이동 최소화 | 균등 분배 | 권장 상황 |
|------|------------|-----------|---------|---------|
| `RangeAssignor` | Eager (Stop-the-World) | X | 토픽 수 많으면 불균등 | 단일 토픽, 간단한 구성 |
| `RoundRobinAssignor` | Eager (Stop-the-World) | X | 균등 (같은 토픽 구독 시) | 모든 컨슈머가 동일 토픽 구독 |
| `StickyAssignor` | Eager (Stop-the-World) | O | 균등 | 이동 최소화 + 허용 가능한 중단 |
| `CooperativeStickyAssignor` | Cooperative (점진적) | O | 균등 | **대부분의 운영 환경에서 권장** |
| KIP-848 (Kafka 4.0) | 브로커 주도 점진적 | O | 균등 | Kafka 4.0 이상 |

### 설정 예시

```python
# CooperativeStickyAssignor (Kafka 4.0 이전 권장)
conf = {
    'partition.assignment.strategy': 'cooperative-sticky',
}

# Kafka 4.0: 새 프로토콜 자동 사용 (별도 설정 불필요)
# group.protocol=consumer 설정으로 명시 가능 (기본값)
```

> **실무 요약**: Kafka 4.0 이전에는 `CooperativeStickyAssignor`를 사용하는 것이 권장된다. Kafka 4.0 이상에서는 새 리밸런스 프로토콜(KIP-848)이 기본 적용되므로 별도 설정 없이도 비중단 리밸런스가 가능하다.
