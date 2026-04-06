# Chapter 04 — kubeadm 클러스터 구축

> **참고:** 이 섹션은 강사가 별도 VM 환경에서 데모로 진행합니다. 수강생은 과정을 관찰하며, 실제 교육 클러스터에는 영향을 주지 않습니다.

## 학습 목표

- kubeadm의 역할과 동작 방식을 이해한다
- 노드 사전 준비 단계를 파악한다
- kubeadm init / join 흐름을 이해한다
- HA(고가용성) 클러스터 구성 방법을 개괄적으로 파악한다

---

## 1. kubeadm이란?

kubeadm은 쿠버네티스 클러스터를 부트스트래핑하는 공식 도구입니다.

### kubeadm이 하는 일

- Control Plane 컴포넌트(API Server, etcd, Scheduler, Controller Manager)를 **Static Pod**로 배포
- 인증서(CA, API Server 인증서 등) 자동 생성
- kubeconfig 파일 생성
- 클러스터 부트스트래핑 (RBAC, CoreDNS 등 기본 addon 설치)
- 워커 노드 조인을 위한 토큰 발급

### kubeadm이 하지 않는 일

- OS 설치 및 설정
- 컨테이너 런타임(containerd) 설치
- CNI 플러그인(Cilium) 설치
- 로드밸런서, 스토리지 등 인프라 구성

---

## 2. 노드 사전 준비 (모든 노드 공통)

### 2.1 스왑 비활성화

쿠버네티스는 메모리 관리의 예측 가능성을 위해 스왑을 비활성화해야 합니다.

```bash
# 즉시 스왑 비활성화
sudo swapoff -a

# 영구 비활성화 (/etc/fstab에서 swap 라인 주석 처리)
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab
```

### 2.2 커널 모듈 로드

```bash
# 필요한 커널 모듈 설정
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# 즉시 로드
sudo modprobe overlay
sudo modprobe br_netfilter
```

### 2.3 커널 파라미터 설정

```bash
# 네트워크 브릿지 및 IP 포워딩 설정
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 적용
sudo sysctl --system
```

### 2.4 containerd 설치

```bash
# containerd 패키지 설치 (배포판에 따라 다름)
sudo apt-get update
sudo apt-get install -y containerd

# 기본 설정 파일 생성
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# SystemdCgroup 활성화 (중요!)
# config.toml에서 SystemdCgroup = true로 변경
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# containerd 재시작
sudo systemctl restart containerd
sudo systemctl enable containerd
```

> **중요:** `SystemdCgroup = true` 설정은 kubelet이 사용하는 cgroup 드라이버와 일치시키기 위해 반드시 필요합니다.

### 2.5 쿠버네티스 패키지 설치

```bash
# 쿠버네티스 APT 저장소 추가 (v1.35 기준)
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# kubeadm, kubelet, kubectl 설치
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# 버전 고정 (자동 업그레이드 방지)
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## 3. kubeadm init (첫 번째 Control Plane 노드)

### 3.1 초기화 명령

```bash
sudo kubeadm init \
  --control-plane-endpoint "10.254.0.10:6443" \
  --upload-certs \
  --pod-network-cidr "10.244.0.0/16"
```

| 옵션 | 설명 |
|------|------|
| `--control-plane-endpoint` | HA 구성을 위한 API Server VIP (로드밸런서 주소) |
| `--upload-certs` | 추가 Control Plane 노드가 인증서를 자동으로 가져갈 수 있도록 설정 |
| `--pod-network-cidr` | Pod 네트워크 CIDR (CNI에 따라 다름) |

### 3.2 init 후 수행되는 작업

1. 인증서 생성 (`/etc/kubernetes/pki/`)
2. kubeconfig 파일 생성 (`/etc/kubernetes/`)
3. Static Pod 매니페스트 생성 (`/etc/kubernetes/manifests/`)
4. etcd 클러스터 부트스트래핑
5. 기본 addon 설치 (CoreDNS, kube-proxy)
6. 조인 토큰 및 명령어 출력

### 3.3 kubectl 설정

```bash
# 일반 사용자로 kubectl 사용
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 4. 추가 Control Plane 노드 조인

HA 구성을 위해 나머지 Control Plane 노드를 조인합니다.

```bash
# kubeadm init 완료 후 출력되는 명령어 사용
sudo kubeadm join 10.254.0.10:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key>
```

---

## 5. Worker 노드 조인

```bash
# Worker 노드 조인 (--control-plane 옵션 없이)
sudo kubeadm join 10.254.0.10:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

## 6. CNI 설치 (Cilium)

kubeadm init 후, 노드가 Ready 상태가 되려면 CNI 플러그인을 설치해야 합니다.

```bash
# Cilium CLI 설치 후
cilium install --version 1.19.2

# 또는 Helm으로 설치
helm install cilium cilium/cilium --version 1.19.2 \
  --namespace kube-system \
  --set kubeProxyReplacement=true
```

> CNI가 설치되어야 노드 상태가 `NotReady`에서 `Ready`로 변경됩니다.

---

## 7. kubeadm 전체 흐름 요약

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    사전 준비 (모든 노드)                       │
  │  1. 스왑 비활성화                                             │
  │  2. 커널 모듈 및 파라미터 설정                                 │
  │  3. containerd 설치 및 설정                                   │
  │  4. kubeadm, kubelet, kubectl 설치                           │
  └──────────────────────────┬───────────────────────────────────┘
                             │
                             ▼
  ┌──────────────────────────────────────────────────────────────┐
  │           kubeadm init (첫 번째 Control Plane)                │
  │  • 인증서 생성                                                │
  │  • Static Pod 매니페스트 생성                                  │
  │  • etcd 부트스트래핑                                          │
  │  • 기본 addon 설치                                            │
  └──────────────────────────┬───────────────────────────────────┘
                             │
                  ┌──────────┴──────────┐
                  ▼                     ▼
  ┌──────────────────────┐  ┌──────────────────────┐
  │ kubeadm join         │  │ kubeadm join         │
  │ --control-plane      │  │ (Worker 노드)         │
  │ (추가 CP 노드)        │  │                      │
  └──────────┬───────────┘  └──────────┬───────────┘
             │                         │
             └────────────┬────────────┘
                          ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                    CNI 설치 (Cilium)                          │
  │             모든 노드가 Ready 상태로 전환                       │
  └──────────────────────────────────────────────────────────────┘
```

---

## 참고: 우리 교육 클러스터

우리 교육 클러스터는 이미 위 과정이 완료된 상태입니다:

```bash
# 클러스터 상태 확인
kubectl get nodes -o wide

# 예상 출력:
# ctrl-0   Ready   control-plane   v1.35.3
# ctrl-1   Ready   control-plane   v1.35.3
# ctrl-2   Ready   control-plane   v1.35.3
# wrk-0    Ready   <none>          v1.35.3
# wrk-1    Ready   <none>          v1.35.3
# wrk-2    Ready   <none>          v1.35.3
```

---

## 핵심 정리

1. **kubeadm**은 쿠버네티스 클러스터를 부트스트래핑하는 공식 도구입니다
2. 노드 준비 → `kubeadm init` → `kubeadm join` → CNI 설치 순서로 진행합니다
3. `--control-plane-endpoint`으로 VIP를 지정하면 HA 구성이 가능합니다
4. CNI(Cilium)가 설치되어야 노드가 Ready 상태가 됩니다
5. kubeadm은 클러스터 부트스트래핑만 담당하며, OS/런타임/CNI/스토리지 등은 별도로 구성합니다

---

> **다음 챕터**: [Ch.05 Service와 쿠버네티스 네트워킹](../ch05-service-networking/README.md)
