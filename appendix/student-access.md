# 수강생 접속 가이드

## 0. kubeconfig 다운로드 (가장 먼저!)

| 항목 | 값 |
|------|-----|
| **URL** | https://lab.basphere.dev |
| **인증** | 비밀번호 (강사가 구두로 안내) |

1. 브라우저에서 **https://lab.basphere.dev** 접속
2. 비밀번호 입력 후 입장
3. **자신의 Lab 번호** 버튼 클릭 → `lab-XX.yaml` 파일 다운로드
4. 이 파일이 kubectl과 Headlamp에서 사용할 인증 정보입니다

> 한 번 선택한 번호는 다른 사람이 사용할 수 없으므로, 강사의 안내에 따라 선택하세요.

---

## 1. Headlamp (클러스터 대시보드)

### 접속 정보

| 항목 | 값 |
|------|-----|
| **URL** | https://headlamp.basphere.dev |
| **인증 방식** | Bearer Token (kubeconfig 파일에서 추출) |

### 접속 방법

1. 웹 브라우저에서 **https://headlamp.basphere.dev** 접속
2. 로그인 화면에서 **Token** 선택
3. 다운로드한 `lab-XX.yaml` 파일을 열어 `token:` 값을 복사
4. Token 입력란에 붙여넣고 **Authenticate** 클릭

### 토큰 추출 방법

```bash
# macOS / Linux
grep "token:" ~/Downloads/lab-XX.yaml | awk '{print $2}'

# Windows PowerShell
(Get-Content ~\Downloads\lab-XX.yaml | Select-String "token:").ToString().Split(" ")[5]
```

> **참고**: 수강생은 본인 네임스페이스(lab-XX)에서 리소스 생성/삭제가 가능하며, 다른 네임스페이스는 조회만 가능합니다.

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

Lab Portal(https://lab.basphere.dev)에서 다운로드한 `lab-XX.yaml` 파일을 사용합니다.

```bash
# macOS / Linux
export KUBECONFIG=~/Downloads/lab-XX.yaml

# Windows (PowerShell)
$env:KUBECONFIG = "$HOME\Downloads\lab-XX.yaml"

# Windows (WSL) — Windows에서 다운로드한 파일을 WSL로 복사
cp /mnt/c/Users/{사용자명}/Downloads/lab-XX.yaml ~/lab-XX.yaml
export KUBECONFIG=~/lab-XX.yaml

# 접속 테스트
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
export KUBECONFIG=~/lab-XX.yaml
kubectl get nodes

# 또는 매 명령어마다 --kubeconfig 플래그 사용
kubectl get nodes --kubeconfig=~/lab-XX.yaml
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
| **접근 방식** | 뷰어 모드 (로그인 불필요), 관리자 접근은 강사만 가능 |

### 사용 방법

1. 웹 브라우저에서 **https://loadtest.basphere.dev** 접속
2. 뷰어 모드로 실시간 메트릭 관찰 가능 (부하 제어는 강사만 가능)
3. `kubectl get hpa -n load-tester -w` 명령으로 오토스케일링 관찰

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
