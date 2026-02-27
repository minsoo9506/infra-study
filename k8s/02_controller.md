# Controller

k8s를 구성하는 object중에서 Controller들에 대해 알아보자.

## Controller란
- pod의 개수를 보장

유저가 예를 들어 3개의 서버를 실행해달라고 API에 요청을 보낸다. 그러면 etcd의 정보를 참고하여 scheduler에게 요청한다. 그럼 어떤 node에 pod를 띄울지 확인하고 API에 알려준다. 그러면 API는 controller에게 3개 container를 보장하도록 요청한다. pod를 띄운 이후 문제가 생기면 controller가 이를 확인하고 API에게 요청을 한다.

- controller를 삭제하면 관련된 pod들도 같이 삭제된다. `--cascade=false` 옵션을 통해서 pod를 남길 수 있다.

### Controller의 종류
- replication controller
- replicaset, deployment(replicaset을 제어하는 상위 object)
- daemon set
- stateful sets
- cronjob, job

## Replication Controller
- pod의 개수를 보장하며 pod를 안정적으로 유지하는 것을 목표
    - 요구하는 pod의 개수가 부족하면 template를 이용해 pod 추가
    - 요구하는 pod 수 보다 많으면 최근에 생성된 pod 삭제
- 기본구성: `replicas`, `selector`, `template`

```yaml
apiVersion: v1
knd: ReplicationController
metadata:
  name: <RC 이름>
spec:
  replicas: <배포개수>
  selector:
    key: value
  template:
    <컨테이너 템플릿>
```
- 아래에서 `spec.selector`에 있는 `app: webui`와 동일한 값으로 `spec.template`에서 `labels`를 가지고 있어야 한다. 다르면 에러가 발생한다.
    - 이는 RC가 `spec.selector`에서 지정한 key:value 값과 동일한 `labels`을 갖는 pod를 띄우기 때문이다.
    - 그래서 예를 들어 이미 RC에서 원하는 pod 갯수가 running되고 있고 `spec.selector`와 같은 `labels`를 갖는 pod를 추가로 실행하면 추가로 실행한 pod는 terminating된다. (가장 최근 실행 pod 종류)
    - `spec.selector`, `labels`의 값은 해당 pod에 대한 tag 같은 것으로 이해하면 된다.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: rc-nginx
spec:
  replicas: 3
  selector:
    app: webui
  template: # template이전까지가 controller, template부터는 pod 정의
    metadata:
      name: mypod
      labels:
        app: webui
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.14
        ports:
        - containerPort: 80
```

- `kubectl create -f rc-repl.yaml`
    - 지금은 minikube로 하는거라 아래와 같지만 pod는 서로 다른 node에 만들어질 수 있다.
```bash
NAME                      READY   STATUS    RESTARTS        AGE     IP           NODE       NOMINATED NODE   READINESS GATES
rc-nginx-kpdt6            1/1     Running   0               69s     172.17.0.9   minikube   <none>           <none>
rc-nginx-qrpzt            1/1     Running   0               69s     172.17.0.8   minikube   <none>           <none>
rc-nginx-x6pjp            1/1     Running   0               69s     172.17.0.7   minikube   <none>           <none>
```

- `kubectl get rc`
```bash
NAME       DESIRED   CURRENT   READY   AGE
rc-nginx   3         3         3       2m53s
```

- `kubectl edit rc rc-nginx`
    - RC의 세부정보를 수정할 수 있다.
    - 예를 들어 `replicas`를 수정하면 실행되는 pod가 달라진다.
    - 예를 들어 원하는 pod가 잘 실행되는 상황에서 container의 버전을 바꾸면 아무런 변화가 없다. controller는 해당 `labels`의 pod 갯수만 보고 있기 때문이다. 하지만 pod를 delete하면 새롭게 pod를 생성하는데 이때는 수정한 container의 버전으로 pod를 생성한다.
- `kubectl scale rc rc-nginx --replicas=2`
    - 이런식으로 pod의 수를 조절할 수도 있다.
- 근데 그냥 yaml 파일을 수정하고 `kubectl apply -f 파일.yaml`이 가장 간단한 것 같다.

하지만 RC보다 주로 ReplicaSet을 사용한다.

## ReplicaSet
- ReplicationController와 마찬가지로 pod를 수를 유지시킨다. (위의 RC에 대한 설명 그대로 적용됨)
- ReplicationController 보다 더 많은 selector 제공
- 아래 `matchExpressions`의 의미는 version이 2.1이거나 2.2이면 된다는 의미이다.
  - operator는 `In`, `NotIn`, `Exists`, `DoesNotExist` 등이 있다.
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webui
    matchExpressions:
    - {key: version, operator: In, values: ["2.1", "2.2"]}
  template: ... 
```

- `kubectl delete rs rs-nginx --cascade=false`
  - `rs-nginx`라는 ReplicaSet은 삭제되지만 실행되고 있는 pod는 살아있다.

## Deployment
- ReplicaSet을 컨트롤해서 pod수를 조절
- rolling update & rolling back
  - rolling update: 서비스 중단없이 pod를 점진적으로 업데이트

Deployment의 실행 예시는 아래와 같다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webui
  template:
    metadata:
      name: deploy-pod
      labels:
        app: webui
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.14
```

위의 yaml 파일을 실행하고 `kubectl get deploy,rs,pod` 명령어를 통해 살펴보면 아래와 같다. 이름을 통해서 이들이 어떤 관계인지도 파악할 수 있다.

```bash
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deploy-nginx   2/2     2            2           2m22s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/deploy-nginx-7c99999d66   2         2         2       2m22s

NAME                                READY   STATUS    RESTARTS   AGE
pod/deploy-nginx-7c99999d66-l5589   1/1     Running   0          2m22s
pod/deploy-nginx-7c99999d66-p4m6d   1/1     Running   0          2m22s
```

그렇다면 이제 rolling update와 rolling back에 대해 알아보자. 위의 예시에서 conatiner의 버전을 올리려고 한다. 먼저 위처럼 이미 pod가 실행되고 있는 상태에서 아래의 명령어를 통해서 업데이트 할 수 있다.

- `kubectl set image deployment deploy-nginx nginx-container=nginx:1.15 --record`
  - `kubectl set image deployment <deploy-name> <container-name>=<new-version-image>`
  - `--record` 옵션을 넣으면 `kubectl rollout history deployment deploy-nginx`로 히스토리를 볼 수 있다.

- `kubectl rollout history deploy deploy-nginx` 명령어를 치면 아래와 같이 나온다.
  - `--record`를 안하면 `REVISION`은 있지만 `CHANGE-CAUSE`은 `<none>`으로 나온다. 처음 deployment를 만들때 옵션을 안줘서 그렇다.
  - 이때, `CHANGE-CAUSE`도 직접 정할 수 있다. yaml파일에서 `metadata`안에 `annotations`안에 `kubernetes.io/chang-cause: version 1.14`이렇게 하면 `version 1.14`라고 기록된다.

```bash
deployment.apps/deploy-nginx 
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment deploy-nginx nginx-container=nginx:1.15 --record=true
```

위의 `set`관련 명령어를 치면 기존의 pod들이 하나씩 terminating되고 동시에 새 버전의 container를 갖고 있는 pod가 running된다. 하지만 기존의 ReplicaSet은 삭제되지는 않는다. 이는 추후에 roll back해야할 수도 있기 때문이다. 이외에도 아래와 같이 `spec`에 rolling update, back과 관련해서 조절할 수 있다.
```yaml
spec:
  progressDeadlineSeconds: 600
  revisionHistoryLimit: 10
  strategy:
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 25%
    type: RollingUpdate
```

- `kubectl rollout pause deployment deployment이름`
  - 업데이트 중단
  - `pause`대신 `resume`을 쓰면 재시작

그렇다면 이제 back을 해보자.
- `kubectl rollout undo deployment deployment이름`
  - 업데이트 바로 전단계로 rollback
  - 근데 `--to-revision=3` 옵션을 통해서 revision을 직접 지정할 수도 있다.

그런데 항상 `set`으로 업데이트해야하는 것은 아니다. yaml 파일을 수정해서 진행할 수도 있다. yaml파일을 수정하고 `kubectl apply -f 파일.yaml` 하면 된다.

## DaemonSet
- 각 worker node마다 pod가 한개씩 실행되도록 보장한다.
- 로그 수집기, 모니터링 에이전트와 같은 프로그램 실행 시 사용하기 적합하다. 
- deployment처럼 rolling update 기능도 있다.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-nginx
spec:
  selector:
    matchLabels:
      app: log-collect
  template:
    metadata:
      name: daemon-pod
      labels:
        app: log-collect
    spec:
      containers:
      - name: daemon-container
        image: nginx:1.14
```

## StatefulSet
- Pod의 상태를 유지해주는 controller
  - pod 이름, pod의 volume(저장소)
- 위에서 봤던 지금까지의 controller는 pod의 갯수를 유지해주지만 예를 들면 pod가 생성될 때마다 이름이 달라진다. StatefulSet의 경우 `name`에 숫자가 순서대로 0,1,2.. 이 붙어서 pod이름이 정해지고 고유하게 식별한다.
- 예를 들어, `rs-nginx-0`인 pod를 지우면 이름이 동일한 pod를 다시 띄운다.
- deployment처럼 rolling update 기능도 있다.

```yaml
apiVersion: apps/v1
kind: Statefulset
metadata:
  name: rs-nginx
spec:
  replicas: 3
  serviceName: sf-nginx
  selector:
    matchLabels:
      app: webui
  template:
...
```

## Job
- k8s는 pod를 running 중인 상태로 유지한다. 그런데 특정 동작을 처리하는 pod는 작업이 완료되면 종료되도록 하는 것이 필요하다.
- Batch 처리에 적합한 contoller로 pod의 성공적인 완료를 보장한다.
  - 비정상종료면 다시 실행, 정상종료면 종료
  - 옵션으로 container를 재실행할 수도 있고 pod를 재실행할 수도 있다.
- pod를 삭제하는 것은 아니다. `STATUS`가 `completed`로 된다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-example
spec:
  completions: 5 # job 성공했다고 여겨지려면 pod가 총 몇개 성공해야하는지
  parallelism: 2 # 동시에 실행할 job의 수
  activeDeadlineSeconds: 15 # 지정시간내에 job 완료해야됨, 넘으면 실패로 생각
  template:
    spec:
      restartPolicy: Never # 실패시 Never면 새로운 pod 실행, OnFailure면 다시 실행
      containers:
      - name: centos-container
        image: centos:7
        command: ["bash"]
        args:
        - "-c"
        - "echo 'hello world'; sleep 3; echo 'Bye'"
```

## CronJob
- 사용자가 원하는 시간에 Job 실행 예약
- Linux의 cronjob의 스케줄링 기능을 Job Controller에 추가한 API
- Cronjob Schedule: `* * * * *`
  - 순서대로 minute, hour, day of the month, month, day of the week
  - 예를 들어, `30 9 * * 1-5`는 매월 월~금 9시 30분에 job을 실행한다는 의미이다.

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: job-example
spec:
  schedule: "0 3 1 * *"
  startingDeadlineSeconds: 300 # 500초동안 job실행 못하면 취소
  concurrencyPolicy: Forbid # job 동시진행을 막음
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: centos-container
            image: centos:7
            command: ["bash"]
            args:
            - "-c"
            - "echo 'hello world'; sleep 3; echo 'Bye'"
```

## HPA (Horizontal Pod Autoscaler)
- pod의 CPU/메모리 사용률 등을 기반으로 pod 수를 자동으로 조절
- Deployment, ReplicaSet, StatefulSet에 적용 가능
- metrics-server가 설치되어 있어야 한다.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
# CLI로 간단하게 HPA 생성
kubectl autoscale deployment api-server --cpu-percent=70 --min=2 --max=10
# HPA 상태 확인
kubectl get hpa
```

- `scaleTargetRef`: 오토스케일링 대상
- `minReplicas` / `maxReplicas`: pod 수 범위
- `averageUtilization`: 목표 사용률 (이 값을 넘으면 scale out)

ML 추론 서버처럼 요청량에 따라 부하가 달라지는 워크로드에서 유용하다. custom metrics(예: GPU 사용률, 요청 큐 길이)를 기반으로 스케일링하려면 Prometheus Adapter 같은 도구를 추가로 사용한다.