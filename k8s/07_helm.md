# Helm

k8s 리소스를 패키지로 관리하는 Helm에 대해 알아보자.

## Helm이란
- k8s의 **패키지 매니저** (apt, yum, brew 같은 역할)
- 여러 k8s 리소스(Deployment, Service, ConfigMap 등)를 하나의 **Chart**로 묶어서 배포/관리
- 템플릿 엔진을 통해 환경별(dev, staging, prod)로 다른 설정을 적용할 수 있다.

### Helm이 필요한 이유
- 하나의 애플리케이션을 배포하려면 Deployment, Service, ConfigMap, Secret, Ingress 등 여러 yaml 파일이 필요하다.
- 이들을 일일이 `kubectl apply`하면 관리가 어렵고, 환경별로 값만 다른 yaml을 중복 작성해야 한다.
- Helm을 사용하면 하나의 Chart로 묶어서 한 번에 배포하고, values만 바꿔서 환경별 배포가 가능하다.

## 주요 개념

| 용어 | 설명 |
|------|------|
| Chart | k8s 리소스를 정의한 패키지 (하나의 디렉토리) |
| Release | Chart를 클러스터에 설치한 인스턴스 |
| Repository | Chart를 저장하고 공유하는 저장소 |
| Values | Chart의 템플릿에 주입할 설정값 |

## Helm 설치

```bash
# macOS
brew install helm

# 스크립트로 설치
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 버전 확인
helm version
```

## Helm 기본 명령어

```bash
# repository 추가
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# chart 검색
helm search repo nginx
helm search hub prometheus  # Artifact Hub에서 검색

# chart 설치
helm install my-release bitnami/nginx

# 설치된 release 목록
helm list
helm list -A  # 모든 namespace

# release 상태 확인
helm status my-release

# release 삭제
helm uninstall my-release

# 설치 전 렌더링된 yaml 미리보기 (dry-run)
helm install my-release bitnami/nginx --dry-run --debug

# 실제 생성될 yaml만 출력 (설치하지 않음)
helm template my-release bitnami/nginx
```

## Chart 구조

```
my-chart/
├── Chart.yaml          # Chart 메타정보 (이름, 버전, 설명)
├── values.yaml         # 기본 설정값
├── templates/          # k8s 리소스 템플릿
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl    # 템플릿 헬퍼 함수
│   └── NOTES.txt       # 설치 후 출력되는 안내 메시지
├── charts/             # 의존성 Chart
└── .helmignore         # 패키징 시 제외할 파일
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: My application Helm chart
type: application
version: 0.1.0        # Chart 버전
appVersion: "1.0.0"   # 애플리케이션 버전
dependencies:
- name: redis
  version: "17.x.x"
  repository: https://charts.bitnami.com/bitnami
```

### values.yaml

```yaml
# 기본 설정값 (설치 시 오버라이드 가능)
replicaCount: 2

image:
  repository: nginx
  tag: "1.24"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  host: app.example.com

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi
```

### 템플릿 예시 (templates/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
  labels:
    app: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: 80
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
```

- `{{ .Values.xxx }}`: values.yaml의 값 참조
- `{{ .Release.Name }}`: release 이름 참조
- `{{ .Chart.Name }}`: Chart.yaml의 name 참조
- `{{- toYaml ... | nindent 12 }}`: yaml 객체를 들여쓰기하여 삽입

## Values 오버라이드

```bash
# 커맨드라인에서 값 지정
helm install my-release bitnami/nginx --set replicaCount=3 --set service.type=NodePort

# 별도 values 파일 사용
helm install my-release bitnami/nginx -f prod-values.yaml

# 여러 파일 조합 (뒤의 파일이 우선)
helm install my-release bitnami/nginx -f values.yaml -f prod-values.yaml
```

환경별 values 파일을 분리하면 같은 Chart로 dev/staging/prod를 관리할 수 있다.

```
my-chart/
├── values.yaml           # 기본값
├── values-dev.yaml       # dev 환경
├── values-staging.yaml   # staging 환경
└── values-prod.yaml      # prod 환경
```

## Helm Upgrade & Rollback

```bash
# release 업그레이드 (values 변경 또는 chart 버전 업)
helm upgrade my-release bitnami/nginx --set replicaCount=5

# install + upgrade 한번에 (없으면 install, 있으면 upgrade)
helm upgrade --install my-release bitnami/nginx -f values.yaml

# 히스토리 확인
helm history my-release

# 이전 버전으로 rollback
helm rollback my-release 1  # revision 1로 복원
```

## 직접 Chart 만들기

```bash
# Chart 스캐폴딩 생성
helm create my-app

# Chart 유효성 검사
helm lint my-app/

# Chart 패키징
helm package my-app/

# 로컬 Chart 설치
helm install my-release ./my-app
```

## 실무 활용 팁

### 자주 사용하는 Chart들
```bash
# 모니터링: Prometheus + Grafana
helm install monitoring prometheus-community/kube-prometheus-stack

# 로그 수집: EFK 스택
helm install elasticsearch elastic/elasticsearch
helm install kibana elastic/kibana

# Ingress Controller
helm install ingress-nginx ingress-nginx/ingress-nginx

# Redis
helm install redis bitnami/redis
```

### Chart에서 조건부 리소스 생성

```yaml
# values.yaml에서 ingress.enabled: true로 설정한 경우에만 생성
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
spec:
  ...
{{- end }}
```