# 트러블슈팅 실습

> 💻 **수강생 실습** — 각자의 lab 네임스페이스에서 진행합니다.

## 학습 목표

- Pod가 정상 동작하지 않을 때 원인을 진단하는 방법을 익힌다
- `kubectl describe`, `kubectl logs` 명령어로 문제를 파악한다

---

## 실습: 고장난 Pod 디버깅

아래 3개의 YAML 파일은 **의도적으로 문제가 있는** Pod입니다.
각 Pod를 배포하고, 어떤 문제인지 진단한 후 원인을 찾아보세요.

### 문제 1

```bash
kubectl apply -f examples/broken-crash.yaml
kubectl get pod broken-probe
```

30초 정도 기다린 후 상태를 확인하세요:
```bash
kubectl get pod broken-probe
kubectl describe pod broken-probe
```

<details>
<summary>💡 힌트 (클릭하여 펼치기)</summary>

`kubectl describe`의 Events에서 Probe 관련 메시지를 확인하세요. nginx에 `/healthz` 경로가 있을까요?

</details>

<details>
<summary>✅ 정답</summary>

**CrashLoopBackOff (Liveness Probe 실패)** — nginx에는 기본적으로 `/healthz` 경로가 없습니다. Readiness Probe가 계속 404를 받아 `READY 0/1` 상태가 되고, Liveness Probe도 실패하여 컨테이너가 반복 재시작됩니다.

**해결**: Probe 경로를 nginx가 응답하는 `/`로 변경하면 됩니다.

</details>

---

### 문제 2

```bash
kubectl apply -f examples/broken-image.yaml
kubectl get pod broken-image
```

상태를 확인하고, 원인을 찾아보세요:
```bash
kubectl describe pod broken-image
```

<details>
<summary>💡 힌트</summary>

Events에서 `Failed to pull image` 메시지를 확인하세요. 이미지 태그가 맞나요?

</details>

<details>
<summary>✅ 정답</summary>

**ImagePullBackOff** — `nginx:99.99.99` 이미지가 존재하지 않습니다. Docker Hub에 해당 태그가 없어서 이미지를 다운로드할 수 없습니다.

**해결**: 존재하는 태그로 변경합니다 (예: `nginx:1.27`).

</details>

---

### 문제 3

```bash
kubectl apply -f examples/broken-pending.yaml
kubectl get pod broken-pending
```

상태를 확인하고, 원인을 찾아보세요:
```bash
kubectl describe pod broken-pending
```

<details>
<summary>💡 힌트</summary>

Events에서 `FailedScheduling` 메시지를 확인하세요. `resources.requests`가 얼마나 되나요?

</details>

<details>
<summary>✅ 정답</summary>

**Pending** — `cpu: "100"` (100코어), `memory: "1000Gi"` (1TB)를 요청했습니다. 클러스터의 어떤 노드도 이만큼의 리소스를 제공할 수 없어서 스케줄링이 불가능합니다.

**해결**: 현실적인 값으로 변경합니다 (예: `cpu: 100m`, `memory: 128Mi`).

</details>

---

### 정리

```bash
kubectl delete pod broken-probe broken-image broken-pending
```

## 디버깅 요약

| 증상 | 확인 명령어 | 흔한 원인 |
|------|-----------|----------|
| **CrashLoopBackOff** | `kubectl logs` | 잘못된 command, 앱 에러, 설정 누락 |
| **ImagePullBackOff** | `kubectl describe` → Events | 이미지 이름/태그 오타, 레지스트리 인증 실패 |
| **Pending** | `kubectl describe` → Events | 리소스 부족, PVC 미바인딩, nodeSelector 불일치 |
| **Running (0/1)** | `kubectl describe` → Readiness Probe | Probe 실패 (앱이 아직 준비 안 됨) |

> 📖 추가 트러블슈팅 가이드: [부록 — 트러블슈팅](../../appendix/troubleshooting.md)

---

> **다음 챕터**: [Ch.16 미니 프로젝트: 나만의 웹 서비스 배포](../ch16-mini-project/README.md)
