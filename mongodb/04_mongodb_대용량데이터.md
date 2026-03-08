# MongoDB 대용량 데이터

## 목차
- [Sharding 활성화](#1-sharding-활성화)
- [MongoDB와 대용량 데이터에 적합한가](#2-mongodb와-대용량-데이터에-적합한가)
- [MongoDB vs Wide Column Database](#3-mongodb-vs-wide-column-database)

---

## 1. Sharding 활성화

Sharding은 데이터를 여러 서버(Shard)에 분산 저장하여 **수평 확장(Scale-out)**을 가능하게 하는 MongoDB의 핵심 기능이다.

### Sharding 아키텍처

```
Client
  |
mongos (Query Router)   <-- 쿼리를 적절한 Shard로 라우팅
  |
Config Server (3대 ReplicaSet)  <-- 클러스터 메타데이터 저장
  |
+-----------+-----------+
|           |           |
Shard 1    Shard 2    Shard 3
(ReplicaSet)(ReplicaSet)(ReplicaSet)
```

| 구성 요소 | 역할 |
|----------|------|
| `mongos` | 클라이언트 요청을 받아 적절한 Shard로 라우팅하는 프록시 |
| Config Server | 각 Shard의 데이터 범위(Chunk 정보)와 클러스터 메타데이터 관리 |
| Shard | 실제 데이터를 저장하는 ReplicaSet |

### Sharding 활성화 절차

#### 1단계: Config Server ReplicaSet 구성

```js
// config server mongod 설정
mongod --configsvr --replSet configRS --port 27019 --dbpath /data/configdb

// config server ReplicaSet 초기화
rs.initiate({
  _id: "configRS",
  configsvr: true,
  members: [
    { _id: 0, host: "cfg1:27019" },
    { _id: 1, host: "cfg2:27019" },
    { _id: 2, host: "cfg3:27019" }
  ]
})
```

#### 2단계: Shard ReplicaSet 구성

```js
// shard mongod 설정
mongod --shardsvr --replSet shard1RS --port 27018 --dbpath /data/shard1

// shard ReplicaSet 초기화
rs.initiate({
  _id: "shard1RS",
  members: [
    { _id: 0, host: "shard1a:27018" },
    { _id: 1, host: "shard1b:27018" },
    { _id: 2, host: "shard1c:27018" }
  ]
})
```

#### 3단계: mongos 실행

```bash
mongos --configdb configRS/cfg1:27019,cfg2:27019,cfg3:27019 --port 27017
```

#### 4단계: Shard 추가 및 Sharding 설정

```js
// mongos에 접속 후 Shard 추가
sh.addShard("shard1RS/shard1a:27018,shard1b:27018,shard1c:27018")
sh.addShard("shard2RS/shard2a:27018,shard2b:27018,shard2c:27018")

// 데이터베이스 sharding 활성화
sh.enableSharding("myDatabase")

// 컬렉션 sharding 활성화 (Shard Key 지정)
sh.shardCollection("myDatabase.orders", { userId: "hashed" })
```

### Shard Key 선택

Shard Key는 데이터를 어떤 Shard에 배치할지 결정하는 핵심 필드다. 잘못 선택하면 **Hotspot(특정 Shard에 부하 집중)** 문제가 발생한다.

| 전략 | 설명 | 장점 | 단점 |
|------|------|------|------|
| Hashed Sharding | 필드 값을 해시하여 균등 분산 | 균등한 데이터 분산 | 범위 쿼리 비효율 |
| Ranged Sharding | 값의 범위로 Shard 결정 | 범위 쿼리 효율적 | Hotspot 위험 |
| Zone Sharding | 특정 범위를 특정 Shard에 고정 | 지역별 데이터 배치 가능 | 설정 복잡 |

**좋은 Shard Key 조건**:
- 카디널리티(Cardinality)가 높을 것 - 값의 종류가 많아야 분산이 잘됨
- 쓰기 분산이 고를 것 - 특정 값에 쓰기가 몰리지 않아야 함
- 쿼리 격리가 가능할 것 - Shard Key가 쿼리 조건에 포함되면 특정 Shard만 조회

```js
// 나쁜 Shard Key: 카디널리티 낮음 (status는 몇 가지 값만 존재)
sh.shardCollection("db.orders", { status: 1 })

// 좋은 Shard Key: 카디널리티 높고 균등 분산
sh.shardCollection("db.orders", { userId: "hashed" })

// 복합 Shard Key: 범위 쿼리 + 분산 모두 고려
sh.shardCollection("db.events", { userId: 1, timestamp: 1 })
```

### Chunk와 Balancing

- 데이터는 **Chunk** 단위로 관리된다 (기본 128MB).
- Shard 간 Chunk 수 불균형이 발생하면 **Balancer**가 자동으로 Chunk를 이동시킨다.
- Balancer는 백그라운드에서 실행되며, 필요 시 비활성화할 수 있다.

```js
// Balancer 상태 확인
sh.getBalancerState()

// Balancer 비활성화 (점검 시)
sh.stopBalancer()

// Chunk 분포 확인
db.printShardingStatus()
```

---

## 2. MongoDB와 대용량 데이터에 적합한가

### MongoDB가 대용량에 강한 이유

#### 수평 확장 (Scale-out)
RDBMS는 주로 수직 확장(Scale-up, 더 좋은 서버로 교체)에 의존하지만, MongoDB는 Sharding을 통해 서버를 추가하는 방식으로 선형에 가깝게 확장할 수 있다.

#### 유연한 스키마
대용량 데이터는 다양한 형태를 가지는 경우가 많다. MongoDB의 도큐먼트 모델은 스키마 변경 없이 다양한 구조의 데이터를 저장할 수 있다.

#### 내장 도큐먼트(Embedding)로 JOIN 회피
RDBMS에서 JOIN은 대용량에서 성능 병목이 된다. MongoDB는 관련 데이터를 하나의 도큐먼트에 내장하여 단일 읽기로 처리할 수 있다.

```js
// JOIN 없이 단일 도큐먼트로 처리
{
  _id: ObjectId("..."),
  orderId: "ORD-001",
  userId: "user123",
  items: [
    { productId: "P1", name: "상품A", qty: 2, price: 10000 },
    { productId: "P2", name: "상품B", qty: 1, price: 5000 }
  ],
  address: { city: "Seoul", zip: "12345" }
}
```

### MongoDB가 대용량에 불리한 경우

#### 강한 트랜잭션이 필요한 경우
MongoDB는 4.0부터 멀티 도큐먼트 트랜잭션을 지원하지만, RDBMS에 비해 성능 오버헤드가 크다. 금융, 결제처럼 ACID가 핵심인 시스템에서는 부적합할 수 있다.

#### 복잡한 집계/분석 쿼리
대규모 Aggregation, 복잡한 JOIN이 많은 분석 워크로드는 OLAP 시스템(Redshift, BigQuery 등)이 더 적합하다. MongoDB는 OLTP에 최적화되어 있다.

#### 메모리 의존도
MongoDB는 WiredTiger 스토리지 엔진을 사용하며, **Working Set(자주 접근하는 데이터)이 RAM에 올라와 있어야** 최적 성능이 나온다. Working Set이 RAM을 초과하면 디스크 I/O가 급증하여 성능이 급격히 저하된다.

### 대용량 데이터 처리 시 고려사항

| 항목 | 권장 사항 |
|------|----------|
| Sharding | 단일 서버 용량의 60~70% 도달 전에 Sharding 계획 수립 |
| Working Set | 자주 쓰는 데이터가 RAM에 올라올 수 있도록 메모리 확보 |
| 인덱스 | 인덱스 전체가 RAM에 올라올 수 있어야 함 |
| 쓰기 패턴 | 대량 쓰기는 `bulkWrite()`로 배치 처리 |
| 읽기 패턴 | Replica Set의 Secondary를 읽기에 활용하여 Primary 부하 분산 |

```js
// 대량 쓰기 최적화: bulkWrite 사용
db.orders.bulkWrite([
  { insertOne: { document: { ... } } },
  { updateOne: { filter: { _id: 1 }, update: { $set: { status: "done" } } } },
  { deleteOne: { filter: { _id: 2 } } }
], { ordered: false })  // ordered: false -> 병렬 처리로 성능 향상
```

---

## 3. MongoDB vs Wide Column Database

대용량 데이터 처리에서 MongoDB(Document DB)와 HBase 같은 Wide Column DB는 자주 비교된다.

### 데이터 모델 비교

| 항목 | MongoDB (Document) | HBase (Wide Column) |
|------|--------------------|--------------------|
| 데이터 단위 | JSON 도큐먼트 | Row + 동적 Column |
| 스키마 | 유연한 도큐먼트 | Row Key + Column Family 중심 |
| 중첩 구조 | 지원 (배열, 서브 도큐먼트) | 미지원 (Flat 구조) |
| 쿼리 언어 | MQL (MongoDB Query Language) | Get/Scan API (Java, HBase Shell) |

### 쓰기/읽기 성능 비교

#### Wide Column DB의 강점: 대규모 순차 쓰기
HBase는 **LSM Tree 구조(MemStore -> HFile)**로 순차 쓰기가 매우 빠르다. HDFS 위에서 동작하며, 수백억 건의 로그, 이벤트, 시계열 데이터 저장에 적합하다.

```
HBase 쓰기 경로:
Write -> WAL (Write-Ahead Log, HDFS) -> MemStore (메모리) -> HFile (HDFS)
        (장애 복구용 영속성 보장)        (빠른 메모리 쓰기)   (백그라운드 플러시)
```

읽기는 Row Key 기반 Get이 가장 빠르고, 범위 Scan도 Row Key 정렬 덕분에 효율적이다. 단, Row Key 설계가 나쁘면 **Hotspot(특정 RegionServer에 부하 집중)** 문제가 생긴다.

#### MongoDB의 강점: 유연한 쿼리와 집계
MongoDB는 풍부한 쿼리 언어와 Aggregation Pipeline을 통해 복잡한 데이터 조회/변환이 가능하다. HBase는 기본적으로 **Row Key 기반 조회**가 주이며, 복잡한 조건 필터링이나 집계는 MapReduce나 Spark 연동이 필요하다.

### 확장성 비교

| 항목 | MongoDB | HBase |
|------|---------|-------|
| 확장 방식 | Sharding (Config Server 필요) | RegionServer 추가 (HMaster 관리) |
| 노드 추가 | mongos, Config Server 필요 | RegionServer 추가 후 Region 자동 이동 |
| 단일 실패 지점 | mongos, Config Server | HMaster (Active/Standby 구성 가능) |
| 운영 복잡도 | 높음 (구성 요소 다수) | 높음 (ZooKeeper, HDFS, HMaster 필요) |

### 일관성(Consistency) 비교

| 항목 | MongoDB | HBase |
|------|---------|-------|
| 기본 일관성 | Strong (Primary에서 읽기) | Strong (Row 단위 원자적 연산) |
| 설정 가능 | Read Preference로 조정 가능 | 설정 불가 (Row 단위 강한 일관성 고정) |
| 트랜잭션 | 멀티 도큐먼트 트랜잭션 지원 | Row 단위 원자성만 보장 (멀티 Row 트랜잭션 없음) |

### 사용 사례별 선택 기준

| 사용 사례 | 권장 DB | 이유 |
|----------|---------|------|
| 사용자 프로필, 상품 카탈로그 | MongoDB | 유연한 스키마, 복잡한 쿼리 |
| 수백억 건 로그, 이벤트 저장 | HBase | LSM Tree 순차 쓰기, HDFS 대용량 저장 |
| 실시간 분석 + 쓰기 혼합 | MongoDB | Aggregation Pipeline 활용 |
| 하둡 기반 배치 분석 + 랜덤 읽기 | HBase | HDFS 연동, MapReduce/Spark 연계 |
| 시계열 데이터 (Row Key = 센서ID+시간) | HBase | 범위 Scan으로 시간 범위 조회 효율적 |

### 요약

```
수백억 건 대용량 순차 쓰기 -> HBase
복잡한 쿼리 + 유연한 스키마 -> MongoDB
하둡 에코시스템 연동 (MapReduce, Spark) -> HBase
Row Key 기반 단순 조회 + 대용량 -> HBase
```

MongoDB는 대용량을 다룰 수 있지만, **쓰기 처리량보다 쿼리 유연성과 개발 생산성이 중요한 영역**에서 진가를 발휘한다. 수백억 건의 단순 순차 쓰기와 하둡 연동이 핵심이라면 HBase가 더 적합하다.
