# MongoDB 개요

## 목차
- [SQL vs NoSQL](#sql-vs-nosql)
- [MongoDB 구조](#mongodb-구조)
  - [MongoDB 용어](#mongodb-용어)
  - [기본 database (admin, local, config)](#기본-database)
  - [Collection 특징](#collection-특징)
  - [Document 특징](#document-특징)
- [MongoDB 배포 형태](#mongodb-배포-형태)
- [Replica Set](#replica-set)
- [Sharded Cluster](#sharded-cluster)
- [Replica Set vs Sharded Cluster 배포 방식](#replica-set-vs-sharded-cluster-배포-방식)
- [MongoDB Storage Engines](#mongodb-storage-engines)

---

## SQL vs NoSQL

| 항목 | SQL (RDBMS) | NoSQL (MongoDB) |
|------|------------|-----------------|
| 데이터 모델 | 테이블 (행/열) | 문서 (JSON/BSON) |
| 스키마 | 고정 스키마 | 유연한 스키마 |
| 관계 | JOIN으로 연결 | 중첩 문서 또는 참조 |
| 확장 방식 | 수직 확장 (Scale-Up) | 수평 확장 (Scale-Out) |
| 트랜잭션 | 강한 ACID 보장 | 단일 문서 원자성 (다중 문서는 v4.0+) |
| 쿼리 언어 | SQL | MQL (MongoDB Query Language) |
| 적합한 케이스 | 정형 데이터, 복잡한 관계 | 비정형 데이터, 빠른 개발, 대용량 |

**NoSQL을 선택하는 이유**
- 스키마가 자주 바뀌는 요구사항
- 대용량 데이터의 수평 확장 필요
- 계층적/중첩 구조의 데이터 표현이 자연스러울 때
- 빠른 읽기/쓰기 처리량이 필요할 때

---

## MongoDB 구조

### MongoDB 용어

RDB와 MongoDB의 개념 대응표:

| RDB | MongoDB |
|-----|---------|
| Database | Database |
| Table | Collection |
| Row | Document |
| Column | Field |
| Index | Index |
| JOIN | $lookup (Aggregation) |
| Primary Key | `_id` |

### 기본 Database

MongoDB 설치 시 자동으로 생성되는 3개의 내장 DB:

| Database | 역할 |
|----------|------|
| `admin` | 관리자 계정, 권한 관리. 이 DB에 접속한 사용자만 서버 전체 관리 명령 실행 가능 |
| `local` | Replica Set의 oplog 저장. 복제되지 않으며 해당 노드에만 존재 |
| `config` | Sharded Cluster에서 샤딩 메타데이터(청크, 라우팅 정보) 저장 |

### Collection 특징

- **스키마 없음(Schema-less)**: 같은 Collection 안에 구조가 다른 Document가 공존 가능
- **동적 생성**: 명시적으로 생성하지 않아도 첫 Document 삽입 시 자동 생성
- **Capped Collection**: 크기 제한을 설정하면 FIFO 방식으로 오래된 데이터 자동 삭제 (로그 저장 등에 활용)
- **Validator**: JSON Schema를 이용해 선택적으로 스키마 유효성 검사 적용 가능

### Document 특징

#### BSON (Binary JSON)

- MongoDB가 내부적으로 데이터를 저장하는 포맷
- JSON의 상위 집합으로, JSON에 없는 타입 지원: `Date`, `ObjectId`, `Binary`, `Decimal128`, `int32/int64` 등
- 텍스트 JSON보다 파싱이 빠르고, 타입 정보를 명시적으로 포함

#### `_id` 필드

- 모든 Document는 반드시 `_id` 필드를 가짐
- 삽입 시 `_id`를 명시하지 않으면 MongoDB가 **ObjectId** 자동 생성
- **ObjectId 구조** (12 bytes):
  ```
  [4 bytes: timestamp] [5 bytes: random] [3 bytes: incrementing counter]
  ```
  - 생성 시각이 포함되어 있어 `_id` 정렬 = 삽입 시간 순 정렬
- Collection 내에서 유일함을 보장 (Unique Index 자동 생성)

#### 16MB 제한

- 단일 Document의 최대 크기는 **16MB**
- 대용량 파일(이미지, 동영상 등) 저장 시 **GridFS** 사용 (16MB 초과 파일을 청크로 분할 저장)

---

## MongoDB 배포 형태

### Standalone

- 단일 mongod 프로세스로 구성
- 고가용성, 복제 없음
- 개발/테스트 환경에 적합, 프로덕션 비권장

### Replica Set

- 동일 데이터를 복제하는 mongod 그룹
- 고가용성(HA) 및 데이터 이중화 목적
- 프로덕션 환경의 기본 배포 단위

### Sharded Cluster

- 데이터를 여러 Shard(Replica Set)로 분산 저장
- 수평 확장(Scale-Out)을 통한 대용량 처리
- Replica Set + 라우터(mongos) + Config Server로 구성

---

## Replica Set

하나의 Primary와 하나 이상의 Secondary로 구성된 mongod 그룹.

```
          Write/Read
Client ──────────────► Primary
                          │ oplog 복제
                ┌─────────┴─────────┐
             Secondary           Secondary
           (Read 가능)          (Read 가능)
```

### 구성 멤버

| 멤버 | 역할 |
|------|------|
| **Primary** | 모든 쓰기 요청 처리. oplog에 변경사항 기록 |
| **Secondary** | Primary의 oplog를 비동기적으로 복제. 읽기 분산 가능 |
| **Arbiter** | 데이터 저장 없이 투표에만 참여. 홀수 멤버 수 유지용 |

### 자동 장애 복구 (Automatic Failover)

- Primary 장애 시 Secondary들이 **선거(Election)** 를 통해 새 Primary 선출
- 과반수 투표(majority)로 결정 → 최소 3개 멤버 권장 (홀수 유지)
- 선출 완료까지 보통 10~30초 소요

### oplog (Operations Log)

- `local.oplog.rs` Collection에 저장되는 Capped Collection
- Primary의 모든 쓰기 연산을 기록, Secondary가 이를 읽어 동일 연산 재수행
- 크기가 고정되어 있어 오래된 oplog는 덮어써짐 (Secondary 복제 지연 시 동기화 불가 위험)

### Read Preference

Secondary에서도 읽기 가능. 옵션:

| 옵션 | 설명 |
|------|------|
| `primary` (기본값) | 항상 Primary에서 읽기 |
| `primaryPreferred` | Primary 우선, 불가 시 Secondary |
| `secondary` | 항상 Secondary에서 읽기 |
| `secondaryPreferred` | Secondary 우선, 불가 시 Primary |
| `nearest` | 네트워크 지연이 가장 낮은 멤버 |

> Secondary 읽기는 복제 지연으로 인해 최신 데이터를 보장하지 않음 (Eventual Consistency)

---

## Sharded Cluster

대용량 데이터를 여러 Shard에 분산 저장하는 구조.

```
Client
  │
  ▼
mongos (Query Router)
  │
  ├──► Shard 1 (Replica Set)
  ├──► Shard 2 (Replica Set)
  └──► Shard 3 (Replica Set)
       ▲
Config Servers (Replica Set)
(샤딩 메타데이터 저장)
```

### 구성 요소

| 구성 요소 | 역할 |
|-----------|------|
| **Shard** | 실제 데이터를 저장하는 Replica Set |
| **mongos** | 클라이언트 요청을 받아 적절한 Shard로 라우팅하는 쿼리 라우터 |
| **Config Server** | 샤딩 메타데이터(청크 정보, 라우팅 테이블) 저장. Replica Set으로 구성 |

### Shard Key

- 데이터를 어느 Shard에 저장할지 결정하는 기준 필드
- 한 번 설정하면 변경 불가 → **신중하게 선택**
- 좋은 Shard Key 조건:
  - 높은 카디널리티(cardinality)
  - 균등한 쓰기 분산
  - 쿼리 패턴과 일치 (targeted query 가능)

### 청크 (Chunk)

- Shard Key 범위를 기준으로 데이터를 나눈 단위 (기본 128MB)
- **Balancer**가 백그라운드에서 Shard 간 청크 수를 균등하게 유지

### 샤딩 방식

#### Range Sharding
- Shard Key의 **값 범위**를 기준으로 청크를 분할
- 예: `age 0~30` → Shard 1, `age 31~60` → Shard 2, `age 61~` → Shard 3
- 범위 쿼리(`$gt`, `$lt`)에 효율적 (targeted query 가능)
- 단점: Shard Key가 단조 증가하는 값(timestamp, ObjectId 등)이면 특정 Shard에만 쓰기가 몰리는 **핫스팟** 발생

#### Hashed Sharding
- Shard Key 값을 **해시 변환**한 결과를 기준으로 분산
- 데이터가 Shard 전체에 균등하게 분산되어 핫스팟 방지
- 단점: 범위 쿼리 시 모든 Shard를 조회하는 **scatter-gather** 발생
- 단조 증가 필드(ObjectId 등)를 Shard Key로 써야 할 때 적합

#### Zone Sharding
- 특정 Shard Key 범위를 특정 Shard(Zone)에 **고정** 배치
- 예: 지역별 데이터를 해당 지역 데이터센터의 Shard에만 저장
- 데이터 지역성(Locality), 규정 준수(GDPR 등), 하드웨어 티어 분리에 활용
- Range/Hashed Sharding과 함께 사용 가능

| 방식 | 분산 기준 | 범위 쿼리 | 핫스팟 위험 | 주 용도 |
|------|-----------|-----------|-------------|---------|
| Range | Shard Key 값 범위 | 효율적 | 있음 | 범위 탐색이 많은 경우 |
| Hashed | Shard Key 해시값 | 비효율 (scatter-gather) | 없음 | 균등 분산이 중요한 경우 |
| Zone | 사용자 정의 범위 고정 | 경우에 따라 다름 | 설계에 따라 | 지역성/규정 준수 |

---

## Replica Set vs Sharded Cluster 배포 방식

| 항목 | Replica Set | Sharded Cluster |
|------|-------------|-----------------|
| 목적 | 고가용성, 데이터 이중화 | 수평 확장, 대용량 처리 |
| 구성 복잡도 | 낮음 | 높음 |
| 데이터 분산 | 모든 노드가 전체 데이터 보유 | Shard별로 데이터 분산 |
| 쓰기 확장 | 불가 (Primary 단일) | 가능 (Shard별 분산) |
| 읽기 확장 | Secondary로 분산 가능 | 각 Shard의 Secondary 활용 |
| 운영 비용 | 낮음 | 높음 |
| 권장 데이터 크기 | ~수 TB | 수 TB 이상 |

**선택 기준**
- 데이터가 단일 서버에 들어가고 쓰기 처리량이 충분하다면 → **Replica Set**
- 단일 서버 용량을 초과하거나 쓰기 처리량이 병목이라면 → **Sharded Cluster**

---

## MongoDB Storage Engines

Storage Engine은 데이터의 메모리/디스크 저장 방식을 담당하는 컴포넌트.

### WiredTiger (기본값, v3.2+)

| 특징 | 내용 |
|------|------|
| **압축** | Snappy(기본), zlib, zstd 압축 지원으로 디스크 사용량 절감 |
| **동시성** | Document 수준 잠금(Lock)으로 높은 동시 쓰기 성능 |
| **캐시** | WiredTiger 내부 캐시 (기본: RAM의 50% 또는 256MB 중 큰 값) |
| **체크포인트** | 60초마다 디스크에 일관된 스냅샷 기록 |
| **저널링** | WAL(Write-Ahead Log) 방식으로 크래시 복구 지원 |

### In-Memory Storage Engine

- 데이터를 메모리에만 저장 (디스크 영속성 없음)
- 극도로 낮은 지연시간 필요 시 사용
- MongoDB Enterprise 전용
