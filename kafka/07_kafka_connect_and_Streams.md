- kafka v4 기준 설명

# Kafka Connect & Kafka Streams

## 목차

- [Kafka Connect](#kafka-connect)
  - [Kafka Connect란](#kafka-connect란)
  - [아키텍처](#아키텍처)
  - [Source Connector vs Sink Connector](#source-connector-vs-sink-connector)
  - [Standalone vs Distributed 모드](#standalone-vs-distributed-모드)
  - [Connector, Task, Worker](#connector-task-worker)
  - [SMT (Single Message Transforms)](#smt-single-message-transforms)
  - [REST API로 Connector 관리](#rest-api로-connector-관리)
  - [주요 Connector 목록](#주요-connector-목록)
  - [Kafka Connect 주요 설정](#kafka-connect-주요-설정)
- [Kafka Streams](#kafka-streams)
  - [Kafka Streams란](#kafka-streams란)
  - [아키텍처](#아키텍처-1)
  - [KStream vs KTable vs GlobalKTable](#kstream-vs-ktable-vs-globaltable)
  - [Topology와 처리 파이프라인](#topology와-처리-파이프라인)
  - [Stateless 연산](#stateless-연산)
  - [Stateful 연산](#stateful-연산)
  - [윈도잉 (Windowing)](#윈도잉-windowing)
  - [State Store](#state-store)
  - [Interactive Queries](#interactive-queries)
  - [Exactly-Once 처리](#exactly-once-처리)
  - [Kafka Streams 주요 설정](#kafka-streams-주요-설정)
- [Connect vs Streams 비교](#connect-vs-streams-비교)

---

# Kafka Connect

## Kafka Connect란

Kafka Connect는 **외부 시스템과 Kafka 사이의 데이터 이동을 표준화된 방식으로 처리**하는 프레임워크이다. 데이터베이스, 파일 시스템, 클라우드 스토리지, 검색 엔진 등 다양한 시스템과의 연동을 코드 없이 설정만으로 구성할 수 있다.

```
Kafka Connect의 위치:

외부 시스템                    Kafka                     외부 시스템
(MySQL, PostgreSQL,     ←─────────────────→     (Elasticsearch,
 MongoDB, S3, ...)              │                 S3, BigQuery, ...)
                                │
                      ┌─────────────────┐
                      │  Kafka Connect   │
                      │                 │
                      │  Source         │  Sink
                      │  Connector      │  Connector
                      │  (외부→Kafka)    │  (Kafka→외부)
                      └─────────────────┘
```

### Kafka Connect를 쓰는 이유

```
직접 구현 vs Kafka Connect:

직접 구현:
  - 프로듀서/컨슈머 코드를 직접 작성
  - 오프셋 관리, 재시도, 스키마 변환 직접 처리
  - 외부 시스템 변경 시 코드 수정 필요
  - 운영 모니터링 직접 구축

Kafka Connect:
  - 설정 파일만으로 파이프라인 구성
  - 오프셋 관리, 재시도, 장애 복구 내장
  - REST API로 실시간 관리
  - 표준화된 모니터링 (JMX, REST)
  - 수천 개의 오픈소스 커넥터 활용 가능
```

## 아키텍처

```
Kafka Connect Distributed 모드 아키텍처:

┌───────────────────────────────────────────────────────────────────┐
│  Kafka Connect 클러스터                                             │
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐│
│  │    Worker 1       │  │    Worker 2       │  │    Worker 3       ││
│  │                  │  │                  │  │                  ││
│  │ Source Task 1    │  │ Source Task 2    │  │ Sink Task 1      ││
│  │ Source Task 2    │  │ Sink Task 1      │  │ Sink Task 2      ││
│  └──────────────────┘  └──────────────────┘  └──────────────────┘│
│          │                      │                      │          │
└──────────┼──────────────────────┼──────────────────────┼──────────┘
           │                      │                      │
           ▼                      ▼                      ▼
     ┌───────────────────────────────────────────────────────┐
     │                  Kafka Cluster                         │
     │  connect-configs   connect-offsets   connect-status    │
     │  (커넥터 설정 저장)  (오프셋 저장)   (상태 저장)         │
     └───────────────────────────────────────────────────────┘
```

Connect 클러스터는 내부적으로 다음 3개의 Kafka 토픽을 사용한다:

| 내부 토픽 | 용도 |
|----------|------|
| `connect-configs` | 커넥터 및 Task 설정 저장 (컴팩션 정책) |
| `connect-offsets` | Source Connector의 읽기 오프셋 저장 (컴팩션 정책) |
| `connect-status` | 커넥터/Task 상태 저장 (컴팩션 정책) |

## Source Connector vs Sink Connector

### Source Connector (외부 → Kafka)

```
Source Connector:

외부 시스템 (MySQL)
     │
     │ 변경 감지 (폴링 or CDC)
     ▼
Source Task
  - poll() 호출로 데이터 읽기
  - SourceRecord 생성 (토픽, 키, 값, 오프셋)
     │
     ▼
Kafka Topic (새 메시지 발행)

주요 Source Connectors:
  - Debezium MySQL/PostgreSQL (CDC 기반)
  - Kafka Connect JDBC Source (폴링 기반)
  - Kafka Connect S3 Source
  - Kafka Connect File Source
```

### Sink Connector (Kafka → 외부)

```
Sink Connector:

Kafka Topic
     │
     │ 컨슈머처럼 메시지 소비
     ▼
Sink Task
  - put(records) 호출로 데이터 처리
  - 외부 시스템에 쓰기
  - 완료 후 오프셋 커밋
     │
     ▼
외부 시스템 (Elasticsearch, S3, BigQuery 등)

주요 Sink Connectors:
  - Elasticsearch Sink
  - S3 Sink (Apache Parquet, JSON 등)
  - BigQuery Sink
  - JDBC Sink (데이터베이스)
  - HTTP Sink
```

### CDC (Change Data Capture) - Debezium 예시

```
Debezium (CDC 기반 Source Connector) 동작:

MySQL binlog:
  INSERT INTO orders VALUES (1, 'user_1', 1000)
  UPDATE orders SET amount=2000 WHERE id=1
  DELETE FROM orders WHERE id=1

Debezium이 binlog를 읽어 Kafka로 발행:

INSERT → {"op":"c", "before":null, "after":{"id":1,"user":"user_1","amount":1000}}
UPDATE → {"op":"u", "before":{"id":1,"amount":1000}, "after":{"id":1,"amount":2000}}
DELETE → {"op":"d", "before":{"id":1,"amount":2000}, "after":null}

→ DB 변경을 실시간으로 Kafka 토픽에 발행
→ 이중 쓰기(dual write) 없이 DB와 메시지 일관성 보장
```

## Standalone vs Distributed 모드

### Standalone 모드

```
Standalone 모드:
  - Worker 1개에서 모든 Task 실행
  - 설정 파일로 커넥터 구성
  - 오프셋을 로컬 파일에 저장
  - 장애 복구 없음 (Worker 죽으면 Task 전체 중단)
  - 개발/테스트 환경에 적합

실행 방법:
  connect-standalone.sh worker.properties connector.properties
```

### Distributed 모드

```
Distributed 모드:
  - 여러 Worker에서 Task를 분산 실행
  - 설정을 Kafka 토픽(connect-configs)에 저장
  - Worker 장애 시 Task가 다른 Worker로 자동 이동
  - REST API로 런타임 커넥터 관리
  - 운영 환경 표준

실행 방법:
  connect-distributed.sh worker.properties
  (커넥터 추가/수정은 REST API로)

Worker 장애 시 자동 복구:
  Worker1 (Task A, B) → 장애
  Worker2, Worker3 → Task A, B를 나눠 재배분
  → 서비스 중단 없이 복구
```

## Connector, Task, Worker

```
구성 요소 계층:

Connector (논리적 단위)
  │
  ├── Task 1  (실제 작업 단위, 병렬 실행 가능)
  ├── Task 2
  └── Task 3
          │
          Worker (Task를 실행하는 JVM 프로세스)

예시) MySQL Source Connector, tasks.max=3:

Connector: mysql-source
  Task 0: 테이블 orders 읽기 → 브로커로 전송
  Task 1: 테이블 products 읽기 → 브로커로 전송
  Task 2: 테이블 users 읽기 → 브로커로 전송

→ tasks.max로 병렬도 조절
→ 각 Task는 서로 다른 Worker에서 실행될 수 있음
```

## SMT (Single Message Transforms)

SMT는 메시지가 Kafka에 저장되기 전(Source) 또는 외부로 나가기 전(Sink)에 **가벼운 변환**을 수행한다.

```
SMT 처리 위치:

Source Connector:
  외부 시스템 → [SMT 변환] → Kafka 토픽

Sink Connector:
  Kafka 토픽 → [SMT 변환] → 외부 시스템
```

### 주요 내장 SMT

| SMT | 설명 | 예시 |
|-----|------|------|
| `ReplaceField` | 필드 추가/제거/이름 변경 | `blacklist`: 필드 제거 |
| `MaskField` | 필드값을 마스킹 | PII 데이터 마스킹 |
| `InsertField` | 정적 값 또는 메타데이터 삽입 | 타임스탬프, 토픽명 삽입 |
| `TimestampConverter` | 타임스탬프 형식 변환 | epoch → ISO 8601 |
| `ValueToKey` | 값의 필드를 메시지 키로 변경 | `user_id`를 키로 |
| `ExtractField` | Struct에서 특정 필드만 추출 | |
| `Filter` | 조건에 맞는 메시지만 통과 | |
| `Flatten` | 중첩 구조를 평탄화 | |
| `Cast` | 필드 타입 변환 | String → Integer |

```json
// SMT 설정 예시 (커넥터 설정 JSON)
{
  "name": "mysql-source",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "transforms": "unwrap,addTimestamp",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.addTimestamp.type": "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.addTimestamp.timestamp.field": "kafka_ingestion_time"
  }
}
```

## REST API로 Connector 관리

Distributed 모드에서는 REST API로 커넥터를 관리한다. (기본 포트: 8083)

```bash
# 커넥터 목록 조회
curl http://localhost:8083/connectors

# 커넥터 생성
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "mysql-source",
    "config": {
      "connector.class": "io.debezium.connector.mysql.MySqlConnector",
      "database.hostname": "mysql",
      "database.port": "3306",
      "database.user": "debezium",
      "database.password": "dbz",
      "database.server.name": "mydb",
      "database.include.list": "orders",
      "topic.prefix": "cdc"
    }
  }'

# 커넥터 상태 조회
curl http://localhost:8083/connectors/mysql-source/status

# 커넥터 일시 정지
curl -X PUT http://localhost:8083/connectors/mysql-source/pause

# 커넥터 재시작
curl -X POST http://localhost:8083/connectors/mysql-source/restart

# 커넥터 삭제
curl -X DELETE http://localhost:8083/connectors/mysql-source
```

## 주요 Connector 목록

| 종류 | 커넥터 | 방향 | 설명 |
|------|-------|------|------|
| Database | Debezium MySQL | Source | MySQL binlog CDC |
| Database | Debezium PostgreSQL | Source | PostgreSQL CDC |
| Database | JDBC Source/Sink | Source/Sink | 표준 JDBC 연동 |
| Cloud | S3 Sink | Sink | Kafka → S3 (Parquet, JSON 등) |
| Cloud | S3 Source | Source | S3 → Kafka |
| Cloud | GCS Sink | Sink | Kafka → Google Cloud Storage |
| Search | Elasticsearch Sink | Sink | Kafka → Elasticsearch |
| Search | OpenSearch Sink | Sink | Kafka → OpenSearch |
| Analytics | BigQuery Sink | Sink | Kafka → BigQuery |
| Analytics | Snowflake Sink | Sink | Kafka → Snowflake |
| Messaging | MirrorMaker 2 | Source/Sink | Kafka → Kafka 클러스터 간 복제 |
| File | File Sink/Source | Source/Sink | 파일 시스템 연동 |

## Kafka Connect 주요 설정

### Worker 설정 (distributed)

| 설정 | 기본값 | 설명 |
|------|--------|------|
| `group.id` | - | Connect 클러스터 ID. 같은 ID의 Worker가 클러스터 구성 |
| `bootstrap.servers` | - | Kafka 브로커 주소 |
| `config.storage.topic` | `connect-configs` | 커넥터 설정 저장 토픽 |
| `offset.storage.topic` | `connect-offsets` | 오프셋 저장 토픽 |
| `status.storage.topic` | `connect-status` | 상태 저장 토픽 |
| `key.converter` | `JsonConverter` | 메시지 키 직렬화 방식 |
| `value.converter` | `JsonConverter` | 메시지 값 직렬화 방식 |
| `plugin.path` | - | 커넥터 JAR 파일 경로 |
| `rest.port` | 8083 | REST API 포트 |

### Converter 설정

```
Converter: Kafka 메시지의 직렬화/역직렬화 담당

JsonConverter:
  → JSON 형식으로 직렬화
  → 스키마 정보를 메시지에 포함 가능 (schemas.enable=true)

AvroConverter (Confluent Schema Registry 필요):
  → Avro 형식으로 직렬화
  → 스키마를 Schema Registry에 등록/조회
  → 스키마 진화(evolution) 지원

ProtobufConverter:
  → Protobuf 형식
  → 스키마 Registry 필요

ByteArrayConverter:
  → 바이트 배열 그대로 전달
  → 이미 직렬화된 데이터를 그대로 쓸 때
```

> **실무 요약**: CDC 파이프라인에는 Debezium + SMT(ExtractNewRecordState)를 조합하여 before/after 구조를 단순화하는 것이 일반적이다. 스키마 관리가 중요한 경우 Avro + Schema Registry를 사용하고, 간단한 파이프라인에서는 JSON으로 시작하는 것이 무난하다.

---

# Kafka Streams

## Kafka Streams란

Kafka Streams는 **Kafka 토픽을 입력/출력으로 하는 스트림 처리를 위한 Java 라이브러리**이다. 별도의 클러스터(Flink, Spark 등)가 필요 없으며, 일반 Java 애플리케이션에 라이브러리로 포함하여 사용한다.

```
Kafka Streams의 특징:

일반 Java 라이브러리:
  → 별도 처리 클러스터 불필요
  → 애플리케이션에 직접 내장 (마이크로서비스 친화)
  → JVM 애플리케이션에서 바로 사용 가능

Kafka 생태계 완전 통합:
  → 입력/출력이 모두 Kafka 토픽
  → Exactly-Once 처리 보장
  → Kafka의 파티션 기반 병렬 처리 자동 활용

Stateful 처리 지원:
  → RocksDB 기반 로컬 State Store
  → 윈도잉, 조인, 집계 등 복잡한 처리 가능
  → State Store를 Kafka 토픽(changelog)으로 백업
```

## 아키텍처

```
Kafka Streams 아키텍처:

입력 토픽 (orders)
  파티션 0 ──▶ StreamThread 0 (Task 0)
  파티션 1 ──▶ StreamThread 1 (Task 1)
  파티션 2 ──▶ StreamThread 2 (Task 2)

각 Task:
  ┌─────────────────────────────────────┐
  │           Stream Task               │
  │                                     │
  │  Source ──▶ Processor ──▶ Sink      │
  │             (Topology)              │
  │                 │                  │
  │           State Store (RocksDB)     │
  │           (로컬 디스크)              │
  └─────────────────────────────────────┘
       │                         │
  입력 토픽                   출력 토픽
  (Kafka)                    (Kafka)
```

### 병렬 처리

```
병렬 처리 = 파티션 수 × 인스턴스 수

예시) 파티션 6개, 인스턴스 2개, 스레드 3개:

인스턴스 1 (StreamThread 3개):
  Thread 0 → Task 0 (파티션 0)
  Thread 1 → Task 1 (파티션 1)
  Thread 2 → Task 2 (파티션 2)

인스턴스 2 (StreamThread 3개):
  Thread 0 → Task 3 (파티션 3)
  Thread 1 → Task 4 (파티션 4)
  Thread 2 → Task 5 (파티션 5)

→ 인스턴스를 늘리면 자동으로 Task가 재분배
→ 파티션 수가 최대 병렬도의 상한
```

## KStream vs KTable vs GlobalKTable

Kafka Streams는 데이터를 세 가지 추상화로 표현한다.

### KStream - 이벤트 스트림

```
KStream: 무한한 이벤트의 흐름

orders 토픽:
  t=1: key=order-1, value={amount: 1000, status: "created"}
  t=2: key=order-2, value={amount: 2000, status: "created"}
  t=3: key=order-1, value={amount: 1000, status: "paid"}
  t=4: key=order-1, value={amount: 1000, status: "shipped"}

→ 각 레코드는 독립적인 이벤트
→ 같은 키의 레코드도 모두 개별 이벤트로 처리
→ "Order-1에 대한 이벤트 3개가 있었다"
```

### KTable - 변경 로그 (최신 상태)

```
KTable: 각 키의 최신 상태를 나타내는 테이블

같은 토픽을 KTable로 읽으면:
  t=1: key=order-1 → {status: "created"}  (삽입)
  t=3: key=order-1 → {status: "paid"}     (업데이트)
  t=4: key=order-1 → {status: "shipped"}  (업데이트)

현재 KTable 상태:
  order-1 → {status: "shipped"}  ← 최신 값
  order-2 → {status: "created"}

→ 각 키의 최신 값만 유지 (컴팩션 개념)
→ "Order-1의 현재 상태는 shipped"
→ 내부적으로 State Store에 저장
```

### GlobalKTable

```
GlobalKTable: 모든 파티션의 데이터를 로컬에 복사

KTable vs GlobalKTable:

KTable:
  → 자신에게 할당된 파티션의 데이터만 로컬에 보유
  → 파티션 코로케이션이 필요한 조인에 사용

GlobalKTable:
  → 모든 파티션의 전체 데이터를 로컬에 복사
  → 어떤 키와도 조인 가능 (파티션 무관)
  → 데이터셋이 작은 경우 (참조 데이터, 룩업 테이블) 적합
  → 예: 사용자 정보, 상품 카탈로그 등

주의: 토픽 크기만큼 모든 인스턴스에 복제 → 큰 토픽엔 부적합
```

## Topology와 처리 파이프라인

Kafka Streams의 처리 로직은 **Topology**로 표현된다. Topology는 Source → Processor → Sink의 DAG(방향성 비순환 그래프)이다.

```java
StreamsBuilder builder = new StreamsBuilder();

// Source: 입력 토픽
KStream<String, Order> orders = builder.stream("orders");

// Processor: 변환/필터/집계
KStream<String, Order> paidOrders = orders
    .filter((key, order) -> order.getStatus().equals("paid"))
    .mapValues(order -> enrichOrder(order));

// Sink: 출력 토픽
paidOrders.to("paid-orders");

// Topology 빌드
Topology topology = builder.build();

// 실행
KafkaStreams streams = new KafkaStreams(topology, config);
streams.start();
```

```
위 코드의 Topology:

orders (Source 토픽)
    │
    ▼
Filter (status == "paid")
    │
    ▼
MapValues (enrichOrder)
    │
    ▼
paid-orders (Sink 토픽)
```

## Stateless 연산

상태를 유지하지 않고 각 레코드를 독립적으로 처리한다.

| 연산 | 설명 | 예시 |
|------|------|------|
| `filter` | 조건에 맞는 레코드만 통과 | `status == "paid"` |
| `filterNot` | 조건에 맞지 않는 레코드 통과 | `status != "cancelled"` |
| `map` | key와 value 모두 변환 | 새 KeyValue 반환 |
| `mapValues` | value만 변환 | `order → order.getAmount()` |
| `selectKey` | key만 변환 | `userId를 새 key로` |
| `flatMap` | 1개 레코드를 N개로 변환 | 배열 평탄화 |
| `flatMapValues` | value만 N개로 변환 | |
| `foreach` | 부수 효과 (로깅 등) | |
| `peek` | 통과시키면서 부수 효과 | 모니터링 |
| `branch` | 조건별로 스트림 분기 | |
| `merge` | 두 스트림 합치기 | |

```java
// 예시
KStream<String, Order> stream = builder.stream("orders");

stream
    .filter((k, v) -> v.getAmount() > 1000)        // 금액 1000 초과만
    .mapValues(v -> v.toSummary())                   // 요약 객체로 변환
    .selectKey((k, v) -> v.getUserId())             // userId를 key로
    .to("high-value-orders");
```

## Stateful 연산

이전 레코드의 상태를 유지하며 처리한다. 내부적으로 **State Store**를 사용한다.

### 집계 (Aggregation)

```java
// 사용자별 주문 총액 집계
KTable<String, Long> totalByUser = builder
    .stream("orders")
    .groupByKey()                           // key(userId)로 그룹핑
    .aggregate(
        () -> 0L,                          // 초기값
        (key, order, total) ->             // 집계 함수
            total + order.getAmount(),
        Materialized.as("total-by-user")   // State Store 이름
    );

totalByUser.toStream().to("user-totals");
```

### 카운트 (Count)

```java
// 파티션별 메시지 수 카운트
KTable<String, Long> countByStatus = builder
    .stream("orders")
    .groupBy((key, order) ->
        KeyValue.pair(order.getStatus(), order))
    .count(Materialized.as("count-by-status"));
```

### 조인 (Join)

```java
// KStream-KTable 조인 (주문 + 사용자 정보)
KStream<String, Order> orders = builder.stream("orders");
KTable<String, User> users = builder.table("users");

KStream<String, EnrichedOrder> enriched = orders.join(
    users,
    (order, user) -> new EnrichedOrder(order, user)
);
```

| 조인 종류 | 설명 |
|----------|------|
| `KStream-KStream` | 두 스트림을 윈도우 내에서 조인 |
| `KStream-KTable` | 스트림의 각 레코드를 KTable의 최신 값과 조인 |
| `KStream-GlobalKTable` | 파티션 관계 없이 GlobalKTable과 조인 |
| `KTable-KTable` | 두 KTable을 조인하여 새 KTable 생성 |

## 윈도잉 (Windowing)

시간 범위를 기반으로 이벤트를 그룹화한다.

### Tumbling Window (비겹침 고정 윈도우)

```
[0s─────5s) [5s───10s) [10s──15s) ...

→ 겹치지 않는 고정 크기 윈도우
→ 각 이벤트가 정확히 하나의 윈도우에 속함
→ 예: 5분마다 집계

KGroupedStream.windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
```

### Hopping Window (겹치는 슬라이딩 윈도우)

```
[0s──────10s)
     [5s──────15s)
          [10s──────20s)

→ 윈도우 크기 > 이동 간격 → 겹침 발생
→ 각 이벤트가 여러 윈도우에 속할 수 있음
→ 예: 10분 윈도우를 5분마다 슬라이딩

TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(10))
           .advanceBy(Duration.ofMinutes(5))
```

### Session Window (활동 기반 동적 윈도우)

```
사용자 활동:
  t=0: 클릭
  t=3: 클릭   ─── 세션 1 (비활동 5초 이내)
  t=4: 클릭
  t=10: 클릭              → 비활동 6초 → 세션 2 시작
  t=12: 클릭  ─── 세션 2

→ 일정 시간 비활동(Gap) 후 새 세션 시작
→ 세션 길이가 가변적
→ 사용자 행동 분석, 이벤트 클러스터링에 유용

SessionWindows.ofInactivityGapWithNoGrace(Duration.ofSeconds(5))
```

### Grace Period (늦은 데이터 처리)

```java
// 늦게 도착한 데이터를 2분까지 허용
TimeWindows.ofSizeAndGrace(
    Duration.ofMinutes(5),   // 윈도우 크기
    Duration.ofMinutes(2)    // Grace Period
)
```

## State Store

Stateful 처리에서 상태를 저장하는 로컬 저장소이다.

```
State Store 구조:

┌──────────────────────────────────────────┐
│  Kafka Streams 인스턴스                    │
│                                          │
│  ┌───────────────────┐                   │
│  │  State Store       │ ← RocksDB (기본) │
│  │  (로컬 디스크)      │   또는 In-Memory  │
│  │                   │                   │
│  │  key=user-1 → 5000│                   │
│  │  key=user-2 → 3000│                   │
│  └───────────────────┘                   │
│           │                              │
│           │ Changelog (변경 사항 기록)     │
│           ▼                              │
└──────────────────────────────────────────┘
           │
           ▼
  Kafka 토픽 (changelog 토픽)
  예: my-app-total-by-user-changelog

→ State Store의 변경이 Kafka 토픽에 기록됨
→ 인스턴스 재시작 시 changelog로부터 State 복구
→ 인스턴스 이동(리밸런스) 시 새 인스턴스가 changelog로 State 재구축
```

### State Store 유형

```java
// 영구 State Store (RocksDB, 기본)
Materialized.as("my-store")
    .withKeySerde(Serdes.String())
    .withValueSerde(Serdes.Long())

// In-Memory State Store (재시작 시 초기화, changelog로 복구)
Materialized.<String, Long, KeyValueStore<Bytes, byte[]>>as("my-store")
    .withStoreType(Materialized.StoreType.IN_MEMORY)

// 로깅 비활성화 (changelog 토픽 미생성, 복구 불가)
Materialized.as("my-store").withLoggingDisabled()
```

## Interactive Queries

State Store에 저장된 데이터를 **외부에서 직접 조회**할 수 있다. 마이크로서비스에서 실시간 집계 결과를 API로 제공할 때 유용하다.

```java
// 집계 결과를 State Store에 저장
KTable<String, Long> totalByUser = orders
    .groupByKey()
    .aggregate(..., Materialized.as("total-by-user"));

// State Store에서 직접 조회
KafkaStreams streams = new KafkaStreams(topology, config);
streams.start();

// 특정 사용자의 총 주문액 조회
ReadOnlyKeyValueStore<String, Long> store =
    streams.store(StoreQueryParameters.fromNameAndType(
        "total-by-user",
        QueryableStoreTypes.keyValueStore()
    ));

Long totalAmount = store.get("user-1");  // 실시간 조회

// 전체 조회
KeyValueIterator<String, Long> all = store.all();
while (all.hasNext()) {
    KeyValue<String, Long> kv = all.next();
    System.out.println(kv.key + ": " + kv.value);
}
```

```
분산 환경에서 Interactive Queries:

인스턴스 1: user-1 ~ user-500 (파티션 0)
인스턴스 2: user-501 ~ user-1000 (파티션 1)

"user-600"를 조회하려면 인스턴스 2에 요청해야 함

→ streams.queryMetadataForKey()로 어느 인스턴스에 있는지 파악
→ 해당 인스턴스에 HTTP 요청 (커스텀 구현 필요)
```

## Exactly-Once 처리

Kafka Streams는 `processing.guarantee` 설정으로 Exactly-Once 처리를 지원한다.

```
처리 보장 수준:

at_least_once (기본값):
  → 장애 시 메시지를 다시 처리 (중복 가능)
  → 높은 처리량
  → 멱등한 처리 로직 필요

exactly_once_v2 (권장, Kafka 2.6+):
  → 메시지를 정확히 한 번 처리
  → 내부적으로 트랜잭션 프로듀서 사용
  → 소비 + 처리 + 발행을 하나의 트랜잭션으로 묶음
  → 약간의 성능 오버헤드

exactly_once (구버전, Kafka 2.5 이하):
  → exactly_once_v2 이전 버전
```

```java
Properties config = new Properties();
config.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG,
    StreamsConfig.EXACTLY_ONCE_V2);  // 권장
```

```
Exactly-Once 내부 동작:

poll (Kafka Input) ──────────────────────────────────┐
                                                      │ 하나의 트랜잭션
State Store 업데이트                                   │
(로컬 RocksDB + changelog 토픽 쓰기)                   │
                                                      │
produce (Kafka Output) ──────────────────────────────┘

commit ─── 트랜잭션 커밋 + 오프셋 커밋 원자적으로 수행

→ 장애 시 트랜잭션 전체 롤백
→ 재처리 시 중복 발행 없음
```

## Kafka Streams 주요 설정

| 설정 | 기본값 | 설명 |
|------|--------|------|
| `application.id` | - | 애플리케이션 고유 ID. 컨슈머 group.id로도 사용됨 |
| `bootstrap.servers` | - | Kafka 브로커 주소 |
| `num.stream.threads` | 1 | 스트림 스레드 수. CPU 코어 수만큼 늘려 병렬도 향상 |
| `processing.guarantee` | `at_least_once` | 처리 보장 수준. `exactly_once_v2` 권장 |
| `state.dir` | `/tmp/kafka-streams` | State Store 로컬 저장 경로 |
| `replication.factor` | 1 | 내부 토픽(changelog, repartition) 복제 팩터 |
| `cache.max.bytes.buffering` | 10485760 (10MB) | 레코드 캐시 크기. 0이면 캐시 비활성화 |
| `commit.interval.ms` | 30000 (30초) | 커밋 주기 (at_least_once), EOS에서는 100ms |
| `default.key.serde` | `Serdes.ByteArray()` | 기본 Key Serde |
| `default.value.serde` | `Serdes.ByteArray()` | 기본 Value Serde |

### 실전 설정 예시

```java
Properties config = new Properties();
config.put(StreamsConfig.APPLICATION_ID_CONFIG, "order-processor");
config.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "broker1:9092,broker2:9092");
config.put(StreamsConfig.NUM_STREAM_THREADS_CONFIG, 4);
config.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG,
    StreamsConfig.EXACTLY_ONCE_V2);
config.put(StreamsConfig.STATE_DIR_CONFIG, "/data/kafka-streams");
config.put(StreamsConfig.REPLICATION_FACTOR_CONFIG, 3);
config.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG,
    Serdes.String().getClass());
config.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG,
    Serdes.String().getClass());
```

> **실무 요약**: Kafka Streams는 Kafka 기반의 실시간 처리를 별도 클러스터 없이 구현할 수 있어 운영 복잡도가 낮다. 그러나 복잡한 조인이나 대규모 상태 관리가 필요한 경우 Flink나 Spark Streaming과 비교 검토가 필요하다. 간단한 스트림 처리, 이벤트 보강, 집계에는 Kafka Streams가 매우 효과적이다.

---

# Connect vs Streams 비교

| | Kafka Connect | Kafka Streams |
|---|---|---|
| **목적** | 외부 시스템 ↔ Kafka 데이터 이동 | Kafka 토픽 내 데이터 처리/변환 |
| **입력** | 외부 시스템 (DB, S3, 등) | Kafka 토픽 |
| **출력** | Kafka 토픽 또는 외부 시스템 | Kafka 토픽 |
| **복잡도** | 설정(JSON/Properties) 기반 | Java 코드 기반 |
| **상태 관리** | 오프셋만 관리 | 복잡한 State Store 지원 |
| **변환 능력** | SMT로 간단한 변환 | 복잡한 집계/조인/윈도잉 |
| **스케일링** | Worker 추가로 수평 확장 | 인스턴스 추가로 수평 확장 |
| **언어** | JVM (Java/Kotlin) | JVM (Java/Kotlin) |
| **적합한 케이스** | ETL, CDC, 데이터 파이프라인 | 실시간 집계, 이벤트 보강, 복잡 처리 |

### 함께 사용하는 패턴

```
대표적인 데이터 파이프라인 패턴:

MySQL (원천)
    │
    │ Debezium Source Connector (CDC)
    ▼
Kafka (raw-orders 토픽)
    │
    │ Kafka Streams (주문 금액 집계, 사용자 보강)
    ▼
Kafka (enriched-orders 토픽)
    │
    │ Elasticsearch Sink Connector
    ▼
Elasticsearch (검색/분석)

→ Connect: 데이터 수집/적재
→ Streams: 비즈니스 로직 처리
→ Connect: 결과 외부 저장소로 전달
```
