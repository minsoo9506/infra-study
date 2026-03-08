# MongoDB 주요 Query

## 목차
- [SQL vs MQL](#sql-vs-mql)
- [Query Filter 와 Operator](#query-filter-와-operator)
- [기본 CRUD](#기본-crud)
- [유용한 Query 함수들](#유용한-query-함수들)
- [배열과 내장 Document 를 다루는 방법](#배열과-내장-document-를-다루는-방법)
- [Query 예제](#query-예제)
- [집계 프레임워크 Aggregation](#집계-프레임워크-aggregation)
- [자주 사용되는 Aggregation Stage](#자주-사용되는-aggregation-stage)
- [배포형태에 따른 CRUD 특징](#배포형태에-따른-crud-특징-replica-set-vs-sharded-cluster)

---

## SQL vs MQL

MongoDB의 쿼리 언어인 MQL(MongoDB Query Language)은 SQL과 개념은 유사하지만 문법이 다릅니다.

| 개념 | SQL | MongoDB (MQL) |
|------|-----|---------------|
| 데이터 저장 단위 | Table / Row | Collection / Document |
| 스키마 | 고정 스키마 | 유연한 스키마 (Schema-less) |
| 조회 | `SELECT * FROM users WHERE age > 20` | `db.users.find({ age: { $gt: 20 } })` |
| 삽입 | `INSERT INTO users (name) VALUES ('Alice')` | `db.users.insertOne({ name: 'Alice' })` |
| 수정 | `UPDATE users SET name='Bob' WHERE _id=1` | `db.users.updateOne({ _id: 1 }, { $set: { name: 'Bob' } })` |
| 삭제 | `DELETE FROM users WHERE _id=1` | `db.users.deleteOne({ _id: 1 })` |
| 조인 | `JOIN` | `$lookup` (Aggregation) |
| 그룹화 | `GROUP BY` | `$group` (Aggregation) |

---

## Query Filter 와 Operator

### Query Filter
MongoDB에서 조건을 지정할 때 사용하는 객체입니다. `find()`, `updateOne()`, `deleteOne()` 등 대부분의 쿼리에 사용됩니다.

```js
// 기본 필터: name이 "Alice"인 document 조회
db.users.find({ name: "Alice" })

// 여러 조건: name이 "Alice"이고 age가 30인 document 조회
db.users.find({ name: "Alice", age: 30 })
```

### 비교 연산자 (Comparison Operators)

| Operator | 의미 | 예시 |
|----------|------|------|
| `$eq` | 같음 (=) | `{ age: { $eq: 25 } }` |
| `$ne` | 다름 (!=) | `{ age: { $ne: 25 } }` |
| `$gt` | 초과 (>) | `{ age: { $gt: 20 } }` |
| `$gte` | 이상 (>=) | `{ age: { $gte: 20 } }` |
| `$lt` | 미만 (<) | `{ age: { $lt: 30 } }` |
| `$lte` | 이하 (<=) | `{ age: { $lte: 30 } }` |
| `$in` | 배열 내 포함 | `{ age: { $in: [20, 25, 30] } }` |
| `$nin` | 배열 내 미포함 | `{ age: { $nin: [20, 25] } }` |

### 논리 연산자 (Logical Operators)

| Operator | 의미 | 예시 |
|----------|------|------|
| `$and` | 모든 조건 충족 | `{ $and: [{ age: { $gt: 20 } }, { name: "Alice" }] }` |
| `$or` | 하나 이상 조건 충족 | `{ $or: [{ age: 20 }, { age: 30 }] }` |
| `$not` | 조건 부정 | `{ age: { $not: { $gt: 20 } } }` |
| `$nor` | 모든 조건 불충족 | `{ $nor: [{ age: 20 }, { name: "Alice" }] }` |

### 요소 연산자 (Element Operators)

| Operator | 의미 | 예시 |
|----------|------|------|
| `$exists` | 필드 존재 여부 | `{ email: { $exists: true } }` |
| `$type` | 필드 타입 확인 | `{ age: { $type: "int" } }` |

---

## 기본 CRUD

### Create (삽입)

```js
// 단건 삽입
db.users.insertOne({
  name: "Alice",
  age: 25,
  email: "alice@example.com"
})

// 다건 삽입
db.users.insertMany([
  { name: "Bob", age: 30 },
  { name: "Charlie", age: 22 }
])
```

### Read (조회)

```js
// 전체 조회
db.users.find()

// 조건 조회
db.users.find({ age: { $gte: 25 } })

// 단건 조회 (조건에 맞는 첫 번째 document)
db.users.findOne({ name: "Alice" })

// 특정 필드만 반환 (Projection) - 1: 포함, 0: 제외
db.users.find({ age: { $gt: 20 } }, { name: 1, email: 1, _id: 0 })
```

### Update (수정)

```js
// 단건 수정 ($set: 특정 필드만 수정)
db.users.updateOne(
  { name: "Alice" },
  { $set: { age: 26 } }
)

// 다건 수정
db.users.updateMany(
  { age: { $lt: 25 } },
  { $set: { status: "young" } }
)

// 필드 제거
db.users.updateOne(
  { name: "Alice" },
  { $unset: { email: "" } }
)

// 숫자 증가/감소
db.users.updateOne(
  { name: "Alice" },
  { $inc: { age: 1 } }
)

// document 전체 교체
db.users.replaceOne(
  { name: "Alice" },
  { name: "Alice", age: 27, city: "Seoul" }
)
```

### Delete (삭제)

```js
// 단건 삭제
db.users.deleteOne({ name: "Alice" })

// 다건 삭제
db.users.deleteMany({ age: { $lt: 20 } })

// 전체 삭제
db.users.deleteMany({})
```

---

## 유용한 Query 함수들

### sort() - 정렬
```js
// age 오름차순 (1), 내림차순 (-1)
db.users.find().sort({ age: 1 })
db.users.find().sort({ age: -1, name: 1 })
```

### limit() - 결과 수 제한
```js
// 상위 5개만 반환
db.users.find().limit(5)
```

### skip() - 건너뛰기 (페이지네이션에 활용)
```js
// 10개 건너뛰고 다음 10개 반환
db.users.find().skip(10).limit(10)
```

### count() / countDocuments() - 개수 세기
```js
db.users.countDocuments({ age: { $gt: 20 } })
```

### distinct() - 중복 제거
```js
// city 필드의 유니크한 값 목록
db.users.distinct("city")
```

### findOneAndUpdate() - 조회 후 수정
```js
// 수정 후 새로운 document 반환
db.users.findOneAndUpdate(
  { name: "Alice" },
  { $set: { age: 28 } },
  { returnDocument: "after" }
)
```

### findOneAndDelete() - 조회 후 삭제
```js
db.users.findOneAndDelete({ name: "Alice" })
```

### upsert - 없으면 삽입, 있으면 수정
```js
db.users.updateOne(
  { name: "Dave" },
  { $set: { age: 35 } },
  { upsert: true }
)
```

---

## 배열과 내장 Document 를 다루는 방법

### 내장 Document (Embedded Document) 조회
```js
// document 예시
// { name: "Alice", address: { city: "Seoul", zip: "04524" } }

// 점 표기법(dot notation)으로 접근
db.users.find({ "address.city": "Seoul" })
```

### 배열 조회

```js
// document 예시
// { name: "Alice", tags: ["mongodb", "database", "nosql"] }

// 배열에 특정 값이 포함된 document 조회
db.users.find({ tags: "mongodb" })

// 배열이 정확히 일치하는 document 조회
db.users.find({ tags: ["mongodb", "database"] })

// $all: 배열에 여러 값이 모두 포함
db.users.find({ tags: { $all: ["mongodb", "nosql"] } })

// $size: 배열 길이
db.users.find({ tags: { $size: 3 } })

// $elemMatch: 배열 요소가 여러 조건을 동시에 만족
// { scores: [{ subject: "math", score: 90 }, { subject: "eng", score: 85 }] }
db.students.find({ scores: { $elemMatch: { subject: "math", score: { $gte: 80 } } } })
```

### 배열 수정

```js
// $push: 배열에 요소 추가
db.users.updateOne({ name: "Alice" }, { $push: { tags: "backend" } })

// $addToSet: 중복 없이 추가
db.users.updateOne({ name: "Alice" }, { $addToSet: { tags: "mongodb" } })

// $pull: 특정 값 제거
db.users.updateOne({ name: "Alice" }, { $pull: { tags: "nosql" } })

// $pop: 첫 번째(-1) 또는 마지막(1) 요소 제거
db.users.updateOne({ name: "Alice" }, { $pop: { tags: 1 } })

// 배열의 특정 인덱스 요소 수정 (위치 연산자 $)
db.users.updateOne(
  { name: "Alice", tags: "database" },
  { $set: { "tags.$": "db" } }
)
```

---

## Query 예제

### 예제 데이터
```js
db.orders.insertMany([
  { customer: "Alice", product: "Laptop", qty: 1, price: 1200, status: "shipped", date: new Date("2024-01-15") },
  { customer: "Bob", product: "Mouse", qty: 3, price: 25, status: "pending", date: new Date("2024-02-10") },
  { customer: "Alice", product: "Keyboard", qty: 2, price: 75, status: "delivered", date: new Date("2024-02-20") },
  { customer: "Charlie", product: "Monitor", qty: 1, price: 400, status: "shipped", date: new Date("2024-03-05") },
  { customer: "Bob", product: "Laptop", qty: 1, price: 1200, status: "delivered", date: new Date("2024-03-10") }
])
```

### 예제 쿼리

```js
// 1. Alice의 모든 주문 조회
db.orders.find({ customer: "Alice" })

// 2. 상태가 "shipped" 또는 "delivered"인 주문
db.orders.find({ status: { $in: ["shipped", "delivered"] } })

// 3. 가격이 100 이상인 주문을 가격 내림차순 정렬
db.orders.find({ price: { $gte: 100 } }).sort({ price: -1 })

// 4. 2024년 2월 이후 주문 수
db.orders.countDocuments({ date: { $gte: new Date("2024-02-01") } })

// 5. 주문 상태가 "pending"인 주문을 "processing"으로 변경
db.orders.updateMany(
  { status: "pending" },
  { $set: { status: "processing" } }
)

// 6. customer 이름 목록 (중복 제거)
db.orders.distinct("customer")
```

---

## 집계 프레임워크 Aggregation

Aggregation은 MongoDB에서 데이터를 변환하고 집계하는 강력한 프레임워크입니다. 여러 Stage를 파이프라인(Pipeline)으로 연결해 데이터를 단계적으로 처리합니다.

```
Collection → Stage 1 → Stage 2 → Stage 3 → 결과
```

### 기본 구조

```js
db.collection.aggregate([
  { $stage1: { ... } },
  { $stage2: { ... } },
  { $stage3: { ... } }
])
```

### 간단한 예시

```js
// orders 컬렉션에서 고객별 총 주문 금액 계산
db.orders.aggregate([
  { $match: { status: "delivered" } },           // 1. delivered 상태만 필터
  { $group: { _id: "$customer", total: { $sum: { $multiply: ["$price", "$qty"] } } } }, // 2. 고객별 합산
  { $sort: { total: -1 } }                        // 3. 내림차순 정렬
])
```

---

## 자주 사용되는 Aggregation Stage

### $match - 필터링 (SQL의 WHERE)
```js
{ $match: { status: "shipped", price: { $gte: 100 } } }
```

### $group - 그룹화 (SQL의 GROUP BY)
```js
{
  $group: {
    _id: "$customer",               // 그룹 기준 필드
    totalAmount: { $sum: "$price" }, // 합계
    avgPrice: { $avg: "$price" },    // 평균
    count: { $sum: 1 },              // 개수
    maxPrice: { $max: "$price" },    // 최대값
    minPrice: { $min: "$price" }     // 최소값
  }
}
```

### $project - 필드 선택/변환 (SQL의 SELECT)
```js
{
  $project: {
    customer: 1,
    product: 1,
    totalPrice: { $multiply: ["$price", "$qty"] },  // 계산 필드 추가
    _id: 0
  }
}
```

### $sort - 정렬 (SQL의 ORDER BY)
```js
{ $sort: { totalAmount: -1, customer: 1 } }
```

### $limit / $skip - 페이지네이션
```js
{ $skip: 0 },
{ $limit: 10 }
```

### $lookup - 조인 (SQL의 JOIN)
```js
{
  $lookup: {
    from: "customers",        // 조인할 컬렉션
    localField: "customerId", // 현재 컬렉션의 필드
    foreignField: "_id",      // 대상 컬렉션의 필드
    as: "customerInfo"        // 결과를 저장할 필드명
  }
}
```

### $unwind - 배열 펼치기
```js
// { tags: ["a", "b", "c"] } → tags: "a", tags: "b", tags: "c" 3개의 document로 분리
{ $unwind: "$tags" }
```

### $addFields - 필드 추가
```js
{
  $addFields: {
    totalPrice: { $multiply: ["$price", "$qty"] }
  }
}
```

### $facet - 여러 파이프라인 동시 실행
```js
{
  $facet: {
    byStatus: [{ $group: { _id: "$status", count: { $sum: 1 } } }],
    byCustomer: [{ $group: { _id: "$customer", total: { $sum: "$price" } } }]
  }
}
```

---

## 배포형태에 따른 CRUD 특징 (Replica Set vs Sharded Cluster)

### Replica Set

Replica Set은 동일한 데이터를 가진 여러 MongoDB 서버(Primary 1대 + Secondary N대)로 구성됩니다.

```
         Write/Read
Client ──────────────→ Primary
                           │ Replication
                      ┌────┴────┐
                   Secondary  Secondary
                  (Read Only) (Read Only)
```

| 특징 | 설명 |
|------|------|
| **Write** | Primary에서만 가능 |
| **Read** | 기본적으로 Primary, `readPreference` 설정으로 Secondary 읽기 가능 |
| **Failover** | Primary 장애 시 Secondary가 자동으로 Primary로 선출 |
| **읽기 확장** | Secondary로 읽기 분산 가능 (단, 약간의 지연 발생 가능) |

```js
// Secondary에서 읽기 허용
db.users.find().readPref("secondaryPreferred")
```

**Write Concern**: 쓰기 작업의 확인 수준 설정
```js
db.users.insertOne(
  { name: "Alice" },
  { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
  // w: "majority" → 과반수 노드에 복제 완료 후 응답
  // j: true → 디스크 저널에 기록 후 응답
)
```

**Read Concern**: 읽기 일관성 수준 설정
```js
db.users.find().readConcern("majority")
// "local": 현재 노드 데이터 (기본값, 빠름)
// "majority": 과반수 노드에 복제된 데이터 (더 안전)
// "linearizable": 가장 최신 데이터 보장 (느림)
```

---

### Sharded Cluster

Sharded Cluster는 대용량 데이터를 여러 Shard(서버 그룹)에 분산 저장하는 구조입니다.

```
             mongos (Router)
Client ──────────────────────→  ┌─────────┐
                                │  Config │
                           ┌────┤  Server ├────┐
                           ↓    └─────────┘    ↓
                        Shard 1             Shard 2
                      (Replica Set)       (Replica Set)
```

| 구성 요소 | 역할 |
|-----------|------|
| **mongos** | 클라이언트 요청을 적절한 Shard로 라우팅 |
| **Config Server** | 클러스터 메타데이터, Shard 키 범위 정보 저장 |
| **Shard** | 실제 데이터 저장, 각 Shard는 Replica Set으로 구성 |

**Shard Key**: 데이터를 어느 Shard에 저장할지 결정하는 기준 필드

```js
// Shard Key 설정 (컬렉션 샤딩)
sh.shardCollection("mydb.orders", { customerId: 1 })  // Range Sharding
sh.shardCollection("mydb.orders", { customerId: "hashed" })  // Hash Sharding
```

| Sharding 방식 | 설명 | 장단점 |
|---------------|------|--------|
| **Range Sharding** | Shard Key 범위로 분배 | 범위 쿼리에 유리, 핫스팟 위험 |
| **Hash Sharding** | Shard Key를 해시하여 분배 | 균등 분배, 범위 쿼리 비효율 |
| **Zone Sharding** | 특정 범위를 특정 Shard에 고정 | 지역별 데이터 분리 등에 활용 |

**CRUD 특징**

| 작업 | 특징 |
|------|------|
| **Insert** | Shard Key 기반으로 적절한 Shard에 저장 |
| **Query (Shard Key 포함)** | Targeted Query: 특정 Shard에만 요청 (효율적) |
| **Query (Shard Key 미포함)** | Scatter-Gather: 모든 Shard에 요청 후 병합 (비효율적) |
| **Update/Delete** | Shard Key 포함 시 Targeted, 미포함 시 Scatter-Gather |
| **Transaction** | 단일 Shard: 일반 트랜잭션 / 여러 Shard: 분산 트랜잭션 (성능 비용) |

**Targeted Query vs Scatter-Gather**
```js
// Targeted Query (권장): Shard Key인 customerId 포함
db.orders.find({ customerId: "user_123" })

// Scatter-Gather (비효율): Shard Key 미포함
db.orders.find({ product: "Laptop" })
```
