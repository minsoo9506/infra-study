# ArgoCD

k8s를 위한 GitOps 기반 CD(Continuous Delivery) 도구인 ArgoCD에 대해 알아보자.

## GitOps란
- Git 저장소를 **Single Source of Truth**(단일 진실 공급원)로 사용하는 운영 방식
- k8s 클러스터의 원하는 상태(desired state)를 Git에 선언적으로 저장
- Git에 변경이 생기면 자동으로 클러스터에 반영
- 기존 방식: 개발자가 `kubectl apply`나 CI 파이프라인에서 직접 클러스터에 배포
- GitOps 방식: Git에 push하면 ArgoCD가 이를 감지하여 클러스터에 자동 동기화

### GitOps의 장점
- **감사 추적(Audit Trail)**: 모든 변경 이력이 Git commit으로 남음
- **롤백 용이**: `git revert`로 이전 상태 복원 가능
- **협업**: PR/MR 리뷰를 통한 배포 검증
- **보안**: 클러스터 접근 권한 없이 Git 권한만으로 배포 가능

## ArgoCD란
- k8s를 위한 선언적 GitOps CD 도구
- Git 저장소에 정의된 k8s 매니페스트(yaml, Helm Chart, Kustomize 등)와 클러스터 상태를 지속적으로 비교
- 차이가 발생하면 자동 또는 수동으로 동기화(Sync)

## ArgoCD 설치

```bash
# namespace 생성 및 설치
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ArgoCD CLI 설치 (macOS)
brew install argocd

# 초기 admin 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Web UI 접근 (포트 포워딩)
kubectl port-forward svc/argocd-server -n argocd 8080:443
# 브라우저에서 https://localhost:8080 접속 (admin / 위에서 확인한 비밀번호)

# CLI 로그인
argocd login localhost:8080
```

## ArgoCD 아키텍처

```
┌─────────────────────────────────────────────────┐
│                  ArgoCD Server                   │
│                                                  │
│  ┌──────────────┐  ┌──────────────────────────┐ │
│  │   API Server  │  │   Web UI (Dashboard)     │ │
│  └──────────────┘  └──────────────────────────┘ │
│                                                  │
│  ┌──────────────┐  ┌──────────────────────────┐ │
│  │  Repo Server  │  │  Application Controller  │ │
│  │  (Git 연동)   │  │  (상태 비교 & 동기화)     │ │
│  └──────────────┘  └──────────────────────────┘ │
└─────────────────────────────────────────────────┘
         │                        │
         ▼                        ▼
   ┌───────────┐          ┌──────────────┐
   │ Git Repo  │          │  k8s Cluster │
   │ (desired) │          │  (actual)    │
   └───────────┘          └──────────────┘
```

- **API Server**: Web UI와 CLI 요청 처리
- **Repo Server**: Git 저장소에서 매니페스트를 가져와 렌더링
- **Application Controller**: 클러스터의 실제 상태와 Git의 원하는 상태를 비교하고 동기화

## Application 생성

### CLI로 생성

```bash
argocd app create my-app \
  --repo https://github.com/myorg/my-k8s-manifests.git \
  --path k8s/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated
```

### yaml로 생성

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/myorg/my-k8s-manifests.git
    targetRevision: main  # 브랜치, 태그, 또는 커밋 해시
    path: k8s/prod        # 저장소 내 매니페스트 경로

  destination:
    server: https://kubernetes.default.svc
    namespace: my-app

  syncPolicy:
    automated:
      prune: true       # Git에서 삭제된 리소스를 클러스터에서도 삭제
      selfHeal: true     # 수동으로 변경된 리소스를 Git 상태로 되돌림
    syncOptions:
    - CreateNamespace=true  # namespace 자동 생성
```

### 주요 필드 설명
- `source.repoURL`: Git 저장소 URL
- `source.targetRevision`: 추적할 브랜치/태그
- `source.path`: 매니페스트가 위치한 디렉토리
- `destination.server`: 배포 대상 클러스터
- `syncPolicy.automated`: 자동 동기화 설정
  - `prune`: Git에서 삭제된 리소스 자동 정리
  - `selfHeal`: 누군가 `kubectl`로 직접 수정해도 Git 상태로 복원

## Helm Chart와 연동

ArgoCD는 Helm Chart를 직접 지원한다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-helm-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/helm-charts.git
    targetRevision: main
    path: charts/my-app
    helm:
      valueFiles:
      - values.yaml
      - values-prod.yaml
      parameters:
      - name: replicaCount
        value: "3"
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
```

```yaml
# 외부 Helm repository의 Chart 사용
source:
  repoURL: https://charts.bitnami.com/bitnami
  chart: nginx
  targetRevision: 15.0.0
  helm:
    valueFiles:
    - values.yaml
```

## ArgoCD 기본 명령어

```bash
# Application 목록
argocd app list

# Application 상태 확인
argocd app get my-app

# 수동 동기화 (sync)
argocd app sync my-app

# 동기화 상태 확인
argocd app wait my-app

# Application 히스토리
argocd app history my-app

# 이전 버전으로 롤백
argocd app rollback my-app <history-id>

# Application 삭제
argocd app delete my-app
```

## Sync 상태와 Health 상태

### Sync Status
- **Synced**: Git의 상태와 클러스터가 일치
- **OutOfSync**: Git과 클러스터 상태가 다름

### Health Status
- **Healthy**: 모든 리소스가 정상
- **Progressing**: 배포 진행 중
- **Degraded**: 일부 리소스에 문제 발생
- **Missing**: 리소스가 클러스터에 없음

## Project

여러 Application을 그룹으로 관리하고 접근 권한을 제어할 수 있다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: ml-team
  namespace: argocd
spec:
  description: ML team project
  # 허용된 Git 저장소
  sourceRepos:
  - https://github.com/myorg/ml-manifests.git
  # 허용된 배포 대상
  destinations:
  - namespace: ml-*
    server: https://kubernetes.default.svc
  # 허용된 클러스터 리소스
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
```

## 실무에서의 GitOps 저장소 구조

### 단일 저장소 구조 (Monorepo)

```
k8s-manifests/
├── apps/
│   ├── api-server/
│   │   ├── base/              # 공통 매니페스트
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/          # 환경별 설정
│   │       ├── dev/
│   │       ├── staging/
│   │       └── prod/
│   └── ml-serving/
│       ├── base/
│       └── overlays/
└── argocd/
    └── applications.yaml      # ArgoCD Application 정의
```

### CI/CD 파이프라인 흐름

```
1. 개발자가 소스코드 수정 → Git push
2. CI (GitHub Actions, Jenkins 등)
   → 테스트 실행
   → Docker 이미지 빌드 & push
   → k8s 매니페스트 저장소의 image tag 업데이트 (자동 커밋)
3. ArgoCD가 매니페스트 저장소 변경 감지
4. 클러스터에 자동 동기화 (배포)
```

중요한 점은 **소스코드 저장소**와 **매니페스트 저장소**를 분리하는 것이 권장된다. 소스코드 변경과 인프라 설정 변경의 라이프사이클이 다르기 때문이다.

## ArgoCD vs 다른 CD 도구

| 특징 | ArgoCD | Flux | Jenkins |
|------|--------|------|---------|
| 방식 | Pull 기반 (GitOps) | Pull 기반 (GitOps) | Push 기반 |
| UI | Web UI 제공 | CLI 중심 | Web UI 제공 |
| 멀티 클러스터 | 지원 | 지원 | 플러그인 필요 |
| Helm 지원 | 네이티브 | 네이티브 | 플러그인 |
| 학습 곡선 | 중간 | 중간 | 높음 |

- **Pull 기반**: ArgoCD가 Git을 주기적으로 확인하여 변경사항을 가져옴 (클러스터 → Git)
- **Push 기반**: CI 파이프라인이 클러스터에 직접 배포 명령을 보냄 (CI → 클러스터)
- Pull 기반이 보안상 더 안전하다 (CI에 클러스터 접근 권한을 줄 필요 없음)