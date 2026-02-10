# HBase

## HBase 데이터모델

HBase는 Google Bigtable 논문을 기반으로 한 **Column-Oriented(Wide Column) Store**이다.

### 핵심 개념: 데이터의 좌표계

HBase에서 하나의 값(Cell)을 찾으려면 **4개의 좌표**가 필요하다:

```
(Row Key, Column Family, Column Qualifier, Timestamp) → Value
```

```
┌─────────────────────────────────────────────────────────────┐
│ Table                                                       │
│                                                             │
│  Row Key   │    Column Family: info    │  Column Family: stats│
│            │  name     │  email        │  login_count        │
│ ───────────┼───────────┼───────────────┼─────────────────────│
│ user001    │  "Alice"  │ "a@mail.com"  │  142                │
│ user002    │  "Bob"    │ "b@mail.com"  │  87                 │
│ user003    │  "Carol"  │  (없음)        │  256                │
│ ───────────┼───────────┼───────────────┼─────────────────────│
│                                                             │
│  * user003의 email은 존재하지 않아도 된다 (Sparse)             │
└─────────────────────────────────────────────────────────────┘
```

### 1. Table

- 데이터를 담는 최상위 컨테이너
- 여러 개의 **Row**로 구성된다
- 물리적으로는 여러 **Region**으로 분할되어 클러스터에 분산 저장된다

### 2. Row Key

Row를 고유하게 식별하는 키이며, HBase 데이터 모델에서 **가장 중요한 설계 요소**이다.

- **사전순(Lexicographic order)** 으로 정렬되어 저장된다
- Row Key 설계가 곧 조회 성능을 결정한다
- byte 배열로 저장되므로 어떤 데이터 타입이든 가능하다

```
Row Key 정렬 예시:

  "aaa"  ← 먼저
  "aab"
  "b"
  "baa"
  "user001"
  "user002"
  "user100"  ← 주의: "user2"보다 뒤에 온다 (사전순)
```

> **주의**: 숫자를 문자열로 저장하면 `"user2"` > `"user100"` 이 된다.
> 따라서 zero-padding(`"user002"`)을 사용하거나, 고정 길이 바이트 인코딩을 적용해야 한다.

### 3. Column Family

컬럼을 **물리적으로 그룹화**하는 단위이다. RDBMS에는 없는 HBase 고유의 개념이다.

- 테이블 생성 시 **미리 정의**해야 한다 (스키마의 일부)
- 출력 가능한 문자로 구성 되어야 한다
- 같은 Column Family의 데이터는 **동일한 HFile에 물리적으로 함께 저장**된다
- Column Family마다 별도의 설정이 가능하다:
  - 압축 방식 (Snappy, LZO, GZIP 등)
  - 블룸 필터 (Bloom Filter)
  - TTL (Time To Live)
  - 최대 버전 수

```
✅ 좋은 설계: Column Family는 2~3개 이내로 유지
❌ 나쁜 설계: Column Family를 수십 개 만드는 것 (Flush/Compaction이 CF 단위로 발생)
```

### 4. Column Qualifier (컬럼)

Column Family 내에서 실제 컬럼을 구분하는 이름이다.

- **사전 정의가 필요 없다** — 언제든 새로운 Qualifier를 추가할 수 있다
- 이것이 HBase가 "Schema-less"라고 불리는 이유이다
- Row마다 서로 다른 Qualifier를 가질 수 있다 (Sparse 구조)

```
Column 전체 이름 = Column Family : Column Qualifier

예시: info:name,  info:email,  stats:login_count
      ────       ────         ─────
      Family     Family       Family
           ────       ─────        ───────────
           Qualifier  Qualifier    Qualifier
```

### 5. Timestamp (버전)

모든 Cell은 **Timestamp를 키의 일부**로 가진다. 이를 통해 동일한 Cell의 여러 버전을 유지할 수 있다.

```
(user001, info:email, t3) → "alice_new@mail.com"   ← 최신
(user001, info:email, t2) → "alice@company.com"
(user001, info:email, t1) → "alice@old.com"         ← 가장 오래됨
```

- 기본적으로 조회 시 **가장 최신 버전**을 반환한다
- `VERSIONS => n` 설정으로 유지할 버전 수를 지정한다
- 너무 많은 수의 버전은 성능부하가 발생한기에 실제로는 잘 사용하지 않거나, 2개 이하로 사용한다.
- 오래된 버전은 Major Compaction 시 삭제된다

### 6. Cell

위 좌표가 가리키는 **최종 값**이다.

- 타입이 없다 — 모든 값은 **byte 배열**로 저장된다
- 직렬화/역직렬화는 클라이언트 책임이다

### RDBMS vs HBase 데이터 모델 비교

| 구분 | RDBMS | HBase |
|------|-------|-------|
| 스키마 | 엄격한 스키마 (DDL 필요) | Column Family만 사전 정의, Qualifier는 자유 |
| 행 구조 | 모든 행이 동일한 컬럼 보유 | 행마다 다른 컬럼 가능 (Sparse) |
| 데이터 타입 | INT, VARCHAR 등 명시적 타입 | 모두 byte[] |
| 정렬 | 인덱스 기반 | Row Key 사전순 자동 정렬 |
| 버전 관리 | 별도 설계 필요 | Timestamp로 자동 버전 관리 |
| NULL 처리 | NULL 값이 공간 차지 | 해당 컬럼이 아예 존재하지 않음 |
| 인덱스 | 다중 보조 인덱스 지원 | Row Key가 유일한 기본 인덱스 |

### 논리적 뷰 vs 물리적 뷰

**논리적 뷰** (사람이 보는 관점):
```
Row Key  │ info:name │ info:email      │ stats:login_count
─────────┼───────────┼─────────────────┼──────────────────
user001  │ Alice     │ a@mail.com      │ 142
user002  │ Bob       │ b@mail.com      │ 87
```

**물리적 뷰** (실제 저장 방식 — Column Family별로 분리):
```
── HFile (Column Family: info) ──────────────────────────
Row Key  │ Column        │ Timestamp │ Value
user001  │ info:name     │ t1        │ Alice
user001  │ info:email    │ t1        │ a@mail.com
user002  │ info:name     │ t1        │ Bob
user002  │ info:email    │ t1        │ b@mail.com

── HFile (Column Family: stats) ─────────────────────────
Row Key  │ Column             │ Timestamp │ Value
user001  │ stats:login_count  │ t1        │ 142
user002  │ stats:login_count  │ t1        │ 87
```

같은 Column Family의 데이터가 물리적으로 함께 저장되기 때문에, **자주 함께 조회되는 컬럼은 같은 Column Family에 배치**하는 것이 성능상 유리하다.