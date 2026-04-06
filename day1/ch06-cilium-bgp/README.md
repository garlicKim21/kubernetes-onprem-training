# Chapter 06 — Cilium CNI와 BGP 기반 LoadBalancer

## 학습 목표

- Cilium의 역할과 eBPF 기반 동작 원리를 이해한다
- 기존 kube-proxy와 Cilium의 차이를 파악한다
- 온프레미스에서 LoadBalancer 서비스를 구현하는 방법을 이해한다
- Cilium LB-IPAM과 BGP 설정을 파악한다
- Hubble UI를 통한 네트워크 관측 방법을 익힌다

---

## 1. Cilium이란?

Cilium은 **eBPF(extended Berkeley Packet Filter)** 기반의 CNI(Container Network Interface) 플러그인입니다.

### eBPF란?

eBPF는 Linux 커널 내부에서 사용자 정의 프로그램을 실행할 수 있는 기술입니다.

```
  ┌──────────────────────────────────────────┐
  │               사용자 공간                  │
  │  ┌──────────────────────────────┐        │
  │  │ Cilium Agent (cilium-agent)  │        │
  │  │ • eBPF 프로그램 로드/관리      │        │
  │  │ • 정책 관리                   │        │
  │  └──────────────┬───────────────┘        │
  ├─────────────────┼────────────────────────┤
  │               커널 공간                    │
  │  ┌──────────────▼───────────────┐        │
  │  │ eBPF 프로그램                 │        │
  │  │ • 패킷 필터링/라우팅          │        │
  │  │ • 서비스 로드밸런싱            │        │
  │  │ • 네트워크 정책 적용           │        │
  │  │ • 네트워크 관측 (Hubble)      │        │
  │  └──────────────────────────────┘        │
  │  ┌──────────────────────────────┐        │
  │  │ Linux 네트워크 스택           │        │
  │  └──────────────────────────────┘        │
  └──────────────────────────────────────────┘
```

### Cilium의 주요 기능

| 기능 | 설명 |
|------|------|
| **CNI** | Pod 네트워크 인터페이스 설정 및 IP 할당 |
| **kube-proxy 대체** | eBPF로 Service 로드밸런싱 (iptables/ipvs 불필요) |
| **Network Policy** | L3/L4/L7 수준의 네트워크 정책 |
| **LB-IPAM** | LoadBalancer Service에 IP 자동 할당 |
| **BGP** | 외부 라우터와 BGP 피어링으로 경로 광고 |
| **Hubble** | eBPF 기반 네트워크 관측성(Observability) |

---

## 2. kube-proxy vs Cilium

### 기존 kube-proxy 방식

```
  Pod → iptables/ipvs 규칙 → 대상 Pod
         (커널 네트워크 스택 전체 통과)
```

- **iptables 모드**: 규칙이 선형 검색, Service가 많으면 성능 저하
- **ipvs 모드**: 해시 테이블 기반으로 개선되었지만, 여전히 커널 네트워크 스택 전체를 통과

### Cilium (eBPF) 방식

```
  Pod → eBPF (커널 초기 단계에서 처리) → 대상 Pod
         (불필요한 커널 스택 바이패스)
```

| 비교 항목 | kube-proxy (iptables) | kube-proxy (ipvs) | Cilium (eBPF) |
|----------|----------------------|-------------------|---------------|
| Service 조회 | O(n) 선형 | O(1) 해시 | O(1) eBPF map |
| 처리 위치 | 커널 네트워크 스택 | 커널 네트워크 스택 | TC/XDP (커널 초기) |
| 연결 추적 | conntrack | conntrack | eBPF CT map |
| 확장성 | 수천 Service에서 성능 저하 | 양호 | 매우 우수 |
| 관측성 | 제한적 | 제한적 | Hubble (L3-L7) |

### 우리 클러스터 설정

```bash
# Cilium이 kube-proxy를 대체하고 있음을 확인
cilium status

# 예상 출력 (일부):
# KubeProxyReplacement:   true
# ...
```

---

## 3. 온프레미스 LoadBalancer 문제

### 클라우드 환경

```
  LoadBalancer Service 생성
       │
       ▼
  클라우드 제공자 (AWS ELB, GCP CLB 등)
  → 자동으로 외부 LB 생성
  → 외부 IP 할당
  → 트래픽 라우팅
```

### 온프레미스 환경의 문제

```
  LoadBalancer Service 생성
       │
       ▼
  ??? 클라우드 제공자가 없음 ???
  → EXTERNAL-IP가 영원히 <pending> 상태
```

**해결책:**

| 솔루션 | 설명 |
|--------|------|
| ~~MetalLB~~ | 전통적인 온프레미스 LB 솔루션 (별도 설치 필요) |
| **Cilium LB-IPAM + BGP** | Cilium에 내장된 IP 할당 + BGP 경로 광고 (우리 클러스터 방식) |

---

## 4. Cilium LB-IPAM (LoadBalancer IP Address Management)

### CiliumLoadBalancerIPPool

LoadBalancer Service에 할당할 IP 주소 풀을 정의합니다.

```yaml
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: lb-pool
spec:
  blocks:
    - cidr: "172.16.200.0/24"
  allowFirstLastIPs: "No"
```

| 필드 | 설명 |
|------|------|
| `blocks[].cidr` | 할당 가능한 IP 대역 |
| `allowFirstLastIPs` | 네트워크/브로드캐스트 주소 사용 여부 (No 권장) |

### 동작 흐름

```
  1. LoadBalancer Service 생성
       │
       ▼
  2. Cilium LB-IPAM이 IP Pool에서 IP 할당
     (172.16.200.0/24 대역에서 선택)
       │
       ▼
  3. Service의 EXTERNAL-IP에 할당된 IP 설정
       │
       ▼
  4. BGP로 외부 라우터에 경로 광고
     "172.16.200.x는 이 클러스터로 오세요"
       │
       ▼
  5. 외부 트래픽이 LB IP로 도달 → Pod에 전달
```

---

## 5. BGP (Border Gateway Protocol)

### BGP 기본 개념

BGP는 **자율 시스템(AS) 간에 네트워크 경로 정보를 교환**하는 프로토콜입니다.

| 용어 | 설명 |
|------|------|
| **ASN (Autonomous System Number)** | 네트워크 그룹의 고유 번호 |
| **피어(Peer)** | BGP 세션을 맺는 상대방 |
| **경로 광고 (Advertisement)** | "이 IP 대역으로의 트래픽은 나에게 보내라" |

### 우리 클러스터의 BGP 토폴로지

```
  ┌───────────────────────────────────────────┐
  │            OPNsense Router                 │
  │            ASN 65000                       │
  │            10.254.0.1                      │
  │                                            │
  │   "172.16.200.0/24로의 트래픽은             │
  │    Kubernetes 클러스터(ASN 65100)로"        │
  └──────┬──────────┬──────────┬──────────────┘
         │          │          │
    BGP Session  BGP Session  BGP Session
         │          │          │
  ┌──────▼────┐ ┌───▼──────┐ ┌▼───────────┐
  │  wrk-0    │ │  wrk-1   │ │  wrk-2     │
  │  ASN 65100│ │  ASN 65100│ │  ASN 65100 │
  │  Cilium   │ │  Cilium   │ │  Cilium    │
  │  BGP Agent│ │  BGP Agent│ │  BGP Agent │
  └───────────┘ └──────────┘ └────────────┘
```

- **클러스터 ASN**: 65100
- **라우터 ASN**: 65000 (OPNsense at 10.254.0.1)
- **광고 대상**: LoadBalancer Service에 할당된 IP (172.16.200.0/24 대역)
- **BGP 피어링**: Worker 노드만 참여 (Control Plane 노드는 제외)

---

## 6. BGP 설정 상세

### 우리 클러스터에 적용된 BGP 매니페스트

아래는 실제 클러스터에 적용된 BGP 설정입니다:

**examples/bgp-config.yaml:**

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: lb-pool
spec:
  blocks:
    - cidr: "172.16.200.0/24"
  allowFirstLastIPs: "No"
---
apiVersion: cilium.io/v2
kind: CiliumBGPClusterConfig
metadata:
  name: bgp-cluster
spec:
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist
  bgpInstances:
    - name: default
      localASN: 65100
      peers:
        - name: opnsense
          peerASN: 65000
          peerAddress: 10.254.0.1
          peerConfigRef:
            name: peer-config
---
apiVersion: cilium.io/v2
kind: CiliumBGPPeerConfig
metadata:
  name: peer-config
spec:
  families:
    - afi: ipv4
      safi: unicast
      advertisements:
        matchLabels:
          advertise: lb
  gracefulRestart:
    enabled: true
    restartTimeSeconds: 120
---
apiVersion: cilium.io/v2
kind: CiliumBGPAdvertisement
metadata:
  name: lb-advertisement
  labels:
    advertise: lb
spec:
  advertisements:
    - advertisementType: Service
      service:
        addresses:
          - LoadBalancerIP
      selector:
        matchExpressions:
          - key: somekey
            operator: NotIn
            values: ["never-match"]
```

### 리소스별 설명

#### CiliumLoadBalancerIPPool

```yaml
spec:
  blocks:
    - cidr: "172.16.200.0/24"    # LoadBalancer IP 할당 대역
  allowFirstLastIPs: "No"        # .0과 .255는 사용하지 않음
```

→ 172.16.200.1 ~ 172.16.200.254 범위에서 IP가 할당됩니다.

#### CiliumBGPClusterConfig

```yaml
spec:
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist   # Worker 노드에서만 BGP 실행
  bgpInstances:
    - name: default
      localASN: 65100            # 클러스터의 ASN
      peers:
        - name: opnsense
          peerASN: 65000         # OPNsense 라우터의 ASN
          peerAddress: 10.254.0.1  # OPNsense 라우터 IP
          peerConfigRef:
            name: peer-config    # 피어 설정 참조
```

→ Worker 노드만 ASN 65100으로 OPNsense(65000)와 BGP 세션을 맺습니다.

#### CiliumBGPPeerConfig

```yaml
spec:
  families:
    - afi: ipv4                  # IPv4 주소 체계
      safi: unicast              # 유니캐스트 라우팅
      advertisements:
        matchLabels:
          advertise: lb          # "advertise: lb" 라벨의 광고 설정 사용
  gracefulRestart:
    enabled: true                # Graceful Restart 활성화
    restartTimeSeconds: 120      # 재시작 대기 시간 120초
```

#### CiliumBGPAdvertisement

```yaml
metadata:
  labels:
    advertise: lb                # PeerConfig에서 참조하는 라벨
spec:
  advertisements:
    - advertisementType: Service
      service:
        addresses:
          - LoadBalancerIP       # LoadBalancer Service의 IP를 광고
      selector:
        matchExpressions:
          - key: somekey
            operator: NotIn
            values: ["never-match"]   # 사실상 모든 Service에 적용
```

→ 모든 LoadBalancer Service의 External IP를 BGP로 광고합니다.

---

## 7. 실습 데모

### Cilium 상태 확인

```bash
# Cilium 전체 상태
cilium status

# 예상 출력 (주요 항목):
#     /¯¯\
#  /¯¯\__/¯¯\    Cilium:             OK
#  \__/¯¯\__/    Operator:           OK
#  /¯¯\__/¯¯\    Envoy DaemonSet:    OK
#  \__/¯¯\__/    Hubble Relay:       OK
#  \__/
#
# KubeProxyReplacement:   true
```

### BGP 피어 상태 확인

```bash
# BGP 피어링 상태
cilium bgp peers

# 예상 출력:
# Node    Local AS   Peer AS   Peer Address   Session State   ...
# wrk-0   65100      65000     10.254.0.1     established     ...
# wrk-1   65100      65000     10.254.0.1     established     ...
# wrk-2   65100      65000     10.254.0.1     established     ...
```

> **핵심 확인 포인트**: 모든 Worker 노드의 Session State가 `established`이어야 합니다.

### IP Pool 확인

```bash
# LB IP Pool 확인
kubectl get CiliumLoadBalancerIPPool

# 예상 출력:
# NAME      DISABLED   CONFLICTING   IPS AVAILABLE   AGE
# lb-pool   false      False         253             ...
```

### LoadBalancer Service 생성 및 확인

```bash
# 1. nginx Deployment 생성
kubectl create deployment nginx-bgp-demo --image=nginx:1.27 --replicas=3

# 2. LoadBalancer Service 생성
kubectl expose deployment nginx-bgp-demo --type=LoadBalancer --port=80

# 3. External IP 할당 확인
kubectl get svc nginx-bgp-demo -w

# 예상 출력:
# NAME             TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE
# nginx-bgp-demo   LoadBalancer   10.96.xx.xx    172.16.200.x   80:3xxxx/TCP   5s

# 4. 할당된 IP로 접근 테스트
curl http://172.16.200.x

# 5. BGP 경로 광고 확인
cilium bgp routes advertised ipv4 unicast

# 예상 출력에서 172.16.200.x에 대한 경로가 광고됨을 확인
```

### IP Pool 사용량 확인

```bash
# IP Pool 사용량 변화 확인
kubectl get CiliumLoadBalancerIPPool

# IPS AVAILABLE이 줄어든 것을 확인
```

### 정리

```bash
kubectl delete deployment nginx-bgp-demo
kubectl delete svc nginx-bgp-demo
```

---

## 8. Hubble UI: 네트워크 관측

Hubble은 Cilium에 내장된 네트워크 관측성(Observability) 도구입니다.

### Hubble UI 접속

브라우저에서 [https://hubble.basphere.dev](https://hubble.basphere.dev) 접속

### Hubble에서 확인 가능한 정보

- **네트워크 플로우**: Pod 간 통신 흐름 실시간 확인
- **서비스 맵**: 서비스 간 연결 관계 시각화
- **DNS 쿼리**: Pod의 DNS 조회 이력
- **정책 적용**: Network Policy에 의한 허용/차단 여부

### Hubble CLI (참고)

```bash
# Pod 간 네트워크 플로우 관찰
hubble observe --namespace default

# 특정 Pod의 트래픽 관찰
hubble observe --to-pod default/nginx-bgp-demo-xxxxx

# DNS 관련 플로우만 필터
hubble observe --type l7 --protocol DNS
```

---

## 전체 동작 흐름 요약

```
  1. CiliumLoadBalancerIPPool 생성 → 172.16.200.0/24 대역 등록
  2. CiliumBGPClusterConfig 생성 → Worker 노드가 OPNsense와 BGP 세션 수립
  3. LoadBalancer Service 생성 → Cilium LB-IPAM이 172.16.200.x IP 할당
  4. CiliumBGPAdvertisement → BGP로 "172.16.200.x → 이 클러스터" 경로 광고
  5. OPNsense 라우터가 경로를 학습
  6. 외부 클라이언트 → OPNsense → Worker 노드 → Pod로 트래픽 도달
```

---

## 핵심 정리

1. **Cilium**은 eBPF 기반 CNI로, kube-proxy를 대체하여 더 효율적인 네트워킹을 제공합니다
2. 온프레미스에서는 클라우드 LB가 없으므로, **Cilium LB-IPAM + BGP**로 LoadBalancer Service를 구현합니다
3. **CiliumLoadBalancerIPPool**: IP 할당 대역 정의 (172.16.200.0/24)
4. **CiliumBGPClusterConfig**: BGP 피어링 설정 (ASN 65100 ↔ ASN 65000)
5. **CiliumBGPAdvertisement**: LoadBalancer IP를 BGP로 외부에 광고
6. **Hubble**: eBPF 기반 네트워크 플로우 관측 도구 (hubble.basphere.dev)

---

> **다음 챕터**: [Ch.07 Gateway API를 이용한 HTTP 라우팅](../ch07-gateway-api/README.md)
