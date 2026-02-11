# HBase

## Row Key 설계

HBase는 Row Key의 사전순(Lexicographic order)으로 데이터를 정렬하고, Row Key 범위 기준으로 Region을 나누어 분산 저장한다. 따라서 **Row Key 설계 = 분산 전략 = 성능**이다.

### Region 분할과 Row Key의 관계

```
Table: user_activity
Row Key 범위에 따라 Region이 자동으로 나뉜다

┌──────────────────────────────────────────────────────────┐
│                     HBase Table                          │
├──────────────────┬──────────────────┬────────────────────┤
│    Region 1      │    Region 2      │    Region 3        │
│  RegionServer A  │  RegionServer B  │  RegionServer C    │
│                  │                  │                    │
│ Row Key: aaa~fff │ Row Key: fff~nnn │ Row Key: nnn~zzz   │
└──────────────────┴──────────────────┴────────────────────┘

→ Row Key가 "abc"인 데이터 → Region 1 (RegionServer A)
→ Row Key가 "hello"인 데이터 → Region 2 (RegionServer B)
→ Row Key가 "zoo"인 데이터 → Region 3 (RegionServer C)
```

클라이언트는 이 과정을 의식할 필요가 없다. HBase가 **META 테이블**을 통해 자동으로 올바른 RegionServer로 라우팅한다.

### 나쁜 Row Key 설계: Hotspot 문제

가장 흔하고 치명적인 실수는 **Hotspot(핫스팟)** 이다. 특정 Region에 쓰기/읽기가 집중되는 현상이다.

#### 예시: 타임스탬프를 Row Key로 사용

```
Row Key = timestamp

2024-01-01T00:00:01
2024-01-01T00:00:02
2024-01-01T00:00:03   ← 시간순이라 항상 마지막 Region에 쓰기 집중
...
2024-01-01T23:59:59

┌──────────────────┬──────────────────┬────────────────────┐
│    Region 1      │    Region 2      │    Region 3        │
│   (한가함)        │   (한가함)        │   (과부하!!!)       │
│                  │                  │  ← 모든 신규 데이터  │
│ 2024-01~2024-04  │ 2024-04~2024-08  │ 2024-08~현재        │
└──────────────────┴──────────────────┴────────────────────┘
```

시간은 항상 증가하므로 최신 데이터가 **항상 마지막 Region 한 곳에만** 몰린다. 나머지 RegionServer들은 놀고 있게 된다.

#### 예시: 순차적 ID를 Row Key로 사용

```
Row Key = auto_increment_id

00000001
00000002
...
99999999   ← 역시 마지막 Region에만 집중

→ 타임스탬프와 같은 문제 발생
```

### 좋은 Row Key 설계 기법

#### 기법 1: Salting (솔팅)

Row Key 앞에 **랜덤 접두사(salt)** 를 붙여 데이터를 강제로 분산시킨다.

```
원본 Row Key:  2024-01-01T12:00:00_user001
Salt 적용:     hash("2024-01-01T12:00:00_user001") % 10 = 3

최종 Row Key:  3_2024-01-01T12:00:00_user001
```

```
Salt 값에 따라 자동으로 여러 Region에 분산:

0_2024-01-01T12:00:00_user005  → Region 1
1_2024-01-01T12:00:02_user003  → Region 1
2_2024-01-01T12:00:01_user007  → Region 2
3_2024-01-01T12:00:00_user001  → Region 2
4_2024-01-01T12:00:03_user002  → Region 3
...

→ 쓰기가 모든 Region에 고르게 분산된다
```

- **장점**: 쓰기 분산 효과가 확실하다
- **단점**: 범위 스캔(Scan)이 어려워진다. 특정 시간대 데이터를 조회하려면 모든 salt 값(0~9)에 대해 각각 스캔해야 한다

#### 기법 2: Hashing (해싱)

Row Key 자체를 해시값으로 변환한다.

```
원본:     user001
MD5 해시: a1b2c3d4e5f6...

Row Key = MD5("user001") = a1b2c3d4e5f6...
```

- **장점**: 데이터가 매우 균등하게 분산된다
- **단점**: 원본 값의 순서가 완전히 사라진다. Get(단건 조회)만 가능하고 범위 스캔이 사실상 불가능하다

#### 기법 3: Reversing (키 뒤집기)

Row Key를 뒤집어서 저장한다. 앞부분이 고정되고 뒷부분만 변하는 데이터에 효과적이다.

```
원래 Row Key (전화번호):
  010-1234-0001
  010-1234-0002    ← 앞자리 "010-1234"가 동일 → 한 Region에 몰림
  010-1234-0003

뒤집은 Row Key:
  1000-4321-010
  2000-4321-010    ← 앞자리가 달라져서 분산됨
  3000-4321-010
```

- **장점**: 구현이 간단하고, 원본 데이터 복원이 쉽다
- **단점**: 뒤집힌 순서로 정렬되므로 원래 순서의 범위 스캔이 불가능하다

#### 기법 4: Composite Key (복합 키)

여러 필드를 조합하여 Row Key를 구성한다. 가장 실무에서 많이 사용되는 패턴이다.

```
Row Key = <user_id>_<reverse_timestamp>

예시:
  user001_9999999999-1704067200  (2024-01-01 12:00:00)
  user001_9999999999-1704067260  (2024-01-01 12:01:00)
  user002_9999999999-1704067200  (2024-01-01 12:00:00)

→ user_id로 분산 + reverse_timestamp로 최신 데이터가 먼저 조회
```

**Reverse Timestamp**란?
```
reverse_ts = Long.MAX_VALUE - timestamp

→ 최신 데이터일수록 값이 작아진다
→ 사전순 정렬에서 최신 데이터가 앞에 온다
→ Scan 시 자연스럽게 "최신순" 조회가 된다
```

### 실전 예시: SNS 타임라인 서비스

**요구사항**: 특정 유저의 최신 게시물을 빠르게 조회하고 싶다

```
❌ 나쁜 설계: Row Key = post_id (순차 증가)
   → 모든 쓰기가 마지막 Region에 집중 (Hotspot)
   → 특정 유저의 게시물만 조회하기 어렵다

❌ 아쉬운 설계: Row Key = user_id + timestamp
   → 유저별로는 묶이지만, 최신 글 조회 시 전체 Scan 필요

✅ 좋은 설계: Row Key = user_id + reverse_timestamp
   → 유저별 데이터가 연속 저장 (Scan 효율적)
   → 최신 글이 앞에 정렬 (소량만 Scan하면 됨)
   → user_id가 다양하므로 Region 분산도 자연스럽다
```

```
Row Key                              │ Column          │ Value
user001_9999999998295732800          │ post:content    │ "오늘 점심 맛집 발견!"
user001_9999999998295732860          │ post:content    │ "날씨가 좋다"
user001_9999999998295732920          │ post:content    │ "HBase 공부 중"
user002_9999999998295732800          │ post:content    │ "새해 복 많이 받으세요"
...

→ Scan(startRow="user001_", stopRow="user001_~", limit=10)
→ user001의 최신 게시물 10개를 즉시 조회 가능
```

### Row Key 설계 체크리스트

```
1. Hotspot이 발생하지 않는가?
   → 순차적 키(timestamp, auto_increment)를 단독으로 사용하지 않았는가

2. 주요 조회 패턴과 일치하는가?
   → 가장 빈번한 쿼리가 Row Key prefix로 처리 가능한가

3. Row Key 길이가 적절한가?
   → Row Key는 모든 Cell에 반복 저장되므로 짧을수록 좋다
   → 일반적으로 10~100 bytes 권장

4. 분산이 균등한가?
   → 특정 prefix에 데이터가 편중되지 않는가

5. Scan 패턴을 고려했는가?
   → 범위 조회가 필요한 필드가 Row Key에 포함되어 있는가
   → 정렬 순서가 조회 요구사항과 일치하는가
```
