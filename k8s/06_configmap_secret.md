# ConfigMap과 Secret

설정 데이터와 민감 정보를 container 이미지와 분리하여 관리하는 방법에 대해 알아보자.

## ConfigMap

### ConfigMap이란
- container에 필요한 환경 설정을 container와 분리하여 제공하는 기능
- pod 생성 시 ConfigMap을 참조하여 환경변수나 설정 파일을 주입
- container 이미지를 다시 빌드하지 않고 설정만 변경할 수 있다.

### ConfigMap 생성

```bash
# literal 값으로 생성
kubectl create configmap my-config --from-literal=DB_HOST=mysql --from-literal=DB_PORT=3306

# 파일로 생성
kubectl create configmap app-config --from-file=config.properties

# 디렉토리 전체로 생성
kubectl create configmap dir-config --from-file=./configs/
```

```yaml
# yaml로 생성
apiVersion: v1
kind: ConfigMap
metadata:
  name: ml-config
data:
  BATCH_SIZE: "32"
  LEARNING_RATE: "0.001"
  MODEL_NAME: "resnet50"
  EPOCHS: "100"
  config.yaml: |
    training:
      optimizer: adam
      scheduler: cosine
    data:
      num_workers: 4
      prefetch_factor: 2
```

### ConfigMap 사용 방법

#### 환경변수로 주입

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-train
spec:
  containers:
  - name: trainer
    image: ml-training:v1
    envFrom:  # ConfigMap의 모든 key-value를 환경변수로 주입
    - configMapRef:
        name: ml-config
```

```yaml
# 특정 key만 선택적으로 주입
env:
- name: BATCH_SIZE
  valueFrom:
    configMapKeyRef:
      name: ml-config
      key: BATCH_SIZE
```

#### 볼륨으로 마운트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-train
spec:
  containers:
  - name: trainer
    image: ml-training:v1
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: ml-config
```

- 볼륨으로 마운트하면 ConfigMap의 각 key가 파일이 된다.
- ConfigMap 업데이트 시 마운트된 파일도 자동으로 갱신된다 (환경변수는 pod 재시작 필요).

## Secret

### Secret이란
- ConfigMap과 비슷하지만 비밀번호, API 키, 인증서 등 민감한 데이터를 저장
- base64로 인코딩되어 저장 (암호화가 아님에 주의!)
- etcd에 저장될 때 encryption at rest 설정을 통해 실제 암호화 가능

### Secret 생성

```bash
# literal 값으로 생성
kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password=p@ssw0rd

# 파일로 생성
kubectl create secret generic tls-secret --from-file=cert.pem --from-file=key.pem
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ml-secret
type: Opaque
data:
  # base64로 인코딩된 값
  AWS_ACCESS_KEY_ID: QUtJQVhYWFhYWFhYWFhYWA==
  AWS_SECRET_ACCESS_KEY: c2VjcmV0a2V5MTIzNDU2Nzg5MA==
```

```bash
# base64 인코딩/디코딩
echo -n 'mypassword' | base64         # 인코딩
echo 'bXlwYXNzd29yZA==' | base64 -d  # 디코딩
```

### Secret 타입
- `Opaque`: 일반적인 key-value (기본값)
- `kubernetes.io/tls`: TLS 인증서
- `kubernetes.io/dockerconfigjson`: Docker registry 인증 정보
- `kubernetes.io/service-account-token`: ServiceAccount 토큰

### Secret 사용 방법

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-train
spec:
  containers:
  - name: trainer
    image: ml-training:v1
    env:
    - name: AWS_ACCESS_KEY_ID
      valueFrom:
        secretKeyRef:
          name: ml-secret
          key: AWS_ACCESS_KEY_ID
    - name: AWS_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: ml-secret
          key: AWS_SECRET_ACCESS_KEY
```

### Docker Registry Secret
private Docker registry에서 이미지를 pull할 때 필요하다.

```bash
kubectl create secret docker-registry my-registry-secret \
  --docker-server=myregistry.com \
  --docker-username=user \
  --docker-password=password \
  --docker-email=user@example.com
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-pod
spec:
  imagePullSecrets:
  - name: my-registry-secret
  containers:
  - name: app
    image: myregistry.com/my-app:v1
```

## ML 워크로드에서의 활용

### 하이퍼파라미터 관리
- 학습 하이퍼파라미터를 ConfigMap으로 관리하면 이미지 재빌드 없이 실험 설정 변경 가능
- 여러 실험을 서로 다른 ConfigMap으로 정의하여 관리

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: experiment-001
data:
  config.yaml: |
    model: resnet50
    batch_size: 64
    learning_rate: 0.001
    optimizer: adamw
    weight_decay: 0.01
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: experiment-002
data:
  config.yaml: |
    model: resnet101
    batch_size: 32
    learning_rate: 0.0005
    optimizer: sgd
    momentum: 0.9
```

### 클라우드 인증 정보 관리
- S3, GCS 등 클라우드 스토리지 접근 키를 Secret으로 관리
- 모델 레지스트리, 실험 추적 서버(MLflow 등)의 인증 정보도 Secret으로 관리
- 운영 환경에서는 HashiCorp Vault, AWS Secrets Manager 등 외부 Secret 관리 도구와 연동하는 것을 권장