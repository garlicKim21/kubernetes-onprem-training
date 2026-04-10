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
