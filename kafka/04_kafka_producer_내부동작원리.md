- kafka v4 기준 설명

# 카프카 프로듀서 내부 동작 원리

## 목차

- [프로듀서 전체 흐름](#프로듀서-전체-흐름)
- [파티셔너](#파티셔너)
  - [키가 있는 경우](#키가-있는-경우)
  - [키가 없는 경우 - 라운드 로빈 (구버전 방식)](#키가-없는-경우---라운드-로빈-구버전-방식)
  - [키가 없는 경우 - 스티키 파티셔닝 (Kafka 2.4+)](#키가-없는-경우---스티키-파티셔닝-kafka-24)
- [프로듀서의 배치](#프로듀서의-배치)
  - [RecordAccumulator](#recordaccumulator)
  - [배치 전송 조건](#배치-전송-조건)
  - [압축](#압축)
- [Sender Thread](#sender-thread)
  - [Sender Thread의 동작 루프](#sender-thread의-동작-루프)
  - [브로커별 요청 묶음 (drain)](#브로커별-요청-묶음-drain)
  - [In-flight 요청 관리](#in-flight-요청-관리)
  - [재시도 (Retry) 메커니즘](#재시도-retry-메커니즘)
  - [메타데이터 갱신](#메타데이터-갱신)
- [중복 없는 전송 (멱등성 프로듀서)](#중복-없는-전송-멱등성-프로듀서)
- [정확히 한 번 전송 (Exactly-Once / 트랜잭션)](#정확히-한-번-전송-exactly-once--트랜잭션)

---

## 프로듀서 전체 흐름

프로듀서가 메시지를 Kafka 브로커로 전송하는 과정은 단순한 네트워크 호출이 아니라, 여러 단계를 거치는 파이프라인이다.

```
프로듀서 애플리케이션
        │
        │ producer.send(ProducerRecord)
        ▼
┌─────────────────────────────────────────────────────────────┐
│                      프로듀서 내부                            │
│                                                             │
│  1. Serializer       2. Partitioner      3. RecordAccumulator│
│  ┌──────────┐       ┌──────────────┐    ┌────────────────┐  │
│  │key/value │──────▶│  파티션 결정  │───▶│  배치 버퍼      │  │
│  │직렬화     │       │  (0, 1, 2...)│    │  (파티션별)     │  │
│  └──────────┘       └──────────────┘    └───────┬────────┘  │
│                                                 │           │
│                                    4. Sender Thread         │
│                                         ┌───────▼────────┐  │
│                                         │ 배치 전송 결정   │  │
│                                         │ (batch.size /  │  │
│                                         │  linger.ms)    │  │
│                                         └───────┬────────┘  │
└─────────────────────────────────────────────────┼───────────┘
                                                  │
                                    5. NetworkClient (TCP)
                                                  │
                                                  ▼
                                           카프카 브로커
                                       (파티션 리더에 저장)
```

### 처리 단계 요약

| 단계 | 구성요소 | 역할 |
|------|---------|------|
| 1 | Serializer | ProducerRecord의 key/value를 byte[]로 변환 |
| 2 | Partitioner | 메시지를 저장할 파티션 번호 결정 |
| 3 | RecordAccumulator | 파티션별로 메시지를 배치로 묶어 메모리에 버퍼링 |
| 4 | Sender Thread | 배치 조건이 충족되면 브로커로 전송 |
| 5 | NetworkClient | TCP 통신으로 브로커에 실제 전달, 응답 처리 |

---

# 파티셔너

파티셔너는 **메시지를 어느 파티션에 저장할지 결정**하는 컴포넌트이다. 파티션 결정 방식에 따라 처리량과 순서 보장 수준이 달라진다.

## 키가 있는 경우

키가 지정된 메시지는 `hash(key) % numPartitions` 공식으로 파티션이 결정된다.

```
key = "user-123"
hash("user-123") % 3 = 1  → 파티션 1에 저장

→ 같은 키를 가진 메시지는 항상 같은 파티션에 저장됨
→ 파티션 내에서 순서가 보장됨 (e.g., 특정 사용자 이벤트 순서 보장)

주의: 파티션 수가 변경되면 같은 키가 다른 파티션으로 이동할 수 있다.
```

## 키가 없는 경우 - 라운드 로빈 (구버전 방식)

Kafka 2.3 이하의 기본 파티셔너는 키가 없으면 **라운드 로빈** 방식으로 파티션을 순환 배정한다.

```
메시지 1 → 파티션 0
메시지 2 → 파티션 1
메시지 3 → 파티션 2
메시지 4 → 파티션 0
...

문제점:
  메시지 1개 → 파티션 0 배치
  메시지 1개 → 파티션 1 배치
  메시지 1개 → 파티션 2 배치

→ 배치 크기가 1개짜리로 쪼개져 배치 효율이 매우 낮음
→ linger.ms=0이면 모든 메시지가 개별 요청으로 전송됨
→ 브로커에 요청 수가 급격히 증가
```

## 키가 없는 경우 - 스티키 파티셔닝 (Kafka 2.4+)

**스티키 파티셔닝(Sticky Partitioning)**은 배치가 가득 차거나 전송될 때까지 **하나의 파티션에 메시지를 계속 쌓는** 방식이다. Kafka 2.4부터 도입되어 Kafka 3.3부터는 기본 동작이 되었다.

```
[라운드 로빈]                  [스티키 파티셔닝]

메시지 m1 → 파티션 0 (배치: m1)   메시지 m1 → 파티션 0 (배치: m1)
메시지 m2 → 파티션 1 (배치: m2)   메시지 m2 → 파티션 0 (배치: m1, m2)
메시지 m3 → 파티션 2 (배치: m3)   메시지 m3 → 파티션 0 (배치: m1, m2, m3)
메시지 m4 → 파티션 0 (배치: m4)   배치 전송! → 다음 파티션으로 전환
                                  메시지 m4 → 파티션 1 (배치: m4)
                                  메시지 m5 → 파티션 1 (배치: m4, m5)

→ 배치 크기가 커져 브로커 요청 수 감소
→ 처리량(throughput) 대폭 향상
→ 전환 시점: batch.size 도달 또는 linger.ms 만료
```

### 파티셔닝 방식 비교

| 방식 | 적용 시점 | 배치 효율 | 순서 보장 | 균등 분산 |
|------|---------|---------|---------|---------|
| 키 기반 해시 | 키 있음 | 높음 | 파티션 내 보장 | 키 분포에 의존 |
| 라운드 로빈 | 키 없음 (구버전) | 낮음 | 보장 안 됨 | 균등 |
| 스티키 파티셔닝 | 키 없음 (2.4+) | 높음 | 보장 안 됨 | 장기적으로 균등 |

> **커스텀 파티셔너**: `Partitioner` 인터페이스를 구현하여 비즈니스 로직에 맞는 파티션 배정 규칙을 직접 정의할 수 있다. 예를 들어 VIP 사용자 요청을 특정 파티션으로 라우팅하는 등의 용도로 사용된다.

---

# 프로듀서의 배치

## RecordAccumulator

파티셔너가 파티션을 결정하면 메시지는 즉시 전송되지 않고, **RecordAccumulator**라는 메모리 버퍼에 파티션별로 누적된다.

```
RecordAccumulator 내부 구조:

파티션 0: [RecordBatch] → [RecordBatch] → [RecordBatch(현재 쌓는 중)]
파티션 1: [RecordBatch] → [RecordBatch(현재 쌓는 중)]
파티션 2: [RecordBatch(현재 쌓는 중)]

→ 각 파티션마다 Deque(양방향 큐)로 배치를 관리
→ 새 메시지는 맨 뒤 배치에 추가
→ Sender Thread는 맨 앞 배치부터 전송
```

## 배치 전송 조건

Sender Thread는 다음 두 조건 중 하나가 충족되면 배치를 브로커로 전송한다.

```
조건 1: batch.size 초과
  → 배치 크기가 설정값에 도달하면 즉시 전송
  → 기본값: 16,384 bytes (16KB)

조건 2: linger.ms 만료
  → batch.size 미달이더라도 대기 시간이 지나면 전송
  → 기본값: 0ms (즉시 전송)

배치가 전송되는 시나리오:

linger.ms=0 (기본):
  메시지 도착 → 즉시 전송 (배치 효과 없음)

linger.ms=5:
  메시지 도착 후 최대 5ms 대기 → 그 사이 도착한 메시지 모두 묶어 전송
  → 약간의 지연을 허용하되 처리량을 높이는 전략
```

## 주요 배치 관련 설정

| 설정 | 기본값 | 설명 |
|------|--------|------|
| `batch.size` | 16384 (16KB) | 배치 최대 크기. 초과 시 즉시 전송 |
| `linger.ms` | 0 | 배치 대기 시간. 이 시간 동안 추가 메시지를 기다림 |
| `buffer.memory` | 33554432 (32MB) | RecordAccumulator 전체 메모리 크기 |
| `max.block.ms` | 60000 (60초) | 버퍼가 꽉 찼을 때 send() 블로킹 최대 시간 |
| `compression.type` | none | 배치 압축 방식 (none, gzip, snappy, lz4, zstd) |

## 압축

압축은 **배치 단위**로 적용된다. 배치가 클수록 압축률이 높아지므로, 스티키 파티셔닝 + linger.ms > 0 + 압축을 함께 사용하면 시너지 효과가 크다.

```
압축 방식 비교:

| 압축 | CPU 사용 | 압축률 | 권장 상황 |
|------|---------|--------|---------|
| none | 없음 | - | 네트워크 충분, 단순 환경 |
| snappy | 낮음 | 중간 | 범용 (처리량 우선) |
| lz4 | 낮음 | 중간 | 저지연 우선 |
| gzip | 높음 | 높음 | 네트워크 비용 절감 우선 |
| zstd | 중간 | 높음 | 처리량 + 압축률 균형 |

→ 압축/해제는 프로듀서/컨슈머에서 수행 (브로커는 압축된 상태로 저장)
→ 단, 브로커가 메시지 검증/재압축이 필요한 경우 브로커에서도 CPU 사용
```

---

# Sender Thread

Sender Thread는 프로듀서 내부에서 **메인 스레드와 분리된 백그라운드 스레드**로 동작한다. 메인 스레드가 `send()`를 호출해 RecordAccumulator에 메시지를 쌓는 동안, Sender Thread는 독립적으로 실행되며 배치를 꺼내 브로커로 전송한다.

```
메인 스레드                         Sender Thread (백그라운드)
─────────────────────              ────────────────────────────────────
producer.send(record1)  ──쌓음──▶  ① ready()  : 전송 준비된 파티션 확인
producer.send(record2)  ──쌓음──▶  ② drain()  : 배치를 RecordAccumulator에서 꺼냄
producer.send(record3)  ──쌓음──▶  ③ send()   : 브로커별로 ProduceRequest 전송
                                   ④ poll()   : 응답 수신 및 콜백 처리
                                   ↑ 위 과정을 무한 루프로 반복
```

## Sender Thread의 동작 루프

Sender Thread는 내부적으로 `run()` 루프를 실행하며 다음 4단계를 반복한다.

```
[루프 시작]
    │
    ▼
① ready() ── RecordAccumulator를 조회하여 전송 가능한 파티션 목록 수집
    │
    │  전송 가능 조건 (OR):
    │  - 배치 크기가 batch.size 이상
    │  - linger.ms 경과
    │  - RecordAccumulator 버퍼가 꽉 참 (buffer.memory 초과)
    │  - flush() 또는 close() 호출됨
    │  - max.in.flight 여유 있음
    │
    ▼
② drain() ── 전송 가능한 파티션의 배치를 꺼냄 (브로커별로 묶음)
    │
    │  파티션 0 (리더: 브로커1), 파티션 3 (리더: 브로커1)
    │    → 브로커1에게 보낼 ProduceRequest 하나로 묶음
    │  파티션 1 (리더: 브로커2)
    │    → 브로커2에게 보낼 ProduceRequest
    │
    ▼
③ sendProduceRequests() ── NetworkClient를 통해 각 브로커로 요청 전송
    │
    ▼
④ NetworkClient.poll() ── 응답 수신 대기 및 처리
    │
    │  성공: 콜백 실행, 배치 해제
    │  실패: 재시도 가능 에러 → 배치를 RecordAccumulator에 다시 넣음
    │        재시도 불가 에러 → 콜백에 예외 전달
    │
    └── [루프 시작으로]
```

## 브로커별 요청 묶음 (drain)

Sender Thread는 파티션별이 아닌 **브로커별**로 요청을 묶어 전송한다. 이를 통해 네트워크 요청 수를 최소화한다.

```
예시) 파티션 6개, 브로커 3개

파티션 0 → 브로커1 (리더)    ┐
파티션 1 → 브로커2 (리더)    │
파티션 2 → 브로커3 (리더)    │  drain() 결과
파티션 3 → 브로커1 (리더)    │
파티션 4 → 브로커2 (리더)    │
파티션 5 → 브로커3 (리더)    ┘

→ ProduceRequest 3개 (브로커당 1개)
  브로커1: 파티션0 배치 + 파티션3 배치
  브로커2: 파티션1 배치 + 파티션4 배치
  브로커3: 파티션2 배치 + 파티션5 배치

→ 파티션 수가 늘어나도 브로커 수만큼의 요청만 발생
```

## In-flight 요청 관리

**In-flight 요청**이란 브로커에 전송했지만 아직 ACK를 받지 못한 요청이다. `max.in.flight.requests.per.connection`으로 브로커 연결당 동시 전송 가능한 요청 수를 제한한다.

```
max.in.flight.requests.per.connection = 3 인 경우:

시간 →
요청1 ──────────────────────────▶ [브로커] → ACK1 ──▶
요청2 ──────────────────▶        [브로커] → ACK2 ──▶
요청3 ──────▶                    [브로커] → ACK3 ──▶
요청4 (대기) ← in-flight 3개 꽉 참, ACK 받을 때까지 대기

→ in-flight이 클수록 처리량 증가
→ in-flight이 클수록 장애 시 재전송 범위 증가
```

### 순서 역전 문제

`max.in.flight.requests.per.connection > 1`이면 재시도 시 메시지 순서가 역전될 수 있다.

```
순서 역전 시나리오 (max.in.flight=2):

요청A (offset 0, 1) ──▶ 브로커 (실패 → 재시도 대기)
요청B (offset 2, 3) ──▶ 브로커 (성공 → offset 2, 3 저장)
요청A (재시도)       ──▶ 브로커 (성공 → offset 0, 1 저장)

결과: 브로커 로그 = [2, 3, 0, 1] ← 순서 역전!

해결:
  방법1: max.in.flight.requests.per.connection=1 → 순서 보장, 처리량 감소
  방법2: enable.idempotence=true → max.in.flight≤5에서도 시퀀스 번호로 순서 보장
```

## 재시도 (Retry) 메커니즘

전송 실패 시 Sender Thread는 에러 종류에 따라 재시도 여부를 결정한다.

```
에러 분류:

재시도 가능 (Retriable) 에러:
  - LEADER_NOT_AVAILABLE  → 리더 선출 중, 잠시 후 재시도
  - NOT_LEADER_FOR_PARTITION → 메타데이터 갱신 후 재시도
  - REQUEST_TIMEOUT       → 네트워크 지연, 재시도
  - NETWORK_EXCEPTION     → 연결 장애, 재시도

재시도 불가 (Non-retriable) 에러:
  - INVALID_TOPIC         → 토픽 이름 오류, 재시도해도 동일
  - MESSAGE_TOO_LARGE     → 메시지 크기 초과
  - UNKNOWN_PRODUCER_ID  → 트랜잭션 ID 불일치
  → 콜백에 예외 전달, 재시도 없음

재시도 흐름:
  실패 배치 → RetryQueue에 다시 삽입
           → retry.backoff.ms(기본 100ms) 대기
           → 재전송 시도
           → retries 횟수 초과 시 최종 실패 처리
```

### 재시도 관련 설정

| 설정 | 기본값 | 설명 |
|------|--------|------|
| `retries` | Integer.MAX_VALUE | 재시도 최대 횟수 (멱등성 활성화 시 자동 설정) |
| `retry.backoff.ms` | 100 | 재시도 전 대기 시간 |
| `retry.backoff.max.ms` | 1000 | 재시도 대기 시간 상한 (지수 백오프 적용 시) |
| `delivery.timeout.ms` | 120000 (2분) | send() 호출부터 최종 성공/실패까지 총 허용 시간 |
| `request.timeout.ms` | 30000 (30초) | 브로커 응답 대기 최대 시간 |

```
delivery.timeout.ms 의 역할:

send() 호출
    │
    ├── linger.ms 대기
    ├── 전송
    ├── request.timeout.ms 대기
    ├── 실패 시 retry.backoff.ms 대기 후 재전송
    ├── ...반복...
    │
    └── delivery.timeout.ms 초과 → TimeoutException 발생

→ retries 횟수가 아무리 많아도 delivery.timeout.ms를 넘으면 실패 처리
→ 실질적인 "포기 시한"은 delivery.timeout.ms로 제어하는 것이 권장됨
```

## 메타데이터 갱신

Sender Thread는 브로커로부터 **클러스터 메타데이터**(토픽, 파티션, 리더 브로커 정보)를 주기적으로 갱신한다.

```
메타데이터가 필요한 상황:
  - 프로듀서 최초 초기화 시
  - LEADER_NOT_AVAILABLE 에러 수신 시 (리더 변경 감지)
  - metadata.max.age.ms 경과 시 (주기적 갱신)

갱신 흐름:
  Sender Thread → 임의의 브로커에 MetadataRequest 전송
               ← 모든 토픽/파티션/리더 정보 응답
  → 메타데이터 캐시 갱신
  → 이후 요청은 최신 리더에게 전송

리더 변경 시 자동 복구:
  브로커1 장애 → 파티션0의 리더가 브로커2로 변경
  프로듀서: LEADER_NOT_AVAILABLE 수신
         → 메타데이터 갱신
         → 이후 파티션0 요청을 브로커2로 전송 (자동)
```

| 설정 | 기본값 | 설명 |
|------|--------|------|
| `metadata.max.age.ms` | 300000 (5분) | 강제 메타데이터 갱신 주기 |
| `metadata.max.idle.ms` | 300000 (5분) | 사용하지 않는 토픽의 메타데이터 캐시 유지 시간 |

---

# 중복 없는 전송 (멱등성 프로듀서)

## 기본 전송의 중복 문제

기본 설정(`enable.idempotence=false`)에서 네트워크 장애 발생 시 메시지가 중복될 수 있다.

```
중복 발생 시나리오:

1. 프로듀서 → 브로커: 메시지 m1 전송
2. 브로커: m1을 디스크에 저장
3. 브로커 → 프로듀서: ACK 전송
4. 네트워크 장애로 ACK 유실
5. 프로듀서: ACK 미수신 → 타임아웃 후 m1 재전송
6. 브로커: 이미 저장된 m1이 또 저장됨 → 중복!

결과: m1이 2번 저장됨
```

## 멱등성 프로듀서 (enable.idempotence=true)

Kafka 3.0부터 `enable.idempotence`의 기본값이 `true`로 변경되었다. 멱등성이 활성화되면 프로듀서는 **PID(Producer ID)**와 **시퀀스 번호**를 이용해 브로커가 중복을 감지하고 무시하도록 한다.

```
멱등성 프로듀서의 중복 방지:

프로듀서 초기화 시:
  → 브로커로부터 PID (Producer ID) 할당받음

메시지 전송 시:
  ProducerRecord + PID=3, Seq=0 → 브로커 저장 (seq 0 처음 도착)
  ProducerRecord + PID=3, Seq=1 → 브로커 저장
  ProducerRecord + PID=3, Seq=2 → 브로커 저장
  ...

ACK 유실로 인한 재전송:
  ProducerRecord + PID=3, Seq=1 → 브로커: "Seq=1은 이미 받았음" → 무시!

→ 브로커는 파티션별로 각 PID의 마지막 시퀀스 번호를 추적
→ 중복 메시지는 저장하지 않고 ACK만 재전송
```

### 멱등성 보장 범위

```
보장됨:
  → 단일 프로듀서 세션 내에서 단일 파티션에 대한 중복 제거

보장 안 됨:
  → 프로듀서가 재시작되면 PID가 새로 발급됨 (세션 간 중복 가능)
  → 여러 파티션/토픽에 걸친 원자적 쓰기 (→ 트랜잭션으로 해결)
```

### 멱등성 관련 설정

| 설정 | 값 | 설명 |
|------|-----|------|
| `enable.idempotence` | true (기본) | 멱등성 활성화 |
| `max.in.flight.requests.per.connection` | 5 | 응답 대기 중인 요청 수 상한 (멱등성 활성화 시 자동으로 ≤5 강제) |
| `retries` | Integer.MAX_VALUE | 멱등성 활성화 시 자동 설정 |
| `acks` | all | 멱등성 활성화 시 자동 설정 |

---

# 정확히 한 번 전송 (Exactly-Once / 트랜잭션)

## 멱등성만으로 부족한 경우

멱등성 프로듀서는 단일 파티션에 대한 중복을 막지만, **여러 파티션/토픽에 원자적으로 쓰는 것**은 보장하지 않는다.

```
문제 시나리오: 입력 토픽 → 처리 → 출력 토픽 (스트림 처리)

1. 컨슈머: 입력 토픽에서 메시지 읽기
2. 프로듀서: 처리 결과를 출력 토픽에 쓰기
3. 컨슈머: 오프셋 커밋

장애 발생 시:
  → 출력 토픽에는 썼지만 오프셋 커밋 전에 장애
  → 재시작 후 입력 메시지를 다시 처리 → 출력 토픽에 중복 기록
  → 멱등성만으로는 해결 불가 (PID가 새로 발급되므로)
```

## 트랜잭션 프로듀서

`transactional.id`를 설정하면 트랜잭션 기능이 활성화된다. 여러 파티션/토픽에 대한 쓰기와 오프셋 커밋을 하나의 원자적 단위로 묶을 수 있다.

```
트랜잭션 사용 코드 패턴:

producer.initTransactions()          // 초기화 (트랜잭션 코디네이터에 등록)

while (true) {
    producer.beginTransaction()      // 트랜잭션 시작

    // 여러 파티션/토픽에 쓰기
    producer.send(record1)           // 토픽A 파티션 0
    producer.send(record2)           // 토픽B 파티션 1

    // 컨슈머 오프셋도 트랜잭션에 포함
    producer.sendOffsetsToTransaction(offsets, consumerGroupMetadata)

    producer.commitTransaction()     // 원자적 커밋
    // 또는
    producer.abortTransaction()      // 롤백
}
```

## 트랜잭션 내부 동작

```
트랜잭션 코디네이터 (브로커 내 특수 컴포넌트):

1. initTransactions()
   → 브로커: transactional.id로 트랜잭션 코디네이터 결정
   → PID 발급 (재시작해도 같은 transactional.id면 동일 PID 재사용)
   → epoch 번호 증가 (이전 프로듀서 세션 무효화)

2. beginTransaction()
   → 프로듀서 내부 상태만 변경 (브로커 통신 없음)

3. send() 호출 시
   → 브로커: 해당 파티션이 진행 중 트랜잭션에 포함됨을 로그에 기록

4. commitTransaction()
   → 트랜잭션 코디네이터에 커밋 요청
   → 코디네이터: __transaction_state 토픽에 COMMIT 마커 기록
   → 모든 관련 파티션에 트랜잭션 완료 마커(COMMIT marker) 기록
   → 컨슈머는 COMMIT 마커를 본 후에야 해당 메시지 읽기 가능

5. abortTransaction()
   → ABORT 마커 기록
   → 관련 메시지는 컨슈머에게 보이지 않음
```

## 트랜잭션과 컨슈머 isolation level

트랜잭션 메시지를 제대로 읽으려면 컨슈머의 `isolation.level` 설정이 필요하다.

```
isolation.level=read_uncommitted (기본값):
  → 커밋되지 않은 트랜잭션 메시지도 읽음
  → Exactly-Once 의미론 보장 안 됨

isolation.level=read_committed:
  → 커밋 완료된 트랜잭션 메시지만 읽음
  → 진행 중인 트랜잭션의 메시지는 COMMIT 마커 도착까지 대기
  → Exactly-Once 의미론 완전히 보장

예시:
  트랜잭션 A: offset 10, 11, 12 쓰기 (커밋 안 됨)
  트랜잭션 B: offset 13, 14 쓰기 (커밋 완료)

  read_uncommitted: offset 10, 11, 12, 13, 14 모두 읽음
  read_committed:   offset 13, 14만 읽음 (10~12는 커밋 대기)
```

## 전달 보장 수준 비교

| 보장 수준 | 설정 | 특징 | 성능 |
|---------|------|------|------|
| At-Most-Once | acks=0, retries=0 | 유실 가능, 중복 없음 | 최고 |
| At-Least-Once | acks=1 또는 all, retries>0 | 유실 없음, 중복 가능 | 중간 |
| Exactly-Once (단일 파티션) | enable.idempotence=true | 유실 없음, 중복 없음 | 약간 낮음 |
| Exactly-Once (멀티 파티션) | transactional.id 설정 | 원자적 쓰기 + 유실/중복 없음 | 가장 낮음 |

### 트랜잭션 관련 주요 설정

| 설정 | 기본값 | 설명 |
|------|--------|------|
| `transactional.id` | 없음 | 설정 시 트랜잭션 모드 활성화. 재시작 간 동일 값 사용 |
| `transaction.timeout.ms` | 60000 (60초) | 트랜잭션 최대 유지 시간. 초과 시 브로커가 강제 ABORT |
| `isolation.level` (컨슈머) | read_uncommitted | `read_committed`로 설정 시 커밋된 메시지만 읽음 |

> **실무 요약**: 단순 유실 방지에는 `acks=all + enable.idempotence=true`로 충분하다. 스트림 처리(Kafka Streams, Flink)처럼 consume-process-produce 패턴이 필요한 경우에만 트랜잭션을 사용한다. 트랜잭션은 성능 오버헤드가 있으므로 꼭 필요한 토픽에만 적용한다.
