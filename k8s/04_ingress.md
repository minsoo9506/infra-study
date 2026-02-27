# Ingress

외부의 요청과 k8s 내부를 연결해주는 Ingress에 대해 알아보자.

## Ingress
- HTTP, HTTPS를 통해 cluster 내부의 서비스를 외부로 노출
- 기능
    - Service에 외부 URL 제공
    - 트래픽을 로드밸런싱
    - SSL 인증서 처리
    - Virtual hosting을 지정
- 이런 기능과 관련한 규칙들을 정의해둔 것이 Ingress 자체이고 실제로 동작시키는 것은 Ingress Controller이다.
- 아래의 yaml 파일로 Ingress를 생성해도 아무 일도 일어나지 않는다. Ingress는 단지 Ingress 규칙을 정의하는 선언적인 object일 뿐, 외부 요청을 받는 서버가 아니다. Ingress Controller가 외부 요청을 수신했을 때, Ingress 규칙에 기반해 이 요청을 어떻게 처리할지를 결정한다.

### Ingress 설치
[다양한 Ingress controller 구현체](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)가 있고 원하는 것을 선택해서 설치해서 사용하면 된다. minikube를 사용하고 있어서 `minikube addons enable ingress` 명령어로 ingress-nginx를 설치했다.

- `kubectl get svc -n ingress-nginx`
```bash
NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.108.58.44    <none>        80:31542/TCP,443:30202/TCP   4m54s
ingress-nginx-controller-admission   ClusterIP   10.108.234.41   <none>        443/TCP                      4m54s
```
- `kubectl  get pod -n ingress-nginx`
```bash
NAME                                        READY   STATUS      RESTARTS      AGE
ingress-nginx-admission-create-l6fcc        0/1     Completed   0             21h
ingress-nginx-admission-patch-6lp2h         0/1     Completed   0             21h
ingress-nginx-controller-5959f988fd-q2jn2   0/1     Running     1 (48s ago)   21h
```

### Ingress 사용하는 방법
1. 위처럼 ingress controller를 생성한다.
2. 외부로 노출하고 싶은 어플리케이션을 생성한다.
3. 아래와 같은 yaml 파일을 통해 ingress object를 생성한다.
4. 클라이언트에서의 요청이 ingress controller에게 전달되고 규칙에 따라 적절한 pod(endpoint)로 전달된다.

```yaml
apiVersion: extensions/v1beta1 # 지금 버전은 다를수도
kind: Ingress
metadata:
  name: marvel-ingress
spec:
  rules:
  - http:
    paths:
    - path: /
      backend:
        serviceName: marvel-service
        servicePort: 80
    - path: /pay
      backend:
        serviceName: pay-service
        servicePort: 80
```

- `/`경로로 들어온 요청을 `marvel0-service`라는 서비스의 80 port로 전달한다.

### Ingress pathType
최신 API 버전(`networking.k8s.io/v1`)에서는 pathType을 반드시 명시해야 한다.

- `Exact`: 정확히 일치하는 경로만 매칭
- `Prefix`: 해당 경로로 시작하는 모든 요청 매칭
- `ImplementationSpecific`: Ingress controller 구현체에 따라 다름

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ml-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: ml.example.com
    http:
      paths:
      - path: /predict
        pathType: Prefix
        backend:
          service:
            name: inference-service
            port:
              number: 80
      - path: /train
        pathType: Prefix
        backend:
          service:
            name: training-dashboard
            port:
              number: 8080
```

### TLS 설정
Ingress에서 HTTPS를 처리하려면 TLS 인증서를 Secret으로 등록하고 Ingress에서 참조한다.

```bash
# TLS Secret 생성
kubectl create secret tls my-tls-secret --cert=tls.crt --key=tls.key
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - ml.example.com
    secretName: my-tls-secret
  rules:
  - host: ml.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ml-service
            port:
              number: 80
```

### Ingress Annotation 활용
nginx ingress controller 기준으로 자주 사용하는 annotation들이다.

```yaml
metadata:
  annotations:
    # 요청 body 크기 제한 (대용량 데이터 전송 시)
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    # 타임아웃 설정 (ML 추론처럼 응답이 느린 경우)
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    # CORS 설정
    nginx.ingress.kubernetes.io/enable-cors: "true"
    # URL rewrite
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    # rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
```

- ML 모델 서빙에서 대용량 입력(이미지, 비디오 등)을 받거나 추론 시간이 긴 경우 `proxy-body-size`와 `proxy-read-timeout` 설정이 필수적이다.