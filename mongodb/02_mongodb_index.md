# MongoDB Index

## 목차
- index 기본 구조와 효율적인 탐색
- compound index 와 ESR rule
- multikey index
- index 의 다양한 속성
- index 생성 주의사항

---

## 1. Index 기본 구조와 효율적인 탐색

MongoDB의 인덱스는 **B-Tree** 자료구조를 기반으로 합니다.

- **없을 때**: 쿼리 시 컬렉션 전체를 순차적으로 스캔 (**Collection Scan / COLLSCAN**) → O(n)
- **있을 때**: B-Tree를 통해 빠르게 탐색 (**Index Scan / IXSCAN**) → O(log n)

```js
// 인덱스 생성
db.users.createIndex({ age: 1 })  // 1: 오름차순, -1: 내림차순

// 실행 계획 확인
db.users.find({ age: 25 }).explain("executionStats")
```

**_id 필드**에는 기본적으로 unique index가 자동 생성됩니다.

---

## 2. Compound Index와 ESR Rule

**Compound Index**는 여러 필드를 묶어 하나의 인덱스로 만드는 것입니다.

```js
db.users.createIndex({ status: 1, age: 1, name: 1 })
```

### ESR Rule (Equality → Sort → Range)

Compound Index 필드 순서의 황금 법칙:

| 순서 | 종류 | 설명 |
|------|------|------|
| 1st | **E**quality | 정확히 일치하는 조건 (`status: "active"`) |
| 2nd | **S**ort | 정렬 조건 (`sort({ age: 1 })`) |
| 3rd | **R**ange | 범위 조건 (`age: { $gt: 20 }`) |

**Prefix Rule**: Compound Index `{ a, b, c }`는 `{ a }`, `{ a, b }`, `{ a, b, c }` 쿼리에 활용 가능. `{ b }` 또는 `{ b, c }` 단독으로는 사용 불가.

---

## 3. Multikey Index

**배열 필드**에 인덱스를 걸면 자동으로 Multikey Index가 됩니다.

```js
// tags가 배열인 경우
db.products.createIndex({ tags: 1 })

// { tags: ["mongodb", "database", "nosql"] }
// → 각 배열 요소마다 인덱스 엔트리 생성
```

**주의사항**:
- Compound Index에서 **두 개 이상의 배열 필드**를 함께 인덱싱할 수 없음
- 배열 크기가 클수록 인덱스 엔트리가 많아져 쓰기 성능 저하 가능

---

## 4. Index의 다양한 속성

| 속성 | 설명 | 예시 |
|------|------|------|
| **Unique** | 중복 값 허용 안 함 | `{ unique: true }` |
| **Sparse** | null/존재하지 않는 필드 제외 | `{ sparse: true }` |
| **TTL** | 일정 시간 후 도큐먼트 자동 삭제 | `{ expireAfterSeconds: 3600 }` |
| **Partial** | 조건을 만족하는 도큐먼트만 인덱싱 | `{ partialFilterExpression: { age: { $gt: 18 } } }` |
| **Text** | 전문 검색(Full-text search) | `{ "$**": "text" }` |
| **Wildcard** | 동적 필드 구조에 대응 | `{ "$**": 1 }` |
| **2dsphere** | 지리 공간 쿼리 | GeoJSON 필드에 사용 |

```js
// TTL Index 예시 (로그 자동 삭제)
db.logs.createIndex({ createdAt: 1 }, { expireAfterSeconds: 86400 })
```

---

## 5. Index 생성 주의사항

### 운영 중 인덱스 생성

- **Foreground** 방식 (기본, 구버전): 컬렉션 Lock 발생 → 서비스 중단 위험
- **Background** 방식 또는 `Rolling Index Build` 권장
- MongoDB 4.2+ 부터는 기본적으로 **non-blocking** 방식으로 개선됨

### 인덱스가 많으면 좋을까?

- **읽기 성능** 향상 vs **쓰기 성능** 저하 (INSERT/UPDATE/DELETE 시 인덱스도 갱신)
- **메모리(RAM) 사용량** 증가 → 인덱스는 가능한 RAM에 올라와 있어야 효율적
- 불필요한 인덱스는 주기적으로 제거 필요

### 인덱스 활용 도구

```js
// 인덱스 목록 확인
db.users.getIndexes()

// 미사용 인덱스 파악
db.users.aggregate([{ $indexStats: {} }])

// 느린 쿼리 분석
db.users.find({ ... }).explain("executionStats")
```

---

> 좋은 인덱스 전략: **ESR Rule 준수** + **필요한 인덱스만 최소한으로 유지** + **explain()으로 실행 계획 검증**
