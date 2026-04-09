# BusyBox nslookup DNS Search Domain 버그

## 요약

BusyBox v1.29 이상의 `nslookup`은 `/etc/resolv.conf`의 `search` 지시문과 `ndots` 옵션을 제대로 처리하지 못합니다. Kubernetes 환경에서 짧은 서비스 이름(예: `web-0.headless-svc`)으로 DNS 조회 시 NXDOMAIN이 반환되어, 정상 동작하는 DNS를 "고장났다"고 오인할 수 있습니다.

---

## 현상

Pod 내부에서 동일한 이름을 다른 도구로 조회하면 결과가 다릅니다:

```bash
# busybox nslookup → 실패
/ # nslookup web-0.headless-svc
** server can't find web-0.headless-svc: NXDOMAIN

# ping → 성공
/ # ping web-0.headless-svc
PING web-0.headless-svc (10.244.0.60): 56 data bytes
64 bytes from 10.244.0.60: seq=0 ttl=63 time=0.050 ms

# FQDN으로 nslookup → 성공
/ # nslookup web-0.headless-svc.default.svc.cluster.local
Name:   web-0.headless-svc.default.svc.cluster.local
Address: 10.244.0.60
```

---

## 원인

### Pod의 DNS 설정

```bash
/ # cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

`ndots:5`는 "쿼리할 이름에 점(.)이 5개 미만이면 FQDN이 아니라고 판단하고, search domain을 붙여서 시도한다"는 설정입니다.

### 정상 동작 (ping, wget 등)

`ping`, `wget` 등은 시스템의 C 라이브러리(musl libc)의 `getaddrinfo()` 함수를 사용하여 DNS를 해석합니다. musl libc는 `/etc/resolv.conf`의 `search`와 `ndots`를 정상적으로 처리합니다:

```
"web-0.headless-svc" (점 1개, ndots:5 미만)
  → search domain 적용
  → "web-0.headless-svc.default.svc.cluster.local"로 쿼리
  → 성공
```

### busybox nslookup의 문제

BusyBox의 nslookup에는 두 가지 구현이 있습니다 (컴파일 시 선택):

| 구현 | DNS 해석 방식 | search/ndots 지원 |
|------|-------------|-----------------|
| **Small** (`!ENABLE_FEATURE_NSLOOKUP_BIG`) | 시스템 `getaddrinfo()` 사용 | ✅ 정상 |
| **Big** (`ENABLE_FEATURE_NSLOOKUP_BIG`, 기본값) | **자체 DNS 클라이언트** (`res_mkquery()`로 직접 UDP 패킷 생성) | ❌ 부분적 |

대부분의 busybox Docker 이미지는 **Big 버전**을 사용합니다. 이 구현은:

- v1.29에서 재작성되면서 **search domain 지원이 완전히 빠짐** (회귀 버그)
- v1.30에서 `add_query_with_search()` 함수로 search domain이 부분적으로 복구됨
- 하지만 **`ndots` 옵션은 여전히 미구현** — 점이 1개 이상이면 search domain을 적용하지 않음

| 점(.) 개수 | busybox nslookup | 정상 resolver (ndots:5) |
|:---:|---|---|
| **0개** | search domain 적용 (순서/중복 문제 있음) | search domain 적용 |
| **1~4개** | search domain **미적용** | search domain 적용 |
| **5개 이상** | FQDN으로 직접 쿼리 | FQDN으로 직접 쿼리 |

---

## 영향 받는 버전

| 버전 | 상태 |
|------|------|
| BusyBox 1.28 이하 | ✅ 정상 (구 nslookup 구현) |
| BusyBox 1.29 | ❌ search domain 완전 미지원 |
| BusyBox 1.30~1.37 | ⚠️ 부분 수정 (search 복구, ndots 미구현) |

---

## 워크어라운드

### 1. FQDN 사용 (권장)

```bash
# 짧은 이름 대신 FQDN 사용
nslookup web-0.headless-svc.default.svc.cluster.local
```

### 2. busybox 1.28 사용

```bash
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup web-0.headless-svc
```

> Kubernetes 공식 문서에서도 DNS 디버깅 시 `busybox:1.28` 사용을 권장합니다.

### 3. 다른 이미지 사용

```bash
# nicolaka/netshoot — 네트워크 디버깅 전용 이미지 (dig, nslookup, curl 등 포함)
kubectl run dns-test --image=nicolaka/netshoot --rm -it --restart=Never -- nslookup web-0.headless-svc

# Alpine — ash 쉘 + 정상 동작하는 nslookup
kubectl run dns-test --image=alpine --rm -it --restart=Never -- nslookup web-0.headless-svc
```

---

## 관련 이슈 및 참고 자료

### BusyBox 공식 버그 트래커

- [Bug #11161 — nslookup doesn't work in kubernetes](https://bugs.busybox.net/show_bug.cgi?id=11161) — 핵심 버그 리포트. `parse_resolvconf`가 search domain을 무시하던 문제. 2018-08-01 부분 수정됨.

### Docker / BusyBox 이미지

- [docker-library/busybox#48 — Nslookup does not work in latest busybox image](https://github.com/docker-library/busybox/issues/48) — 가장 상세한 논의. v1.28 → v1.29 회귀 확인. 메인테이너: "새 resolver는 DNS search domain을 전혀 지원하지 않는다."
- [docker-library/busybox#61 — Nslookup does not work in Kubernetes](https://github.com/docker-library/busybox/issues/61) — 메인테이너: "nslookup은 완전히 deprecated로 간주해야 한다."
- [docker-library/busybox#85 — Can't resolve DNS in Kubernetes](https://github.com/docker-library/busybox/issues/85) — v1.31.1에서도 문제 지속 확인.

### Kubernetes GitHub

- [kubernetes/dns#109 — Short-form DNS query not working](https://github.com/kubernetes/dns/issues/109) — busybox가 ndots와 search domain을 무시하는 문제 분석.
- [kubernetes/kubernetes#66924 — DNS can't resolve kubernetes.default](https://github.com/kubernetes/kubernetes/issues/66924) — Kubernetes DNS 해석 실패 보고.
- [kubernetes/minikube#4475 — DNS Test Fail: nslookup kubernetes.default: NXDOMAIN](https://github.com/kubernetes/minikube/issues/4475) — minikube에서 동일 현상.

### Kubernetes 공식 문서

- [Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/) — 공식 DNS 디버깅 가이드. busybox가 아닌 `registry.k8s.io/e2e-test-images/agnhost:2.39` 이미지를 사용합니다. 과거에는 `busybox:1.28`을 권장했으나, 현재는 busybox 자체를 DNS 테스트에 사용하지 않도록 변경되었습니다.

### BusyBox 소스 코드

- [networking/nslookup.c](https://git.busybox.net/busybox/plain/networking/nslookup.c) — Big/Small 두 가지 구현 확인 가능. `parse_resolvconf()` 함수에서 search domain 파싱, `ndots` 미구현 확인.

---

## 교훈

1. **DNS 디버깅 시 도구 선택에 주의하세요.** busybox nslookup의 결과만으로 "DNS가 고장났다"고 판단하지 마세요.
2. **FQDN을 사용하면 어떤 도구에서든 정상 동작합니다.** Kubernetes에서 DNS FQDN 형식: `{서비스명}.{네임스페이스}.svc.cluster.local`
3. **프로덕션 디버깅에는 전용 이미지를 사용하세요.** `nicolaka/netshoot`이 네트워크 디버깅에 가장 적합합니다.
