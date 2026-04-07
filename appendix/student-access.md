# 수강생 접속 가이드

## 1. Headlamp (클러스터 대시보드)

### 접속 정보

| 항목 | 값 |
|------|-----|
| **URL** | https://headlamp.basphere.dev |
| **인증 방식** | Bearer Token |

### 접속 방법

1. 웹 브라우저에서 **https://headlamp.basphere.dev** 접속
2. 로그인 화면에서 **Token** 입력란에 강사가 제공한 토큰을 붙여넣기
3. **Authenticate** 버튼 클릭

### 토큰 입력

강사가 배포한 토큰을 아래와 같이 입력합니다:

```
eyJhbGciOiJSUzI1NiIs...  (강사가 제공하는 전체 토큰 문자열)
```

> **참고**: 수강생 토큰은 읽기 전용(view) 권한만 부여되어 있습니다. 리소스를 조회할 수 있지만 생성/수정/삭제는 불가능합니다.

### Headlamp 주요 기능

| 메뉴 | 설명 |
|------|------|
| **Cluster** | 클러스터 전체 상태 (노드, 네임스페이스) |
| **Workloads** | Pod, Deployment, StatefulSet, DaemonSet 등 |
| **Network** | Service, Ingress, NetworkPolicy 등 |
| **Storage** | PV, PVC, StorageClass |
| **Configuration** | ConfigMap, Secret |

---

## 2. Grafana (모니터링 대시보드)

### 접속 정보

| 항목 | 값 |
|------|-----|
| **URL** | https://grafana.basphere.dev |
| **수강생 계정** | 사용자명: `student` / 비밀번호: `k8s-training` |
| **관리자 계정** | 사용자명: `admin` / 비밀번호: `Basphere2026!` |

### 접속 방법

1. 웹 브라우저에서 **https://grafana.basphere.dev** 접속
2. 로그인 화면에서:
   - **Email or username**: `student`
   - **Password**: `k8s-training`
3. **Log in** 버튼 클릭

### 수강생 계정 권한

- **Viewer (읽기 전용)**: 대시보드 조회만 가능
- 대시보드 생성/수정/삭제 불가
- 알림 설정 변경 불가

### 주요 대시보드 위치

왼쪽 메뉴에서 **Dashboards** 아이콘을 클릭한 후:

| 대시보드 경로 | 용도 |
|---------------|------|
| Kubernetes / Compute Resources / Cluster | 클러스터 전체 리소스 사용량 |
| Kubernetes / Compute Resources / Namespace (Pods) | 네임스페이스별 Pod 리소스 |
| Kubernetes / Compute Resources / Node (Pods) | 노드별 Pod 리소스 |
| Kubernetes / Networking / Cluster | 클러스터 네트워크 트래픽 |
| Node Exporter / Nodes | 노드 시스템 메트릭 (CPU, 메모리, 디스크) |

---

## 3. kubectl 사용 방법

### API 서버 엔드포인트

| 항목 | 값 |
|------|-----|
| **API Server** | `api.basphere.dev:6443` |
| **Kubernetes 버전** | v1.35.3 |

### kubeconfig 설정

강사가 배포한 kubeconfig 파일을 사용합니다.

```bash
# 1. kubeconfig 파일을 홈 디렉토리에 저장
mkdir -p ~/.kube
cp student-kubeconfig.yaml ~/.kube/config

# 2. 접속 테스트
kubectl cluster-info
```

**예상 출력:**
```
Kubernetes control plane is running at https://api.basphere.dev:6443
CoreDNS is running at https://api.basphere.dev:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

```bash
# 3. 노드 확인
kubectl get nodes
```

**예상 출력:**
```
NAME     STATUS   ROLES           AGE   VERSION
ctrl-0   Ready    control-plane   30d   v1.35.3
ctrl-1   Ready    control-plane   30d   v1.35.3
ctrl-2   Ready    control-plane   30d   v1.35.3
wrk-0    Ready    <none>          30d   v1.35.3
wrk-1    Ready    <none>          30d   v1.35.3
wrk-2    Ready    <none>          30d   v1.35.3
wrk-3    Ready    <none>          30d   v1.35.3
wrk-4    Ready    <none>          30d   v1.35.3
wrk-5    Ready    <none>          30d   v1.35.3
```

### kubeconfig를 기본 경로에 두지 않는 경우

```bash
# KUBECONFIG 환경변수 사용
export KUBECONFIG=/path/to/student-kubeconfig.yaml
kubectl get nodes

# 또는 매 명령어마다 --kubeconfig 플래그 사용
kubectl get nodes --kubeconfig=/path/to/student-kubeconfig.yaml
```

### 수강생 kubectl 권한

수강생 계정은 **본인 네임스페이스(lab-XX)에서는 리소스를 생성/수정/삭제**할 수 있으며, **그 외 네임스페이스는 읽기 전용(view)**입니다.

**본인 네임스페이스 (lab-XX) — 읽기/쓰기 가능:**

| 허용 작업 | 대상 리소스 |
|-----------|-------------|
| `kubectl apply -f` | Pod, Deployment, Service, ConfigMap, Secret, PVC, HPA, HTTPRoute |
| `kubectl create` | 위와 동일 |
| `kubectl delete` | 위와 동일 |
| `kubectl get/describe/logs` | 모든 리소스 |

**다른 네임스페이스 — 읽기 전용:**

| 허용 (읽기) | 불가 (쓰기) |
|-------------|-------------|
| `kubectl get` | `kubectl create` |
| `kubectl describe` | `kubectl apply` |
| `kubectl logs` | `kubectl delete` |
| `kubectl top` | `kubectl edit` |

> **참고**: 강사가 데모를 진행할 때 수강생들은 Headlamp 또는 kubectl을 통해 실시간으로 변경 사항을 확인할 수 있습니다.

---

## 4. 부하 테스트 도구

### 접속 정보

| 항목 | 값 |
|------|-----|
| **URL** | https://loadtest.basphere.dev |
| **계정** | 사용자명: `admin` / 비밀번호: `Basphere2026!` |

### 사용 방법

1. 웹 브라우저에서 **https://loadtest.basphere.dev** 접속
2. `admin` / `Basphere2026!`으로 로그인
3. CPU 또는 메모리 부하 생성 버튼 클릭
4. `kubectl get hpa -n load-tester -w` 명령으로 오토스케일링 관찰

---

## 5. Hubble UI (네트워크 관측)

### 접속 정보

| 항목 | 값 |
|------|-----|
| **URL** | https://hubble.basphere.dev |
| **인증** | 별도 인증 없음 |

### 주요 기능

- Pod 간 네트워크 트래픽 실시간 시각화
- DNS 쿼리 추적
- 네트워크 정책 적용 현황 확인
- 서비스 맵(Service Map) 표시

---

## 문제 해결

### 접속이 안 될 때

1. **DNS 확인**: `nslookup headlamp.basphere.dev` 명령으로 DNS 해석이 되는지 확인
2. **네트워크 확인**: 교육장 Wi-Fi 또는 유선 네트워크에 연결되어 있는지 확인
3. **브라우저 캐시**: 브라우저의 캐시를 삭제하고 다시 시도
4. **강사에게 문의**: 위 방법으로 해결되지 않으면 강사에게 문의하세요
