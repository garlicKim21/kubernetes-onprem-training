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
cilium config view | grep kube-proxy-replacement

# 예상 출력:
# kube-proxy-replacement    true
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

### 리소스별 상세 설명 (필드별 해설)

#### CiliumLoadBalancerIPPool

```yaml
apiVersion: cilium.io/v2           # Cilium CRD API 버전
kind: CiliumLoadBalancerIPPool     # LoadBalancer Service에 할당할 IP 풀을 정의하는 CRD
metadata:
  name: lb-pool                    # IP Pool 이름 (클러스터 내 유일해야 함)
spec:
  blocks:                          # IP 주소 블록 목록 (여러 개 정의 가능)
    - cidr: "172.16.200.0/24"      # 할당 가능한 IP 대역
                                   # /24 = 256개 IP (172.16.200.0 ~ 172.16.200.255)
  allowFirstLastIPs: "No"          # 네트워크 주소(.0)와 브로드캐스트 주소(.255) 사용 여부
                                   # "No" → .0과 .255를 제외하여 네트워크 충돌 방지
                                   # 실제 사용 가능: 172.16.200.1 ~ 172.16.200.254 (253개)
```

> IP Pool은 여러 개 정의할 수 있으며, `serviceSelector`를 사용하면 특정 Service에만 특정 Pool을 적용할 수도 있습니다.

#### CiliumBGPClusterConfig

```yaml
apiVersion: cilium.io/v2           # Cilium CRD API 버전
kind: CiliumBGPClusterConfig       # 클러스터 수준의 BGP 설정을 정의하는 CRD
metadata:
  name: bgp-cluster               # BGP 클러스터 설정 이름
spec:
  nodeSelector:                    # BGP를 실행할 노드를 선택하는 조건
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist     # ★ "DoesNotExist" 의미:
                                   #   이 라벨 키가 "존재하지 않는" 노드만 선택
                                   #   = Control Plane 라벨이 없는 노드 = Worker 노드만
                                   #
                                   # Control Plane 노드에는 "node-role.kubernetes.io/control-plane"
                                   # 라벨이 자동으로 붙어 있으므로, 이 조건에서 제외됩니다.
                                   # → Worker 노드(wrk-0, wrk-1, wrk-2)에서만 BGP 실행
  bgpInstances:                    # BGP 인스턴스 목록 (보통 1개)
    - name: default                # BGP 인스턴스 이름
      localASN: 65100              # 이 클러스터의 AS 번호 (Autonomous System Number)
                                   # 65100은 프라이빗 ASN 범위(64512~65534) 내의 값
      peers:                       # BGP 피어(상대방 라우터) 목록
        - name: opnsense           # 피어 이름 (식별용)
          peerASN: 65000           # 피어(OPNsense 라우터)의 AS 번호
          peerAddress: 10.254.0.1  # 피어의 IP 주소 (OPNsense 라우터)
          peerConfigRef:           # 이 피어에 적용할 세부 설정을 참조
            name: peer-config      # CiliumBGPPeerConfig 리소스의 이름
```

> `nodeSelector`의 `DoesNotExist` 연산자는 해당 키의 라벨이 아예 존재하지 않는 노드를 선택합니다. `Exists`는 반대로 라벨이 존재하는 노드를 선택합니다.

#### CiliumBGPPeerConfig

```yaml
apiVersion: cilium.io/v2           # Cilium CRD API 버전
kind: CiliumBGPPeerConfig          # BGP 피어와의 세션 세부 설정
metadata:
  name: peer-config                # CiliumBGPClusterConfig에서 peerConfigRef로 참조됨
spec:
  families:                        # BGP Address Family 설정
                                   # Address Family = BGP에서 교환하는 라우팅 정보의 종류
    - afi: ipv4                    # AFI (Address Family Identifier)
                                   # ipv4 = IPv4 주소 체계의 경로를 교환
                                   # (ipv6도 가능하지만 이 클러스터에서는 IPv4만 사용)
      safi: unicast                # SAFI (Subsequent Address Family Identifier)
                                   # unicast = 일반적인 유니캐스트 라우팅
                                   # (multicast 등도 가능하지만 일반적으로 unicast 사용)
      advertisements:              # 이 Family에서 사용할 광고(Advertisement) 설정 선택
        matchLabels:
          advertise: lb            # "advertise: lb" 라벨을 가진 CiliumBGPAdvertisement를 사용
                                   # → 아래 CiliumBGPAdvertisement의 labels와 매칭됨
  gracefulRestart:                 # BGP Graceful Restart 설정
                                   # BGP 세션이 일시적으로 끊겼을 때 경로를 즉시 삭제하지 않는 기능
    enabled: true                  # Graceful Restart 활성화
                                   # → Cilium Agent가 재시작되어도 트래픽 단절 최소화
    restartTimeSeconds: 120        # 피어가 세션 복구를 기다리는 시간 (초)
                                   # 120초 내에 세션이 재수립되면 경로를 유지
                                   # 120초가 지나면 피어가 해당 경로를 삭제
```

> **Graceful Restart**는 프로덕션 환경에서 매우 중요합니다. Cilium 업그레이드나 노드 재부팅 시 BGP 세션이 일시적으로 끊기더라도 라우터가 바로 경로를 삭제하지 않아 서비스 중단을 방지합니다.

#### CiliumBGPAdvertisement

```yaml
apiVersion: cilium.io/v2           # Cilium CRD API 버전
kind: CiliumBGPAdvertisement       # BGP로 어떤 경로를 광고할지 정의
metadata:
  name: lb-advertisement           # 광고 설정 이름
  labels:
    advertise: lb                  # ★ 이 라벨이 CiliumBGPPeerConfig의
                                   #   advertisements.matchLabels와 매칭됨
                                   #   → peer-config가 이 광고 설정을 사용
spec:
  advertisements:                  # 광고할 경로 목록
    - advertisementType: Service   # 광고 유형: "Service"
                                   # = 쿠버네티스 Service 리소스의 IP를 광고
                                   # (다른 옵션: "PodCIDR" = Pod 네트워크 대역 광고)
      service:
        addresses:                 # 광고할 Service IP의 종류
          - LoadBalancerIP         # LoadBalancer 타입 Service의 External IP만 광고
                                   # (다른 옵션: "ClusterIP", "ExternalIP")
      selector:                    # 어떤 Service를 광고 대상으로 선택할지
        matchExpressions:
          - key: somekey           # ★★ "모든 Service 매칭" 패턴 ★★
            operator: NotIn        #
            values: ["never-match"]# 이 조건의 의미:
                                   # "somekey 라벨의 값이 'never-match'가 아닌 Service"
                                   # → somekey 라벨이 없는 Service도 이 조건을 만족함!
                                   #   (라벨이 없으면 NotIn 조건에 걸리지 않으므로 통과)
                                   # → 결과적으로 모든 Service가 매칭됨
                                   #
                                   # 왜 이런 패턴을 쓰나?
                                   # selector를 비워두면 "아무것도 매칭 안 됨"이 기본값.
                                   # 모든 Service를 매칭하려면 이처럼 절대 매칭되지 않는
                                   # 값으로 NotIn을 사용하는 트릭이 필요합니다.
                                   #
                                   # 특정 Service만 광고하려면:
                                   #   matchLabels:
                                   #     expose-via-bgp: "true"
                                   # 처럼 명시적으로 라벨을 지정할 수 있습니다.
```

> **"NotIn + never-match" 패턴 요약**: `selector`에서 `matchExpressions`의 `NotIn`은 "이 값이 아닌 것"을 선택합니다. 라벨이 아예 없는 리소스도 `NotIn` 조건을 만족하므로, 실질적으로 **모든 리소스를 선택**하는 "match all" 패턴이 됩니다.

### CRD 간 참조 관계 요약

```
  CiliumLoadBalancerIPPool          CiliumBGPClusterConfig
  (IP 풀 정의)                      (BGP 피어 + 노드 선택)
  cidr: 172.16.200.0/24                   │
          │                               │ peerConfigRef
          │                               ▼
          │                     CiliumBGPPeerConfig
          │                     (AFI/SAFI + Graceful Restart)
          │                               │
          │                               │ advertisements.matchLabels
          │                               ▼
          │                     CiliumBGPAdvertisement
          │                     (어떤 Service IP를 광고할지)
          │                               │
          └──── IP 할당 ──────── Service ──┘── BGP 경로 광고
```

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
# Cluster Pods:   xx/xx managed by Cilium
# Helm chart version:    1.19.2

# kube-proxy 대체 여부 확인
cilium config view | grep kube-proxy-replacement
# 예상 출력:
# kube-proxy-replacement    true
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

## 핵심 요약

1. **Cilium**은 eBPF 기반 CNI로, kube-proxy를 대체하여 더 효율적인 네트워킹을 제공합니다
2. 온프레미스에서는 클라우드 LB가 없으므로, **Cilium LB-IPAM + BGP**로 LoadBalancer Service를 구현합니다
3. **CiliumLoadBalancerIPPool**: IP 할당 대역 정의 (172.16.200.0/24)
4. **CiliumBGPClusterConfig**: BGP 피어링 설정 (ASN 65100 ↔ ASN 65000)
5. **CiliumBGPAdvertisement**: LoadBalancer IP를 BGP로 외부에 광고
6. **Hubble**: eBPF 기반 네트워크 플로우 관측 도구 (hubble.basphere.dev)

---

> **다음 챕터**: [Ch.07 Gateway API를 이용한 HTTP 라우팅](../ch07-gateway-api/README.md)
