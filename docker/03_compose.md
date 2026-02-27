# compose

이전에 docker network를 이용하여 다수의 컨테이너를 다뤘었다. 하지만 하나씩 하지 않고 docker-compose를 이용하면 더 편하다. 여러개의 이미지를 빌드할 수 있고 여러개의 컨테이너를 띄울 수 있게한다.

- (참고) 리눅스에서는 도커 컴포즈를 따로 다운받아야한다.

### docker-compose.yaml 작성

- 설명이 잘 되어있음: https://meetup.toast.com/posts/277
- `docker-compose.yaml` 파일을 만든다.
  - 2개의 공백을 사용

```yaml
version: "3.8"
services:
  mongodb:
    image: "mongo"
    # container_name: mongodb
    volumes:
      - data:/data/db
    # environment:
    # MONGO_INITDB_ROOT_USERNAME: root
    # MONGO_INITDB_ROOT_PASSWORD: secret
    # - MONGO_INITDB_ROOT_USERNAME=root
    env_file:
      - ./env/mongo.env
  backend:
    build: ./backend
    # build:
    #   context: ./backend
    #   dockerfile: Dockerfile
    #   args:
    #     some-args: 80
    ports:
      - "80:80"
    volumes:
      - logs:/app/logs # named volume
      - ./backend:/app # bind mount, compose에서는 상대경로 이용
      - /app/node_modules # anonymous volume
    env_file:
      - ./env/backend.env
    depends_on:
      - mongodb
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    volumes: -./frontend/src:/app/src
    stdin_open: true # -i
    tty: true # -t
    depends_on:
      - backend

volumes:
  data:
  logs:
```

- `version`: docker-compose.yaml의 버전을 의미한다.
- `services`: 컨테이너들을 묶어놓은 단위이다.
- 위에서 `mongodb`, `backend`, `front` 는 컨테이너를 의미한다.
  - 나중에 도커컴포즈로 컨테이너를 띄우면 해당 이름이 컨테이너 이름의 일부로 들어가 있는 것을 알 수 있다.
  - 물론 이름을 정하고 싶으면 `container_name`을 이용하면 된다.
- `environment`부분에서 보면 `~: ~` 이 아니라 `=`을 사용하고 싶으면 맨 앞에 `-`을 붙여야 한다.
- network를 따로 추가하지 않아도 docker-compose가 모든 서비스를 디폴트 네트워크에 자동으로 추가해준다.
  - 물론 원하는 network를 추가해서 사용할 수 있다.
- `volumes`: service에 있는 컨테이너에서 명시한 volume을 작성한다.
  - named volume만 작성이 필요하다. (anonymous, bind mount는 불필요)
  - 위에서 named volume은 `data`이고 `data:` 처럼 작성하면 된다. (`:`뒤에 아무것도 없는게 정상)
  - 동일한 volume을 여러개의 컨테이너가 공유할 수도 있다.
- `build`: 해당 위치에 있는 Dockerfile 이용하여 이미지를 빌드하고 컨테이너를 띄운다.
- `depends_on`: 의존성을 나타내고 아래에 해당하는 컨테이너들이 먼저 생성된다.

### 재시작 정책 (Restart Policy)

- 컨테이너가 비정상 종료되었을 때 자동으로 재시작할지 정할 수 있다.
- `docker run`에서는 `--restart` 옵션을 사용하고, compose에서는 `restart` 키를 사용한다.

| 정책 | 설명 |
|-----|------|
| `no` | 재시작하지 않음 (기본값) |
| `always` | 항상 재시작 (수동 중지 시에도 Docker 데몬 재시작 시 다시 시작) |
| `on-failure` | 비정상 종료(exit code != 0)일 때만 재시작 |
| `unless-stopped` | 수동으로 중지하지 않는 한 항상 재시작 |

```yaml
services:
  backend:
    build: ./backend
    restart: unless-stopped
```

- 프로덕션 환경에서는 `unless-stopped` 또는 `always`를 사용하는 것이 일반적이다.

### 헬스체크 (Healthcheck)

- 컨테이너가 실행중이더라도 내부 애플리케이션이 정상 동작하는지 확인할 수 있다.
- 헬스체크에 실패하면 컨테이너 상태가 `unhealthy`로 표시된다.

```yaml
services:
  backend:
    build: ./backend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s    # 체크 간격
      timeout: 10s     # 타임아웃
      retries: 3       # 실패 허용 횟수
      start_period: 40s # 컨테이너 시작 후 대기 시간
```

### 리소스 제한

- compose에서 컨테이너의 CPU, 메모리 사용량을 제한할 수 있다.

```yaml
services:
  backend:
    build: ./backend
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
```

- `limits`: 최대 사용 가능한 리소스
- `reservations`: 최소 보장되는 리소스

### 로그 관리

- `docker-compose logs`로 모든 서비스의 로그를 한번에 볼 수 있다.
  - `docker-compose logs -f backend`처럼 특정 서비스만 follow할 수도 있다.
  - `--tail 50`으로 마지막 50줄만 볼 수도 있다.
- compose에서 로그 드라이버를 설정하여 로그 파일 크기를 제한할 수 있다. (로그가 디스크를 가득 채우는 것을 방지)

```yaml
services:
  backend:
    build: ./backend
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### Override 파일

- `docker-compose.override.yml` 파일을 만들면 `docker-compose.yml`의 설정을 자동으로 덮어쓴다.
- 개발 환경과 프로덕션 환경의 설정을 분리할 때 유용하다.
- `docker-compose up`을 하면 `docker-compose.yml` + `docker-compose.override.yml`이 자동으로 합쳐진다.
- 특정 파일을 지정하고 싶으면 `docker-compose -f docker-compose.yml -f docker-compose.prod.yml up`처럼 `-f` 옵션을 사용한다.

```yaml
# docker-compose.yml (공통 설정)
services:
  backend:
    build: ./backend
    ports:
      - "80:80"

# docker-compose.override.yml (개발 환경, 자동 적용)
services:
  backend:
    volumes:
      - ./backend:/app  # 소스코드 실시간 반영
    environment:
      - NODE_ENV=development

# docker-compose.prod.yml (프로덕션 환경, -f로 명시)
services:
  backend:
    restart: always
    environment:
      - NODE_ENV=production
```

### 실행

- `docker-compose up`으로 실행할 수 있다.
  - `-d`을 추가하면 detach모드
  - `-p 프로젝트이름`을 추가하면 동일한 docker-compose.yaml로 이름이 다른 여러 개의 프로젝트를 생성할 수 있다.
- `docker-compose down`하면 volume을 제외하고 컨테이너들을 모두 내리고 삭제한다.
  - `docker-compose down -v`하면 volume도 삭제된다.
- `docker-compose up --build`를 하면 강제로 이미지를 다시 빌드해서 컨테이너를 띄우게 한다. (위에서 backend, frontend처럼 build하는 과정이 있는 경우)
  - `--build` 옵션이 없고 이미지가 이미있다면 이미지를 빌드하지 않고 기존의 이미지를 통해서 컨테이너를 띄운다.
  - `docker-compose build`를 하면 이미지만 빌드하고 컨테이너를 띄우지는 않는다.