- kafka v4 기준 설명

# kafka 핵심 개념, 구성 요소

## 목차

- [kafka 핵심 개념, 구성 요소](#kafka-핵심-개념-구성-요소)
  - [1. Kafka 서버와 클라이언트](#1-kafka-서버와-클라이언트)
  - [2. 분산 시스템](#2-분산-시스템)
  - [3. 페이지 캐시 (Page Cache)](#3-페이지-캐시-page-cache)
  - [4. 배치 전송 처리](#4-배치-전송-처리)
  - [5. 압축 전송](#5-압축-전송)
  - [6. 브로커 (Broker)](#6-브로커-broker)
  - [7. 레플리케이션, 토픽, 파티션, 세그먼트, 오프셋](#7-레플리케이션-토픽-파티션-세그먼트-오프셋)
  - [8. 이벤트 (Event / Record)](#8-이벤트-event--record)
  - [9. 프로듀서 (Producer)](#9-프로듀서-producer)
  - [10. 컨슈머 (Consumer)](#10-컨슈머-consumer)
  - [11. 고가용성 보장](#11-고가용성-보장)
- [프로듀서 기본 동작과 예시](#프로듀서-기본-동작과-예시)
  - [프로듀서 디자인](#프로듀서-디자인)
  - [프로듀서 주요 옵션](#프로듀서-주요-옵션)
  - [프로듀서 예제 (Python, confluent-kafka)](#프로듀서-예제-python-confluent-kafka)
  - [실무 가이드: poll(), flush() 사용 전략](#실무-가이드-poll-flush-사용-전략)
- [컨슈머 기본 동작과 예시](#컨슈머-기본-동작과-예시)
  - [컨슈머 디자인](#컨슈머-디자인)
  - [컨슈머 그룹과 파티션 할당](#컨슈머-그룹과-파티션-할당)
  - [컨슈머 주요 옵션](#컨슈머-주요-옵션)
  - [오프셋 커밋 전략](#오프셋-커밋-전략)
  - [컨슈머 예제 (Python, confluent-kafka)](#컨슈머-예제-python-confluent-kafka)
  - [실무 가이드: poll() 루프 설계](#실무-가이드-poll-루프-설계)

---

## 1. Kafka 서버와 클라이언트
- **서버(브로커)**: 메시지를 수신, 저장, 전달하는 역할을 하는 Kafka 프로세스. 여러 대의 브로커가 모여 클러스터를 구성한다.
- **클라이언트**: 서버와 통신하는 애플리케이션으로, 크게 **프로듀서**(메시지 발행)와 **컨슈머**(메시지 소비)로 나뉜다.
- Kafka 4.0부터는 ZooKeeper가 제거되고, 브로커 중 일부가 **KRaft 컨트롤러** 역할을 겸하여 클러스터 메타데이터를 자체 관리한다.

## 2. 분산 시스템
- Kafka는 여러 브로커로 구성된 **분산 클러스터**로 동작한다.
- 하나의 토픽을 여러 파티션으로 나누고, 이 파티션들을 서로 다른 브로커에 분산 배치하여 **부하 분산**과 **병렬 처리**를 달성한다.
- 브로커 일부에 장애가 발생해도 나머지 브로커가 서비스를 계속할 수 있다.

## 3. 페이지 캐시 (Page Cache)
- Kafka는 메시지를 디스크에 저장하지만, OS의 **페이지 캐시**를 적극 활용하여 높은 처리량을 달성한다.
- 프로듀서가 보낸 메시지는 먼저 OS 페이지 캐시에 기록되고, OS가 비동기적으로 디스크에 flush한다.
- 컨슈머가 읽는 데이터도 페이지 캐시에 있으면 디스크 I/O 없이 메모리에서 바로 전달된다.
- JVM 힙 메모리가 아닌 OS 레벨 캐시를 사용하므로, GC 부담 없이 대용량 데이터를 처리할 수 있다.

## 4. 배치 전송 처리
- 프로듀서는 메시지를 하나씩 보내지 않고 **배치(batch)로 묶어** 전송한다.
- `linger.ms`(배치 대기 시간)와 `batch.size`(배치 크기) 설정으로 배치 동작을 제어한다.
- 네트워크 왕복 횟수를 줄여 처리량을 극대화하고, 브로커 측에서도 배치 단위로 디스크에 기록하여 I/O 효율을 높인다.

## 5. 압축 전송
- 배치 단위로 메시지를 **압축**하여 네트워크 대역폭과 디스크 사용량을 절약한다.
- 지원 압축 방식: `gzip`, `snappy`, `lz4`, `zstd`
- 일반적으로 **lz4** 또는 **zstd**가 압축률과 속도의 균형이 좋아 많이 사용된다.
- 프로듀서에서 압축하고, 브로커는 압축된 상태 그대로 저장하며, 컨슈머가 읽을 때 해제한다.

## 6. 브로커 (Broker)
- Kafka 클러스터를 구성하는 **개별 서버 프로세스**이다.
- 각 브로커는 고유한 `broker.id`를 가지며, 하나 이상의 파티션의 리더 또는 팔로워 역할을 수행한다.
- Kafka 4.0에서는 브로커가 **KRaft 모드**로만 동작하며, 컨트롤러 역할을 담당하는 브로커가 리더 선출과 메타데이터 관리를 수행한다.

## 7. 레플리케이션, 토픽, 파티션, 세그먼트, 오프셋

### 토픽 (Topic)
- 메시지를 분류하는 논리적 단위. 프로듀서는 특정 토픽에 메시지를 발행하고, 컨슈머는 토픽을 구독한다.

### 파티션 (Partition)
- 토픽은 하나 이상의 파티션으로 나뉜다. 파티션은 메시지가 실제로 저장되는 단위이다.
- 각 파티션은 **순서가 보장된 불변의 로그**이며, 파티션 수를 늘려 병렬 처리 성능을 확장한다.
- 메시지에 **Key가 있으면** 같은 Key는 항상 같은 파티션으로 라우팅된다. **Key가 없으면** 라운드 로빈/Sticky 방식으로 골고루 분배된다.
- 각 파티션에는 **서로 다른 메시지**가 들어가고, 순서 보장은 파티션 내에서만 유효하다.

```
예시) 토픽 "orders" - 파티션 3개

파티션 0: [msg-A, msg-D, msg-G, ...]  ← key=user_1 인 메시지
파티션 1: [msg-B, msg-E, msg-H, ...]  ← key=user_2 인 메시지
파티션 2: [msg-C, msg-F, msg-I, ...]  ← key=user_3 인 메시지

→ 파티션 = 데이터를 "나누는" 것 (병렬 처리, 처리량 향상)
```

### 세그먼트 (Segment)
- 파티션은 물리적으로 여러 **세그먼트 파일**로 나뉘어 디스크에 저장된다.
- 일정 크기(`log.segment.bytes`) 또는 시간(`log.roll.ms`)이 되면 새 세그먼트가 생성된다.
- 오래된 세그먼트는 retention 정책에 따라 삭제되거나 compact된다.

### 오프셋 (Offset)
- 파티션 내에서 각 메시지에 부여되는 **순차적인 고유 번호**(0, 1, 2, ...).
- 컨슈머는 자신이 마지막으로 읽은 오프셋을 기록(commit)하여, 재시작 시 이어서 소비할 수 있다.
- 오프셋은 `__consumer_offsets` 내부 토픽에 저장된다.

### 레플리케이션 (Replication)
- 각 파티션은 설정된 수(`replication.factor`)만큼 **복제본**을 가진다.
- **리더 파티션**: 프로듀서/컨슈머의 읽기·쓰기 요청을 처리한다.
- **팔로워 파티션**: 리더의 데이터를 지속적으로 복제하며, 리더 장애 시 팔로워 중 하나가 새로운 리더로 승격된다.
- **ISR(In-Sync Replica)**: 리더와 동기화된 복제본 집합. ISR에 포함된 팔로워만 리더 승격 대상이 된다.
- 복제는 **토픽 단위가 아니라 파티션 단위**로 이루어진다.

```
예시) 토픽 "orders" - 파티션 3개, replication.factor=3, 브로커 3대

브로커 1:  파티션0(리더)    파티션1(팔로워)  파티션2(팔로워)
브로커 2:  파티션0(팔로워)  파티션1(리더)    파티션2(팔로워)
브로커 3:  파티션0(팔로워)  파티션1(팔로워)  파티션2(리더)

→ 각 파티션마다 리더 1개 + 팔로워 2개가 서로 다른 브로커에 분산
→ 프로듀서/컨슈머는 리더에게만 읽기·쓰기
→ 팔로워는 리더 데이터를 계속 복제하다가, 리더 장애 시 승격
→ 레플리케이션 = 파티션을 "복사"하는 것 (장애 대비, 고가용성)
```

## 8. 이벤트 (Event / Record)
- Kafka에서 주고받는 데이터의 기본 단위이다. (메시지 또는 레코드라고도 부른다.)
- 구성 요소:
  - **Key**: 파티션 라우팅에 사용 (같은 키 → 같은 파티션)
  - **Value**: 실제 데이터 (JSON, Avro, Protobuf 등)
  - **Timestamp**: 이벤트 발생 시각
  - **Headers**: 메타데이터 (key-value 쌍)

## 9. 프로듀서 (Producer)
- 토픽에 메시지를 **발행**하는 클라이언트이다.
- 메시지 키에 따라 파티셔너가 대상 파티션을 결정한다 (키가 없으면 라운드 로빈 또는 Sticky Partitioner).
- 전송 보장 수준을 `acks` 설정으로 제어한다:
  - `acks=0`: 응답을 기다리지 않음 (가장 빠르지만 유실 가능)
  - `acks=1`: 리더에 기록 확인 후 응답
  - `acks=all(-1)`: ISR 전체에 기록 확인 후 응답 (가장 안전)

## 10. 컨슈머 (Consumer)
- 토픽에서 메시지를 **소비**하는 클라이언트이다.
- **컨슈머 그룹**: 같은 `group.id`를 가진 컨슈머들은 파티션을 나누어 소비한다. 하나의 파티션은 그룹 내 하나의 컨슈머에게만 할당된다.
- Kafka 4.0의 **새로운 리밸런스 프로토콜(KIP-848)**: 리밸런스 로직이 브로커로 이동하여 Stop-the-World 없이 부드럽게 파티션이 재할당된다.
- Kafka 4.0의 **Share Group(KIP-932, Early Access)**: 같은 파티션을 여러 컨슈머가 협력적으로 소비할 수 있는 큐 방식도 도입되었다.

## 11. 고가용성 보장
- **레플리케이션**: 파티션 복제를 통해 브로커 장애 시에도 데이터 유실 없이 서비스를 지속한다.
- **자동 리더 선출**: 리더 브로커가 다운되면 ISR 내의 팔로워가 자동으로 리더로 승격된다.
- **KRaft 컨트롤러 쿼럼**: Kafka 4.0에서는 컨트롤러 노드들이 Raft 합의 프로토콜로 메타데이터를 관리하므로, 단일 장애점(SPOF) 없이 고가용성을 유지한다.
- **`min.insync.replicas`**: 최소 동기화 복제본 수를 지정하여, 충분한 복제가 이루어지지 않으면 쓰기를 거부하도록 설정할 수 있다.

# 프로듀서 기본 동작과 예시

## 프로듀서 디자인

프로듀서는 메시지를 브로커로 전송하기까지 내부적으로 여러 단계를 거친다.

```
┌─────────────────────────────────────────────────────────────────┐
│  Producer 애플리케이션                                            │
│                                                                 │
│  send(topic, key, value)                                        │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────┐    직렬화 실패 시                                │
│  │  Serializer  │───→ SerializationException                    │
│  │ (Key+Value)  │                                               │
│  └──────┬──────┘                                                │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────┐   파티션 결정 로직:                             │
│  │  Partitioner  │  - key 있음 → hash(key) % partition 수        │
│  │              │  - key 없음 → Sticky Partitioner (배치 단위)    │
│  └──────┬───────┘                                               │
│         │                                                       │
│         ▼                                                       │
│  ┌───────────────────────────────────┐                          │
│  │  RecordAccumulator (배치 버퍼)      │                          │
│  │                                   │                          │
│  │  파티션0 배치: [msg1, msg2, ...]   │                          │
│  │  파티션1 배치: [msg3, msg4, ...]   │                          │
│  │  파티션2 배치: [msg5, ...]         │                          │
│  └──────────────┬────────────────────┘                          │
│                 │                                               │
│    batch.size 도달 또는 linger.ms 경과 시                         │
│                 │                                               │
│                 ▼                                               │
│  ┌──────────────────┐                                           │
│  │   Sender Thread   │  (별도 I/O 스레드)                        │
│  │  배치를 브로커별로  │                                           │
│  │  묶어서 전송       │                                           │
│  └────────┬─────────┘                                           │
└───────────┼─────────────────────────────────────────────────────┘
            │
            ▼
     ┌──────────────┐
     │  Kafka Broker  │
     │  (리더 파티션)   │
     └──────┬───────┘
            │
            ▼
    acks 설정에 따라 응답
    - acks=0: 응답 안 기다림
    - acks=1: 리더 기록 후 응답
    - acks=all: ISR 전체 기록 후 응답
            │
            ▼
    성공 → RecordMetadata (토픽, 파티션, 오프셋) 반환
    실패 → retries 설정만큼 재시도 후 최종 실패 시 예외 발생
```

### 핵심 포인트
- `send()`는 **비동기**이다. 호출 즉시 Future를 반환하고, 실제 전송은 Sender 스레드가 담당한다.
- Serializer → Partitioner → RecordAccumulator → Sender 순서로 처리된다.
- RecordAccumulator는 파티션별로 배치를 관리하며, `batch.size` 또는 `linger.ms` 조건 충족 시 Sender가 배치를 가져간다.
- 전송 실패 시 `retries` 설정에 따라 자동 재시도하며, `delivery.timeout.ms` 내에서만 재시도한다.

## 프로듀서 주요 옵션

| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `bootstrap.servers` | - | 브로커 접속 주소 (host:port). 최소 2개 이상 권장 |
| `key.serializer` | - | 메시지 키 직렬화 클래스 |
| `value.serializer` | - | 메시지 값 직렬화 클래스 |
| `acks` | `all` | 전송 보장 수준. `0`, `1`, `all` |
| `batch.size` | 16384 (16KB) | 파티션별 배치 최대 크기 (바이트) |
| `linger.ms` | 0 | 배치 전송 전 대기 시간. 늘리면 배치에 더 많은 메시지가 모인다 |
| `buffer.memory` | 33554432 (32MB) | 프로듀서 전체 전송 버퍼 크기. 초과 시 `send()` 블로킹 |
| `compression.type` | `none` | 압축 방식. `gzip`, `snappy`, `lz4`, `zstd` |
| `retries` | 2147483647 | 전송 실패 시 재시도 횟수 |
| `delivery.timeout.ms` | 120000 (2분) | `send()` 호출 후 최종 성공/실패까지 허용 시간 |
| `max.in.flight.requests.per.connection` | 5 | 브로커 응답 없이 동시 전송 가능한 요청 수 |
| `enable.idempotence` | `true` | 멱등성 프로듀서 활성화. 네트워크 재시도 시 중복 방지 |
| `transactional.id` | `null` | 트랜잭션 프로듀서 ID. 설정 시 Exactly-Once 전송 가능 |

### 주요 옵션 조합 가이드

**높은 처리량 (throughput 우선)**
```
linger.ms=50
batch.size=65536
compression.type=lz4
acks=1
```

**데이터 안전성 (durability 우선)**
```
acks=all
enable.idempotence=true
min.insync.replicas=2   # 브로커 측 설정
retries=2147483647
max.in.flight.requests.per.connection=5  # 멱등성 활성화 시 순서 보장됨
```

## 프로듀서 예제 (Python, confluent-kafka)

### 기본 프로듀서

```python
from confluent_kafka import Producer

conf = {
    'bootstrap.servers': 'localhost:9092',
    'acks': 'all',
    'linger.ms': 10,
    'batch.size': 32768,
    'compression.type': 'lz4',
}

producer = Producer(conf)


# 전송 결과 콜백
def delivery_callback(err, msg):
    if err:
        print(f'전송 실패: {err}')
    else:
        print(f'전송 성공: topic={msg.topic()}, '
              f'partition={msg.partition()}, offset={msg.offset()}')


# 메시지 전송
for i in range(10):
    producer.produce(
        topic='orders',
        key=f'user_{i % 3}',        # 같은 유저는 같은 파티션으로
        value=f'{{"order_id": {i}, "item": "product_{i}"}}',
        callback=delivery_callback,
    )

    # 내부 큐가 가득 차지 않도록 주기적으로 poll
    producer.poll(0)

# 버퍼에 남은 메시지를 모두 전송하고 대기
producer.flush()
```

### 동기 전송 (결과를 즉시 확인)

```python
from confluent_kafka import Producer, KafkaError
import time

producer = Producer({'bootstrap.servers': 'localhost:9092', 'acks': 'all'})

result = {}


def sync_callback(err, msg):
    result['err'] = err
    result['msg'] = msg


producer.produce(
    topic='orders',
    key='user_1',
    value='{"order_id": 100}',
    callback=sync_callback,
)
producer.flush()  # 콜백이 실행될 때까지 대기

if result['err']:
    print(f'실패: {result["err"]}')
else:
    print(f'성공: partition={result["msg"].partition()}, '
          f'offset={result["msg"].offset()}')
```

### Exactly-Once 트랜잭션 프로듀서

```python
from confluent_kafka import Producer

conf = {
    'bootstrap.servers': 'localhost:9092',
    'transactional.id': 'my-tx-producer-001',  # 트랜잭션 ID 필수
    'acks': 'all',
    'enable.idempotence': True,
}

producer = Producer(conf)
producer.init_transactions()  # 트랜잭션 초기화 (최초 1회)

try:
    producer.begin_transaction()

    # 여러 메시지를 하나의 트랜잭션으로 묶어 전송
    producer.produce('orders', key='user_1', value='{"order_id": 1}')
    producer.produce('payments', key='user_1', value='{"payment_id": 1}')
    producer.produce('notifications', key='user_1', value='{"type": "order_created"}')

    producer.commit_transaction()
    print('트랜잭션 커밋 완료')

except Exception as e:
    print(f'트랜잭션 실패, 롤백: {e}')
    producer.abort_transaction()
```

> **트랜잭션 프로듀서**는 여러 토픽/파티션에 걸친 메시지를 **원자적(all-or-nothing)**으로 전송한다. 컨슈머 측에서 `isolation.level=read_committed`로 설정하면 커밋된 트랜잭션의 메시지만 읽게 된다.

## 실무 가이드: poll(), flush() 사용 전략

### poll()의 동작 원리

`produce()`로 메시지를 보내면 Sender 스레드가 백그라운드에서 브로커로 전송하고, 브로커 응답은 내부 **완료 큐**에 쌓인다. 하지만 **응답이 와도 콜백이 자동으로 실행되지 않는다.** `poll()`을 호출해야 완료 큐에서 응답을 꺼내 콜백 함수를 실행한다.

```
produce() → 내부 버퍼에 적재
                ↓
      Sender 스레드가 브로커로 전송 (백그라운드)
                ↓
      브로커 응답 → 내부 "완료 큐"에 쌓임   ← 여기까지 자동
                ↓
      poll() 호출 → 완료 큐에서 꺼내서 콜백 실행   ← 명시적 호출 필요
```

- `poll(0)`: 완료 큐에 이미 도착한 응답만 처리하고 **즉시 반환** (비블로킹)
- `poll(5)`: 완료 큐가 비어있으면 최대 **5초까지 대기** 후 반환

> confluent-kafka는 C 라이브러리(librdkafka) 기반으로, 멀티스레드 안전성을 위해 "응답 수신"과 "콜백 실행"을 분리한 구조이다. 콜백은 반드시 `poll()` 또는 `flush()`를 호출하는 애플리케이션 스레드에서 실행된다.

### flush()의 동작 원리

`flush()`는 내부적으로 **`poll()`을 반복 호출**하면서, 프로듀서 내부 버퍼(RecordAccumulator)에 남아있는 모든 메시지가 브로커에 전달되고 응답까지 완료될 때까지 블로킹한다.

```
flush() 호출
    │
    ├─→ 내부 버퍼에 미전송 메시지가 있는가?
    │       YES → Sender 스레드가 전송할 때까지 대기
    │       NO ──┐
    │            │
    ├─→ 완료 큐에 미처리 응답이 있는가?
    │       YES → 콜백 실행 (= poll() 동작)
    │       NO ──┐
    │            │
    └─→ 버퍼 비어있음 + 완료 큐 비어있음 → 반환
```

즉, `flush()` = **"버퍼가 완전히 비워질 때까지 poll()을 반복"**이다.

- 콜백이 등록되어 있으면 `flush()` 내부에서 콜백도 실행됨
- 콜백이 없어도 전송 완료 대기 + 내부 리소스 정리는 동일하게 수행됨
- `flush(timeout)`: 최대 timeout(초)만큼 대기. 시간 초과 시 미완료 메시지가 남아있을 수 있음

### poll(0) vs flush() 차이

| | `poll(0)` | `flush()` |
|---|---|---|
| **동작** | 완료 큐에 도착한 응답의 콜백만 즉시 처리 | 버퍼의 **모든** 메시지 전송 완료까지 대기 |
| **블로킹** | 안 함 | 함 (모두 완료될 때까지) |
| **용도** | 루프 중간에 콜백 큐 비우기 | 프로그램 종료 전, 트랜잭션 커밋 전 등 전송 보장이 필요한 시점 |

### 매 메시지마다 flush()하는 패턴

```python
# 건별 flush 패턴 — 사실상 동기 전송
producer.produce(topic, value=message)
producer.flush()
```

**단점**: 배치가 무력화되어 메시지 1건마다 네트워크 왕복(RTT)이 발생하므로 처리량이 낮다.

**장점**:
- `flush()` 반환 시점에 해당 메시지의 전송 성공/실패가 확정됨
- API 요청 → 처리 → Kafka 전송 → 응답 같은 요청-응답 흐름에서 "보냈음"을 보장한 뒤 다음 단계로 진행 가능
- 버퍼에 메시지가 남아 프로세스 종료 시 유실되는 걱정이 없음
- 구현이 단순하고 에러 핸들링이 직관적

### 처리량별 권장 전략

| 초당 처리량 | 권장 방식 | 이유 |
|------------|----------|------|
| ~수십 건 | 건별 `flush()` 가능 | RTT 누적이 미미하여 성능 차이 체감 없음. 단순함과 전송 보장이 더 가치 있음 |
| 수백~수천 건 | 배치 전송 + 주기적 `poll(0)` | 배치 효과가 유의미해지는 구간. `linger.ms`, `batch.size` 튜닝 필요 |
| 수만 건 이상 | 배치 + 압축 + 비동기 | 반드시 배치/압축 최적화 필요. 건별 flush는 병목 |

> **실무 요약**: 초당 수십 건 이하의 서비스(예: 결과 1건씩 Kafka로 전송하는 API)에서 건별 `flush()`는 오버엔지니어링 없이 안전한 선택이다. 처리량이 병목이 되는 시점에 배치 방식으로 전환하면 된다.

# 컨슈머 기본 동작과 예시

## 컨슈머 디자인

컨슈머는 브로커로부터 메시지를 가져오기까지 내부적으로 여러 단계를 거친다. 프로듀서와 달리 컨슈머는 **pull 방식**으로 동작한다. 즉, 브로커가 메시지를 밀어주는 것이 아니라 컨슈머가 주기적으로 브로커에 요청하여 메시지를 가져온다.

```
┌─────────────────────────────────────────────────────────────────┐
│  Consumer 애플리케이션                                            │
│                                                                 │
│  poll(timeout)                                                  │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────────┐                                           │
│  │  Coordinator 통신  │  ← 그룹 가입, 파티션 할당, 하트비트          │
│  └──────┬───────────┘                                           │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────────┐                                           │
│  │  Fetcher          │  → 할당된 파티션의 리더 브로커에              │
│  │  (데이터 요청)     │    Fetch 요청 전송                         │
│  └──────┬───────────┘                                           │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────────┐                                           │
│  │  Deserializer     │  Key/Value 역직렬화                       │
│  │  (Key + Value)    │  실패 시 SerializationException            │
│  └──────┬───────────┘                                           │
│         │                                                       │
│         ▼                                                       │
│  ConsumerRecords 반환                                            │
│  (토픽, 파티션, 오프셋, 키, 값, 타임스탬프)                         │
│         │                                                       │
│         ▼                                                       │
│  애플리케이션 로직 처리                                             │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────────┐                                           │
│  │  Offset Commit    │  → 처리 완료된 오프셋을 브로커에 커밋          │
│  │                   │    (__consumer_offsets 토픽에 저장)          │
│  └──────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘
```

### 핵심 포인트
- 컨슈머는 **pull 모델**이다. `poll()`을 호출해야 브로커에서 메시지를 가져온다.
- `poll()`은 데이터 가져오기뿐 아니라 **하트비트 전송**, **리밸런스 참여**, **오프셋 커밋(자동 커밋 시)** 등 컨슈머의 모든 백그라운드 작업을 트리거한다.
- 컨슈머 그룹 내에서 파티션 할당은 **Group Coordinator**(브로커 중 하나)가 관리한다.
- `poll()` 호출 간격이 `max.poll.interval.ms`를 초과하면 컨슈머가 죽은 것으로 간주되어 리밸런스가 발생한다.

## 컨슈머 그룹과 파티션 할당

### 컨슈머 그룹 (Consumer Group)
- 같은 `group.id`를 가진 컨슈머들은 하나의 **컨슈머 그룹**을 형성한다.
- 그룹 내 각 파티션은 **단 하나의 컨슈머**에게만 할당된다 → 메시지 중복 소비 방지.
- 서로 다른 컨슈머 그룹은 **독립적으로** 같은 토픽을 소비할 수 있다 → pub/sub 패턴.

```
예시) 토픽 "orders" - 파티션 3개

컨슈머 그룹 A (group.id = "order-processor")
  컨슈머 A-1 ← 파티션 0, 파티션 1
  컨슈머 A-2 ← 파티션 2

컨슈머 그룹 B (group.id = "order-analytics")
  컨슈머 B-1 ← 파티션 0, 파티션 1, 파티션 2

→ 그룹 A와 B는 독립적으로 모든 메시지를 소비
→ 그룹 내에서는 파티션을 나누어 병렬 처리
```

### 파티션 할당 전략

| 전략 | 설명 |
|------|------|
| `RangeAssignor` | 토픽별로 파티션을 연속 범위로 나누어 할당. 토픽이 많으면 앞쪽 컨슈머에 편중될 수 있음 |
| `RoundRobinAssignor` | 모든 파티션을 라운드 로빈으로 균등 배분 |
| `StickyAssignor` | 기존 할당을 최대한 유지하면서 균등하게 재배분. 리밸런스 비용 최소화 |
| `CooperativeStickyAssignor` | Sticky + 점진적(incremental) 리밸런스. 전체 파티션을 한꺼번에 해제하지 않고 변경이 필요한 파티션만 이동 |

> **Kafka 4.0 새 리밸런스 프로토콜(KIP-848)**: 파티션 할당 로직이 브로커(Group Coordinator)로 이동하여 클라이언트 측 Assignor 설정이 불필요해지고, Stop-the-World 없이 더 부드럽게 리밸런스가 수행된다.

### 컨슈머 수와 파티션 수의 관계

```
파티션 3개인 토픽에 대해:

컨슈머 1개:  C1 ← P0, P1, P2          (1개가 모두 담당)
컨슈머 2개:  C1 ← P0, P1 / C2 ← P2   (불균등하지만 정상)
컨슈머 3개:  C1 ← P0 / C2 ← P1 / C3 ← P2  (이상적)
컨슈머 4개:  C1 ← P0 / C2 ← P1 / C3 ← P2 / C4 ← (유휴!)

→ 컨슈머 수가 파티션 수를 초과하면 초과된 컨슈머는 놀게 된다
→ 병렬 처리를 늘리려면 파티션 수를 먼저 늘려야 한다
```

## 컨슈머 주요 옵션

| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `bootstrap.servers` | - | 브로커 접속 주소 (host:port). 최소 2개 이상 권장 |
| `group.id` | - | 컨슈머 그룹 ID. 같은 ID를 가진 컨슈머끼리 파티션을 나누어 소비 |
| `key.deserializer` | - | 메시지 키 역직렬화 클래스 |
| `value.deserializer` | - | 메시지 값 역직렬화 클래스 |
| `auto.offset.reset` | `latest` | 커밋된 오프셋이 없을 때 시작 위치. `earliest`, `latest`, `none` |
| `enable.auto.commit` | `true` | 오프셋 자동 커밋 여부 |
| `auto.commit.interval.ms` | 5000 (5초) | 자동 커밋 주기 |
| `max.poll.records` | 500 | `poll()` 한 번에 반환할 최대 레코드 수 |
| `max.poll.interval.ms` | 300000 (5분) | `poll()` 호출 간 최대 허용 간격. 초과 시 리밸런스 발생 |
| `session.timeout.ms` | 45000 (45초) | 하트비트가 이 시간 내에 도착하지 않으면 컨슈머가 죽은 것으로 판단 |
| `heartbeat.interval.ms` | 3000 (3초) | 하트비트 전송 주기. `session.timeout.ms`의 1/3 이하 권장 |
| `fetch.min.bytes` | 1 | Fetch 요청 시 브로커가 최소 이 크기만큼 데이터가 모일 때까지 대기 |
| `fetch.max.wait.ms` | 500 | `fetch.min.bytes`가 충족되지 않아도 이 시간이 지나면 응답 |
| `max.partition.fetch.bytes` | 1048576 (1MB) | 파티션당 Fetch 응답 최대 크기 |
| `isolation.level` | `read_uncommitted` | 트랜잭션 메시지 읽기 수준. `read_committed`면 커밋된 메시지만 소비 |

## 오프셋 커밋 전략

컨슈머가 어디까지 읽었는지를 기록하는 것이 **오프셋 커밋**이다. 커밋 전략에 따라 **메시지 유실**과 **중복 소비**의 트레이드오프가 달라진다.

### 자동 커밋 (Auto Commit)

```python
# enable.auto.commit=True (기본값)
# auto.commit.interval.ms=5000 (기본값)
# → poll() 호출 시 이전 poll()에서 받은 오프셋을 자동으로 커밋
```

- **장점**: 코드가 단순하다.
- **단점**: `poll()`로 메시지를 받은 후 처리 완료 전에 커밋이 될 수 있다. 이 경우 장애 발생 시 **메시지 유실** 가능.

### 수동 커밋 - 동기 (commitSync)

```python
consumer.commit()  # 동기 커밋. 커밋 완료까지 블로킹
```

- **장점**: 커밋 성공을 확인할 수 있어 안전하다.
- **단점**: 커밋 요청마다 브로커 응답을 기다리므로 처리량이 낮아질 수 있다.

### 수동 커밋 - 비동기 (commitAsync)

```python
consumer.commit(asynchronous=True)  # 비동기 커밋. 즉시 반환
```

- **장점**: 블로킹이 없어 처리량이 높다.
- **단점**: 커밋 실패 시 자동 재시도하지 않는다 (재시도 시 순서 문제 발생 가능). 보통 종료 시 마지막에 `commitSync()`를 한 번 호출하는 패턴을 사용한다.

### 오프셋 커밋과 메시지 처리의 관계

```
시나리오 1: 커밋 후 처리 (At-Most-Once)
  poll() → commit() → process()
  → 처리 전 장애 시 메시지 유실 (이미 커밋했으므로 다시 안 읽음)

시나리오 2: 처리 후 커밋 (At-Least-Once) ← 가장 일반적
  poll() → process() → commit()
  → 처리 후 커밋 전 장애 시 메시지 중복 소비 (같은 오프셋부터 다시 읽음)
  → 따라서 컨슈머 로직은 멱등성(idempotent)을 갖추는 것이 좋다

시나리오 3: Exactly-Once
  → 트랜잭션 프로듀서 + read_committed 컨슈머 조합으로 구현
  → 또는 외부 저장소에 오프셋과 처리 결과를 원자적으로 저장
```

## 컨슈머 예제 (Python, confluent-kafka)

### 기본 컨슈머

```python
from confluent_kafka import Consumer

conf = {
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'order-processor',
    'auto.offset.reset': 'earliest',
    'enable.auto.commit': True,
}

consumer = Consumer(conf)
consumer.subscribe(['orders'])

try:
    while True:
        msg = consumer.poll(timeout=1.0)  # 최대 1초 대기

        if msg is None:
            continue  # 메시지 없음

        if msg.error():
            print(f'컨슈머 에러: {msg.error()}')
            continue

        print(f'수신: topic={msg.topic()}, partition={msg.partition()}, '
              f'offset={msg.offset()}, key={msg.key()}, '
              f'value={msg.value().decode("utf-8")}')

except KeyboardInterrupt:
    pass
finally:
    consumer.close()  # 그룹 탈퇴 + 오프셋 커밋 + 리소스 정리
```

### 수동 커밋 컨슈머

```python
from confluent_kafka import Consumer

conf = {
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'order-processor',
    'auto.offset.reset': 'earliest',
    'enable.auto.commit': False,  # 자동 커밋 비활성화
}

consumer = Consumer(conf)
consumer.subscribe(['orders'])

try:
    while True:
        msg = consumer.poll(timeout=1.0)

        if msg is None:
            continue

        if msg.error():
            print(f'컨슈머 에러: {msg.error()}')
            continue

        # 메시지 처리
        print(f'처리 중: offset={msg.offset()}, '
              f'value={msg.value().decode("utf-8")}')

        # 처리 완료 후 수동 커밋 (At-Least-Once)
        consumer.commit(message=msg)

except KeyboardInterrupt:
    pass
finally:
    consumer.close()
```

### 배치 처리 + 비동기 커밋 컨슈머

```python
from confluent_kafka import Consumer

conf = {
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'order-batch-processor',
    'auto.offset.reset': 'earliest',
    'enable.auto.commit': False,
    'max.poll.records': 100,  # 배치 크기 제어
}

consumer = Consumer(conf)
consumer.subscribe(['orders'])

try:
    while True:
        # 여러 메시지를 한 번에 가져오기
        messages = consumer.consume(num_messages=100, timeout=1.0)

        if not messages:
            continue

        for msg in messages:
            if msg.error():
                print(f'에러: {msg.error()}')
                continue

            # 메시지 처리
            print(f'처리: partition={msg.partition()}, '
                  f'offset={msg.offset()}')

        # 배치 처리 완료 후 비동기 커밋
        consumer.commit(asynchronous=True)

except KeyboardInterrupt:
    pass
finally:
    # 종료 전 마지막 동기 커밋으로 유실 방지
    consumer.commit()
    consumer.close()
```

### 특정 파티션/오프셋에서 소비 (assign 방식)

```python
from confluent_kafka import Consumer, TopicPartition

conf = {
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'manual-assign-consumer',
    'enable.auto.commit': False,
}

consumer = Consumer(conf)

# subscribe() 대신 assign()으로 특정 파티션을 직접 지정
consumer.assign([
    TopicPartition('orders', partition=0, offset=100),  # 파티션 0의 오프셋 100부터
    TopicPartition('orders', partition=1, offset=0),    # 파티션 1의 처음부터
])

try:
    while True:
        msg = consumer.poll(timeout=1.0)

        if msg is None:
            continue
        if msg.error():
            print(f'에러: {msg.error()}')
            continue

        print(f'파티션={msg.partition()}, 오프셋={msg.offset()}, '
              f'값={msg.value().decode("utf-8")}')

except KeyboardInterrupt:
    pass
finally:
    consumer.close()
```

> **assign vs subscribe**: `subscribe()`는 컨슈머 그룹을 통해 자동으로 파티션이 할당되고 리밸런스에 참여한다. `assign()`은 그룹 조정 없이 특정 파티션을 직접 지정하며, 리밸런스가 발생하지 않는다. 리플레이, 특정 오프셋 조회 등 특수한 용도에 사용한다.

## 실무 가이드: poll() 루프 설계

### poll() 타임아웃의 의미

`poll(timeout)`의 타임아웃은 **Fetch 응답이 없을 때 최대 대기 시간**이다.

- `poll(0)`: 즉시 반환. 이미 내부 버퍼에 있는 데이터만 반환. CPU를 많이 사용할 수 있다.
- `poll(1.0)`: 데이터가 없으면 최대 1초 대기. 일반적으로 권장되는 값.
- `poll(5.0)`: 대기 시간이 길어 메시지가 적을 때 CPU를 절약하지만, 종료 반응이 느려진다.

### 주의: poll() 호출 간격과 리밸런스

```
poll() 호출 간격이 max.poll.interval.ms(기본 5분)를 초과하면:
  → Group Coordinator가 해당 컨슈머를 죽은 것으로 판단
  → 리밸런스 발생 → 파티션 재할당
  → 해당 컨슈머는 CommitFailedException 발생

대응 방법:
  1. max.poll.records를 줄여 poll() 당 처리 시간 단축
  2. max.poll.interval.ms를 처리 시간에 맞게 늘림
  3. 메시지 처리 로직을 별도 스레드로 분리 (주의: 오프셋 관리 복잡)
```

### consumer.close()의 중요성

`close()`는 단순 리소스 정리가 아니라, 그룹에서 **즉시 탈퇴(LeaveGroup)**를 요청한다. 이를 통해 `session.timeout.ms`를 기다리지 않고 바로 리밸런스가 시작되어 다른 컨슈머가 파티션을 빠르게 인계받는다.

```python
# 올바른 패턴
try:
    while True:
        msg = consumer.poll(1.0)
        # 처리 로직
except KeyboardInterrupt:
    pass
finally:
    consumer.close()  # 반드시 호출!

# close()를 호출하지 않으면:
# → session.timeout.ms(기본 45초) 동안 해당 컨슈머의 파티션이 방치됨
# → 그 시간 동안 해당 파티션의 메시지가 처리되지 않음
```

### 처리량별 권장 전략

| 초당 처리량 | 권장 방식 | 이유 |
|------------|----------|------|
| ~수십 건 | 자동 커밋 + 단건 처리 | 단순하고 충분히 빠름. 중복 허용 가능한 경우 적합 |
| 수백~수천 건 | 수동 커밋 + 배치 처리 | `consume(num_messages=N)` + 배치 완료 후 커밋으로 효율 향상 |
| 수만 건 이상 | 배치 + 비동기 커밋 + 멀티 컨슈머 | 파티션 수를 늘리고 컨슈머를 스케일 아웃. 멱등 처리 필수 |

> **실무 요약**: 대부분의 서비스에서는 `enable.auto.commit=False` + 처리 후 `commit()` (At-Least-Once) 패턴이 안전하고 충분하다. 메시지 처리 로직을 멱등하게 구현하면 중복 소비에도 문제가 없다.
