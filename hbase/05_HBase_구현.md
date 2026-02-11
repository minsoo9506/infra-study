# HBase

## 구현

### HFile 특징 & 원리

**HFile**은 HBase가 디스크에 데이터를 저장하는 파일 포맷으로, HDFS 위에 저장됩니다. Google Bigtable의 SSTable(Sorted String Table)에서 영감을 받았습니다.

#### HFile 구조

```
┌──────────────────────────────┐
│        Data Blocks           │  ← 실제 KeyValue 데이터 (정렬된 상태)
│  ┌────────────────────────┐  │
│  │ Block 1 (64KB 기본)    │  │
│  │  Key1:Value1           │  │
│  │  Key2:Value2           │  │
│  │  ...                   │  │
│  ├────────────────────────┤  │
│  │ Block 2                │  │
│  │  Key100:Value100       │  │
│  │  ...                   │  │
│  └────────────────────────┘  │
├──────────────────────────────┤
│        Meta Blocks           │  ← Bloom Filter 데이터
├──────────────────────────────┤
│     Meta Index Block         │  ← Meta Block의 인덱스
├──────────────────────────────┤
│     Data Index Block         │  ← Data Block의 인덱스 (각 블록의 첫 번째 key)
├──────────────────────────────┤
│        File Info             │  ← 메타정보 (AVG_KEY_LEN, comparator 등)
├──────────────────────────────┤
│        Trailer              │  ← 각 섹션의 오프셋, 버전 정보
└──────────────────────────────┘
```

#### HFile의 핵심 특징

| 특징 | 설명 |
|---|---|
| **정렬 저장** | 모든 KeyValue가 row key → column family → qualifier → timestamp(내림차순) 순으로 정렬 |
| **불변(Immutable)** | 한번 쓰여진 HFile은 절대 수정되지 않음. 갱신/삭제는 새로운 파일로 처리 |
| **블록 단위 I/O** | 기본 64KB 블록 단위로 읽기 수행, 블록 단위 압축/캐싱 가능 |
| **Bloom Filter** | 특정 row key가 이 HFile에 존재하는지 빠르게 판별 (불필요한 디스크 I/O 방지) |
| **Block Index** | 이진 탐색으로 원하는 Data Block을 O(log N)에 찾음 |
| **압축 지원** | Snappy, LZ4, GZip, ZSTD 등 블록 단위 압축 |

#### KeyValue 내부 구조

HFile에 저장되는 하나의 셀(Cell)은 다음과 같은 구조입니다.

```
┌─────────┬───────────┬──────────┬────────────────┬───────────┬───────┬───────┐
│ Row Len │  Row Key  │ CF Len   │ CF + Qualifier │ Timestamp │ Type  │ Value │
│ (4byte) │ (가변)    │ (1byte)  │ (가변)         │ (8byte)   │(1byte)│(가변) │
└─────────┴───────────┴──────────┴────────────────┴───────────┴───────┴───────┘
```

- **Type**: Put(4), Delete(8), DeleteColumn(12), DeleteFamily(14)
- Key에 row key, column family, qualifier, timestamp가 모두 포함되어 있어서 HFile만으로 자체 완결적

#### 읽기 최적화 메커니즘

```
조회 요청 (row key = "user_001")
    ↓
① Bloom Filter 체크 → "이 HFile에 없음" → 스킵 (디스크 I/O 절약)
    ↓ (있을 수 있음)
② Block Index 이진 탐색 → 해당 Data Block 오프셋 찾기
    ↓
③ Block Cache 확인 → 캐시 히트 시 디스크 읽기 생략
    ↓ (캐시 미스)
④ HDFS에서 Data Block 읽기 → 블록 내 순차 탐색
```

---

### 데이터 갱신 과정

#### Write Path (쓰기 경로)

```
Client Put 요청
    ↓
┌─────────────────────────────────────┐
│ 1. WAL(Write-Ahead Log) 기록       │  ← 디스크 (순차 쓰기, 빠름)
│    - 장애 복구용 Redo Log           │
│    - RegionServer당 1개의 WAL 파일  │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ 2. MemStore에 기록                  │  ← 메모리 (ConcurrentSkipListMap)
│    - Column Family별 1개            │
│    - 정렬된 상태로 유지              │
└─────────────────────────────────────┘
    ↓ (클라이언트에 성공 응답)
    ↓
    ↓ MemStore가 임계값(128MB 기본)에 도달하면
    ↓
┌─────────────────────────────────────┐
│ 3. Flush → HFile 생성              │  ← HDFS에 새 HFile 기록
│    - MemStore의 정렬된 데이터를      │
│      그대로 순차 기록 (매우 빠름)    │
│    - flush 후 MemStore 비움         │
│    - 대응하는 WAL 엔트리 삭제        │
└─────────────────────────────────────┘
```

#### 갱신(Update)과 삭제(Delete)가 처리되는 방식

HBase는 **제자리 갱신(in-place update)을 하지 않습니다**. 모든 변경은 새로운 데이터 추가입니다.

```
시점1: Put(row1, cf:name, ts=100, "Alice")   → HFile-1에 저장
시점2: Put(row1, cf:name, ts=200, "Bob")     → HFile-2에 저장 (갱신)
시점3: Delete(row1, cf:name, ts=300)         → HFile-3에 tombstone 마커 저장
```

**읽기 시** 모든 HFile + MemStore를 병합하여 가장 최신 타임스탬프의 값을 반환합니다.

```
Read(row1, cf:name)
    ↓
MemStore에서 검색 + HFile-3 + HFile-2 + HFile-1 병합
    ↓
ts=300 Delete 발견 → 이 셀은 삭제됨 → 결과 없음
```

#### Compaction (병합)

flush가 반복되면 HFile이 계속 쌓이므로, 읽기 성능이 저하됩니다. 이를 해결하는 것이 Compaction입니다.

**Minor Compaction**

```
HFile-1 ─┐
HFile-2 ─┼──→ 병합 ──→ HFile-new
HFile-3 ─┘
```

- 작은 HFile 여러 개를 하나로 병합
- tombstone은 **유지**됨 (아직 삭제하지 않음)
- 비교적 가볍고 자주 수행

**Major Compaction**

```
HFile-A ─┐
HFile-B ─┼──→ 병합 ──→ HFile-final (1개)
HFile-C ─┘
  - 삭제된 데이터(tombstone) 물리 제거
  - 만료된 버전(TTL/max_versions 초과) 제거
  - 해당 Region의 모든 HFile → 1개로 통합
```

- 무거운 작업 (전체 데이터 재작성)
- 기본 7일 주기로 자동 실행
- 이 시점에서야 디스크 공간이 실제로 회수됨

#### Read Path (읽기 경로) 전체 흐름

```
Client Get/Scan 요청
    ↓
┌─────────────────────────────┐
│ 1. MemStore 검색            │  ← 메모리 (가장 최신 데이터)
└─────────────────────────────┘
    ↓
┌─────────────────────────────┐
│ 2. BlockCache 검색          │  ← 메모리 (자주 읽는 HFile 블록 캐시)
└─────────────────────────────┘
    ↓
┌─────────────────────────────┐
│ 3. HFile 검색               │  ← 디스크 (Bloom Filter → Index → Data Block)
│    - 최신 HFile부터 탐색    │
│    - Bloom Filter로 빠른    │
│      존재 여부 판별         │
└─────────────────────────────┘
    ↓
결과 병합 (가장 최신 timestamp 기준)
    ↓
클라이언트에 응답
```

#### 전체 과정 요약

```
Write: Client → WAL → MemStore → (flush) → HFile → (compaction) → 병합된 HFile
Read:  Client → MemStore + BlockCache + HFiles → 병합 → 최신 값 반환
```

| 단계 | 매체 | 역할 |
|---|---|---|
| WAL | 디스크 (순차쓰기) | 장애 복구 보장 |
| MemStore | 메모리 | 쓰기 버퍼, 정렬 유지 |
| Flush | 메모리 → 디스크 | MemStore → HFile 변환 |
| HFile | 디스크 (HDFS) | 영구 저장, 불변 파일 |
| Minor Compaction | 디스크 | 작은 HFile 병합, 읽기 성능 유지 |
| Major Compaction | 디스크 | 전체 병합, 삭제 데이터 물리 제거 |
| BlockCache | 메모리 | HFile 블록 읽기 캐시 |
