# HBase

## 아키텍쳐

### 전체 아키텍쳐 개요

```
┌─────────────────────────────────────────────────────────────────┐
│                          Client                                 │
│              (HBase Shell / Java API / Thrift)                  │
└────────┬──────────────────────────────────┬─────────────────────┘
         │ ① meta 테이블 위치 조회            │ ③ 데이터 읽기/쓰기
         ▼                                   ▼
┌─────────────────┐               ┌─────────────────────────────┐
│   ZooKeeper     │               │      RegionServer           │
│                 │               │  ┌───────────────────────┐  │
│ - Master 선출   │               │  │ Region (Table 파티션) │  │
│ - meta 위치 관리│               │  │  ├─ Store (CF별)      │  │
│ - RS 생존 감지  │               │  │  │  ├─ MemStore       │  │
│                 │               │  │  │  └─ StoreFile(HFile)│ │
│                 │               │  │  └─ WAL              │  │
│                 │               │  └───────────────────────┘  │
└────────┬────────┘               └──────────────┬──────────────┘
         │                                       │
         │ ② Region 할당/밸런싱                    │ ④ 데이터 저장
         ▼                                       ▼
┌─────────────────┐               ┌─────────────────────────────┐
│   HMaster       │               │           HDFS              │
│                 │               │                             │
│ - Region 할당   │               │  - HFile 영구 저장           │
│ - 로드 밸런싱   │               │  - WAL 저장                  │
│ - DDL 처리      │               │  - 3중 복제로 내구성 보장     │
│ - 장애 복구     │               │                             │
└─────────────────┘               └─────────────────────────────┘
```

| 구성 요소 | 역할 |
|---|---|
| **Client** | ZooKeeper를 통해 meta 테이블 위치를 알아낸 후, RegionServer에 직접 요청 |
| **ZooKeeper** | 클러스터 코디네이션 — Master 선출, RegionServer 생존 감지, meta 테이블 위치 관리 |
| **HMaster** | DDL 처리(테이블 생성/삭제), Region 할당 및 로드 밸런싱, 장애 복구 |
| **RegionServer** | 실제 데이터 읽기/쓰기 처리. 여러 Region을 호스팅하며 각 Region은 MemStore + HFile로 구성 |
| **HDFS** | HFile과 WAL을 영구 저장하는 분산 파일 시스템 |

#### 구성 요소 간 통신 프로토콜

| 구간 | 프로토콜 | 특징 |
|---|---|---|
| Client → RegionServer | **Protobuf RPC** | 직접 연결. ZooKeeper나 Master를 거치지 않음 |
| RegionServer → HDFS | **HDFS Client** | WAL과 HFile 모두 HDFS에 기록 |
| RegionServer → ZooKeeper | **ZooKeeper Session** | 하트비트로 생존 보고. 데이터 전송은 없음 |
| HMaster → RegionServer | **Protobuf RPC** | Region 할당/이동 명령 전송 |

#### Region 이동 (로드 밸런싱)

HMaster가 RegionServer 간 부하를 균등하게 분배하기 위해 Region을 이동시킵니다.

```
[상황] RegionServer-1에 Region이 몰려있음

HMaster의 Balancer가 감지
    ↓
① RegionServer-1에 Region-5를 unassign 명령
    ↓
② RegionServer-1이 Region-5의 MemStore를 flush → HFile 생성
    ↓
③ Region-5를 close
    ↓
④ HMaster가 RegionServer-3에 Region-5를 assign
    ↓
⑤ RegionServer-3이 Region-5를 open
    - HDFS에서 HFile을 읽음 (데이터 복사 없음! HFile은 HDFS에 있으므로)
    - 새로운 MemStore 초기화
    ↓
⑥ 클라이언트 캐시 무효화 → 다음 요청 시 meta 테이블 재조회
```

**핵심**: Region 이동 시 **데이터 복사가 발생하지 않습니다**. HFile은 HDFS에 저장되어 있어 어떤 RegionServer에서든 접근 가능합니다. 이동되는 것은 "어떤 RegionServer가 이 Region을 서빙하느냐"라는 메타데이터뿐입니다.

---

### B+ 트리 — 전통적 RDBMS의 인덱스 구조

B+ 트리는 MySQL InnoDB, PostgreSQL 등 전통적 RDBMS가 사용하는 인덱스 자료구조입니다.

#### B+ 트리 구조

```
                    ┌─────────────────┐
                    │   [30 | 60]     │  ← Root (내부 노드)
                    └──┬─────┬─────┬──┘
                       │     │     │
            ┌──────────┘     │     └──────────┐
            ▼                ▼                ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │ [10 | 20]  │  │ [40 | 50]  │  │ [70 | 80]  │  ← 내부 노드
     └─┬───┬───┬──┘  └─┬───┬───┬──┘  └─┬───┬───┬──┘
       │   │   │        │   │   │        │   │   │
       ▼   ▼   ▼        ▼   ▼   ▼        ▼   ▼   ▼
      [1] [15] [25]    [35] [45] [55]    [65] [75] [90]  ← 리프 노드 (데이터)
       └──→└──→└──→     └──→└──→└──→     └──→└──→         ← 리프 간 연결 리스트
```

#### B+ 트리의 특징

| 특징 | 설명 |
|---|---|
| **구조** | 균형 트리(Balanced Tree). 루트에서 모든 리프까지 깊이가 동일 |
| **데이터 위치** | 실제 데이터는 리프 노드에만 저장. 내부 노드는 키(라우팅 정보)만 보유 |
| **리프 연결** | 리프 노드끼리 연결 리스트로 연결 → 범위 스캔(Range Scan)에 효율적 |
| **읽기 성능** | O(log N) — 트리 깊이만큼 디스크 I/O. 수억 건도 3~4번의 I/O로 탐색 |
| **쓰기 성능** | O(log N) — 단, **랜덤 I/O 발생**. 리프 노드를 찾아가 제자리 갱신(in-place update) |

#### B+ 트리의 쓰기 과정

```
Insert(key=45, value="data")
    ↓
① 루트부터 리프까지 탐색 (랜덤 읽기 I/O)
    ↓
② 해당 리프 노드에 데이터 삽입 (랜덤 쓰기 I/O)
    ↓
③ 노드가 꽉 차면 분할(split) 발생 → 부모 노드도 갱신
```

**핵심 문제점**: 쓰기마다 디스크의 **랜덤 위치**에 접근해야 합니다. 디스크(HDD/SSD) 특성상 랜덤 I/O는 순차 I/O보다 수십~수백 배 느립니다. 대규모 쓰기 워크로드에서 병목이 됩니다.

---

### LSM 트리 — HBase의 핵심 자료구조

**LSM 트리(Log-Structured Merge Tree)**는 쓰기 성능을 극대화하기 위해 설계된 자료구조입니다. HBase, Cassandra, RocksDB, LevelDB 등이 사용합니다.

#### LSM 트리의 핵심 아이디어

> **"랜덤 쓰기를 순차 쓰기로 변환한다"**

디스크에 직접 랜덤 쓰기를 하지 않고, 메모리에 버퍼링한 후 순차적으로 디스크에 flush합니다.

#### LSM 트리 구조와 동작

```
                        ┌─────────────────────────┐
  Write ──────────────→ │  MemTable (메모리)        │  ← 정렬된 구조 (SkipList)
                        │  - 모든 쓰기가 여기로     │
                        │  - 읽기도 여기서 먼저 탐색 │
                        └────────────┬────────────┘
                                     │ 임계값 도달 시 flush
                                     ▼
                   ┌──────────────────────────────────┐
          Level 0  │  SSTable-1  SSTable-2  SSTable-3  │  ← 최근 flush된 파일들
                   │  (각각 내부 정렬, 파일 간 겹침 가능) │
                   └──────────────────┬───────────────┘
                                      │ Minor Compaction
                                      ▼
          Level 1  ┌──────────────────────────────────┐
                   │  SSTable-A       SSTable-B        │  ← 파일 간 키 범위 겹침 없음
                   │  (aaa~mmm)       (nnn~zzz)        │
                   └──────────────────┬───────────────┘
                                      │ Major Compaction
                                      ▼
          Level 2  ┌──────────────────────────────────┐
                   │  SSTable-X  SSTable-Y  SSTable-Z  │  ← 더 큰 단위로 병합
                   └──────────────────────────────────┘
```

#### LSM 트리의 쓰기 과정

```
Put(key="user_100", value="Alice")
    ↓
① WAL 기록 (디스크 순차 쓰기 — 장애 복구용)
    ↓
② MemTable에 삽입 (메모리 — 즉시 완료, 클라이언트에 응답)
    ↓
    ... (MemTable이 임계값에 도달하면) ...
    ↓
③ MemTable → SSTable(HFile)로 flush (디스크 순차 쓰기)
    - 이미 메모리에서 정렬되어 있으므로 그대로 순차 기록
    - 랜덤 I/O가 전혀 발생하지 않음
```

#### LSM 트리의 읽기 과정

```
Get(key="user_100")
    ↓
① MemTable 검색 (메모리 — 가장 최신 데이터)
    ↓ 없으면
② Level 0 SSTable들 검색 (Bloom Filter로 빠른 판별)
    ↓ 없으면
③ Level 1 → Level 2 → ... 순서로 탐색
    ↓
결과 반환 (여러 레벨에서 동일 키 발견 시 가장 최신 값 반환)
```

**읽기의 단점**: 최악의 경우 여러 레벨의 SSTable을 모두 확인해야 합니다. 이를 보완하기 위해 **Bloom Filter**, **Block Index**, **Block Cache** 등의 최적화 기법을 사용합니다.

#### LSM 트리에서 Compaction의 역할

```
Compaction이 없다면:
  읽기 시 모든 SSTable을 검색해야 함 → 파일이 쌓일수록 읽기 성능 저하

Compaction의 효과:
  ┌─────────┐  ┌─────────┐  ┌─────────┐
  │SSTable-1│  │SSTable-2│  │SSTable-3│     파일 3개 검색 필요
  └────┬────┘  └────┬────┘  └────┬────┘
       └────────────┼────────────┘
                    ▼
             ┌────────────┐
             │ SSTable-new │     파일 1개만 검색하면 됨
             └────────────┘
  + 중복 키 제거, tombstone 처리, 오래된 버전 삭제
```

---

### B+ 트리 vs LSM 트리 비교

| 항목 | B+ 트리 (RDBMS) | LSM 트리 (HBase) |
|---|---|---|
| **쓰기 방식** | 제자리 갱신 (in-place update) | 추가 전용 (append-only) |
| **쓰기 I/O** | 랜덤 I/O | 순차 I/O |
| **쓰기 성능** | O(log N) — 느림 (랜덤 I/O) | O(1) 분할상환 — 빠름 (메모리 + 순차 쓰기) |
| **읽기 성능** | O(log N) — 빠름 (1회 탐색) | O(N × log N) 최악 — 여러 파일 탐색 필요 |
| **공간 효율** | 높음 (제자리 갱신) | 낮음 (중복 데이터 존재, Compaction 전까지) |
| **쓰기 증폭** | 낮음 | 높음 (Compaction으로 인한 재기록) |
| **읽기 증폭** | 낮음 (1개 위치) | 높음 (여러 SSTable 탐색) |
| **적합한 워크로드** | 읽기 중심, OLTP | 쓰기 중심, 대규모 데이터 |

```
쓰기 처리량 (Throughput)

B+ 트리:  ████████░░░░░░░░░░░░  (랜덤 I/O 병목)
LSM 트리: ██████████████████░░  (순차 I/O로 최적화)

읽기 지연시간 (Latency)

B+ 트리:  ██░░░░░░░░░░░░░░░░░░  (단일 탐색으로 빠름)
LSM 트리: ██████░░░░░░░░░░░░░░  (여러 레벨 탐색 필요)
```

**HBase가 LSM 트리를 선택한 이유**: HBase의 주요 사용 사례(로그, 시계열, 메시징 등)는 **쓰기가 매우 빈번**합니다. LSM 트리는 이런 쓰기 중심 워크로드에서 디스크 I/O를 최소화하여 처리량을 극대화합니다.

---

### 탐색 — 클라이언트가 데이터를 찾는 과정

HBase에서 특정 row key의 데이터를 읽는 전체 탐색 경로입니다.

#### 1단계: Region 위치 탐색

클라이언트가 어떤 RegionServer에 요청을 보내야 하는지 알아내는 과정입니다.

```
Client: Get(row key = "user_500")
    ↓
① ZooKeeper에 hbase:meta 테이블의 위치를 질의
    ↓
   ZooKeeper 응답: "meta 테이블은 RegionServer-3에 있음"
    ↓
② RegionServer-3의 hbase:meta에서 "user_500"이 속한 Region 조회
    ↓
   meta 응답: "user_500은 Region-7에 있고, Region-7은 RegionServer-5가 서빙중"
    ↓
③ 클라이언트가 Region 위치를 로컬 캐시에 저장
    ↓
④ 이후 같은 row key 범위 요청은 캐시를 사용 (ZooKeeper/meta 조회 생략)
```

#### hbase:meta 테이블의 구조

```
┌──────────────────────────────────────────────────────────────────┐
│ hbase:meta 테이블                                                │
├──────────────┬───────────────┬──────────────┬───────────────────┤
│   Row Key    │ Region 시작키  │ Region 끝키  │ RegionServer 위치  │
├──────────────┼───────────────┼──────────────┼───────────────────┤
│ table1,,ts1  │ (empty)       │ ddd          │ rs1:16020         │
│ table1,ddd,ts│ ddd           │ ggg          │ rs2:16020         │
│ table1,ggg,ts│ ggg           │ kkk          │ rs3:16020         │
│ table1,kkk,ts│ kkk           │ (empty)      │ rs1:16020         │
└──────────────┴───────────────┴──────────────┴───────────────────┘
```

#### 2단계: Region 내부 데이터 탐색

RegionServer에 도달한 후 실제 데이터를 찾는 과정입니다.

```
RegionServer-5에서 Get(row="user_500", cf:name) 처리
    ↓
┌─────────────────────────────────────────────────┐
│ ① MemStore 검색 (메모리)                         │
│    - ConcurrentSkipListMap에서 O(log N) 탐색    │
│    - 가장 최신 데이터가 여기에 있을 수 있음        │
└─────────────────────┬───────────────────────────┘
                      ↓ (찾았으면 바로 반환, 없으면 계속)
┌─────────────────────────────────────────────────┐
│ ② BlockCache 검색 (메모리)                       │
│    - LRU 캐시에서 이전에 읽었던 HFile 블록 확인   │
│    - 캐시 히트 시 디스크 I/O 없이 반환            │
└─────────────────────┬───────────────────────────┘
                      ↓ (캐시 미스 시 계속)
┌─────────────────────────────────────────────────┐
│ ③ HFile 검색 (디스크)                            │
│                                                  │
│    HFile-3 (최신)                                │
│    ├─ Bloom Filter 체크 → "없음" → 스킵         │
│                                                  │
│    HFile-2                                       │
│    ├─ Bloom Filter 체크 → "있을 수 있음"         │
│    ├─ Block Index 이진 탐색 → 오프셋 찾기        │
│    └─ Data Block 읽기 → 블록 내 순차 탐색        │
│                                                  │
│    HFile-1 (가장 오래된)                          │
│    └─ (이미 찾았으면 여기까지 안 옴)              │
└─────────────────────┬───────────────────────────┘
                      ↓
결과 병합 → 가장 최신 timestamp 값 반환
```

#### 탐색 최적화 기법 정리

| 기법 | 작동 레벨 | 효과 |
|---|---|---|
| **클라이언트 캐시** | Region 위치 | meta 테이블 반복 조회 방지 |
| **MemStore** | RegionServer | 최신 데이터 메모리 탐색 |
| **BlockCache** | RegionServer | 자주 읽는 HFile 블록 메모리 캐싱 |
| **Bloom Filter** | HFile | 존재하지 않는 키에 대한 불필요한 디스크 I/O 제거 |
| **Block Index** | HFile | 이진 탐색으로 O(log N)에 블록 위치 특정 |
| **Compaction** | Store | HFile 수 감소 → 탐색할 파일 수 감소 |

---

### 쓰기 — 클라이언트가 데이터를 저장하는 과정

HBase에서 Put 요청이 처리되는 전체 경로입니다.

#### 1단계: Region 위치 결정

쓰기도 읽기와 동일하게 먼저 대상 Region을 찾아야 합니다.

```
Client: Put(row key = "user_500", cf:name = "Alice")
    ↓
① ZooKeeper → hbase:meta → RegionServer 위치 확인
   (탐색과 동일한 경로. 캐시가 있으면 즉시 사용)
    ↓
② 해당 RegionServer에 RPC로 Put 요청 전송
```

#### 2단계: RegionServer 내부 처리

```
RegionServer-5에서 Put(row="user_500", cf:name, "Alice") 처리
    ↓
┌─────────────────────────────────────────────────────┐
│ ① Row Lock 획득                                     │
│    - 같은 row에 대한 동시 쓰기를 직렬화              │
│    - 단일 row 수준의 원자성(Atomicity) 보장           │
└─────────────────────┬───────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────┐
│ ② WAL(Write-Ahead Log)에 기록                       │
│    - HDFS에 순차 쓰기 (append)                       │
│    - 이 시점에서 데이터 내구성이 보장됨               │
│    - RegionServer 장애 시 이 로그로 복구              │
│                                                      │
│    WAL 엔트리 구조:                                   │
│    ┌──────────┬──────────┬──────────┬──────────────┐ │
│    │ Region   │ Table    │ Sequence │ KeyValue     │ │
│    │ 정보     │ 이름     │ ID       │ 데이터       │ │
│    └──────────┴──────────┴──────────┴──────────────┘ │
└─────────────────────┬───────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────┐
│ ③ MemStore에 삽입                                    │
│    - ConcurrentSkipListMap에 정렬 삽입               │
│    - O(log N) 시간 복잡도                             │
│    - 이 시점에서 읽기 요청에 즉시 반영됨              │
└─────────────────────┬───────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────┐
│ ④ Row Lock 해제                                      │
└─────────────────────┬───────────────────────────────┘
                      ↓
           클라이언트에 성공 응답 반환
```

#### 3단계: MemStore Flush

MemStore가 임계값에 도달하면 디스크로 내려갑니다.

```
MemStore 크기 ≥ 128MB (기본값)
    ↓
┌─────────────────────────────────────────────────────┐
│ ① 현재 MemStore를 "snapshot"으로 전환                │
│    - 스냅샷은 읽기 전용 (더 이상 쓰기 불가)           │
│    - 새로운 빈 MemStore 생성 → 새 쓰기는 여기로      │
│    - 쓰기가 차단되지 않음 (논블로킹)                  │
└─────────────────────┬───────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────┐
│ ② 스냅샷의 정렬된 데이터를 HFile로 순차 기록         │
│    - 이미 정렬되어 있으므로 그대로 쓰면 됨            │
│    - 디스크 순차 쓰기 → 매우 빠름                     │
│    - 새 HFile이 HDFS에 생성됨                         │
└─────────────────────┬───────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────┐
│ ③ flush 완료 후 정리                                 │
│    - 스냅샷 메모리 해제                               │
│    - 대응하는 WAL 엔트리를 "flush 완료"로 마킹        │
│    - WAL이 더 이상 필요 없으면 oldWALs/로 이동        │
└─────────────────────────────────────────────────────┘
```

#### 4단계: Compaction

HFile이 누적되면 병합 작업이 수행됩니다.

```
[트리거 조건]
- Minor: Store 내 HFile 수가 임계값(기본 3개) 이상
- Major: 기본 7일 주기 또는 수동 실행

Minor Compaction:
┌─────────┐  ┌─────────┐  ┌─────────┐
│HFile-1  │  │HFile-2  │  │HFile-3  │
│key:1~50 │  │key:30~80│  │key:60~99│
└────┬────┘  └────┬────┘  └────┬────┘
     └────────────┼────────────┘
                  ▼
           ┌────────────┐
           │ HFile-new  │   정렬 병합 (merge sort)
           │ key:1~99   │   중복 키 → 최신 값만 유지
           └────────────┘   tombstone은 아직 유지

Major Compaction:
  Region의 모든 HFile → 1개로 통합
  + tombstone 물리 삭제
  + TTL 만료 데이터 제거
  + max_versions 초과분 제거
```

#### 갱신(Update)과 삭제(Delete)의 처리

HBase는 제자리 갱신을 하지 않습니다. 모든 변경은 **새로운 데이터 추가**입니다.

```
[갱신] 같은 row key에 새 timestamp로 Put
Put(row1, cf:name, ts=100, "Alice")  → HFile-1
Put(row1, cf:name, ts=200, "Bob")    → HFile-2  ← 이것이 최신

읽기 시: ts=200의 "Bob" 반환 (최신 timestamp 우선)

[삭제] tombstone 마커를 Put
Delete(row1, cf:name, ts=300)        → HFile-3에 Delete 마커 기록

읽기 시: ts=300 Delete 발견 → "삭제됨" → 결과 없음
실제 물리 삭제: Major Compaction 시점까지 대기
```

#### 쓰기 최적화 기법 정리

| 기법 | 작동 레벨 | 효과 |
|---|---|---|
| **WAL 순차 쓰기** | RegionServer | 랜덤 I/O 대신 append로 디스크 오버헤드 최소화 |
| **MemStore 버퍼링** | RegionServer | 디스크 쓰기 없이 메모리에서 즉시 응답 |
| **Flush 순차 쓰기** | Store | 정렬된 데이터를 그대로 순차 기록 → 빠른 HFile 생성 |
| **비동기 Flush** | Store | 스냅샷 전환으로 쓰기 차단 없이 flush 수행 |
| **Compaction** | Store | HFile 수 관리 → 후속 읽기 성능 보장 |
| **Row Lock** | Region | 동일 row 원자성 보장. row 간에는 lock 경합 없음 |

#### 쓰기 vs 읽기 경로 비교

```
쓰기 (Put):
Client → RegionServer → WAL(디스크) → MemStore(메모리) → 응답
                                          ↓ (비동기)
                                     Flush → HFile

읽기 (Get):
Client → RegionServer → MemStore → BlockCache → HFile(디스크) → 병합 → 응답
```

| 항목 | 쓰기 (Put) | 읽기 (Get) |
|---|---|---|
| **디스크 I/O** | WAL append만 (순차) | HFile 블록 읽기 (랜덤, 캐시 미스 시) |
| **지연시간** | 매우 낮음 (메모리 + 순차쓰기) | 낮음~보통 (캐시 히트 여부에 따라) |
| **병목** | WAL sync (fsync 대기) | HFile 수 증가 시 다중 파일 탐색 |

---

### WAL (Write-Ahead Log) 상세

WAL은 **데이터를 MemStore에 쓰기 전에 먼저(Ahead) 디스크에 기록하는 로그**입니다. RegionServer가 장애로 죽더라도 MemStore에 있던 데이터를 WAL로부터 복구할 수 있습니다.

#### WAL이 필요한 이유

```
WAL이 없다면:

Client → Put → MemStore(메모리)에만 기록 → 응답
                    ↓
            RegionServer 장애 발생!
                    ↓
            MemStore 데이터 전부 유실 (메모리는 휘발성)
                    ↓
            flush 안 된 데이터는 영구 손실

WAL이 있으면:

Client → Put → WAL(디스크) 기록 → MemStore(메모리) 기록 → 응답
                    ↓
            RegionServer 장애 발생!
                    ↓
            MemStore 데이터 유실
                    ↓
            WAL은 HDFS에 남아 있음 (3중 복제)
                    ↓
            다른 RegionServer가 WAL을 replay → MemStore 복구
```

#### WAL 파일의 물리적 구조

```
/hbase/WALs/<regionserver_name>/
    └── <regionserver_name>.<timestamp>

예시:
/hbase/WALs/rs1.example.com,16020,1234567890/
    ├── rs1.example.com%2C16020%2C1234567890.1700000001  (현재 WAL)
    └── rs1.example.com%2C16020%2C1234567890.1700000002  (롤오버된 WAL)
```

WAL 파일 내부는 WALEntry의 연속입니다.

```
┌────────────────────────────────────────────────────────┐
│                     WAL 파일                            │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │ WALEntry 1                                       │  │
│  │ ┌────────────┬─────────────────────────────────┐ │  │
│  │ │ WALKey     │ WALEdit                         │ │  │
│  │ │            │                                 │ │  │
│  │ │ - Region   │ - KeyValue 1 (Put row1/cf:a)   │ │  │
│  │ │ - Table    │ - KeyValue 2 (Put row1/cf:b)   │ │  │
│  │ │ - SeqId    │   (하나의 행 mutation에 포함된   │ │  │
│  │ │ - Timestamp│    모든 셀 변경)                │ │  │
│  │ └────────────┴─────────────────────────────────┘ │  │
│  ├──────────────────────────────────────────────────┤  │
│  │ WALEntry 2                                       │  │
│  │ ┌────────────┬─────────────────────────────────┐ │  │
│  │ │ WALKey     │ WALEdit                         │ │  │
│  │ │ - Region   │ - KeyValue 1 (Delete row2/cf:a)│ │  │
│  │ │ - Table    │                                 │ │  │
│  │ │ - SeqId    │                                 │ │  │
│  │ └────────────┴─────────────────────────────────┘ │  │
│  ├──────────────────────────────────────────────────┤  │
│  │ WALEntry 3 ...                                   │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘

WALKey 상세:
┌────────────────┬────────────────────────────────────────┐
│ 필드           │ 설명                                    │
├────────────────┼────────────────────────────────────────┤
│ Region 정보    │ 어떤 Region에 대한 변경인지              │
│ Table 이름     │ 어떤 테이블인지                          │
│ Sequence ID    │ 단조 증가 ID — 순서 보장 & 복구 기준점   │
│ Write Time     │ 기록 시각                                │
│ Cluster IDs    │ 복제(replication) 시 출처 클러스터 식별   │
└────────────────┴────────────────────────────────────────┘
```

#### WAL의 생명주기

```
① 생성 (Creation)
   - RegionServer 시작 시 WAL 파일 생성
   - 모든 Region의 쓰기가 이 하나의 WAL에 순차 기록됨

         ↓

② 기록 (Writing)
   - Put/Delete 요청마다 WALEntry를 append
   - Durability 설정에 따라 sync 수행

         ↓

③ 롤오버 (Roll)
   - WAL 파일이 일정 크기(기본 ~512MB)에 도달하거나
   - 일정 시간(기본 1시간)이 경과하면
   - 현재 WAL을 닫고 새 WAL 파일 생성

         ↓

④ 아카이브 (Archive)
   - WAL에 기록된 모든 Region의 MemStore가 flush되면
   - 해당 WAL은 더 이상 복구에 필요 없음
   - /hbase/WALs/ → /hbase/oldWALs/로 이동

         ↓

⑤ 삭제 (Deletion)
   - oldWALs의 TTL(기본 10분)이 지나면 삭제
   - 단, Replication이 설정된 경우 복제 완료까지 보존
```

```
타임라인 예시:

시간 ──────────────────────────────────────────────────→

WAL-001: [Region-A Put, Region-B Put, Region-A Put, ...]
              ↓
         Region-A flush 완료 (하지만 Region-B는 아직)
              ↓
         WAL-001은 아직 삭제 불가 (Region-B 데이터가 남아있음)
              ↓
         Region-B flush 완료
              ↓
         WAL-001의 모든 데이터가 flush됨 → oldWALs로 이동 → 삭제
```

#### Durability 설정 (쓰기 내구성 수준)

클라이언트가 Put 요청 시 WAL의 동기화 수준을 선택할 수 있습니다.

| 설정 | 동작 | 성능 | 데이터 안전성 |
|---|---|---|---|
| **SYNC_WAL** (기본) | WAL 기록 + fsync (디스크까지 보장) | 보통 | 높음 — RegionServer 장애에도 안전 |
| **ASYNC_WAL** | WAL 기록, fsync는 비동기 | 빠름 | 보통 — OS 크래시 시 최근 데이터 유실 가능 |
| **SKIP_WAL** | WAL 기록 안 함 | 매우 빠름 | 낮음 — RegionServer 장애 시 데이터 유실 |
| **FSYNC_WAL** | SYNC_WAL과 동일 (명시적) | 보통 | 높음 |

```
성능 vs 안전성 트레이드오프:

SKIP_WAL:  성능 ██████████  안전성 ██░░░░░░░░
ASYNC_WAL: 성능 ████████░░  안전성 ██████░░░░
SYNC_WAL:  성능 ██████░░░░  안전성 ██████████
```

```java
// 사용 예시 (Java API)
Put put = new Put(Bytes.toBytes("row1"));
put.addColumn(cf, qualifier, value);

put.setDurability(Durability.SYNC_WAL);   // 기본값 — 안전
put.setDurability(Durability.ASYNC_WAL);  // 빠르지만 위험 증가
put.setDurability(Durability.SKIP_WAL);   // 가장 빠름, 복구 불가
```

#### WAL과 성능

WAL은 쓰기 경로에서 유일한 **동기적 디스크 I/O**이므로 성능 병목이 될 수 있습니다.

```
쓰기 지연시간 분해:

  Row Lock 획득      ~0.01ms
  WAL append + sync   ~1-5ms  ← 대부분의 시간이 여기서 소비
  MemStore 삽입       ~0.01ms
  Row Lock 해제       ~0.01ms
  ─────────────────────────
  총                  ~1-5ms
```

**WAL이 순차 쓰기인데 왜 병목인가?**

```
순차 쓰기 자체는 빠르지만, fsync가 느림:

① append (메모리 버퍼에 쓰기)     → 매우 빠름 (μs 단위)
② fsync  (버퍼 → 실제 디스크 기록) → 느림 (ms 단위)
     ↓
   디스크 헤드/NAND가 실제로 기록을 완료할 때까지 대기
   HDD: ~3-10ms / SSD: ~0.1-1ms
```

**성능 최적화 기법:**

| 기법 | 설명 |
|---|---|
| **WAL 그룹 커밋** | 여러 Put의 WAL 쓰기를 모아서 한 번에 fsync → sync 횟수 감소 |
| **멀티 WAL** | RegionServer당 여러 WAL 파이프라인 운영 → 병렬 쓰기 |
| **ASYNC_WAL** | fsync 대기 생략 → 지연시간 감소 (안전성 트레이드오프) |
| **SSD 사용** | WAL 전용 SSD 디스크 → fsync 지연시간 단축 |

#### WAL Splitting (장애 복구)

RegionServer 장애 시 WAL을 Region별로 분리하여 복구하는 과정입니다.

```
문제: 하나의 WAL에 여러 Region의 데이터가 섞여 있음

WAL 파일 내용:
  Entry 1: Region-A, SeqId=1, Put(row1)
  Entry 2: Region-B, SeqId=2, Put(row5)
  Entry 3: Region-A, SeqId=3, Put(row2)
  Entry 4: Region-C, SeqId=4, Delete(row9)
  Entry 5: Region-B, SeqId=5, Put(row6)

        ↓ WAL Splitting (HMaster가 수행)

Region-A용:                Region-B용:               Region-C용:
  SeqId=1, Put(row1)        SeqId=2, Put(row5)         SeqId=4, Delete(row9)
  SeqId=3, Put(row2)        SeqId=5, Put(row6)

        ↓ 각 Region을 새로운 RegionServer에 할당

RegionServer-X:             RegionServer-Y:
  Region-A open              Region-B open
  → WAL replay               → WAL replay
  → MemStore 복구 완료        → MemStore 복구 완료
```

**Distributed Log Splitting**: 대량의 WAL을 HMaster 혼자 처리하면 복구가 느립니다. 여러 RegionServer가 협력하여 WAL을 분리하는 방식입니다.

```
기본 방식 (HMaster 단독):
  HMaster가 WAL을 읽고 Region별로 분리 → 순차 처리, 느림

분산 방식 (Distributed Log Splitting):
  ┌──────────┐
  │ HMaster  │ WAL splitting 작업을 ZooKeeper에 등록
  └────┬─────┘
       ↓
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │   RS-1   │  │   RS-2   │  │   RS-3   │
  │ WAL 조각1│  │ WAL 조각2│  │ WAL 조각3│  ← 여러 RS가 병렬로 분리
  └──────────┘  └──────────┘  └──────────┘
  → 복구 시간 대폭 단축
```

#### WAL 관련 주요 설정

| 설정 | 기본값 | 설명 |
|---|---|---|
| `hbase.regionserver.hlog.blocksize` | HDFS 블록 크기 | WAL 파일 롤오버 기준 크기 |
| `hbase.regionserver.logroll.period` | 3600000 (1시간) | WAL 롤오버 주기 (ms) |
| `hbase.regionserver.maxlogs` | 32 | 최대 WAL 파일 수. 초과 시 강제 flush |
| `hbase.master.logcleaner.ttl` | 600000 (10분) | oldWALs 보존 시간 (ms) |
| `hbase.wal.provider` | filesystem | WAL 구현체 (filesystem, asyncfs 등) |

---

### 저장소 — 물리적 데이터 저장 구조

#### HDFS 상의 디렉토리 레이아웃

```
/hbase/
├── data/
│   └── <namespace>/
│       └── <table>/
│           └── <region_encoded_name>/
│               ├── .regioninfo              ← Region 메타데이터
│               └── <column_family>/
│                   ├── HFile-001            ← StoreFile (불변)
│                   ├── HFile-002
│                   └── HFile-003
├── WALs/
│   └── <regionserver_name>/
│       ├── WAL.0000000001                   ← Write-Ahead Log
│       └── WAL.0000000002
├── oldWALs/                                 ← flush 완료 후 이동된 WAL
└── hbase.id                                 ← 클러스터 ID
```

#### 저장 계층 구조

```
┌──────────────────────────────────────────────────────┐
│                    RegionServer                       │
│                                                      │
│  ┌─────────────────────────────────────────────────┐ │
│  │                  Region                          │ │
│  │   (row key range: "aaa" ~ "mmm")                │ │
│  │                                                  │ │
│  │  ┌─────────────────┐  ┌─────────────────┐       │ │
│  │  │ Store (cf:info)  │  │ Store (cf:stats) │      │ │
│  │  │                  │  │                  │      │ │
│  │  │ ┌──────────────┐│  │ ┌──────────────┐│      │ │
│  │  │ │  MemStore    ││  │ │  MemStore    ││      │ │
│  │  │ │ (128MB 기본) ││  │ │ (128MB 기본) ││      │ │
│  │  │ └──────────────┘│  │ └──────────────┘│      │ │
│  │  │                  │  │                  │      │ │
│  │  │ ┌──────────────┐│  │ ┌──────────────┐│      │ │
│  │  │ │ HFile-1      ││  │ │ HFile-1      ││      │ │
│  │  │ │ HFile-2      ││  │ │ HFile-2      ││      │ │
│  │  │ │ HFile-3      ││  │ │              ││      │ │
│  │  │ └──────────────┘│  │ └──────────────┘│      │ │
│  │  └─────────────────┘  └─────────────────┘       │ │
│  │                                                  │ │
│  │  ┌──────────────────────────────────────┐       │ │
│  │  │ WAL (Region 공유, RegionServer당 1개) │       │ │
│  │  └──────────────────────────────────────┘       │ │
│  └─────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

#### 저장 계층별 특성

| 계층 | 저장소 | 크기 | 데이터 특성 |
|---|---|---|---|
| **MemStore** | JVM Heap 메모리 | 128MB (기본) / CF | 정렬된 최신 쓰기 데이터. ConcurrentSkipListMap 구조 |
| **BlockCache** | JVM Off-Heap 또는 Heap | RegionServer 힙의 40% (기본) | 자주 읽는 HFile 블록의 LRU 캐시 |
| **WAL** | HDFS (디스크) | RegionServer당 1개 | 순차 기록된 변경 로그. 장애 복구 전용 |
| **HFile** | HDFS (디스크) | flush마다 생성, Compaction으로 병합 | 불변(Immutable). 정렬된 KeyValue 파일 |

#### 데이터 생명주기 요약

```
데이터의 일생:

① Client가 Put 요청
         ↓
② WAL에 순차 기록 (디스크, 내구성 보장)
         ↓
③ MemStore에 기록 (메모리, 빠른 응답)
         ↓  ── 128MB 도달 ──
④ Flush → HFile 생성 (HDFS, 불변 파일)
         ↓  ── HFile 누적 ──
⑤ Minor Compaction (여러 HFile → 하나로 병합)
         ↓  ── 7일 주기 ──
⑥ Major Compaction (전체 HFile 병합 + tombstone 삭제 + 만료 데이터 제거)
```

---

### 전체 정리: 왜 이 구조인가

```
문제: 대규모 데이터를 실시간으로 읽고 쓰려면?

해결 전략:
┌──────────────────────────────────────────────────────────────┐
│ 쓰기 최적화     →  LSM 트리 (랜덤 I/O를 순차 I/O로 변환)     │
│ 읽기 최적화     →  Bloom Filter + Block Index + BlockCache   │
│ 수평 확장       →  Region 기반 자동 샤딩                      │
│ 데이터 내구성   →  WAL + HDFS 3중 복제                        │
│ 고가용성        →  ZooKeeper 코디네이션 + 자동 장애 복구       │
│ 저장 효율       →  Compaction으로 중복 제거 + 블록 단위 압축   │
└──────────────────────────────────────────────────────────────┘
```

| 설계 선택 | 이유 |
|---|---|
| **LSM 트리 (not B+ 트리)** | 쓰기 중심 워크로드에 최적. 순차 I/O로 디스크 처리량 극대화 |
| **Region 기반 샤딩** | Row key range 기반 자동 분할로 수평 확장. 데이터 지역성(locality) 유지 |
| **HDFS 위에 구축** | 데이터 복제와 내구성은 HDFS에 위임. Region 이동 시 데이터 복사 불필요 |
| **Compaction** | LSM 트리의 읽기 성능 저하 보완. 공간 회수 |
| **WAL** | 메모리 데이터 유실 방지. 디스크 순차 쓰기로 오버헤드 최소화 |
| **ZooKeeper** | 분산 환경에서 일관된 메타데이터 관리와 장애 감지 |