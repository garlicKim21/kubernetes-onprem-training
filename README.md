# Kubernetes On-Premise 실무 교육

## 교육 개요

| 항목 | 내용 |
|------|------|
| **교육명** | Kubernetes On-Premise 클러스터 구축 및 운영 실무 |
| **교육 기관** | Basphere (대구) |
| **교육 기간** | 2일 (강사 주도형 실습 교육) |
| **대상** | 쿠버네티스 입문 ~ 초급 수준의 개발자 및 인프라 엔지니어 |
| **교육 방식** | 강사 데모 + 수강생 실습 혼합. 수강생은 개인 네임스페이스(lab-01~lab-30)에서 직접 실습하며, 강사 데모는 Headlamp/Grafana에서 확인 |
| **Kubernetes 버전** | v1.35.3 |
| **CNI** | Cilium v1.19.2 (kube-proxy 대체) |
| **Container Runtime** | containerd 2.2.2 |

---

## 2일 커리큘럼

### Day 1 — 쿠버네티스 기초와 네트워킹

| 시간 | 챕터 | 주제 |
|------|-------|------|
| 09:00–09:40 | Ch.01 | 컨테이너 및 쿠버네티스 개요 |
| 09:40–10:40 | Ch.02 | 핵심 워크로드: Pod, ReplicaSet, Deployment |
| 10:40–10:50 | — | 휴식 |
| 10:50–11:50 | Ch.03 | 운영 기초: ConfigMap, Secret, 리소스 관리, Probe |
| 11:50–12:30 | Ch.04 | kubeadm 클러스터 구축 데모 |
| 12:30–13:30 | — | 점심 |
| 13:30–14:30 | Ch.05 | Service와 쿠버네티스 네트워킹 |
| 14:30–15:30 | Ch.06 | Cilium CNI와 BGP 기반 LoadBalancer |
| 15:30–15:40 | — | 휴식 |
| 15:40–16:40 | Ch.07 | Gateway API를 이용한 HTTP 라우팅 |
| 16:40–17:00 | — | Day 1 정리 및 Q&A |

### Day 2 — 스토리지, 오토스케일링, 모니터링, 실무

| 시간 | 챕터 | 주제 |
|------|-------|------|
| 09:00–09:50 | Ch.08 | 스토리지 기초: Volume, PV, PVC |
| 09:50–10:40 | Ch.09 | StorageClass와 동적 프로비저닝 |
| 10:40–10:50 | — | 휴식 |
| 10:50–11:50 | Ch.10 | 데이터베이스 on Kubernetes: StatefulSet과 MySQL |
| 11:50–12:30 | Ch.11 | HPA: Horizontal Pod Autoscaler |
| 12:30–13:30 | — | 점심 |
| 13:30–14:30 | Ch.12 | 모니터링과 관측성: Prometheus & Grafana |
| 14:30–15:30 | Ch.13 | 종합 데모: 배포부터 오토스케일링까지 |
| 15:30–15:40 | — | 휴식 |
| 15:40–16:40 | Ch.14 | 실무 적용 가이드 |
| 16:40–17:00 | — | 전체 정리 및 Q&A |

---

## 클러스터 아키텍처

```
                        ┌──────────────────────────────────────────────────────────┐
                        │                      OPNsense Router                     │
                        │                  ASN 65000 · 10.254.0.1                  │
                        │                                                          │
                        │   BGP Peering ←→ Cluster ASN 65100                       │
                        │   LB IP Pool: 172.16.200.0/24                            │
                        └────────────┬────────────────────────────┬────────────────┘
                                     │                            │
                    ─────────────────┴────────────────────────────┴─────────────────
                    │          Control Plane VIP: 10.254.0.10 (API Server)         │
                    ────────────────────────────────────────────────────────────────
                         │                    │                    │
                  ┌──────┴──────┐      ┌──────┴──────┐      ┌──────┴──────┐
                  │   ctrl-0    │      │   ctrl-1    │      │   ctrl-2    │
                  │ Control     │      │ Control     │      │ Control     │
                  │ Plane       │      │ Plane       │      │ Plane       │
                  │             │      │             │      │             │
                  │ • API Server│      │ • API Server│      │ • API Server│
                  │ • etcd      │      │ • etcd      │      │ • etcd      │
                  │ • Scheduler │      │ • Scheduler │      │ • Scheduler │
                  │ • Ctrl Mgr  │      │ • Ctrl Mgr  │      │ • Ctrl Mgr  │
                  └─────────────┘      └─────────────┘      └─────────────┘

                  ┌──────┴──────┐      ┌──────┴──────┐      ┌──────┴──────┐
                  │   wrk-0     │      │   wrk-1     │      │   wrk-2     │
                  │ Worker      │      │ Worker      │      │ Worker      │
                  │             │      │             │      │             │
                  │ • kubelet   │      │ • kubelet   │      │ • kubelet   │
                  │ • Cilium    │      │ • Cilium    │      │ • Cilium    │
                  │ • containerd│      │ • containerd│      │ • containerd│
                  └─────────────┘      └─────────────┘      └─────────────┘

                  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
                  │   wrk-3     │      │   wrk-4     │      │   wrk-5     │
                  │ Worker      │      │ Worker      │      │ Worker      │
                  │             │      │             │      │             │
                  │ • kubelet   │      │ • kubelet   │      │ • kubelet   │
                  │ • Cilium    │      │ • Cilium    │      │ • Cilium    │
                  │ • containerd│      │ • containerd│      │ • containerd│
                  └─────────────┘      └─────────────┘      └─────────────┘
```

---

## 수강생 접속 정보

### 웹 대시보드

| 서비스 | URL | 비고 |
|--------|-----|------|
| **Headlamp** (클러스터 대시보드) | [https://headlamp.basphere.dev](https://headlamp.basphere.dev) | 읽기 전용 |
| **Grafana** (모니터링) | [https://grafana.basphere.dev](https://grafana.basphere.dev) | 아래 로그인 정보 참조 |
| **Hubble UI** (네트워크 관측) | [https://hubble.basphere.dev](https://hubble.basphere.dev) | Cilium 네트워크 플로우 |
| **부하 테스트 엔드포인트** | [https://loadtest.basphere.dev](https://loadtest.basphere.dev) | Day 2 사용 |

### Grafana 로그인 정보

```
사용자명: student
비밀번호: k8s-training
```

### kubectl 접속

API 서버 엔드포인트: `api.basphere.dev:6443`

강사가 배포하는 kubeconfig 파일을 다운로드한 후 아래와 같이 설정합니다:

```bash
# kubeconfig 파일 저장
mkdir -p ~/.kube
cp student-kubeconfig.yaml ~/.kube/config

# 접속 테스트
kubectl cluster-info
kubectl get nodes
```

> **참고:** 수강생 kubeconfig는 본인 네임스페이스(lab-XX)에서 Pod, Deployment, Service, ConfigMap, Secret, PVC, HPA, HTTPRoute를 생성/삭제할 수 있으며, 그 외 네임스페이스는 읽기 전용(view)입니다.

---

## 사전 준비 사항 (Prerequisites)

### 수강생 PC 요구사항

- 최신 웹 브라우저 (Chrome, Firefox, Edge 등)
- 터미널 환경 (Windows: PowerShell 또는 WSL, macOS/Linux: 기본 터미널)
- `kubectl` 설치 권장 (선택사항 — Headlamp으로 대체 가능)

### kubectl 설치 (선택사항)

```bash
# Linux
curl -LO "https://dl.k8s.io/release/v1.35.3/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# macOS
brew install kubectl

# Windows
choco install kubernetes-cli
```

### 사전 지식

- Linux 기본 명령어 (ls, cd, cat, vi 등)
- YAML 문법 기초
- 네트워크 기본 개념 (IP, Port, DNS)
- Docker 또는 컨테이너 기본 개념 (있으면 좋음)

---

## 디렉토리 구조

```
kubernetes-onprem-training/
├── README.md                          ← 이 파일
├── day1/
│   ├── ch01-container-overview/       컨테이너 및 쿠버네티스 개요
│   ├── ch02-core-workloads/           핵심 워크로드 리소스
│   ├── ch03-operations-basics/        운영 기초
│   ├── ch04-kubeadm/                  kubeadm 클러스터 구축
│   ├── ch05-service-networking/       Service와 네트워킹
│   ├── ch06-cilium-bgp/              Cilium과 BGP LoadBalancer
│   └── ch07-gateway-api/             Gateway API
├── day2/
│   ├── ch08-storage-basics/           스토리지 기초: Volume, PV, PVC
│   ├── ch09-storageclass/             StorageClass와 동적 프로비저닝
│   ├── ch10-db-on-k8s/                데이터베이스 on Kubernetes
│   ├── ch11-hpa-autoscaling/          HPA 오토스케일링
│   ├── ch12-monitoring/               모니터링과 관측성: Prometheus & Grafana
│   ├── ch13-comprehensive-demo/       종합 데모: 배포부터 오토스케일링까지
│   └── ch14-real-world/               실무 적용 가이드
└── appendix/                          부록 자료
```
