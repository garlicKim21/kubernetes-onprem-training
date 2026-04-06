# 트러블슈팅 가이드

## 일반적인 문제 해결 절차

쿠버네티스에서 문제가 발생하면 다음 순서로 진단합니다:

```
1. kubectl get ... → 리소스 상태 확인
2. kubectl describe ... → 상세 정보 및 이벤트 확인
3. kubectl logs ... → 컨테이너 로그 확인
4. kubectl exec ... → 컨테이너 내부 진입하여 디버깅
```

---

## 1. Pod 관련 문제

### Pod가 Pending 상태에서 멈춤

**증상:**
```
NAME        READY   STATUS    RESTARTS   AGE
my-pod      0/1     Pending   0          5m
```

**원인 및 해결:**

| 원인 | 확인 방법 | 해결 |
|------|-----------|------|
| 리소스 부족 (CPU/메모리) | `kubectl describe pod <이름>` → Events에서 `Insufficient cpu/memory` 확인 | 리소스 요청(requests) 줄이기 또는 노드 추가 |
| 노드 셀렉터/Affinity 불일치 | `kubectl describe pod <이름>` → Events에서 `didn't match node selector` 확인 | nodeSelector 또는 affinity 수정 |
| PVC 바인딩 대기 | `kubectl describe pod <이름>` → Events에서 `waiting for volume` 확인 | PVC 상태 확인: `kubectl get pvc` |
| 노드 Taint | `kubectl describe pod <이름>` → Events에서 `had taint` 확인 | Toleration 추가 또는 Taint 제거 |

**진단 명령어:**
```bash
kubectl describe pod <Pod이름> -n <네임스페이스>
# → Events 섹션을 확인하세요
```

---

### Pod가 CrashLoopBackOff 상태

**증상:**
```
NAME        READY   STATUS             RESTARTS   AGE
my-pod      0/1     CrashLoopBackOff   5          3m
```

**원인 및 해결:**

| 원인 | 확인 방법 | 해결 |
|------|-----------|------|
| 애플리케이션 에러 | `kubectl logs <Pod이름>` | 애플리케이션 코드/설정 수정 |
| 잘못된 command/args | `kubectl describe pod <Pod이름>` → Containers 섹션 | command 또는 args 수정 |
| 환경변수 누락 | `kubectl logs <Pod이름>` → 에러 메시지 확인 | ConfigMap/Secret/env 확인 |
| Probe 실패 | `kubectl describe pod <Pod이름>` → Events에서 Liveness probe failed | Probe 설정 수정 (경로, 포트, 시간) |

**진단 명령어:**
```bash
# 현재 로그
kubectl logs <Pod이름> -n <네임스페이스>

# 이전 컨테이너의 로그 (재시작된 경우)
kubectl logs <Pod이름> -n <네임스페이스> --previous

# 상세 정보
kubectl describe pod <Pod이름> -n <네임스페이스>
```

---

### Pod가 ImagePullBackOff 상태

**증상:**
```
NAME        READY   STATUS             RESTARTS   AGE
my-pod      0/1     ImagePullBackOff   0          2m
```

**원인 및 해결:**

| 원인 | 확인 방법 | 해결 |
|------|-----------|------|
| 이미지 이름/태그 오타 | `kubectl describe pod` → Events에서 `image not found` | 이미지 이름과 태그 확인 |
| 프라이빗 레지스트리 인증 실패 | Events에서 `unauthorized` | imagePullSecrets 설정 |
| 레지스트리 접근 불가 | Events에서 `connection refused` | 네트워크 또는 레지스트리 상태 확인 |

---

### Pod가 Running이지만 Ready가 아님 (0/1)

**증상:**
```
NAME        READY   STATUS    RESTARTS   AGE
my-pod      0/1     Running   0          2m
```

**원인:** Readiness Probe 실패

**진단:**
```bash
kubectl describe pod <Pod이름> -n <네임스페이스>
# → Conditions 섹션에서 Ready: False 확인
# → Events 섹션에서 Readiness probe failed 확인
```

**해결:**
- Readiness Probe의 경로, 포트, 초기 지연 시간 확인
- 애플리케이션이 정상적으로 요청에 응답하는지 확인

---

## 2. Service 관련 문제

### Service에 접근할 수 없음

**진단 절차:**

```bash
# 1. Service 확인
kubectl get svc <이름> -n <네임스페이스>

# 2. Endpoints 확인 — Endpoints가 비어 있으면 Pod와 연결이 안 된 것
kubectl get endpoints <이름> -n <네임스페이스>

# 3. Pod 레이블 확인
kubectl get pods -n <네임스페이스> --show-labels

# 4. Service의 selector와 Pod의 레이블이 일치하는지 확인
kubectl describe svc <이름> -n <네임스페이스>
```

**자주 발생하는 원인:**

| 원인 | 해결 |
|------|------|
| Service selector와 Pod 레이블 불일치 | selector와 Pod의 labels가 동일한지 확인 |
| targetPort가 컨테이너 포트와 다름 | targetPort를 컨테이너가 listen하는 포트로 수정 |
| Pod가 Ready 상태가 아님 | Pod의 Readiness Probe 확인 |

---

### ClusterIP Service에 외부에서 접근하려는 경우

ClusterIP는 **클러스터 내부에서만 접근 가능**합니다.

**해결:**
- LoadBalancer 타입으로 변경
- Gateway API HTTPRoute 설정
- `kubectl port-forward` 사용 (임시 테스트)

```bash
# port-forward로 임시 접근
kubectl port-forward svc/<이름> 8080:80 -n <네임스페이스>
# 이후 localhost:8080으로 접근
```

---

## 3. 스토리지 관련 문제

### PVC가 Pending 상태에서 멈춤

**진단:**
```bash
kubectl describe pvc <이름> -n <네임스페이스>
# → Events 섹션 확인
```

**원인 및 해결:**

| 원인 | Events 메시지 | 해결 |
|------|---------------|------|
| 일치하는 PV 없음 (정적 프로비저닝) | `no persistent volumes available` | PV 생성 또는 StorageClass 확인 |
| StorageClass가 존재하지 않음 | `storageclass.storage.k8s.io not found` | `kubectl get sc`로 확인 후 올바른 이름 지정 |
| WaitForFirstConsumer | `waiting for first consumer to be created` | 정상 동작. Pod가 생성되면 바인딩됨 |
| CSI Driver 오류 | `failed to provision volume` | CSI Driver Pod 상태 확인: `kubectl get pods -n vmware-system-csi` |

---

### PVC 삭제가 안 됨 (Terminating 상태)

**원인:** PVC를 사용 중인 Pod가 있으면 삭제가 보류됩니다 (PVC Protection).

**해결:**
```bash
# PVC를 사용 중인 Pod 확인
kubectl get pods -n <네임스페이스> -o json | grep <PVC이름>

# 해당 Pod 먼저 삭제
kubectl delete pod <Pod이름> -n <네임스페이스>

# 그 후 PVC 삭제됨
```

---

## 4. Deployment 관련 문제

### Deployment 업데이트 후 새 Pod가 생성되지 않음

**진단:**
```bash
kubectl rollout status deployment/<이름> -n <네임스페이스>
kubectl describe deployment <이름> -n <네임스페이스>
```

**자주 발생하는 원인:**
- 새 Pod가 Pending/CrashLoopBackOff 상태 → 위의 Pod 문제 해결 참조
- `maxUnavailable: 0`이고 리소스 부족으로 새 Pod 생성 불가

---

### 롤백 방법

```bash
# 롤아웃 히스토리 확인
kubectl rollout history deployment/<이름> -n <네임스페이스>

# 이전 버전으로 롤백
kubectl rollout undo deployment/<이름> -n <네임스페이스>

# 특정 리비전으로 롤백
kubectl rollout undo deployment/<이름> --to-revision=<번호> -n <네임스페이스>
```

---

## 5. HPA 관련 문제

### HPA의 TARGETS이 `<unknown>/50%`로 표시됨

**증상:**
```
NAME      REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS
my-hpa    Deployment/my-app  <unknown>/50%   1         10        1
```

**원인 및 해결:**

| 원인 | 해결 |
|------|------|
| metrics-server가 미설치/비정상 | `kubectl get deploy metrics-server -n kube-system` 확인 |
| Pod에 resources.requests 미설정 | Deployment의 컨테이너에 `resources.requests.cpu` 추가 |
| Pod가 방금 생성됨 | 1~2분 대기 후 재확인 |

**진단:**
```bash
# metrics-server 상태 확인
kubectl get pods -n kube-system -l k8s-app=metrics-server

# 메트릭 수집 확인
kubectl top pods -n <네임스페이스>
```

---

### HPA가 스케일 업하지 않음

**진단:**
```bash
kubectl describe hpa <이름> -n <네임스페이스>
# → Conditions 및 Events 확인
```

**가능한 원인:**
- 이미 maxReplicas에 도달
- CPU 사용률이 목표값 이하
- `ScalingActive` 조건이 `False`

---

## 6. 네트워크 관련 문제

### Pod 간 통신이 안 됨

**진단:**
```bash
# Pod에서 다른 Pod/Service로 연결 테스트
kubectl exec -it <Pod이름> -n <네임스페이스> -- wget -qO- http://<서비스이름>.<네임스페이스>.svc.cluster.local
# 또는
kubectl exec -it <Pod이름> -n <네임스페이스> -- nslookup <서비스이름>.<네임스페이스>.svc.cluster.local
```

**원인 및 해결:**

| 원인 | 해결 |
|------|------|
| DNS 해석 실패 | CoreDNS Pod 상태 확인: `kubectl get pods -n kube-system -l k8s-app=kube-dns` |
| NetworkPolicy로 차단됨 | `kubectl get networkpolicy -n <네임스페이스>` 확인 |
| Cilium Agent 오류 | `kubectl get pods -n kube-system -l k8s-app=cilium` 확인 |

---

### DNS 해석 실패

```bash
# CoreDNS Pod 상태 확인
kubectl get pods -n kube-system -l k8s-app=kube-dns

# CoreDNS 로그 확인
kubectl logs -n kube-system -l k8s-app=kube-dns

# Pod 내에서 DNS 테스트
kubectl exec -it <Pod이름> -- nslookup kubernetes.default.svc.cluster.local
```

---

## 7. kubectl 접속 문제

### kubectl이 API 서버에 연결할 수 없음

**증상:**
```
Unable to connect to the server: dial tcp: lookup api.basphere.dev: no such host
```

**해결:**
```bash
# DNS 확인
nslookup api.basphere.dev

# 직접 IP로 연결 테스트
curl -k https://10.254.0.10:6443/healthz

# kubeconfig 파일 확인
kubectl config view
```

---

### 인증/인가 오류

**증상:**
```
Error from server (Forbidden): pods is forbidden: User "student" cannot list resource "pods" in API group "" in the namespace "default"
```

**원인:** 해당 사용자/ServiceAccount에 권한이 없습니다.

**확인:**
```bash
# 현재 사용자 확인
kubectl auth whoami

# 특정 동작의 권한 확인
kubectl auth can-i list pods -n default
kubectl auth can-i create deployments -n default
```

---

## 8. 유용한 디버깅 명령어 모음

```bash
# 모든 리소스 한 번에 확인
kubectl get all -n <네임스페이스>

# 이벤트 최신순으로 보기
kubectl get events -n <네임스페이스> --sort-by='.lastTimestamp'

# 모든 네임스페이스의 비정상 Pod 찾기
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded

# 노드에서 실행 중인 Pod 확인
kubectl get pods -A -o wide --field-selector spec.nodeName=<노드이름>

# 리소스 사용량 높은 순으로 Pod 확인
kubectl top pods -n <네임스페이스> --sort-by=cpu
kubectl top pods -n <네임스페이스> --sort-by=memory

# 컨테이너 내부에 셸이 없을 때 디버깅
kubectl debug -it <Pod이름> --image=busybox:1.37 -n <네임스페이스>
```

---

## 빠른 참조: 상태별 체크리스트

| Pod 상태 | 첫 번째로 확인할 것 |
|----------|---------------------|
| **Pending** | `kubectl describe pod` → Events (리소스 부족, PVC 대기) |
| **ContainerCreating** | `kubectl describe pod` → Events (이미지 풀, 볼륨 마운트) |
| **CrashLoopBackOff** | `kubectl logs --previous` (애플리케이션 에러) |
| **ImagePullBackOff** | 이미지 이름/태그 오타, 레지스트리 인증 |
| **Running (0/1)** | Readiness Probe 실패 |
| **Terminating** | Finalizer 확인, `kubectl delete pod --force --grace-period=0` (최후 수단) |
| **Evicted** | 노드 리소스 부족 (메모리/디스크), `kubectl describe node` |
| **OOMKilled** | 메모리 제한(limits.memory) 초과, 메모리 제한 늘리기 |
