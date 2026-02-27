# Volume

k8s에서 데이터를 영속적으로 저장하기 위한 Volume에 대해 알아보자.

## Volume이 필요한 이유
- pod는 일시적(ephemeral)이기 때문에 pod가 삭제되면 내부 데이터도 사라진다.
- ML 학습 데이터, 모델 체크포인트, 로그 등은 pod가 죽어도 보존되어야 한다.
- 여러 pod가 동일한 데이터에 접근해야 하는 경우도 있다.

## Volume 종류

### emptyDir
- pod가 생성될 때 만들어지고, pod가 삭제되면 같이 사라지는 임시 볼륨
- 같은 pod 내의 여러 container가 데이터를 공유할 때 유용

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "echo hello > /data/hello.txt; sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: busybox
    command: ["sh", "-c", "cat /data/hello.txt; sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}
```

- `emptyDir.medium: Memory`로 설정하면 tmpfs(RAM 기반)를 사용하여 더 빠르게 동작한다.

### hostPath
- worker node의 파일시스템을 pod에 마운트
- node에 종속적이므로 pod가 다른 node로 옮겨지면 데이터에 접근 불가
- 로그 수집, node 모니터링 등 특수한 용도로만 사용 권장

```yaml
volumes:
- name: host-volume
  hostPath:
    path: /data/logs
    type: DirectoryOrCreate  # 없으면 생성
```

- `type`: `Directory`, `DirectoryOrCreate`, `File`, `FileOrCreate` 등

## PV와 PVC

### PersistentVolume (PV)
- 클러스터 관리자가 미리 프로비저닝하는 스토리지 리소스
- namespace에 속하지 않는 클러스터 레벨 리소스
- 실제 스토리지(NFS, AWS EBS, GCP PD 등)와 연결

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:  # 테스트용, 실제 운영에서는 NFS, EBS 등 사용
    path: /mnt/data
```

### PersistentVolumeClaim (PVC)
- 사용자(개발자)가 스토리지를 요청하는 object
- PVC를 생성하면 조건에 맞는 PV가 자동으로 바인딩된다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

### Pod에서 PVC 사용

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: my-storage
      mountPath: /usr/share/nginx/html
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

### AccessModes
- `ReadWriteOnce (RWO)`: 하나의 node에서만 읽기/쓰기 가능
- `ReadOnlyMany (ROX)`: 여러 node에서 읽기만 가능
- `ReadWriteMany (RWX)`: 여러 node에서 읽기/쓰기 가능 (NFS 등 필요)

ML 학습에서 여러 worker pod가 동시에 학습 데이터를 읽어야 하는 경우 `ReadOnlyMany`가 필요하고, 체크포인트를 공유해야 하면 `ReadWriteMany`가 필요하다.

### PV Reclaim Policy
PVC가 삭제된 후 PV를 어떻게 처리할지 결정한다.
- `Retain`: PV와 데이터를 보존 (수동으로 삭제해야 함)
- `Delete`: PV와 연결된 외부 스토리지도 함께 삭제
- `Recycle`: 데이터를 삭제하고 PV를 재사용 (deprecated)

## StorageClass
- PV를 동적으로 프로비저닝하기 위한 object
- PVC 생성 시 StorageClass를 지정하면 PV를 미리 만들 필요 없이 자동으로 생성
- 클라우드 환경에서는 기본 StorageClass가 이미 설정되어 있는 경우가 많다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "50"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

```yaml
# StorageClass를 사용하는 PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ssd  # StorageClass 이름
  resources:
    requests:
      storage: 50Gi
```

- `volumeBindingMode: WaitForFirstConsumer`: PVC를 사용하는 pod가 스케줄링될 때까지 PV 생성을 지연. pod가 배치될 node의 zone에 맞는 스토리지를 생성하므로 멀티존 환경에서 중요하다.

## ML 워크로드에서의 Volume 활용 팁

### 학습 데이터 저장소
- 대용량 데이터셋은 NFS나 클라우드 스토리지(S3, GCS)를 PV로 마운트하여 사용
- CSI(Container Storage Interface) 드라이버를 통해 다양한 스토리지 연동 가능
  - 예: `s3-csi-driver`, `gcs-fuse-csi-driver`

### 모델 체크포인트
- 학습 중 체크포인트를 PVC에 저장하면 pod가 재시작되어도 이어서 학습 가능
- `Retain` 정책을 사용하여 중요한 체크포인트가 실수로 삭제되지 않도록 보호

### StatefulSet + PVC
- StatefulSet에서는 `volumeClaimTemplates`를 사용하여 각 pod마다 개별 PVC를 자동 생성

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ml-worker
spec:
  serviceName: ml-worker
  replicas: 3
  selector:
    matchLabels:
      app: ml-worker
  template:
    metadata:
      labels:
        app: ml-worker
    spec:
      containers:
      - name: worker
        image: ml-training:v1
        volumeMounts:
        - name: checkpoint
          mountPath: /checkpoints
  volumeClaimTemplates:
  - metadata:
      name: checkpoint
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 20Gi
```