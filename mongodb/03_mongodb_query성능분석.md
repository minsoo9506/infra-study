# MongoDB Query 성능 분석

## 목차
- [Query Planner Logic](#1-query-planner-logic)
- [Query Plan 읽기](#2-query-plan-읽기)
- [Query 성능 최적화 - Aggregation](#3-query-성능-최적화---aggregation)
- [Query 성능 최적화 - Index Bound](#4-query-성능-최적화---index-bound)
- [Index 최적화](#5-index-최적화)

---

## 1. Query Planner Logic

MongoDB의 Query Planner는 쿼리를 실행하기 전에 **최적의 실행 계획(Execution Plan)을 선택**하는 컴포넌트다.

### 동작 방식

1. **후보 계획 생성**: 쿼리에 사용 가능한 인덱스를 기반으로 여러 후보 실행 계획을 생성한다.
2. **경쟁 실행 (Trial Period)**: 후보 계획들을 병렬로 소량 실행하여 성능을 비교한다.
3. **승자 선택**: 가장 먼저 일정 수의 문서(기본 101개)를 반환한 계획을 승자로 선택한다.
4. **캐싱**: 선택된 계획은 **Plan Cache**에 저장되어 동일한 쿼리 패턴에 재사용된다.

### Plan Cache 무효화 조건

Plan Cache는 아래 상황에서 초기화된다:

- 컬렉션의 인덱스 추가/삭제
- `mongod` 프로세스 재시작
- `planCacheClear` 명령 수동 실행
- 컬렉션의 데이터 양이 크게 변경될 때 (replan 트리거)

### Query Planner가 선택하는 기준

| 기준 | 설명 |
|------|------|
| Works | 쿼리 처리 중 수행한 총 작업 수 (낮을수록 좋음) |
| 반환 문서 수 | 더 적은 문서를 스캔해서 결과를 낸 계획 선호 |
| 인덱스 활용 여부 | COLLSCAN보다 IXSCAN을 선호 |

---

## 2. Query Plan 읽기

`explain()`을 사용하면 MongoDB가 쿼리를 어떻게 실행하는지 확인할 수 있다.

### explain() 모드

```js
// 실행 계획만 확인 (실제 실행 X)
db.collection.find({ age: 25 }).explain("queryPlanner")

// 실제 실행 후 통계 포함
db.collection.find({ age: 25 }).explain("executionStats")

// 모든 후보 계획 포함
db.collection.find({ age: 25 }).explain("allPlansExecution")
```

### 주요 필드 설명

#### `queryPlanner.winningPlan`

```json
{
  "stage": "FETCH",
  "inputStage": {
    "stage": "IXSCAN",
    "keyPattern": { "age": 1 },
    "indexName": "age_1",
    "direction": "forward",
    "indexBounds": {
      "age": ["[25, 25]"]
    }
  }
}
```

#### `executionStats` 핵심 지표

| 필드 | 의미 |
|------|------|
| `nReturned` | 실제 반환된 문서 수 |
| `totalKeysExamined` | 인덱스에서 스캔한 키 수 |
| `totalDocsExamined` | 실제 문서를 읽은 수 |
| `executionTimeMillis` | 실행 시간 (ms) |

**이상적인 쿼리**: `nReturned == totalKeysExamined == totalDocsExamined`

#### 주요 Stage 종류

| Stage | 의미 |
|-------|------|
| `COLLSCAN` | 컬렉션 전체 스캔 (풀스캔, 느림) |
| `IXSCAN` | 인덱스 스캔 |
| `FETCH` | 인덱스로 찾은 후 실제 문서 로드 |
| `SORT` | 메모리 정렬 (인덱스 정렬이 아님) |
| `PROJECTION` | 필드 선택 |
| `COUNT_SCAN` | count 쿼리 최적화 |

### 나쁜 쿼리 패턴 예시

```js
// COLLSCAN 발생 - 인덱스 없음
db.users.find({ nickname: "minsoo" }).explain("executionStats")
// totalDocsExamined가 nReturned보다 훨씬 크면 비효율적
```

---

## 3. Query 성능 최적화 - Aggregation

Aggregation Pipeline은 여러 단계(stage)를 거쳐 데이터를 변환한다. 각 단계의 순서와 구성이 성능에 큰 영향을 준다.

### 핵심 원칙: 일찍 줄이고, 늦게 연산하라

```
$match -> $project -> $group -> $sort -> $limit
```

### 최적화 규칙

#### 1. `$match`와 `$sort`는 파이프라인 앞에 배치

```js
// 좋음: 먼저 필터링 후 그룹핑
db.orders.aggregate([
  { $match: { status: "completed" } },  // 먼저 줄임
  { $group: { _id: "$userId", total: { $sum: "$amount" } } }
])

// 나쁨: 전체를 그룹핑 후 필터링
db.orders.aggregate([
  { $group: { _id: "$userId", total: { $sum: "$amount" } } },
  { $match: { status: "completed" } }  // 이미 늦음
])
```

#### 2. `$project`로 불필요한 필드 제거

```js
db.users.aggregate([
  { $match: { age: { $gte: 20 } } },
  { $project: { name: 1, age: 1 } },  // 필요한 필드만 유지
  { $group: { _id: "$age", count: { $sum: 1 } } }
])
```

#### 3. `$limit`은 가능한 빨리

```js
db.logs.aggregate([
  { $match: { level: "ERROR" } },
  { $sort: { timestamp: -1 } },
  { $limit: 100 },   // 100개로 줄인 후 이후 처리
  { $lookup: { ... } }
])
```

#### 4. `$lookup` 최적화

`$lookup` (조인) 전에 반드시 `$match`로 문서를 줄인다. 조인 대상 필드에 인덱스가 있어야 성능이 좋다.

```js
db.orders.aggregate([
  { $match: { status: "pending" } },  // 먼저 줄임
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",   // users._id에 인덱스 필수
      as: "userInfo"
    }
  }
])
```

#### 5. `allowDiskUse` 옵션

Aggregation의 각 stage는 기본적으로 **100MB** 메모리 제한이 있다. 초과 시 에러가 발생하므로 대용량 처리 시 사용한다.

```js
db.bigCollection.aggregate([...], { allowDiskUse: true })
```

### Aggregation explain

```js
db.orders.explain("executionStats").aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$userId", total: { $sum: "$amount" } } }
])
```

---

## 4. Query 성능 최적화 - Index Bound

Index Bound는 **인덱스 스캔 시 실제로 탐색하는 범위**를 의미한다. Index Bound가 좁을수록 스캔 범위가 줄어 성능이 향상된다.

### Index Bound란?

`explain()`의 `indexBounds` 필드에서 확인할 수 있다.

```json
"indexBounds": {
  "age": ["[20, 30]"],
  "status": ["[\"active\", \"active\"]"]
}
```

### Bound가 좁아지는 조건

| 연산자 | Bound 효율 |
|--------|-----------|
| `$eq` (=) | 가장 좁음 (정확한 값) |
| `$gt`, `$lt`, `$gte`, `$lte` | 범위 bound |
| `$in` | 여러 개의 정확한 값 |
| `$regex` | prefix가 있으면 좁아짐, 없으면 넓음 |
| `$ne`, `$nin` | 매우 넓음 (거의 전체 스캔) |

### 복합 인덱스와 Index Bound

복합 인덱스 `{ a: 1, b: 1 }` 에서 **a의 조건이 equality일 때만** b의 bound도 활용된다.

```js
// 인덱스: { age: 1, status: 1 }

// 좋음: age가 equality -> status bound도 좁아짐
db.users.find({ age: 25, status: "active" })
// indexBounds: { age: ["[25,25]"], status: ["[\"active\",\"active\"]"] }

// 나쁨: age가 범위 -> status bound는 전체 범위
db.users.find({ age: { $gte: 20 }, status: "active" })
// indexBounds: { age: ["[20, inf]"], status: ["[MinKey, MaxKey]"] }
```

### Index Prefix 활용

복합 인덱스 `{ a: 1, b: 1, c: 1 }` 에서 중간 필드를 건너뛰면 이후 필드의 인덱스는 활용되지 않는다.

```js
// 인덱스 { a: 1, b: 1, c: 1 }

db.col.find({ a: 1, c: 1 })
// c의 인덱스는 활용 안됨 (b를 건너뜀)

db.col.find({ a: 1, b: 1, c: 1 })
// 세 필드 모두 인덱스 활용
```

### `$or` 쿼리와 Index Bound

`$or`는 각 조건마다 별도의 인덱스를 사용하므로 여러 인덱스가 필요할 수 있다.

```js
// 각 조건에 별도 인덱스가 있어야 최적
db.users.find({
  $or: [
    { age: 25 },
    { status: "active" }
  ]
})
```

---

## 5. Index 최적화

### ESR 규칙 (Equality, Sort, Range)

복합 인덱스를 설계할 때 **ESR 순서**를 따르는 것이 최적이다.

```
인덱스 필드 순서: Equality -> Sort -> Range
```

```js
// 쿼리: age=25이고, score 범위 조건, name으로 정렬
db.users.find({ age: 25, score: { $gte: 80 } }).sort({ name: 1 })

// 최적 인덱스: { age: 1, name: 1, score: 1 }
// E: age (equality)
// S: name (sort)
// R: score (range)
db.users.createIndex({ age: 1, name: 1, score: 1 })
```

### Covered Query (커버드 쿼리)

쿼리에 필요한 모든 필드가 인덱스에 포함되어 **실제 문서를 읽지 않아도 되는 쿼리**다. `FETCH` 단계가 없고 `IXSCAN`만 있으면 커버드 쿼리다.

```js
// 인덱스: { age: 1, name: 1 }
db.users.find({ age: 25 }, { name: 1, _id: 0 })
// age, name 모두 인덱스에 있고 _id를 제외 -> FETCH 없이 IXSCAN만으로 처리
```

### 인덱스 선택성 (Selectivity)

**선택성이 높은 필드(카디널리티가 높은 필드)**를 인덱스 앞에 배치해야 스캔 범위가 줄어든다.

```js
// gender (M/F, 선택성 낮음) + userId (선택성 높음)

// 나쁨: 선택성 낮은 필드가 앞에
db.users.createIndex({ gender: 1, userId: 1 })

// 좋음: 선택성 높은 필드가 앞에 (단, ESR 규칙과 함께 고려)
db.users.createIndex({ userId: 1, gender: 1 })
```

### Sparse Index

값이 없는 문서(null 또는 필드 자체가 없는)를 인덱스에서 제외한다. 인덱스 크기를 줄이는 데 유리하다.

```js
db.users.createIndex({ email: 1 }, { sparse: true })
// email 필드가 없는 문서는 인덱스에 포함되지 않음
```

### Partial Index

특정 조건을 만족하는 문서만 인덱스에 포함한다. Sparse Index보다 더 세밀한 제어가 가능하다.

```js
// status가 "active"인 문서만 인덱스
db.users.createIndex(
  { age: 1 },
  { partialFilterExpression: { status: "active" } }
)
```

### TTL Index

특정 시간이 지난 문서를 자동으로 삭제하는 인덱스다. 로그, 세션, 캐시 데이터에 유용하다.

```js
// createdAt 필드 기준으로 3600초(1시간) 후 자동 삭제
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 })
```

### 인덱스 관리 명령어

```js
// 인덱스 목록 확인
db.collection.getIndexes()

// 인덱스 사용 통계 확인
db.collection.aggregate([{ $indexStats: {} }])

// 불필요한 인덱스 삭제
db.collection.dropIndex("index_name")

// Plan Cache 확인 및 초기화
db.collection.getPlanCache().list()
db.collection.getPlanCache().clear()
```

### 주의사항

| 항목 | 설명 |
|------|------|
| 인덱스 개수 | 인덱스가 많을수록 쓰기 성능 저하 (Write마다 인덱스 갱신) |
| 인덱스 크기 | 인덱스는 RAM에 올라가므로 크기 관리 필요 |
| 복합 인덱스 우선 | 단일 인덱스 여러 개보다 잘 설계된 복합 인덱스 하나가 효율적 |
| 미사용 인덱스 제거 | `$indexStats`로 사용되지 않는 인덱스를 주기적으로 정리 |
