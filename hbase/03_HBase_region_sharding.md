# HBase Region과 샤딩

## HBase Region

**Region**은 HBase에서 데이터를 수평 분할하는 기본 단위입니다. RDBMS의 파티션과 유사한 개념입니다.

### Region의 구조

```
Table
├── Region 1  (row key: aaa ~ ddd)
├── Region 2  (row key: ddd ~ ggg)
├── Region 3  (row key: ggg ~ kkk)
└── Region 4  (row key: kkk ~ zzz)
```

- 하나의 테이블은 **row key 범위**에 따라 여러 Region으로 나뉩니다
- 각 Region은 하나의 **RegionServer**에 할당되어 서빙됩니다
- Region 내부에는 Column Family별로 **Store**가 존재하고, Store는 메모리의 **MemStore**와 디스크의 **StoreFile(HFile)**로 구성됩니다

### Region의 생명주기

1. **테이블 생성 시** — 기본적으로 1개의 Region으로 시작
2. **Auto Split** — Region 크기가 임계값(기본 10GB)을 넘으면 자동으로 2개로 분할
3. **Merge** — 너무 작은 Region은 병합 가능

## 샤딩 (Sharding)

HBase의 샤딩은 **row key 기반의 range partitioning**입니다.

### 동작 방식

```
Client → ZooKeeper → hbase:meta 테이블 → 해당 RegionServer로 직접 요청
```

1. 클라이언트가 특정 row key로 요청
2. `hbase:meta` 테이블에서 해당 row key가 속한 Region과 RegionServer 위치를 조회
3. 해당 RegionServer에 직접 읽기/쓰기 수행 (이후 캐싱)

### Hotspot 문제

row key 설계가 잘못되면 특정 Region에 트래픽이 집중되는 **hotspot** 문제가 발생합니다.

예를 들어 row key가 타임스탬프로 시작하면, 최신 데이터를 담는 Region 하나에만 쓰기가 몰립니다.

### Hotspot 해결 전략

| 전략 | 설명 |
|---|---|
| **Salting** | row key 앞에 해시값(0~N)을 prefix로 붙여 여러 Region에 분산 |
| **Hashing** | row key 전체를 해시하여 균등 분배 (범위 스캔 불가 트레이드오프) |
| **Reversing** | 타임스탬프 등을 뒤집어서 단조 증가 패턴 제거 |
| **Pre-splitting** | 테이블 생성 시 미리 Region을 N개로 분할하여 초기 hotspot 방지 |

### Pre-splitting 예시

```shell
hbase> create 'my_table', 'cf', SPLITS => ['10', '20', '30', '40', '50']
```

이렇게 하면 테이블 생성 시점부터 6개의 Region이 만들어져 여러 RegionServer에 분산됩니다.

## Region vs 일반 샤딩 비교

| 항목 | HBase Region | 일반 DB 샤딩 |
|---|---|---|
| 분할 기준 | Row key range | Hash 또는 range |
| 자동 분할 | Auto split 지원 | 보통 수동 |
| 리밸런싱 | RegionServer 간 Region 이동(balancer) | 수동 resharding 필요 |
| 메타데이터 | `hbase:meta` 테이블이 관리 | 애플리케이션 또는 프록시가 관리 |

핵심은 **row key 설계가 곧 샤딩 전략**이라는 점입니다. HBase에서는 row key를 어떻게 설계하느냐에 따라 데이터 분산, 읽기/쓰기 성능, hotspot 여부가 결정됩니다.
