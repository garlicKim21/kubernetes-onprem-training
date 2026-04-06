# Chapter 07 — Gateway API를 이용한 HTTP 라우팅

## 학습 목표

- Gateway API가 등장한 배경과 Ingress와의 차이를 이해한다
- GatewayClass, Gateway, HTTPRoute 세 가지 핵심 리소스의 역할을 파악한다
- 역할 분리(인프라 관리자 vs 앱 개발자) 모델을 이해한다
- HTTPRoute를 작성하여 HTTP 라우팅을 구성할 수 있다

---

## 1. Gateway API란?

Gateway API는 쿠버네티스의 **차세대 인그레스(Ingress) API**입니다.

- 기존 Ingress 리소스의 한계를 해결하기 위해 설계
- Kubernetes **1.29부터 GA** (General Availability) 상태
- SIG-Network에서 관리하는 공식 프로젝트
- 다양한 구현체 지원: Cilium, Istio, Envoy Gateway, nginx 등

### 기존 Ingress의 문제점

| 문제 | 설명 |
|------|------|
| **기능 제한** | HTTP/HTTPS 라우팅만 지원, TCP/UDP/gRPC 등 미지원 |
| **어노테이션 의존** | 구현체별 기능은 비표준 어노테이션으로 설정 (이식성 저하) |
| **역할 분리 불가** | 인프라와 앱 설정이 하나의 리소스에 혼재 |
| **헤더/가중치 라우팅** | 표준으로 지원하지 않음 (어노테이션 필요) |

---

## 2. Ingress vs Gateway API 비교

| 항목 | Ingress | Gateway API |
|------|---------|-------------|
| **API 성숙도** | 레거시 (기능 동결) | GA (v1.2+, 활발히 개발 중) |
| **리소스 모델** | Ingress 1개 | GatewayClass, Gateway, HTTPRoute (3단계) |
| **역할 분리** | 불가 | 인프라 관리자 / 앱 개발자 분리 |
| **프로토콜** | HTTP/HTTPS만 | HTTP, gRPC, TCP, UDP, TLS |
| **헤더 기반 라우팅** | 어노테이션 (비표준) | 표준 스펙 |
| **가중치 기반 라우팅** | 어노테이션 (비표준) | 표준 스펙 |
| **트래픽 미러링** | 미지원 (대부분) | 표준 스펙 |
| **크로스 네임스페이스** | 제한적 | ReferenceGrant로 명시적 허용 |

---

## 3. Gateway API 핵심 리소스

### 3단계 리소스 모델

```
  ┌─────────────────────────────────────────────────┐
  │              인프라 관리자 영역                    │
  │                                                  │
  │  ┌──────────────────────────────────────────┐   │
  │  │ GatewayClass                              │   │
  │  │ • 어떤 구현체를 사용할지 정의               │   │
  │  │ • 예: Cilium, Istio, Envoy Gateway 등     │   │
  │  └──────────────────┬───────────────────────┘   │
  │                     │                            │
  │  ┌──────────────────▼───────────────────────┐   │
  │  │ Gateway                                   │   │
  │  │ • 리스너(포트, 프로토콜, 호스트) 정의       │   │
  │  │ • 실제 로드밸런서 인스턴스에 대응            │   │
  │  │ • 어떤 네임스페이스의 Route를 허용할지 설정  │   │
  │  └──────────────────┬───────────────────────┘   │
  └─────────────────────┼──────────────────────────┘
                        │
  ┌─────────────────────┼──────────────────────────┐
  │              앱 개발자 영역                       │
  │                     │                            │
  │  ┌──────────────────▼───────────────────────┐   │
  │  │ HTTPRoute                                 │   │
  │  │ • 호스트 기반 라우팅                        │   │
  │  │ • 경로(path) 기반 라우팅                    │   │
  │  │ • 헤더 기반 라우팅                          │   │
  │  │ • 백엔드 Service 연결                      │   │
  │  └──────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────┘
```

### 역할 분리

| 역할 | 담당 리소스 | 설명 |
|------|-----------|------|
| **인프라 관리자** | GatewayClass, Gateway | 네트워크 인프라 설정 (포트, TLS, IP 등) |
| **앱 개발자** | HTTPRoute, GRPCRoute 등 | 앱별 라우팅 규칙 설정 (호스트, 경로 등) |

→ 앱 개발자는 Gateway를 직접 만들 필요 없이, 기존 Gateway에 HTTPRoute만 붙이면 됩니다.

---

## 4. 리소스 상세 설명

### 4.1 GatewayClass

어떤 Gateway 구현체를 사용할지 정의합니다. 클러스터 수준의 리소스입니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium
spec:
  controllerName: io.cilium/gateway-controller
```

- `controllerName`: Gateway를 실제로 구현하는 컨트롤러
- 우리 클러스터에서는 **Cilium**이 Gateway Controller 역할을 합니다

### 4.2 Gateway

실제 리스너(listener)를 정의합니다. 로드밸런서 인스턴스에 대응됩니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: gateway-system
spec:
  gatewayClassName: cilium
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All          # 모든 네임스페이스의 Route를 허용
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: wildcard-tls
      allowedRoutes:
        namespaces:
          from: All
```

- **listeners**: 수신할 포트와 프로토콜 정의
- **allowedRoutes**: 어떤 네임스페이스의 Route를 수용할지 결정
- Gateway가 생성되면 Cilium이 자동으로 **LoadBalancer Service**를 생성하여 외부 IP를 할당합니다

### 4.3 HTTPRoute

앱 개발자가 HTTP 라우팅 규칙을 정의합니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-route
  namespace: default
spec:
  parentRefs:
    - name: shared-gateway
      namespace: gateway-system    # 연결할 Gateway
  hostnames:
    - "myapp.basphere.dev"         # 호스트 기반 라우팅
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /                # 경로 매칭
      backendRefs:
        - name: my-app-service     # 트래픽을 보낼 Service
          port: 80
```

---

## 5. 우리 클러스터의 현재 설정

### Gateway 확인

```bash
# GatewayClass 확인
kubectl get gatewayclass

# 예상 출력:
# NAME     CONTROLLER                    ACCEPTED   AGE
# cilium   io.cilium/gateway-controller  True       ...
```

```bash
# Gateway 확인
kubectl get gateway -n gateway-system

# 예상 출력:
# NAME             CLASS    ADDRESS        PROGRAMMED   AGE
# shared-gateway   cilium   172.16.200.1   True         ...
```

```bash
# Gateway 상세 정보
kubectl describe gateway shared-gateway -n gateway-system
```

> **핵심:** `shared-gateway`의 ADDRESS가 `172.16.200.1`입니다. 이것은 Cilium LB-IPAM에서 할당된 IP이며, BGP를 통해 외부에서 접근 가능합니다.

### 현재 HTTPRoute 확인

```bash
# 모든 네임스페이스의 HTTPRoute 확인
kubectl get httproute -A

# 예상 출력:
# NAMESPACE        NAME              HOSTNAMES                    PARENTREFS           AGE
# monitoring       grafana-route     ["grafana.basphere.dev"]     ["shared-gateway"]   ...
# kube-system      hubble-route      ["hubble.basphere.dev"]      ["shared-gateway"]   ...
# headlamp         headlamp-route    ["headlamp.basphere.dev"]    ["shared-gateway"]   ...
```

> 현재 Grafana, Hubble UI, Headlamp이 모두 `shared-gateway`를 통해 외부에 노출되어 있습니다.

### 특정 HTTPRoute 상세 확인

```bash
# Grafana의 HTTPRoute 확인
kubectl get httproute -n monitoring grafana-route -o yaml
```

---

## 6. 실습 데모: echo-server 배포 및 HTTPRoute 연결

### Step 1: echo-server Deployment 및 Service 생성

```bash
kubectl apply -f examples/echo-server.yaml
```

**examples/echo-server.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-server
  namespace: default
  labels:
    app: echo-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo-server
  template:
    metadata:
      labels:
        app: echo-server
    spec:
      containers:
        - name: echo
          image: ealen/echo-server:0.9.2
          ports:
            - containerPort: 80
          env:
            - name: PORT
              value: "80"
---
apiVersion: v1
kind: Service
metadata:
  name: echo-server
  namespace: default
spec:
  selector:
    app: echo-server
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

echo-server는 요청 정보(헤더, 경로, 메서드 등)를 그대로 응답하는 테스트용 서버입니다.

### Step 2: Deployment 및 Service 확인

```bash
# Pod 상태 확인
kubectl get pods -l app=echo-server

# 예상 출력:
# NAME                           READY   STATUS    RESTARTS   AGE
# echo-server-xxxxxxxxxx-xxxxx   1/1     Running   0          10s
# echo-server-xxxxxxxxxx-yyyyy   1/1     Running   0          10s

# Service 확인
kubectl get svc echo-server

# 예상 출력:
# NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# echo-server   ClusterIP   10.96.xx.xx    <none>        80/TCP    10s
```

### Step 3: HTTPRoute 생성

```bash
kubectl apply -f examples/httproute-echo.yaml
```

**examples/httproute-echo.yaml:**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: echo-route
  namespace: default
spec:
  parentRefs:
    - name: shared-gateway
      namespace: gateway-system
  hostnames:
    - "echo.basphere.dev"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: echo-server
          port: 80
```

### Step 4: HTTPRoute 상태 확인

```bash
# HTTPRoute 확인
kubectl get httproute echo-route

# 예상 출력:
# NAME         HOSTNAMES              PARENTREFS           AGE
# echo-route   ["echo.basphere.dev"]  ["shared-gateway"]   10s

# HTTPRoute 상세 확인 — Accepted/ResolvedRefs 조건 확인
kubectl describe httproute echo-route

# 상태 확인 포인트:
# Conditions:
#   Type: Accepted    Status: True    ← Gateway가 이 Route를 수락함
#   Type: ResolvedRefs Status: True   ← 백엔드 Service를 찾음
```

### Step 5: curl로 테스트

```bash
# echo.basphere.dev로 요청 (DNS가 설정된 경우)
curl -s http://echo.basphere.dev | jq .

# 또는 Gateway IP로 직접 요청 (Host 헤더 지정)
curl -s -H "Host: echo.basphere.dev" http://172.16.200.1 | jq .

# 예상 출력 (echo-server가 요청 정보를 반환):
# {
#   "host": {
#     "hostname": "echo.basphere.dev",
#     ...
#   },
#   "http": {
#     "method": "GET",
#     "baseUrl": "",
#     "originalUrl": "/",
#     ...
#   },
#   ...
# }
```

### Step 6: 경로 기반 라우팅 테스트

```bash
# 다양한 경로로 요청하여 라우팅 확인
curl -s http://echo.basphere.dev/api/test | jq .http.originalUrl
# 예상 출력: "/api/test"

curl -s http://echo.basphere.dev/health | jq .http.originalUrl
# 예상 출력: "/health"
```

---

## 7. 호스트 기반 라우팅 이해

현재 우리 클러스터에서는 **하나의 Gateway(shared-gateway)** 에 여러 HTTPRoute가 연결되어 있습니다. Gateway는 요청의 **Host 헤더**를 보고 어떤 HTTPRoute로 보낼지 결정합니다.

```
  외부 요청
     │
     ▼
  ┌─────────────────────────────────────────────┐
  │ shared-gateway (172.16.200.1)               │
  │                                              │
  │ Host: grafana.basphere.dev  → grafana-route  │──► Grafana Service
  │ Host: hubble.basphere.dev   → hubble-route   │──► Hubble Service
  │ Host: headlamp.basphere.dev → headlamp-route │──► Headlamp Service
  │ Host: echo.basphere.dev     → echo-route     │──► echo-server Service
  └─────────────────────────────────────────────┘
```

### 확인 방법

```bash
# 모든 HTTPRoute 목록 — 각각 다른 호스트에 매핑
kubectl get httproute -A

# 같은 Gateway IP로 다른 Host 헤더를 보내면 다른 서비스로 라우팅됨
curl -s -H "Host: echo.basphere.dev" http://172.16.200.1 | head -5
curl -s -H "Host: grafana.basphere.dev" http://172.16.200.1 | head -5
```

---

## 8. Gateway API 동작 흐름 전체 요약

```
  1. 인프라 관리자가 GatewayClass 생성 (Cilium 컨트롤러 지정)
  2. 인프라 관리자가 Gateway 생성 (리스너, TLS 등 설정)
     → Cilium이 자동으로 LoadBalancer Service 생성
     → Cilium LB-IPAM이 External IP 할당 (172.16.200.1)
     → BGP로 외부 라우터에 경로 광고
  3. 앱 개발자가 HTTPRoute 생성
     → parentRefs로 Gateway에 연결
     → hostnames로 호스트 매칭 규칙 정의
     → backendRefs로 대상 Service 지정
  4. 외부 클라이언트가 echo.basphere.dev 접속
     → DNS → 172.16.200.1
     → BGP 경로를 통해 클러스터 도달
     → Gateway가 Host 헤더 확인 → echo-route 매칭
     → echo-server Service → Pod
```

---

## 9. HTTPRoute 추가 기능 (참고)

### 경로 기반 라우팅

```yaml
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /api
    backendRefs:
      - name: api-service
        port: 80
  - matches:
      - path:
          type: PathPrefix
          value: /web
    backendRefs:
      - name: web-service
        port: 80
```

### 헤더 기반 라우팅

```yaml
rules:
  - matches:
      - headers:
          - name: X-Version
            value: v2
    backendRefs:
      - name: app-v2
        port: 80
  - backendRefs:
      - name: app-v1
        port: 80
```

### 가중치 기반 라우팅 (Canary 배포)

```yaml
rules:
  - backendRefs:
      - name: app-v1
        port: 80
        weight: 90      # 90%의 트래픽
      - name: app-v2
        port: 80
        weight: 10      # 10%의 트래픽
```

### 요청/응답 헤더 수정

```yaml
rules:
  - filters:
      - type: RequestHeaderModifier
        requestHeaderModifier:
          add:
            - name: X-Custom-Header
              value: "added-by-gateway"
    backendRefs:
      - name: my-service
        port: 80
```

---

## 강사 데모 시나리오

```bash
# 1. 현재 Gateway 설정 확인
kubectl get gatewayclass
kubectl get gateway -n gateway-system
kubectl get httproute -A

# 2. echo-server 배포
kubectl apply -f examples/echo-server.yaml
kubectl get pods -l app=echo-server
kubectl get svc echo-server

# 3. HTTPRoute 생성
kubectl apply -f examples/httproute-echo.yaml
kubectl get httproute echo-route
kubectl describe httproute echo-route

# 4. curl 테스트
curl -s http://echo.basphere.dev | jq .
curl -s -H "Host: echo.basphere.dev" http://172.16.200.1 | jq .

# 5. 호스트 기반 라우팅 확인
curl -s -H "Host: grafana.basphere.dev" http://172.16.200.1 | head -5

# 6. Hubble UI에서 네트워크 플로우 확인
# 브라우저에서 https://hubble.basphere.dev 접속
# default 네임스페이스에서 echo-server 트래픽 관찰

# 7. 정리
kubectl delete -f examples/httproute-echo.yaml
kubectl delete -f examples/echo-server.yaml
```

---

## 핵심 정리

1. **Gateway API**는 Ingress의 후속으로, Kubernetes 1.29부터 GA 상태입니다
2. **3단계 리소스**: GatewayClass(구현체) → Gateway(리스너/인프라) → HTTPRoute(라우팅 규칙)
3. **역할 분리**: 인프라 관리자가 GatewayClass/Gateway를, 앱 개발자가 HTTPRoute를 관리합니다
4. 우리 클러스터에서는 `shared-gateway`(172.16.200.1)를 공유하며, 호스트 기반으로 라우팅합니다
5. HTTPRoute는 경로, 헤더, 가중치 기반 라우팅 등 다양한 기능을 **표준 스펙**으로 제공합니다
6. Grafana, Hubble, Headlamp 등 모든 외부 서비스가 Gateway API를 통해 노출되어 있습니다

---

> **Day 1 완료!** 내일 Day 2에서는 스토리지, 모니터링, 보안, Helm, 고급 스케줄링, 오토스케일링을 다룹니다.
