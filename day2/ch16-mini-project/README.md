# 미니 프로젝트: 나만의 웹 서비스 배포

> 💻 **수강생 실습** — 각자의 lab 네임스페이스에서 직접 진행합니다.

## 목표

지금까지 배운 내용을 조합하여 **자기 이름이 표시되는 웹 페이지**를 쿠버네티스에 배포합니다.

---

## Step 1: ConfigMap 만들기 — 나만의 HTML

`examples/my-html-configmap.yaml` 파일을 열어 `[본인 이름]`과 `[소속]`을 자신의 정보로 수정합니다.

```yaml
# examples/my-html-configmap.yaml 에서 이 부분을 수정하세요:
<h1>Hello, I am [본인 이름]!</h1>
<p>Department: [소속]</p>
```

> 💡 예시: `Hello, I am Hong Gildong!`, `Department: DevOps`

수정 후 적용:
```bash
kubectl apply -f examples/my-html-configmap.yaml
```

**확인:**
```bash
kubectl get configmap my-html
```

> 📖 ConfigMap 복습이 필요하면: [Ch03 — ConfigMap](../../day1/ch03-operations-basics/README.md)

---

## Step 2: nginx 설정 ConfigMap — SSI 활성화

Pod 이름이 HTML에 표시되도록 nginx SSI(Server Side Includes)를 활성화합니다.

> **SSI란?** nginx가 HTML을 클라이언트에게 보내기 전에, `<!--#echo var="hostname" -->` 같은 지시문을 서버 측에서 실제 값으로 치환하는 기능입니다. 이를 통해 JavaScript 없이도 Pod의 hostname을 HTML에 표시할 수 있습니다.

```bash
kubectl apply -f examples/my-nginx-conf-configmap.yaml
```

---

## Step 3: Deployment 생성 — 3개 복제본

```bash
kubectl apply -f examples/my-web-deployment.yaml
```

**확인:**
```bash
kubectl get pods -l app=my-web
```

3개 Pod가 모두 `Running`이 될 때까지 기다립니다.

> 📖 Deployment 복습이 필요하면: [Ch02 — Deployment](../../day1/ch02-core-workloads/README.md)

---

## Step 4: Service 생성

```bash
kubectl apply -f examples/my-web-service.yaml
```

**확인:**
```bash
kubectl get svc my-web-svc
```

> 📖 Service 복습이 필요하면: [Ch05 — Service](../../day1/ch05-service-networking/README.md)

---

## Step 5: 내 웹 페이지 확인

```bash
# Pod 안에서 Service로 접근 테스트
kubectl run curl-test --image=busybox:1.37 --rm -it --restart=Never -- wget -qO- my-web-svc
```

자신의 이름과 Pod 이름이 표시되면 성공입니다!

여러 번 실행하면 **다른 Pod 이름**이 표시됩니다 (Service 로드밸런싱).

---

## Step 6: (도전) HPA 적용

```bash
kubectl autoscale deployment my-web --cpu=50% --min=3 --max=10
```

**확인:**
```bash
kubectl get hpa
```

> 📖 HPA 복습이 필요하면: [Ch11 — HPA](../../day2/ch11-hpa-autoscaling/README.md)

---

## 정리

```bash
kubectl delete deployment my-web
kubectl delete svc my-web-svc
kubectl delete configmap my-html my-nginx-conf
kubectl delete hpa my-web
```

---

## 2일간의 교육을 마치며

이번 교육에서 다룬 내용을 정리합니다:

### Day 1
- 컨테이너와 쿠버네티스 기본 개념
- Pod, ReplicaSet, Deployment, StatefulSet, DaemonSet
- ConfigMap, Secret, 리소스 관리, Probe
- kubeadm 클러스터 구축
- Service와 네트워킹 (ClusterIP, NodePort, LoadBalancer, Headless)
- Cilium CNI와 BGP LoadBalancer
- Gateway API HTTP 라우팅

### Day 2
- 스토리지: emptyDir, hostPath, PV, PVC
- StorageClass와 동적 프로비저닝 (vSphere CSI)
- StatefulSet과 데이터베이스 운영 (MySQL, Headless Service, 데이터 영속성)
- HPA 오토스케일링
- Prometheus & Grafana 모니터링
- 종합 데모
- 실무 적용 가이드 (Helm, RBAC, GitOps)
- 트러블슈팅 실습
- 미니 프로젝트

### 다음 단계 제안

1. **kubectl 연습**: 다양한 명령어에 익숙해지세요
2. **개인 환경 구축**: minikube, kind, k3s 등으로 로컬 클러스터를 만들어 실습하세요
3. **Helm 활용**: 오픈소스 Chart를 설치하고 커스터마이징해 보세요
4. **GitOps 도입**: ArgoCD를 설치하고 Git 기반 배포를 경험해 보세요
5. **자격증 도전**: CKAD 또는 CKA 시험에 도전해 보세요

---

> 교육에 참여해 주셔서 감사합니다!
