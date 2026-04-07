# Chapter 02 — 핵심 워크로드: Pod, ReplicaSet, Deployment

## 학습 목표

- Pod의 개념과 라이프사이클을 이해한다
- YAML 매니페스트로 Pod를 생성하는 방법을 익힌다
- Label과 Selector의 역할을 이해한다
- ReplicaSet과 Deployment의 관계와 차이를 파악한다
- Rolling Update와 Rollback을 수행할 수 있다

---

> 💻 **수강생 실습** — 이 섹션은 각자의 lab 네임스페이스에서 직접 실습합니다.

## 1. Pod

### Pod란?

Pod는 쿠버네티스에서 **배포 가능한 가장 작은 단위**입니다.

- 하나 이상의 컨테이너를 포함하며, 보통 1개의 컨테이너를 실행
- 같은 Pod 내 컨테이너는 **네트워크(IP)와 스토리지를 공유**
- Pod 자체는 일시적(ephemeral) — 삭제되면 사라짐

### Pod 라이프사이클

```
  Pending → Running → Succeeded / Failed
                ↑
            CrashLoopBackOff (반복 실패 시)
```

| 단계 | 설명 |
|------|------|
| **Pending** | 스케줄러가 노드를 배정 중이거나, 이미지를 Pull 중 |
| **Running** | 노드에 배치되어 컨테이너가 실행 중 |
| **Succeeded** | 모든 컨테이너가 성공적으로 종료 (보통 Job에서 발생) |
| **Failed** | 컨테이너가 에러로 종료 |
| **CrashLoopBackOff** | 컨테이너가 반복적으로 실패하여 재시작 대기 중 |

### Pod 생성 — kubectl run (명령형)

```bash
# 간단한 nginx Pod 생성
kubectl run my-nginx --image=nginx:1.27

# Pod 확인
kubectl get pods

# 예상 출력:
# NAME       READY   STATUS    RESTARTS   AGE
# my-nginx   1/1     Running   0          10s
```

### Pod 생성 — YAML (선언형)

```bash
kubectl apply -f examples/pod.yaml
```

**examples/pod.yaml:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx
  labels:
    app: nginx
    environment: training
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

### YAML 구조 설명

| 필드 | 설명 |
|------|------|
| `apiVersion` | 사용할 API 버전 (v1, apps/v1 등) |
| `kind` | 리소스 종류 (Pod, Deployment 등) |
| `metadata` | 이름, 라벨, 네임스페이스 등 메타 정보 |
| `spec` | 리소스의 원하는 상태(desired state) 정의 |

---

## 2. Pod 조사 및 디버깅

### kubectl describe — 상세 정보 확인

```bash
kubectl describe pod my-nginx
```

출력에서 확인할 수 있는 주요 정보:
- **Node**: Pod가 실행 중인 노드
- **IP**: Pod에 할당된 IP
- **Containers**: 컨테이너 상태, 이미지, 포트
- **Conditions**: Ready, Initialized, ContainersReady 등
- **Events**: 스케줄링, 이미지 Pull, 컨테이너 시작 등의 이벤트

### kubectl logs — 로그 확인

```bash
# Pod 로그 출력
kubectl logs my-nginx

# 실시간 로그 스트리밍
kubectl logs -f my-nginx

# 마지막 100줄만 출력
kubectl logs --tail=100 my-nginx
```

### kubectl exec — 컨테이너 내부 접속

```bash
# 컨테이너 안에서 명령 실행
kubectl exec my-nginx -- cat /etc/nginx/nginx.conf

# 대화형 셸 접속
kubectl exec -it my-nginx -- /bin/bash
```

### kubectl port-forward — 로컬 포워딩

```bash
# 로컬 8080 포트 → Pod의 80 포트로 포워딩
kubectl port-forward pod/my-nginx 8080:80

# 브라우저에서 http://localhost:8080 으로 접속 가능
```

---

## 3. Label과 Selector

### Label이란?

Label은 쿠버네티스 리소스에 부여하는 **키-값 쌍의 메타데이터**입니다. 리소스를 분류하고 선택하는 데 사용됩니다.

```yaml
metadata:
  labels:
    app: nginx          # 애플리케이션 이름
    environment: prod   # 환경
    tier: frontend      # 역할
    version: v1.2.0     # 버전
```

### Label로 조회 필터링

```bash
# 특정 라벨을 가진 Pod 조회
kubectl get pods -l app=nginx

# 여러 라벨 조건 (AND)
kubectl get pods -l app=nginx,environment=training

# 라벨 값이 아닌 경우
kubectl get pods -l 'environment!=production'

# 라벨 존재 여부
kubectl get pods -l 'app'

# 라벨 컬럼을 추가하여 출력
kubectl get pods --show-labels
```

### Selector

Selector는 Label을 기반으로 리소스를 선택하는 메커니즘입니다. ReplicaSet, Deployment, Service 등에서 사용됩니다.

```yaml
# matchLabels: 정확한 일치
selector:
  matchLabels:
    app: nginx

# matchExpressions: 조건식
selector:
  matchExpressions:
    - key: environment
      operator: In
      values: ["staging", "production"]
```

---

## 4. ReplicaSet

### ReplicaSet이란?

ReplicaSet은 **지정된 수의 Pod 복제본이 항상 실행**되도록 보장합니다.

- Pod가 삭제되면 자동으로 새 Pod를 생성
- Pod 수가 초과하면 초과분을 삭제
- Label Selector로 관리할 Pod를 식별

### ReplicaSet YAML

```bash
kubectl apply -f examples/replicaset.yaml
```

**examples/replicaset.yaml:**

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      managed-by: replicaset
  template:
    metadata:
      labels:
        app: nginx
        managed-by: replicaset
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

### ReplicaSet 관리

```bash
# ReplicaSet 조회
kubectl get replicaset
# 또는
kubectl get rs

# Pod 확인 (3개 생성됨)
kubectl get pods -l managed-by=replicaset

# 예상 출력:
# NAME             READY   STATUS    RESTARTS   AGE
# nginx-rs-xxxxx   1/1     Running   0          30s
# nginx-rs-yyyyy   1/1     Running   0          30s
# nginx-rs-zzzzz   1/1     Running   0          30s

# Pod 하나를 삭제해보기 → ReplicaSet이 자동으로 새 Pod 생성
kubectl delete pod nginx-rs-xxxxx
kubectl get pods -l managed-by=replicaset
# 여전히 3개의 Pod가 존재함을 확인
```

### 수동 스케일링

```bash
# 복제본 수를 5로 변경
kubectl scale replicaset nginx-rs --replicas=5
kubectl get pods -l managed-by=replicaset
# 5개의 Pod 확인

# 다시 2로 축소
kubectl scale replicaset nginx-rs --replicas=2
kubectl get pods -l managed-by=replicaset
# 2개의 Pod만 남음
```

> **참고:** 실제 운영에서는 ReplicaSet을 직접 생성하지 않고, **Deployment**를 통해 관리합니다.

---

## 5. Deployment

### Deployment란?

Deployment는 ReplicaSet의 **상위 컨트롤러**로, 다음 기능을 추가로 제공합니다:

- **Rolling Update**: 무중단 업데이트
- **Rollback**: 이전 버전으로 되돌리기
- **버전 이력 관리**: 업데이트 히스토리 추적

### 관계 구조

```
  Deployment
     │
     └──▶ ReplicaSet (자동 생성/관리)
              │
              ├──▶ Pod
              ├──▶ Pod
              └──▶ Pod
```

### Deployment 생성

```bash
kubectl apply -f examples/deployment.yaml
```

**examples/deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      managed-by: deployment
  template:
    metadata:
      labels:
        app: nginx
        managed-by: deployment
    spec:
      containers:
        - name: nginx
          image: nginx:1.26
          ports:
            - containerPort: 80
```

### Deployment 조회

```bash
# Deployment 확인
kubectl get deployment
kubectl get deploy

# 예상 출력:
# NAME           READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-deploy   3/3     3            3           30s

# 자동 생성된 ReplicaSet 확인
kubectl get rs

# 예상 출력:
# NAME                      DESIRED   CURRENT   READY   AGE
# nginx-deploy-xxxxxxxxxx   3         3         3       30s

# Pod 확인
kubectl get pods -l managed-by=deployment
```

### 수동 스케일링

```bash
kubectl scale deployment nginx-deploy --replicas=5

# 확인
kubectl get deploy nginx-deploy
# READY가 5/5로 변경됨
```

---

## 6. Rolling Update (무중단 업데이트)

### 이미지 업데이트

```bash
# nginx 이미지를 1.26 → 1.27로 업데이트
kubectl set image deployment/nginx-deploy nginx=nginx:1.27

# 또는 YAML 파일 수정 후 apply
# (deployment.yaml에서 image: nginx:1.27로 변경)
kubectl apply -f examples/deployment.yaml
```

### 업데이트 과정 관찰

```bash
# 롤아웃 상태 확인 (실시간)
kubectl rollout status deployment/nginx-deploy

# 예상 출력:
# Waiting for deployment "nginx-deploy" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "nginx-deploy" rollout to finish: 2 out of 3 new replicas have been updated...
# deployment "nginx-deploy" successfully rolled out
```

### Rolling Update 동작 원리

```
  1. 새로운 ReplicaSet 생성 (새 이미지)
  2. 새 ReplicaSet의 Pod를 점진적으로 증가
  3. 기존 ReplicaSet의 Pod를 점진적으로 감소
  4. 완료 후 기존 ReplicaSet은 replicas=0 상태로 유지 (롤백 대비)

  시간 →
  Old RS: ●●● → ●●○ → ●○○ → ○○○  (점진적 축소)
  New RS: ○○○ → ○○● → ○●● → ●●●  (점진적 확장)
```

```bash
# ReplicaSet 두 개 확인 (이전 + 현재)
kubectl get rs

# 예상 출력:
# NAME                      DESIRED   CURRENT   READY   AGE
# nginx-deploy-aaaaaaaaaa   3         3         3       10s   ← 새 ReplicaSet
# nginx-deploy-bbbbbbbbbb   0         0         0       5m    ← 이전 ReplicaSet
```

---

## 7. Rollback (되돌리기)

### 롤아웃 이력 확인

```bash
kubectl rollout history deployment/nginx-deploy

# 예상 출력:
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
```

### 특정 리비전 상세 조회

```bash
kubectl rollout history deployment/nginx-deploy --revision=1
```

### 이전 버전으로 롤백

```bash
# 바로 이전 버전으로 롤백
kubectl rollout undo deployment/nginx-deploy

# 특정 리비전으로 롤백
kubectl rollout undo deployment/nginx-deploy --to-revision=1

# 롤백 확인
kubectl rollout status deployment/nginx-deploy
kubectl get deploy nginx-deploy -o jsonpath='{.spec.template.spec.containers[0].image}'
# 예상 출력: nginx:1.26
```

---

## 정리

```bash
kubectl delete deployment nginx-deploy
kubectl delete replicaset nginx-rs 2>/dev/null
kubectl delete pod my-nginx 2>/dev/null
```

---

## 핵심 요약

1. **Pod**는 쿠버네티스의 최소 배포 단위이며, 일시적(ephemeral)입니다
2. **Label과 Selector**는 리소스를 분류하고 연결하는 핵심 메커니즘입니다
3. **ReplicaSet**은 지정된 수의 Pod를 항상 유지합니다
4. **Deployment**는 ReplicaSet을 관리하며 Rolling Update와 Rollback을 제공합니다
5. 실무에서는 Pod나 ReplicaSet을 직접 만들지 않고, **Deployment를 통해 관리**합니다

---

> **다음 챕터**: [Ch.03 운영 기초: ConfigMap, Secret, 리소스 관리, Probe](../ch03-operations-basics/README.md)
