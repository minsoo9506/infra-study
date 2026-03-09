# Redis 세션 스토어

## 세션이란

세션(Session)은 서버가 클라이언트의 상태를 일정 시간 동안 유지하는 메커니즘이다.
HTTP는 기본적으로 **stateless** 프로토콜이기 때문에, 요청마다 "누가 보냈는지"를 서버가 기억하지 못한다.
세션은 이 문제를 해결하기 위해 서버 측에 상태를 저장하고, 클라이언트에는 세션 ID만 쿠키로 전달한다.

**세션 동작 흐름:**
```
1. 클라이언트 로그인 요청
2. 서버: 세션 생성 → session_id = "abc123", data = {user_id: 1001}
3. 서버: 세션 저장소에 저장
4. 클라이언트: 쿠키에 session_id = "abc123" 저장
5. 이후 요청마다 쿠키에 session_id 포함
6. 서버: session_id로 저장소 조회 → 사용자 식별
```

**세션 vs 쿠키 vs JWT:**

| 구분 | 저장 위치 | 보안 | 서버 부하 | 확장성 |
|------|-----------|------|-----------|--------|
| 쿠키 | 클라이언트 | 낮음 (탈취 위험) | 없음 | 높음 |
| 세션 | 서버 | 높음 | 있음 (조회 필요) | 낮음 (단일 서버) |
| JWT | 클라이언트 | 중간 (서명 검증) | 낮음 | 높음 |

**왜 Redis를 세션 저장소로 쓰는가:**
- **빠른 조회:** 매 요청마다 세션을 조회하므로 속도가 핵심 → in-memory인 Redis가 적합
- **TTL 지원:** 세션 만료를 `EXPIRE` 명령 하나로 처리
- **중앙화된 저장소:** 여러 서버 인스턴스가 동일한 세션에 접근 가능 (클러스터링 핵심)

---

## Redis + Python 에서 세션 관리

### 기본 구조 (직접 구현)

`redis-py` 라이브러리로 세션을 직접 구현하는 예시다.

```python
import redis
import uuid
import json

r = redis.Redis(host='localhost', port=6379, db=0)

SESSION_TTL = 3600  # 1시간


def create_session(user_id: int) -> str:
    """로그인 시 세션 생성"""
    session_id = str(uuid.uuid4())
    session_data = {
        "user_id": user_id,
        "logged_in": True,
    }
    r.setex(
        name=f"session:{session_id}",
        time=SESSION_TTL,
        value=json.dumps(session_data)
    )
    return session_id  # 클라이언트 쿠키에 저장할 값


def get_session(session_id: str) -> dict | None:
    """요청마다 세션 조회"""
    data = r.get(f"session:{session_id}")
    if data is None:
        return None  # 세션 만료 또는 없음
    return json.loads(data)


def refresh_session(session_id: str):
    """세션 TTL 갱신 (슬라이딩 만료)"""
    r.expire(f"session:{session_id}", SESSION_TTL)


def delete_session(session_id: str):
    """로그아웃 시 세션 삭제"""
    r.delete(f"session:{session_id}")
```

**사용 예:**
```python
# 로그인
session_id = create_session(user_id=1001)
# → 쿠키에 session_id 저장

# 이후 요청
session = get_session(session_id)
if session:
    print(session["user_id"])  # 1001
else:
    print("로그인 필요")

# 로그아웃
delete_session(session_id)
```

---

### Flask에서 Redis 세션 사용 (flask-session)

```python
from flask import Flask, session
from flask_session import Session
import redis

app = Flask(__name__)
app.config["SECRET_KEY"] = "secret"
app.config["SESSION_TYPE"] = "redis"
app.config["SESSION_REDIS"] = redis.Redis(host="localhost", port=6379)
app.config["PERMANENT_SESSION_LIFETIME"] = 3600

Session(app)


@app.route("/login")
def login():
    session["user_id"] = 1001
    return "로그인 완료"


@app.route("/profile")
def profile():
    user_id = session.get("user_id")
    if not user_id:
        return "로그인 필요", 401
    return f"user_id: {user_id}"


@app.route("/logout")
def logout():
    session.clear()
    return "로그아웃 완료"
```

---

### 세션 만료 전략

| 전략 | 설명 | 구현 |
|------|------|------|
| **절대 만료** | 생성 후 고정 시간 뒤 만료 | `SETEX session:id 3600 data` |
| **슬라이딩 만료** | 요청마다 TTL 리셋 | 요청 시마다 `EXPIRE session:id 3600` |
| **명시적 삭제** | 로그아웃 시 즉시 삭제 | `DEL session:id` |

> 보안을 위해 절대 만료 + 명시적 삭제를 기본으로 하고, UX를 위해 슬라이딩 만료를 선택적으로 적용한다.

---

## Redis 를 활용한 세션 클러스터링

### 문제: 단일 서버의 세션 한계

서버가 한 대일 때는 세션을 메모리에 저장해도 무방하다.
하지만 서버가 여러 대로 확장되면 **세션 불일치** 문제가 발생한다.

```
# 문제 상황: 서버 메모리에 세션 저장 시
클라이언트 → Load Balancer → 서버 A (세션 있음) → 정상
클라이언트 → Load Balancer → 서버 B (세션 없음) → 로그아웃됨!
```

### 해결: Redis를 공유 세션 저장소로

```
클라이언트 → Load Balancer → 서버 A ┐
                                      ├→ Redis (공유 세션 저장소)
클라이언트 → Load Balancer → 서버 B ┘
```

모든 서버가 동일한 Redis를 바라보므로 어느 서버로 라우팅되어도 세션 조회가 가능하다.

---

### 세션 클러스터링 구성 예시

**여러 Flask 인스턴스 + 공유 Redis:**

```python
# 모든 서버 인스턴스에 동일하게 적용
app.config["SESSION_REDIS"] = redis.Redis(
    host="redis-cluster.internal",  # 공유 Redis 주소
    port=6379
)
```

**Docker Compose 예시:**
```yaml
services:
  web1:
    image: myapp
    environment:
      - REDIS_HOST=redis
  web2:
    image: myapp
    environment:
      - REDIS_HOST=redis
  redis:
    image: redis:7
    ports:
      - "6379:6379"
  nginx:
    image: nginx
    # web1, web2로 로드 밸런싱
```

---

### Redis 세션 클러스터링 시 고려사항

**1. Redis 고가용성 확보**

세션 저장소가 단일 장애점(SPOF)이 되지 않도록 Redis 자체의 이중화가 필요하다.

| 구성 | 특징 |
|------|------|
| **Sentinel** | Master 장애 시 자동 Failover. 소규모에 적합. |
| **Cluster** | 데이터 샤딩 + 이중화. 대규모 트래픽에 적합. |

**2. 세션 데이터 직렬화**

Python 객체를 Redis에 저장하려면 직렬화가 필요하다.
- `json.dumps` / `json.loads`: 가독성 좋음, 타입 제한 있음
- `pickle`: Python 객체 그대로 저장, 보안 주의
- `msgpack`: 빠르고 용량 작음, 별도 라이브러리 필요

**3. 세션 키 네이밍**

```
session:{session_id}          # 기본
session:{app_name}:{session_id}  # 멀티 앱 환경에서 충돌 방지
```

**4. 보안 고려사항**

- `session_id`는 반드시 **충분한 길이의 랜덤값** 사용 (UUID v4 권장)
- Redis에 민감 정보(비밀번호 등) 직접 저장 금지
- Redis에 비밀번호(`requirepass`) 및 네트워크 접근 제어 설정
- HTTPS + `HttpOnly` + `Secure` 쿠키 플래그 필수
