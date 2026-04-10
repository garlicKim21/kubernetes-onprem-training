# 미니 프로젝트: 나만의 웹 서비스 배포

> 💻 **수강생 실습** — 각자의 lab 네임스페이스에서 직접 진행합니다.

## 목표

지금까지 배운 내용을 조합하여 **자기 이름이 표시되는 웹 페이지**를 쿠버네티스에 배포합니다.

---

## Step 1: ConfigMap 만들기 — 나만의 HTML

자신의 이름과 소속이 표시되는 HTML을 ConfigMap으로 만듭니다.

```bash
kubectl create configmap my-html --from-literal=index.html='<html>
<head><title>My K8s App</title></head>
<body style="font-family:sans-serif; text-align:center; padding:50px;">
<h1>Hello, I am [여기에 본인 이름]!</h1>
<p>Department: [소속]</p>
<p>This page is served from Kubernetes</p>
<p>Pod: <!--#echo var="hostname" --></p>
</body></html>'
```

> 💡 `[여기에 본인 이름]`과 `[소속]`을 자신의 정보로 바꾸세요!
>
> 예: `Hello, I am Hong Gildong!`, `Department: DevOps`

**확인:**
```bash
kubectl get configmap my-html
```

> 📖 ConfigMap 복습이 필요하면: [Ch03 — ConfigMap](../../day1/ch03-operations-basics/README.md)

---

## Step 2: nginx 설정 ConfigMap — SSI 활성화

Pod 이름이 HTML에 표시되도록 nginx SSI(Server Side Includes)를 활성화합니다.

```bash
kubectl create configmap my-nginx-conf --from-literal=default.conf='server {
  listen 80;
  root /usr/share/nginx/html;
  ssi on;
}'
```

---

## Step 3: Deployment 생성 — 3개 복제본

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-web
  template:
    metadata:
      labels:
        app: my-web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html/index.html
              subPath: index.html
            - name: conf
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: default.conf
          resources:
            requests:
              cpu: 10m
              memory: 32Mi
      volumes:
        - name: html
          configMap:
            name: my-html
        - name: conf
          configMap:
            name: my-nginx-conf
EOF
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
kubectl expose deployment my-web --port=80 --name=my-web-svc
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
