# Redis Pub/Sub

## Pub/Sub 패턴의 이해

Pub/Sub(Publish/Subscribe)은 **메시지를 보내는 쪽(Publisher)과 받는 쪽(Subscriber)이 서로를 직접 알지 못하고 채널을 통해 통신하는 메시징 패턴**이다.

### 기본 구조

```
Publisher → [채널: "chat:room1"] → Subscriber A
                                  → Subscriber B
                                  → Subscriber C
```

- **Publisher:** 특정 채널에 메시지를 발행. 누가 구독하는지 알 필요 없음.
- **Channel:** 메시지가 흐르는 논리적 파이프. 키처럼 문자열로 이름을 지정.
- **Subscriber:** 특정 채널을 구독하고 메시지가 오면 수신. 누가 보냈는지 알 필요 없음.

### 기존 요청-응답 방식과의 차이

| 구분 | 요청-응답 (REST/RPC) | Pub/Sub |
|------|----------------------|---------|
| 결합도 | 강결합 (서로 알아야 함) | 느슨한 결합 |
| 통신 방향 | 1:1 | 1:N |
| 동기/비동기 | 동기 | 비동기 |
| 수신 보장 | 보장됨 | 보장 안됨 (구독 중인 경우만) |
| 적합한 상황 | 즉각 응답 필요 | 이벤트 브로드캐스트, 실시간 알림 |

### Redis Pub/Sub의 특징

- **Fire and Forget:** 메시지를 저장하지 않음. 구독 중이 아닌 클라이언트는 메시지를 받을 수 없음.
- **채널 기반:** 메시지는 채널 단위로 전달. 하나의 클라이언트가 여러 채널 구독 가능.
- **패턴 구독:** 와일드카드(`*`, `?`)로 여러 채널을 한번에 구독 가능.
- **브로드캐스트:** 한 Publisher가 보낸 메시지는 해당 채널의 모든 Subscriber에게 전달.

### 주요 명령어

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `SUBSCRIBE channel` | 채널 구독 | `SUBSCRIBE chat:room1` |
| `UNSUBSCRIBE channel` | 채널 구독 해제 | `UNSUBSCRIBE chat:room1` |
| `PUBLISH channel message` | 채널에 메시지 발행 | `PUBLISH chat:room1 "hello"` |
| `PSUBSCRIBE pattern` | 패턴으로 채널 구독 | `PSUBSCRIBE chat:*` |
| `PUNSUBSCRIBE pattern` | 패턴 구독 해제 | `PUNSUBSCRIBE chat:*` |
| `PUBSUB CHANNELS pattern` | 활성 채널 목록 | `PUBSUB CHANNELS chat:*` |
| `PUBSUB NUMSUB channel` | 채널별 구독자 수 | `PUBSUB NUMSUB chat:room1` |

### Redis Pub/Sub vs Redis Stream vs Kafka

| 구분 | Redis Pub/Sub | Redis Stream | Kafka |
|------|---------------|--------------|-------|
| 메시지 저장 | X (휘발성) | O (영속) | O (영속) |
| 오프라인 수신 | 불가 | 가능 | 가능 |
| 컨슈머 그룹 | 없음 | 있음 | 있음 |
| 재처리 | 불가 | 가능 | 가능 |
| 복잡도 | 낮음 | 중간 | 높음 |
| 적합한 상황 | 실시간 브로드캐스트 | 이벤트 로그/재처리 | 대규모 데이터 파이프라인 |

> 메시지 유실이 허용되지 않거나 재처리가 필요하면 Redis Stream이나 Kafka를 써야 한다.

---

## Redis Pub/Sub을 이용한 채팅방 예시

### 설계

```
채널 이름 규칙: chat:{room_id}

사용자 A (Publisher/Subscriber) ─┐
사용자 B (Publisher/Subscriber) ─┼─ [채널: chat:room1] ─→ 모든 구독자에게 브로드캐스트
사용자 C (Subscriber만)         ─┘
```

각 사용자는 **메시지를 보낼 때는 PUBLISH**, **메시지를 받을 때는 SUBSCRIBE** 상태.
Redis에서 SUBSCRIBE 상태의 연결은 수신 전용이므로, 실제 구현에서는 **발행용/구독용 연결을 분리**한다.

---

### 구현 (Python + redis-py)

#### subscriber.py — 채팅 수신

```python
import redis
import threading
import json


def listen_chat(room_id: str, username: str):
    """채팅방 구독 (별도 스레드에서 실행)"""
    r = redis.Redis(host='localhost', port=6379, db=0)
    pubsub = r.pubsub()

    channel = f"chat:{room_id}"
    pubsub.subscribe(channel)
    print(f"[{username}] '{channel}' 채널 구독 시작")

    for message in pubsub.listen():
        if message["type"] == "message":
            data = json.loads(message["data"])
            # 자신이 보낸 메시지는 출력 안함
            if data["sender"] != username:
                print(f"[{data['sender']}]: {data['text']}")
```

#### publisher.py — 채팅 발신

```python
import redis
import json
import time


def send_message(room_id: str, username: str, text: str):
    """채팅 메시지 발행"""
    r = redis.Redis(host='localhost', port=6379, db=0)
    channel = f"chat:{room_id}"

    message = {
        "sender": username,
        "text": text,
        "timestamp": time.time(),
    }
    subscriber_count = r.publish(channel, json.dumps(message))
    print(f"→ '{channel}'에 발행 완료 (수신자 수: {subscriber_count})")
```

#### chat_client.py — 클라이언트 통합

```python
import threading
from subscriber import listen_chat
from publisher import send_message


def run_chat(room_id: str, username: str):
    # 구독은 별도 스레드에서 (블로킹)
    thread = threading.Thread(
        target=listen_chat,
        args=(room_id, username),
        daemon=True
    )
    thread.start()

    print(f"=== {room_id} 채팅방 입장 ({username}) ===")
    print("메시지를 입력하세요 (종료: /quit)")

    while True:
        text = input()
        if text == "/quit":
            break
        send_message(room_id, username, text)


if __name__ == "__main__":
    import sys
    room_id = sys.argv[1] if len(sys.argv) > 1 else "room1"
    username = sys.argv[2] if len(sys.argv) > 2 else "익명"
    run_chat(room_id, username)
```

**실행:**
```bash
# 터미널 1
python chat_client.py room1 Alice

# 터미널 2
python chat_client.py room1 Bob

# 터미널 3 (다른 방)
python chat_client.py room2 Charlie
```

---

### 패턴 구독으로 전체 알림 브로드캐스트

운영자가 모든 채팅방에 공지를 보내는 예시:

```python
import redis
import json


def admin_broadcast(message: str):
    """모든 채팅방에 공지 발행"""
    r = redis.Redis(host='localhost', port=6379, db=0)
    # 활성화된 모든 chat:* 채널에 발행
    channels = r.pubsub_channels("chat:*")
    for channel in channels:
        r.publish(channel, json.dumps({
            "sender": "[공지]",
            "text": message,
        }))
    print(f"총 {len(channels)}개 채널에 공지 발행 완료")


def listen_all_rooms(admin_name: str):
    """패턴으로 모든 채팅방 구독 (모니터링용)"""
    r = redis.Redis(host='localhost', port=6379, db=0)
    pubsub = r.pubsub()
    pubsub.psubscribe("chat:*")  # 패턴 구독

    for message in pubsub.listen():
        if message["type"] == "pmessage":
            channel = message["channel"].decode()
            data = json.loads(message["data"])
            print(f"[모니터링] {channel} | {data['sender']}: {data['text']}")
```

---

### 채팅방 입장/퇴장 이벤트 처리

```python
import redis
import json


def join_room(r: redis.Redis, room_id: str, username: str):
    channel = f"chat:{room_id}"
    r.publish(channel, json.dumps({
        "sender": "system",
        "text": f"{username}님이 입장했습니다.",
        "event": "join"
    }))


def leave_room(r: redis.Redis, room_id: str, username: str):
    channel = f"chat:{room_id}"
    r.publish(channel, json.dumps({
        "sender": "system",
        "text": f"{username}님이 퇴장했습니다.",
        "event": "leave"
    }))
```

---

### 주의사항 및 한계

**1. 메시지 유실**

구독 중이 아닐 때 발행된 메시지는 영구적으로 유실된다.
오프라인 사용자를 위한 메시지 보관이 필요하면 Redis List나 DB에 별도 저장해야 한다.

```python
def send_message_with_history(room_id: str, username: str, text: str):
    r = redis.Redis(host='localhost', port=6379, db=0)
    message = json.dumps({"sender": username, "text": text})

    # Pub/Sub으로 실시간 전달
    r.publish(f"chat:{room_id}", message)

    # List에 히스토리 보관 (최근 100개)
    r.lpush(f"history:{room_id}", message)
    r.ltrim(f"history:{room_id}", 0, 99)
```

**2. 연결 수 관리**

Subscriber는 Redis 연결을 점유한다. 채팅방이 많아질수록 연결 수가 증가하므로 연결 풀 관리가 필요하다.

**3. 확장성**

Redis 단일 노드의 Pub/Sub은 단일 서버 내에서만 동작한다.
여러 Redis 노드로 확장할 때는 모든 노드에 구독이 필요하거나, Kafka 같은 전용 메시지 브로커로 전환을 고려해야 한다.
