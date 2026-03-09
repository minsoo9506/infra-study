# Redis 캐시 레이어

## 캐싱의 원리와 목적

캐싱(Caching)은 **자주 쓰이는 데이터를 빠른 저장소에 미리 올려두어** 원본 저장소(DB, 외부 API 등)에 대한 접근을 줄이는 기법이다.

### 왜 캐싱이 필요한가

```
# 캐싱 없을 때
클라이언트 → 서버 → DB (디스크 I/O) → 응답
                     ↑ 매 요청마다 발생, 느리고 DB 부하 큼

# 캐싱 있을 때
클라이언트 → 서버 → Redis (메모리) → 응답  ← 캐시 히트
클라이언트 → 서버 → Redis (miss) → DB → Redis 저장 → 응답  ← 캐시 미스
```

**캐싱의 목적:**
- **응답 속도 개선:** 디스크 I/O 없이 메모리에서 바로 응답
- **DB 부하 감소:** 동일한 쿼리의 반복 실행 차단
- **비용 절감:** DB 스케일 업 없이 처리량 증가
- **외부 API 호출 최소화:** rate limit 회피, 응답 시간 단축

### 캐시 히트율 (Hit Rate)

```
히트율 = 캐시 히트 수 / 전체 요청 수 × 100
```

- 히트율이 높을수록 캐시 효과가 큼
- 일반적으로 **80% 이상**을 목표로 설계

---

### 캐싱 전략 (캐시 읽기/쓰기 패턴)

#### Cache-Aside (Lazy Loading) — 가장 일반적

애플리케이션이 직접 캐시를 관리한다. 캐시 미스 시 DB에서 읽어와 캐시에 저장.

```
읽기:
  1. Redis에서 조회
  2. 있으면 → 반환 (캐시 히트)
  3. 없으면 → DB 조회 → Redis에 저장 → 반환 (캐시 미스)

쓰기:
  1. DB에 저장
  2. 캐시 무효화(DEL) 또는 갱신
```

- **장점:** 실제로 쓰이는 데이터만 캐시됨, 캐시 장애 시 DB로 fallback 가능
- **단점:** 최초 요청은 항상 캐시 미스 (Cold Start), 데이터 불일치 가능

---

#### Write-Through

데이터를 쓸 때 DB와 캐시를 동시에 갱신.

```
쓰기:
  1. DB에 저장
  2. Redis에도 저장 (항상 최신 상태 유지)

읽기:
  1. Redis에서 조회 → 항상 히트
```

- **장점:** 캐시와 DB 일관성 보장
- **단점:** 쓰기 지연 증가, 사용되지 않는 데이터도 캐시됨 (TTL로 보완)

---

#### Write-Behind (Write-Back)

캐시에 먼저 쓰고, 나중에 비동기로 DB에 반영.

```
쓰기:
  1. Redis에 저장 (즉시 응답)
  2. 백그라운드에서 DB에 반영

읽기:
  1. Redis에서 조회
```

- **장점:** 쓰기 성능 최대화
- **단점:** Redis 장애 시 데이터 유실 위험, 구현 복잡도 높음

---

#### Read-Through

캐시 계층이 DB 조회를 대신 수행 (애플리케이션은 캐시만 바라봄).

- Cache-Aside와 비슷하나, 캐시 미스 처리를 캐시 레이어가 담당
- 별도 캐시 미들웨어나 ORM 레벨에서 주로 구현

---

### 캐싱 전략 비교

| 전략 | 읽기 성능 | 쓰기 성능 | 일관성 | 복잡도 | 적합한 상황 |
|------|-----------|-----------|--------|--------|-------------|
| Cache-Aside | 높음 | 보통 | 보통 | 낮음 | 읽기 많은 일반 서비스 |
| Write-Through | 높음 | 낮음 | 높음 | 보통 | 읽기/쓰기 균형, 일관성 중요 |
| Write-Behind | 높음 | 최고 | 낮음 | 높음 | 쓰기 폭발적, 유실 허용 가능 |
| Read-Through | 높음 | 보통 | 보통 | 보통 | 캐시 레이어 추상화 필요 시 |

---

### 캐시 만료 (Eviction) 정책

Redis 메모리가 가득 찼을 때 어떤 데이터를 제거할지 결정하는 정책.

`redis.conf`에서 `maxmemory-policy`로 설정:

| 정책 | 설명 |
|------|------|
| `noeviction` | 메모리 초과 시 쓰기 오류 반환 (기본값) |
| `allkeys-lru` | 전체 키 중 가장 오래 사용되지 않은 것 제거 |
| `volatile-lru` | TTL 설정된 키 중 LRU 제거 |
| `allkeys-lfu` | 전체 키 중 가장 적게 사용된 것 제거 |
| `volatile-lfu` | TTL 설정된 키 중 LFU 제거 |
| `allkeys-random` | 전체 키 중 랜덤 제거 |
| `volatile-random` | TTL 설정된 키 중 랜덤 제거 |
| `volatile-ttl` | TTL이 가장 짧은 키 먼저 제거 |

> 캐시 용도라면 `allkeys-lru` 또는 `allkeys-lfu` 권장.

---

### 캐시 문제 패턴

#### Cache Stampede (캐시 스탬피드)

캐시가 만료되는 순간 다수의 요청이 동시에 DB로 몰리는 현상.

```
해결책:
- 만료 시간에 랜덤 지터(jitter) 추가: TTL = 3600 + random(0~60)
- 분산 락(SETNX)으로 하나의 요청만 DB 조회 허용
- 백그라운드 갱신: 만료 전에 미리 TTL 연장
```

#### Cache Penetration (캐시 관통)

존재하지 않는 키에 대한 요청이 반복되어 매번 DB까지 도달하는 현상.

```
해결책:
- 존재하지 않는 키도 캐시에 null 또는 빈 값으로 저장 (짧은 TTL)
- Bloom Filter로 존재하지 않는 키 사전 차단
```

#### Cache Avalanche (캐시 어밸런치)

다수의 캐시 키가 동시에 만료되어 DB에 대규모 부하가 집중되는 현상.

```
해결책:
- 캐시 TTL에 랜덤 지터 적용 (동시 만료 분산)
- Redis Cluster로 고가용성 확보
- Circuit Breaker 패턴으로 DB 보호
```

---

## Redis 캐싱 예제

### Cache-Aside 패턴 구현 (Python)

```python
import redis
import json
from typing import Optional

r = redis.Redis(host='localhost', port=6379, db=0)
CACHE_TTL = 300  # 5분


def get_user(user_id: int) -> dict:
    cache_key = f"user:{user_id}"

    # 1. 캐시 조회
    cached = r.get(cache_key)
    if cached:
        print("캐시 히트")
        return json.loads(cached)

    # 2. 캐시 미스 → DB 조회
    print("캐시 미스 → DB 조회")
    user = db_get_user(user_id)  # 실제 DB 쿼리

    # 3. 캐시에 저장
    if user:
        r.setex(cache_key, CACHE_TTL, json.dumps(user))

    return user


def update_user(user_id: int, data: dict):
    # DB 업데이트
    db_update_user(user_id, data)

    # 캐시 무효화
    r.delete(f"user:{user_id}")
```

---

### TTL 기반 단순 캐싱 (API 응답 캐싱)

```python
import hashlib

def get_search_results(query: str) -> list:
    # 쿼리를 해시로 캐시 키 생성
    cache_key = "search:" + hashlib.md5(query.encode()).hexdigest()

    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    results = db_search(query)  # 느린 DB 검색
    r.setex(cache_key, 60, json.dumps(results))  # 1분 캐시
    return results
```

---

### 분산 락으로 Cache Stampede 방지

```python
import time

def get_user_safe(user_id: int) -> dict:
    cache_key = f"user:{user_id}"
    lock_key = f"lock:user:{user_id}"

    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    # 분산 락 획득 시도 (SETNX)
    acquired = r.set(lock_key, "1", nx=True, ex=5)  # 5초 락
    if acquired:
        try:
            user = db_get_user(user_id)
            if user:
                r.setex(cache_key, CACHE_TTL, json.dumps(user))
            return user
        finally:
            r.delete(lock_key)
    else:
        # 락 못 얻은 경우: 잠시 대기 후 재시도
        time.sleep(0.1)
        cached = r.get(cache_key)
        return json.loads(cached) if cached else {}
```

---

### Pipeline으로 다중 캐시 조회 최적화

```python
def get_multiple_users(user_ids: list[int]) -> list[dict]:
    # Pipeline으로 여러 GET을 한 번의 네트워크 왕복으로 처리
    pipe = r.pipeline()
    for uid in user_ids:
        pipe.get(f"user:{uid}")
    results = pipe.execute()

    users = []
    miss_ids = []

    for i, (uid, cached) in enumerate(zip(user_ids, results)):
        if cached:
            users.append((i, json.loads(cached)))
        else:
            miss_ids.append((i, uid))

    # 미스된 것만 DB 조회
    if miss_ids:
        db_users = db_get_users([uid for _, uid in miss_ids])
        pipe = r.pipeline()
        for (i, uid), user in zip(miss_ids, db_users):
            users.append((i, user))
            pipe.setex(f"user:{uid}", CACHE_TTL, json.dumps(user))
        pipe.execute()

    # 원래 순서대로 정렬
    return [u for _, u in sorted(users)]
```

---

### FastAPI + Redis 캐싱 미들웨어 패턴

```python
from fastapi import FastAPI, Request
import functools

app = FastAPI()


def cache(ttl: int = 60):
    """캐시 데코레이터"""
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            cache_key = f"api:{func.__name__}:{args}:{kwargs}"
            cached = r.get(cache_key)
            if cached:
                return json.loads(cached)
            result = await func(*args, **kwargs)
            r.setex(cache_key, ttl, json.dumps(result))
            return result
        return wrapper
    return decorator


@app.get("/products/{product_id}")
@cache(ttl=300)
async def get_product(product_id: int):
    return db_get_product(product_id)
```
