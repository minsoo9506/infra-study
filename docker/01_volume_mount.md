# 이미지와 컨테이너의 데이터 다루기

### 데이터 종류

1. 우리가 만든 application(code + environment)은 빌드 할 때, 이미지에 추가된다. 이는 당연히 read-only이다.

2. 그리고 컨테이너가 실행되면서 생기는 temporary app data들이 있다.

- read + write (temporary) 이고 이미지가 아니라 컨테이너에 저장되는 데이터이다.
- 메모리나 임시파일에 저장된다.
- 자주 바뀔 수 있고 지워질 수도 있다.

3. 마지막으로 user account같은 Permanent app data가 있다.

- 이 또한 컨테이너가 실행되면서 생기는 데이터이다.
- read + write (permanent) 이고 컨테이너와 볼륨에 저장된다.
- 하지만 컨테이너가 멈추거나 재시작되어도 지워지지 않아야 해서 DB같은 곳에 저장된다.

### Volume

- 컨테이너를 삭제하면 당연히 그 안에 있던 데이터들을 삭제된다. 그런데 때로는 그 데이터들을 남기고 싶을 수도 있다.
- 볼륨(Volume)은 host machine의 folder를 의미한다. 컨테이너를 여기에 마운트해서 사용하는 것이다.
- 하지만 나중에 공부할 Bind Mount와 다르게 볼륨의 위치를 지정할 수도 없고 host machine중 어디에 있는지 알 수도 없다.
- volume은 docker가 관리하며 2가지 종류로 나눌 수 있다.
  - anonymous volume
  - named volume
- `docker volume` 명령어의 여러 옵션을 이용해서 volume에 대한 정보를 얻을 수 있다.
  - `ls`, `inspect`, `rm`

### Anonymous Volume

- anonymous volume은 컨테이너가 삭제되면 함께 삭제된다.
- 동일한 이미지로 다시 컨테이너를 만들면 기존의 anoymous volume이 아니라 새로운 anonymous volume이 생성된다. 컨테이너가 멈추면 reuse가 불가능한 것이다.
- 하지만 컨테이너가 실행되는 동안에는 컨테이너의 내부가 아니라 host machine에 데이터를 저장할 수 있는 것이다.
- anonymous volume은 `docker run -v containerFolder imageName` 이런식으로 컨테이너를 띄우면 된다.
  - Dockerfile에 `VOLUME ["containerFolder경로"]` 이렇게 추가해도 생성되지만 위의 명령어를 더 많이 쓸 것 같다.

### Named Volume

- named volume은 컨테이너가 삭제되어도 지워지지 않는다.
- named volume을 이용하면 컨테이너가 삭제되도 `docker volume ls`를 통해서 확인하면 볼륨이 그대로 있는 것을 확인할 수 있다.
- 만약에 같은 이미지로 컨테이너를 띄우면 해당 폴더를 그대로 새로운 컨테이너에서 이용할 수 있다. 물론 새로운 컨테이너를 띄울 때, 볼륨의 이름을 동일하게 유지해야한다.
- 또한 다수의 컨테이너가 동일한 named volume에 마운트할 수도 있다.
- host machine의 어느 곳에 마운트되어있는지 알 필요가 없지만 `docker volume inspect VolumeName`으로 알 수는 있다.
- `docker run -v VolumeName:containerFolder imageName`처럼 이용할 수 있다. `-v` 뒤에 먼저 볼륨이름을 원하는 대로 정하고 `:` 뒤에 마운트할 컨테이너 내부의 (폴더)경로를 지정하면 된다.
- `docker volume creat VolumeName`으로 볼륨을 만들 수도 있다.

### Bind Mount

- 볼륨과 다르게 유저가 직접 host machine의 특정 폴더를 정해서 마운트하는 것이다.
- 따라서 host machine에서 파일을 수정할 수 있다. 그러면 당연히 컨테이너에도 바로 반영된다. 반대도 동일하다.
- 위에서 설명한 named volume의 특징을 공통적으로 갖고 있다.
- `docker run -v 절대경로:containerFolder imageName`
  - 컨테이너의 폴더(파일)을 마운트하려면 절대경로로 host machine의 폴더(파일)경로를 사용해야한다.
  - 경로에 공백이 있는 경우 ""로 묶으면 된다.
- volume과 bind mount를 동시에 사용할 수도 있다.
- `docker run -v 절대경로:containerFolder:ro imageName` 에서 `:ro`를 추가하면 컨테이너에서 해당하는 폴더(하위폴더도)를 수정할 수 없다. read-only가 되는 것이다.
  - 하지만 하위폴더 중에서 컨테이너에서 수정을 하고 싶은 폴더가 있을 수 있다.
  - 이런 경우는 볼륨을 사용하면 된다. `docker run -v 절대경로:containerFolder:ro -v 수정하고싶은폴더 imageName` 그 특정한 볼륨은 bind mount가 아니라 볼륨이 된다. 덮어쓰기 같은 개념이다. 컨테이너의 경로가 더 자세한 쪽이 덮어쓴다(살아남는다).
- dockerfile에 있는 `COPY`와 비교해서 생각할 수 있다. bind mount는 개발환경에서 많이 쓰일 수 있고 프로덕션환경에서는 `COPY`를 많이 쓸 것이다.
  - 프로적션 환경에서는 컨테이너를 띄우고 코드수정이 없을 것이기 때문이다. (새로 띄우기 전까지는)

### dockerignore

- dockerfile에서 `COPY` 할 떄, 제외하고 싶은 파일도 있을 것이다.
- 그러면 `.dockerignore` 파일을 만들어서 작성하면 된다.

### tmpfs Mount

- 볼륨과 bind mount 외에 세 번째 마운트 방식이다. (Linux에서만 사용 가능)
- host machine의 **메모리(RAM)**에 데이터를 저장한다. 디스크에 기록되지 않는다.
- 컨테이너가 중지되면 데이터가 사라진다.
- 민감한 데이터(비밀번호, 토큰 등)를 임시로 저장할 때 유용하다. 디스크에 흔적이 남지 않기 때문이다.
- `docker run --tmpfs /app/temp imageName` 또는 `docker run --mount type=tmpfs,destination=/app/temp imageName`으로 사용한다.

### 볼륨 백업과 복원

- named volume의 데이터를 백업하거나 복원할 수 있다. 임시 컨테이너를 이용하는 방법이 일반적이다.

```bash
# 볼륨 백업: myvolume의 데이터를 backup.tar.gz로 압축
docker run --rm \
  -v myvolume:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/backup.tar.gz -C /data .

# 볼륨 복원: backup.tar.gz를 myvolume에 풀기
docker run --rm \
  -v myvolume:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/backup.tar.gz -C /data
```

- 위의 방법은 임시 alpine 컨테이너를 띄워서 볼륨과 현재 디렉토리를 동시에 마운트한 뒤, tar로 압축/해제하는 방식이다.

### 마운트 방식 비교 정리

| 구분 | Anonymous Volume | Named Volume | Bind Mount | tmpfs Mount |
|------|-----------------|--------------|------------|-------------|
| 호스트 위치 | Docker 관리 (알 수 없음) | Docker 관리 (inspect로 확인 가능) | 사용자가 직접 지정 | 메모리(RAM) |
| 컨테이너 삭제 시 | 함께 삭제 | 유지됨 | 유지됨 | 사라짐 |
| 여러 컨테이너 공유 | 불가능 | 가능 | 가능 | 불가능 |
| 주 용도 | 임시 데이터, 덮어쓰기 방지 | DB 데이터, 로그 등 영속 데이터 | 개발 환경 소스코드 반영 | 민감한 임시 데이터 |

## ARGuments, ENVironment variable

- docker에서 build-time ARGuments와 runtime ENVironment 변수를 사용할 수 있다.
- 옵션들을 하드코딩하지 않고 이들을 이용해서 유연한 이미지와 컨테이너를 만들 수 있는 것이다.

먼저, ENV 사용하는 법을 알아보자.

- 컨테이너 띄울 때 사용한다.
- 예를 들어, 내부포트를 변수처럼 사용하고 싶다고 하자. 그럼 Dockerfile에 아래처럼 `ENV`를 이용한다.
- 아래에서 `80`은 디폴트 값을 의미하고 Dockerfile에서 해당 변수를 이용하려면 `$`를 붙이면 된다.
- 이런 변수가 어플리케이션에 사용되는 방법은 언어마다 다르고 각 방법을 이용하면 된다.
- 그리고 `docker run --env PORT=8000 ...`으로 변수를 직접 할당하면 된다.
  - `-e PORT=8080`으로 사용할 수도 있고 여러 개인 경우 `-e var1=val1 -e var2=val2` 이런식으로 이용하면 된다.

```dockerfile
ENV PORT 80
EXPOSE $PORT
```

- Dockerfile이 아니라 환경변수가 있는 파일을 이용할 수도 있다.
- 환경변수가 있는 파일이 `.env`라는 파일이라고 한다면 `docker run --env-file .env ...` 이런식으로 하면 해당 파일에 있는 변수를 읽어서 할당한다.
  - `.env`에는 `PORT=8000` 이런식으로 되어있다.
  - 주로 보안과 관련된 경우에는 파일을 이용한다.

이제 ARG에 대해 알아보자

- 이미지 빌드 할 때 사용하는 것이다.
- 예를 들어, 위에서 `ENV`의 디폴트값이 있는데 이를 더 유동성있게 하고 싶을 수 있다.

```dockerfile
ARG DEFAULT_PORT=80
ENV $DEFAULT_PORT
```

- 위와 같이 Dockerfile을 만들고 `docker build -t imageName --build-arg DEFAULT_PORT=8000 .` 이런식으로 이미지를 빌드할 수 있다.