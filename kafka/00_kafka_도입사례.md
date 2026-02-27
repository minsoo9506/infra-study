# Apache Kafka 도입 사례

## 1. 국내 기업

### 1-1. LINE (라인)

**도입 배경:** 메시징 서버 팀의 아키텍처가 점점 복잡해지면서, 서비스 간 결합도를 낮출 중간 레이어가 필요했다.

**Kafka를 선택한 이유:** 높은 확장성, 가용성, 데이터 영속성 보장, 그리고 다수의 컨슈머를 동시에 지원하는 Pub/Sub 모델.

**규모:**
- 세계 최대 규모의 단일 Kafka 클러스터 중 하나 운영
- 일 **2,500억 건** 이상의 메시지 처리
- 일 **210TB** 데이터 처리
- 피크 처리량: **4GB/sec 이상**
- 약 **50개 이상의 독립 서비스/시스템**이 플랫폼 사용

**활용 사례:**
- **분산 큐잉**: 웹 애플리케이션 서버의 무거운 작업을 백그라운드 프로세서로 오프로드
- **데이터 허브**: 사용자 관련 이벤트(관계, 활동 등)를 통계, 타임라인, 스팸 감지 등 다수의 다운스트림 서비스로 배포

**성과:** 단일 팀의 아키텍처 단순화 도구로 시작했으나, 전사 플랫폼으로 성장. 클라이언트 배칭을 통해 여러 레코드를 하나의 요청으로 묶어 클러스터 부하를 크게 줄임.

> 참고: [LINE에서 Kafka를 사용하는 방법](https://engineering.linecorp.com/ko/blog/how-to-use-kafka-in-line-1/)

---

### 1-2. 우아한형제들 (배달의민족)

**도입 배경:** 음식 배달 주문을 처리하는 MSA 환경에서, 서비스 간 신뢰성 있는 이벤트 기반 통신과 데이터 일관성 유지가 필요했다.

**Kafka를 선택한 이유:** 분산 서비스 간 비동기적이고 신뢰성 있는 메시지 전달 + 순서 보장.

**아키텍처:**
- **Transactional Outbox 패턴**: 비즈니스 연산과 동일한 DB 트랜잭션 내에서 이벤트를 "Outbox" 테이블에 기록. 비즈니스 트랜잭션이 커밋되면 이벤트도 반드시 기록되도록 보장.
- **Debezium MySQL Kafka Connector**: 바이너리 로그(binlog)를 통해 Outbox 테이블의 새 레코드를 감지하고 자동으로 Kafka 메시지로 발행.

**성과:** "이중 쓰기(Dual Write)" 문제를 해결. DB 쓰기와 메시지 발행이라는 두 개의 독립적 작업이 각각 실패할 수 있는 문제를, 이벤트 기록을 DB 트랜잭션의 일부로 만들어 **신뢰성 있는 이벤트 전달**을 달성.

> 참고: [우리 팀은 카프카를 어떻게 사용하고 있을까 | 우아한형제들 기술블로그](https://techblog.woowahan.com/17386/)

---

### 1-3. 토스증권

**도입 배경:** MSA 서비스 서버는 이미 두 데이터센터에 걸쳐 Active-Active 이중화를 구현했으나, **Kafka는 이중화가 안 된 상태**였다. 데이터센터 장애 시 메시징 인프라가 중단되는 위험이 있었다.

**아키텍처 선택:**

| 방식 | 설명 | 결정 |
|------|------|------|
| **Stretched Cluster** | 두 DC에 걸친 단일 논리 클러스터 | 네트워크 파티션 시 스플릿 브레인 위험 + 지연 오버헤드로 **기각** |
| **Active-Active** | 독립된 두 Kafka 클러스터 + 양방향 미러링 | 가용성/성능 우위로 **채택** |

**구현 세부사항:**
- **프로듀서 정책 (Active-Active)**: Split DNS로 로컬 DC의 Kafka 클러스터로 라우팅. 양방향 미러링 도구가 메시지를 상대 DC로 동기화하여 양쪽 모두 100% 메시지 보유.
- **컨슈머 정책 (Active-Standby)**: 양방향 미러링으로 인한 중복 소비 방지를 위해, GSLB DNS를 통해 한쪽 DC의 컨슈머만 활성화.
- **커스텀 미러링 도구**: Kafka Connect Sink Connector로 구현, 원격 DC의 Kafka 클러스터에 쓰기.
- **오프셋 동기화**: 동일 메시지가 DC1에서는 오프셋 10, DC2에서는 9,959,273일 수 있어, 페일오버를 위한 커스텀 오프셋 변환 구축.
- **실시간 주가 전달**: C 기반 장부 시스템 → Kafka → WebSocket 서버 → 사용자

**성과:** 증권 거래 플랫폼에서 데이터센터 수준의 장애 허용(fault tolerance) 달성. 수백 개의 토픽이 양방향 미러링되고, 수천 개의 컨슈머 그룹 오프셋이 동기화.

> 참고: [토스증권 Apache Kafka 데이터센터 이중화 구성 #1](https://toss.tech/article/kafka-distribution-1), [#2](https://toss.tech/article/kafka-distribution-2)

---

## 2. 글로벌 기업

### 2-1. LinkedIn (Kafka의 탄생지)

**도입 배경:** 대규모 활동 데이터(페이지뷰, 검색, 연결 등)와 운영 메트릭을 처리하는 통합 플랫폼이 필요했으나, 기존 시스템은 규모와 실시간성을 감당하지 못했다.

**특이사항:** LinkedIn이 Kafka를 **직접 만들었다**. 2011년에 오픈소스로 공개.

**규모 변화:**

| 연도 | 일일 메시지 수 |
|------|--------------|
| 2011 | 10억 |
| 2012 | 200억 |
| 2013 | 2,000억 |
| ~2020 | 1조 |
| 2024 | **7조** |

**성과:** 10억에서 7조 메시지로 **7,000배 성장**. Kafka는 LinkedIn 전체 데이터 인프라의 백본이 되었고, 세계에서 가장 널리 채택된 분산 시스템 중 하나로 성장.

> 참고: [How LinkedIn customizes Apache Kafka for 7 trillion messages per day](https://www.linkedin.com/blog/engineering/open-source/apache-kafka-trillion-messages)

---

### 2-2. Netflix

**도입 배경:** 190개국 1억 명 이상의 활성 회원으로부터 발생하는 대규모 스트리밍 이벤트(재생, 오류, 사용자 인터랙션)를 실시간으로 분석해야 했다.

**규모:**
- **Keystone Pipeline**: 일 최대 **2조 건**의 메시지 처리
- 일 **3PB 수집**, **7PB 출력**
- 일 **4,500억 건**의 고유 이벤트
- 프로덕션에서 **700개 이상의 Kafka 토픽** 운영
- 개별 토픽에서 초당 최대 **100만 건** 메시지 처리

**성과:** 배치 ETL 워크로드를 Kafka + Flink 기반 스트림 프로세싱으로 마이그레이션하여, 배치 처리 대기 시간 없이 **실시간 데이터 가용성** 달성.

> 참고: [How Netflix Uses Kafka for Distributed Streaming](https://www.confluent.io/blog/how-kafka-is-used-by-netflix/)

---

### 2-3. Uber

**도입 배경:** 글로벌 플랫폼에서 실시간 라이드 매칭, 가격 책정, 사기 탐지, 분석을 위한 데이터 처리가 필요했다.

**규모:** 세계 최대 규모의 Kafka 배포 중 하나 — 일 수조 건의 메시지, 수 PB 데이터.

**아키텍처:**
- **Federated Cluster**: ~150 노드 규모의 소규모 클러스터들로 분산하여 관리성 확보
- **Region Cluster** (로컬 프로듀서 쓰기) + **Aggregate Cluster** (리플리케이션을 통한 글로벌 뷰)
- **uReplicator**: 크로스 리전 Kafka 데이터 복제를 위한 오픈소스 솔루션
- **uForwarder**: gRPC 인터페이스를 제공하는 푸시 기반 컨슈머 프록시. **1,000개 이상**의 컨슈머 서비스 온보딩

**성과:** Federated 아키텍처로 대규모 클러스터 관리 문제 해결. uForwarder가 Kafka의 파티션 기반 모델과 메시지 큐잉 유즈케이스 간의 불일치를 해소.

> 참고: [Disaster recovery for multi-region Kafka at Uber](https://www.uber.com/blog/kafka/), [Introducing uForwarder](https://www.uber.com/blog/introducing-ufowarder/)

---

### 2-4. Airbnb

**도입 배경:** 수억 명의 사용자에게 실시간 콘텐츠 추천, 결제, 리뷰 등의 기능을 제공하기 위해 다양한 소스의 데이터를 결합해야 했다.

**아키텍처:**
- Kafka를 실시간 데이터 수집 및 스트리밍의 기반으로 사용
- **Kafka Connect**로 다양한 소스에서 데이터 수집
- **Kafka Streams**로 실시간 필터링, 집계, 보강
- **Riverbed**: Kafka(온라인) + Spark(오프라인)를 결합한 분산 Materialized View 프레임워크

**규모:**
- Riverbed가 일 **24억 건** 이벤트 처리
- 일 **3.5억 건** 문서 쓰기
- 결제, 검색, 리뷰, 여정 등 **50개 이상의 Materialized View** 구동

> 참고: [Airbnb Riverbed (InfoQ)](https://www.infoq.com/news/2023/10/airbnb-riverbed-introduction/)

---

### 2-5. Walmart

**도입 배경:** 세계 최대 소매업체로서, 수천 개 매장과 온라인 채널에 걸친 실시간 재고 가시성이 필요했다. 기존 배치 처리는 지연을 유발하여 품절과 과잉 재고 문제를 일으켰다.

**규모:**
- 실시간 재고 시스템에 **8,500개 노드**
- 재고 추적을 위한 일 **110억 건** 이벤트
- 전체 배포에서 일 **수조 건**의 Kafka 메시지 처리
- **99.99% 가용성**

**성과:** 배치 기반 재고 보충을 실시간으로 전환하여 품절을 방지하고, 오프라인/온라인 채널 전반의 고객 경험을 개선.

> 참고: [How Walmart Uses Apache Kafka for Real-Time Replenishment](https://www.confluent.io/blog/how-walmart-uses-kafka-for-real-time-omnichannel-replenishment/)

---

## 3. 종합 비교

| 기업 | 일일 메시지 규모 | 주요 활용 | 주목할 아키텍처 |
|------|----------------|----------|---------------|
| **LinkedIn** | 7조 | 활동 데이터, 메트릭 | Kafka 창시자, 커스텀 포크 |
| **Netflix** | 2조 | 스트리밍 분석, 모니터링 | Keystone Pipeline, 700+ 토픽 |
| **Uber** | 수조 | 라이드 이벤트, 분석 | Federated 클러스터, uReplicator |
| **Walmart** | 수조 | 실시간 재고 관리 | 8,500 노드, 99.99% 가용성 |
| **Airbnb** | 24억 | Materialized View, 검색 | Riverbed (Kafka + Spark) |
| **LINE** | 2,500억 | 메시징, 데이터 허브 | 세계 최대급 단일 클러스터, 210TB/일 |
| **토스증권** | - | 주가 전달, 거래 | Active-Active DC 이중화 |
| **우아한형제들** | - | 주문/배달 이벤트 | Transactional Outbox + Debezium CDC |

---

## 4. 사례에서 얻는 핵심 교훈

1. **이벤트 기반 아키텍처의 허브**: 대부분의 기업이 Kafka를 단순 메시지 큐가 아닌, MSA 환경에서의 **이벤트 허브**로 활용하고 있다.

2. **규모에 맞는 아키텍처 설계**: 소규모에서는 단일 클러스터로 충분하지만, 대규모에서는 Federated Cluster(Uber), Active-Active DC 이중화(토스증권) 등 고급 아키텍처가 필요하다.

3. **배치에서 실시간으로의 전환**: Netflix, Walmart 등 여러 기업이 Kafka 도입을 통해 배치 처리를 실시간 스트림 프로세싱으로 전환하여 비즈니스 가치를 크게 향상시켰다.

4. **데이터 일관성 패턴**: 우아한형제들의 Transactional Outbox 패턴처럼, Kafka와 DB 간의 데이터 일관성을 보장하는 패턴이 실무에서 중요하다.

5. **점진적 성장**: LINE의 사례처럼, 하나의 팀에서 시작하여 전사 플랫폼으로 성장하는 경로가 자연스럽다.
