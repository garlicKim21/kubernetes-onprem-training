# Chapter 03 — 운영 기초: ConfigMap, Secret, 리소스 관리, Probe

## 학습 목표

- ConfigMap으로 설정 데이터를 관리하는 방법을 익힌다
- Secret으로 민감한 데이터를 관리하는 방법을 이해한다
- 리소스 requests/limits와 QoS 클래스를 파악한다
- Liveness, Readiness, Startup Probe의 역할과 차이를 이해한다

---

## 1. ConfigMap

### ConfigMap이란?

ConfigMap은 **설정 데이터를 키-값 쌍으로 저장**하는 리소스입니다.

- 컨테이너 이미지와 설정을 분리하여 이식성을 높임
- 환경 변수, 명령줄 인수, 설정 파일 등으로 Pod에 주입 가능
- 민감하지 않은 데이터 전용 (민감한 데이터는 Secret 사용)

### ConfigMap 생성 — 리터럴 값으로

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_PORT=8080 \
  --from-literal=LOG_LEVEL=info

# 확인
kubectl get configmap app-config
kubectl describe configmap app-config
```

### ConfigMap 생성 — 파일로

```bash
# 설정 파일 생성
cat <<'EOF' > /tmp/app.properties
database.host=db.example.com
database.port=5432
database.name=myapp
cache.ttl=300
EOF

# ConfigMap 생성
kubectl create configmap app-properties --from-file=/tmp/app.properties

# 확인
kubectl get configmap app-properties -o yaml
```

### ConfigMap 생성 — YAML 매니페스트

```bash
kubectl apply -f examples/configmap.yaml
```

**examples/configmap.yaml:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-yaml
data:
  APP_ENV: "production"
  APP_PORT: "8080"
  LOG_LEVEL: "info"
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        location /health {
            return 200 'OK';
            add_header Content-Type text/plain;
        }
    }
```

### ConfigMap 사용 — 환경 변수로 주입

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-env-demo
spec:
  containers:
    - name: app
      image: busybox:1.37
      command: ["sh", "-c", "echo APP_ENV=$APP_ENV APP_PORT=$APP_PORT && sleep 3600"]
      env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config-yaml
              key: APP_ENV
        - name: APP_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config-yaml
              key: APP_PORT
      # 또는 ConfigMap의 모든 키를 한번에 주입
      envFrom:
        - configMapRef:
            name: app-config-yaml
```

### ConfigMap 사용 — 볼륨 마운트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-volume-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: config-volume
      configMap:
        name: app-config-yaml
        items:
          - key: nginx.conf
            path: default.conf
```

---

## 2. Secret

### Secret이란?

Secret은 **민감한 데이터(비밀번호, 토큰, 인증서 등)를 저장**하는 리소스입니다.

- ConfigMap과 유사하지만, 데이터가 **Base64 인코딩**되어 저장됨
- etcd에 암호화 설정 가능 (EncryptionConfiguration)
- RBAC으로 접근 제어 가능

> **주의:** Base64는 암호화가 아닌 인코딩입니다. 추가적인 보안 조치가 필요합니다.

### Secret 유형

| 유형 | 용도 |
|------|------|
| `Opaque` | 일반적인 키-값 데이터 (기본값) |
| `kubernetes.io/dockerconfigjson` | 컨테이너 레지스트리 인증 정보 |
| `kubernetes.io/tls` | TLS 인증서 (cert + key) |
| `kubernetes.io/basic-auth` | 기본 인증 (username + password) |
| `kubernetes.io/ssh-auth` | SSH 인증 키 |

### Secret 생성 — 명령형

```bash
# Opaque Secret
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password='S3cur3P@ssw0rd!'

# Docker Registry Secret
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass

# TLS Secret
kubectl create secret tls my-tls-secret \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem
```

### Secret 생성 — YAML 매니페스트

```bash
kubectl apply -f examples/secret.yaml
```

**examples/secret.yaml:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret-yaml
type: Opaque
# stringData는 평문으로 작성 가능 (적용 시 자동으로 Base64 인코딩)
stringData:
  username: admin
  password: "S3cur3P@ssw0rd!"
  connection-string: "postgresql://admin:S3cur3P@ssw0rd!@db.example.com:5432/myapp"
```

> **팁:** `stringData`를 사용하면 평문으로 작성할 수 있어 편리합니다. `data` 필드를 사용하면 직접 Base64로 인코딩해야 합니다.

### Secret 조회

```bash
# Secret 목록
kubectl get secrets

# 상세 조회 (값은 숨겨져 있음)
kubectl describe secret db-secret-yaml

# 값 확인 (Base64 디코딩)
kubectl get secret db-secret-yaml -o jsonpath='{.data.username}' | base64 -d
# 출력: admin

kubectl get secret db-secret-yaml -o jsonpath='{.data.password}' | base64 -d
# 출력: S3cur3P@ssw0rd!
```

### Secret 사용 — Pod에서 참조

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
    - name: app
      image: busybox:1.37
      command: ["sh", "-c", "echo DB_USER=$DB_USER && sleep 3600"]
      # 환경 변수로 주입
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secret-yaml
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-secret-yaml
              key: password
      # 볼륨 마운트로 파일 주입
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: db-secret-yaml
```

---

## 3. 리소스 Requests와 Limits

### 개념

| 설정 | 의미 |
|------|------|
| **requests** | 컨테이너가 **보장받는** 최소 리소스. 스케줄러가 이 값을 기준으로 노드를 선택 |
| **limits** | 컨테이너가 사용할 수 있는 **최대** 리소스. 초과 시 제한됨 (CPU: 쓰로틀링, 메모리: OOMKilled) |

### 리소스 단위

| 리소스 | 단위 | 예시 |
|--------|------|------|
| CPU | 밀리코어 (m) | `500m` = 0.5 CPU, `1000m` = 1 CPU |
| 메모리 | 바이트 (Mi, Gi) | `128Mi` = 128 메비바이트, `1Gi` = 1 기비바이트 |

### 예시

```bash
kubectl apply -f examples/resource-limits.yaml
```

**examples/resource-limits.yaml:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
  labels:
    purpose: resource-demo
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "250m"       # 0.25 CPU 보장
          memory: "128Mi"   # 128Mi 메모리 보장
        limits:
          cpu: "500m"       # 최대 0.5 CPU 사용 가능
          memory: "256Mi"   # 최대 256Mi 메모리 (초과 시 OOMKilled)
```

### QoS (Quality of Service) 클래스

쿠버네티스는 리소스 설정에 따라 자동으로 QoS 클래스를 부여합니다. 노드의 메모리가 부족하면 **BestEffort → Burstable → Guaranteed** 순서로 Pod를 퇴출(evict)합니다.

| QoS 클래스 | 조건 | 우선순위 |
|-----------|------|---------|
| **Guaranteed** | 모든 컨테이너의 requests = limits (CPU와 메모리 모두) | 가장 높음 (마지막에 퇴출) |
| **Burstable** | requests와 limits가 다르거나, 일부만 설정 | 중간 |
| **BestEffort** | requests와 limits가 모두 미설정 | 가장 낮음 (첫 번째로 퇴출) |

```yaml
# Guaranteed 예시 (requests = limits)
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"

# Burstable 예시 (requests < limits)
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"

# BestEffort 예시 (리소스 설정 없음)
# resources 필드를 아예 생략
```

### QoS 클래스 확인

```bash
kubectl get pod resource-demo -o jsonpath='{.status.qosClass}'
# 예상 출력: Burstable
```

---

## 4. Probe (헬스 체크)

### Probe란?

Probe는 kubelet이 컨테이너의 상태를 주기적으로 확인하는 메커니즘입니다.

### 세 가지 Probe 유형

| Probe | 목적 | 실패 시 동작 |
|-------|------|-------------|
| **Liveness Probe** | 컨테이너가 살아있는지 확인 | 컨테이너를 **재시작** |
| **Readiness Probe** | 트래픽을 받을 준비가 되었는지 확인 | Service 엔드포인트에서 **제거** (트래픽 차단) |
| **Startup Probe** | 앱이 시작 완료되었는지 확인 | 시작 완료 전까지 Liveness/Readiness 체크를 **지연** |

### Probe 방식

| 방식 | 설명 |
|------|------|
| **httpGet** | HTTP GET 요청을 보내 2xx/3xx 응답이면 성공 |
| **tcpSocket** | TCP 연결이 성립되면 성공 |
| **exec** | 컨테이너 안에서 명령 실행, 종료 코드 0이면 성공 |
| **grpc** | gRPC 헬스 체크 프로토콜 사용 |

### Probe 예시

```bash
kubectl apply -f examples/probes.yaml
```

**examples/probes.yaml:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
  labels:
    purpose: probe-demo
spec:
  containers:
    - name: app
      image: nginx:1.27
      ports:
        - containerPort: 80

      # Startup Probe: 앱이 완전히 시작될 때까지 대기
      # 최대 대기 시간: failureThreshold * periodSeconds = 30 * 10 = 300초
      startupProbe:
        httpGet:
          path: /
          port: 80
        failureThreshold: 30
        periodSeconds: 10

      # Liveness Probe: 컨테이너가 살아있는지 확인
      # 실패 시 컨테이너를 재시작
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
        failureThreshold: 3
        timeoutSeconds: 1

      # Readiness Probe: 트래픽을 받을 준비가 되었는지 확인
      # 실패 시 Service 엔드포인트에서 제거
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
        timeoutSeconds: 1
```

### Probe 파라미터 설명

| 파라미터 | 기본값 | 설명 |
|---------|--------|------|
| `initialDelaySeconds` | 0 | 컨테이너 시작 후 첫 Probe까지 대기 시간 |
| `periodSeconds` | 10 | Probe 실행 간격 |
| `timeoutSeconds` | 1 | Probe 응답 타임아웃 |
| `failureThreshold` | 3 | 연속 실패 허용 횟수 |
| `successThreshold` | 1 | 연속 성공 필요 횟수 (Readiness만 의미 있음) |

### Probe 동작 시나리오

```
  Pod 시작
     │
     ▼
  ┌──────────────────────┐
  │ Startup Probe 시작    │  ← 이 기간 동안 Liveness/Readiness 비활성
  │ 성공할 때까지 반복      │
  └──────────┬───────────┘
             │ 성공
             ▼
  ┌──────────────────────┐  ┌──────────────────────┐
  │ Liveness Probe       │  │ Readiness Probe      │
  │ 주기적으로 실행        │  │ 주기적으로 실행        │
  │                      │  │                      │
  │ 실패 → 컨테이너 재시작 │  │ 실패 → 트래픽 차단    │
  │ 성공 → 아무 일 없음    │  │ 성공 → 트래픽 허용    │
  └──────────────────────┘  └──────────────────────┘
```

### Probe 실패 관찰

```bash
# Probe 상태 확인
kubectl describe pod probe-demo

# Events 섹션에서 Probe 관련 이벤트 확인:
# Warning  Unhealthy  Liveness probe failed: ...
# Warning  Unhealthy  Readiness probe failed: ...
```

---

## 강사 데모 시나리오

```bash
# 1. ConfigMap 생성 및 확인
kubectl apply -f examples/configmap.yaml
kubectl describe configmap app-config-yaml

# 2. Secret 생성 및 확인
kubectl apply -f examples/secret.yaml
kubectl get secret db-secret-yaml -o jsonpath='{.data.username}' | base64 -d
echo

# 3. 리소스 제한 Pod 생성
kubectl apply -f examples/resource-limits.yaml
kubectl get pod resource-demo -o jsonpath='{.status.qosClass}'
echo

# 4. Probe Pod 생성 및 관찰
kubectl apply -f examples/probes.yaml
kubectl describe pod probe-demo | grep -A 20 "Containers:"

# 5. 정리
kubectl delete -f examples/
kubectl delete configmap app-config app-properties 2>/dev/null
kubectl delete secret db-secret 2>/dev/null
```

---

## 핵심 정리

1. **ConfigMap**은 설정 데이터를 Pod와 분리하여 관리합니다 (환경 변수 또는 볼륨 마운트)
2. **Secret**은 민감한 데이터를 Base64 인코딩하여 저장합니다 (추가 암호화 설정 권장)
3. **requests**는 보장 리소스, **limits**는 최대 리소스입니다
4. **QoS 클래스**(Guaranteed > Burstable > BestEffort)는 메모리 부족 시 퇴출 우선순위를 결정합니다
5. **Liveness Probe**는 재시작, **Readiness Probe**는 트래픽 차단, **Startup Probe**는 시작 대기를 담당합니다

---

> **다음 챕터**: [Ch.04 kubeadm 클러스터 구축](../ch04-kubeadm/README.md)
