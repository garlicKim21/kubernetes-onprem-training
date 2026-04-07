# Kubernetes Hands-on 실습 워크시트

> **대상**: 수강생 개인 실습 (각자 할당된 네임스페이스: `lab-01` ~ `lab-30`)
>
> **권한**: Pod, Deployment, Service, ConfigMap, Secret, PVC, HPA 생성/수정/삭제 가능
>
> **주의**: 모든 명령어에서 `lab-XX`를 본인의 네임스페이스로 변경하세요.

---

## 사전 준비

```bash
# 본인의 네임스페이스 확인 (강사가 안내한 번호 사용)
export MY_NS=lab-XX   # 예: lab-01, lab-02, ...

# 네임스페이스 존재 확인
kubectl get namespace $MY_NS

# 이후 모든 명령어에서 기본 네임스페이스로 설정
kubectl config set-context --current --namespace=$MY_NS

# 확인
kubectl config view --minify | grep namespace
```

---

## 실습 1: kubectl 기본 명령어 연습

### 1.1 클러스터 노드 확인

```bash
kubectl get nodes
```

**예상 출력:**
```
NAME     STATUS   ROLES           VERSION
ctrl-0   Ready    control-plane   v1.35.3
ctrl-1   Ready    control-plane   v1.35.3
ctrl-2   Ready    control-plane   v1.35.3
wrk-0    Ready    <none>          v1.35.3
wrk-1    Ready    <none>          v1.35.3
wrk-2    Ready    <none>          v1.35.3
```

> Control Plane 노드 3개, Worker 노드 3개로 구성된 HA 클러스터입니다.

### 1.2 전체 Pod 확인

```bash
kubectl get pods -A
```

> `-A` 옵션은 `--all-namespaces`의 약어입니다. 클러스터 전체의 Pod를 확인합니다.

### 1.3 네임스페이스 확인

```bash
kubectl get namespaces
```

> 본인의 네임스페이스(`lab-XX`)가 목록에 있는지 확인하세요.

### 1.4 유용한 추가 명령어

```bash
# 노드 상세 정보 확인
kubectl get nodes -o wide

# 클러스터 정보
kubectl cluster-info

# API 리소스 종류 확인
kubectl api-resources | head -20
```

---

## 실습 2: Pod 생성과 관리

### 2.1 Pod 생성 (명령형)

```bash
kubectl run my-nginx --image=nginx:1.27
```

**예상 출력:**
```
pod/my-nginx created
```

### 2.2 Pod 상태 확인

```bash
# Pod 목록 조회
kubectl get pods

# 예상 출력:
# NAME       READY   STATUS    RESTARTS   AGE
# my-nginx   1/1     Running   0          15s
```

### 2.3 Pod 상세 정보 확인

```bash
kubectl describe pod my-nginx
```

> 다음 항목을 확인하세요:
> - **Node**: Pod가 실행 중인 워커 노드
> - **IP**: Pod에 할당된 IP 주소
> - **Events**: 스케줄링, 이미지 Pull, 컨테이너 시작 이벤트

### 2.4 Pod 로그 확인

```bash
kubectl logs my-nginx
```

### 2.5 Pod 내부 접속

```bash
# 컨테이너 내부에서 명령 실행
kubectl exec my-nginx -- cat /etc/nginx/nginx.conf

# 대화형 셸 접속
kubectl exec -it my-nginx -- /bin/bash

# (셸 내부에서) nginx 기본 페이지 확인
curl localhost
exit
```

### 2.6 Pod 삭제

```bash
kubectl delete pod my-nginx
```

**예상 출력:**
```
pod "my-nginx" deleted
```

```bash
# 삭제 확인
kubectl get pods
# 결과: No resources found
```

---

## 실습 3: Deployment 생성과 스케일링

### 3.1 Deployment YAML 생성 및 적용

```bash
kubectl apply -f examples/deployment.yaml
```

> YAML 파일 내용은 `examples/deployment.yaml`을 참고하세요.

```bash
# Deployment 상태 확인
kubectl get deployment

# 예상 출력:
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-deployment   3/3     3            3           30s

# Pod 확인
kubectl get pods -l app=nginx

# ReplicaSet 확인
kubectl get rs
```

### 3.2 스케일링

```bash
# 복제본을 5개로 확장
kubectl scale deployment nginx-deployment --replicas=5

# 확인
kubectl get pods -l app=nginx

# 예상 출력: 5개의 Pod가 Running 상태
```

```bash
# 다시 2개로 축소
kubectl scale deployment nginx-deployment --replicas=2

# 확인
kubectl get pods -l app=nginx

# 예상 출력: 2개의 Pod만 남음
```

### 3.3 Rolling Update

```bash
# 이미지를 nginx:1.26에서 nginx:1.27로 업데이트
kubectl set image deployment/nginx-deployment nginx=nginx:1.27

# 롤아웃 상태 관찰
kubectl rollout status deployment/nginx-deployment

# 예상 출력:
# deployment "nginx-deployment" successfully rolled out
```

### 3.4 롤아웃 이력 확인

```bash
kubectl rollout history deployment/nginx-deployment

# 예상 출력:
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
```

### 3.5 롤백

```bash
# 이전 버전으로 롤백
kubectl rollout undo deployment/nginx-deployment

# 이미지 확인 (nginx:1.26으로 돌아감)
kubectl get deployment nginx-deployment -o jsonpath='{.spec.template.spec.containers[0].image}'
echo

# 예상 출력: nginx:1.26
```

### 3.6 정리

```bash
kubectl delete deployment nginx-deployment
```

---

## 실습 4: Service 생성

### 4.1 Deployment 생성

```bash
kubectl apply -f examples/deployment.yaml
```

### 4.2 ClusterIP Service 생성

```bash
kubectl apply -f examples/service.yaml
```

```bash
# Service 확인
kubectl get svc

# 예상 출력:
# NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# nginx-service   ClusterIP   10.96.xx.xx    <none>        80/TCP    5s
```

### 4.3 Service 접근 테스트 (busybox에서 curl)

```bash
# busybox Pod를 생성하여 Service에 접근
kubectl run busybox --image=busybox:1.37 --rm -it --restart=Never -- wget -qO- nginx-service
```

**예상 출력:**
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

> ClusterIP Service는 클러스터 내부에서만 접근 가능합니다. busybox Pod에서 Service 이름(`nginx-service`)으로 DNS 기반 접근이 가능합니다.

### 4.4 Service DNS 확인

```bash
# DNS로 Service 조회
kubectl run dns-test --image=busybox:1.37 --rm -it --restart=Never -- nslookup nginx-service
```

**예상 출력:**
```
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx-service
Address 1: 10.96.xx.xx nginx-service.lab-XX.svc.cluster.local
```

### 4.5 정리

```bash
kubectl delete -f examples/service.yaml
kubectl delete -f examples/deployment.yaml
```

---

## 실습 5: ConfigMap & Secret 활용

### 5.1 ConfigMap 생성

```bash
kubectl apply -f examples/configmap.yaml
```

```bash
# ConfigMap 확인
kubectl get configmap app-config
kubectl describe configmap app-config
```

### 5.2 ConfigMap을 사용하는 Pod 생성

```bash
kubectl apply -f examples/configmap-pod.yaml
```

```bash
# Pod 상태 확인
kubectl get pod configmap-demo

# 환경 변수로 주입된 값 확인
kubectl exec configmap-demo -- env | grep APP_

# 예상 출력:
# APP_ENV=production
# APP_PORT=8080

# 볼륨 마운트로 주입된 설정 파일 확인
kubectl exec configmap-demo -- cat /etc/config/app.properties

# 예상 출력:
# database.host=db.example.com
# database.port=5432
# cache.ttl=300
```

### 5.3 Secret 생성

```bash
kubectl apply -f examples/secret.yaml
```

```bash
# Secret 확인 (값은 숨겨져 있음)
kubectl get secret db-secret
kubectl describe secret db-secret

# Secret 값 디코딩
kubectl get secret db-secret -o jsonpath='{.data.username}' | base64 -d
echo
# 예상 출력: admin
```

### 5.4 Secret을 사용하는 Pod 생성

```bash
kubectl apply -f examples/secret-pod.yaml
```

```bash
# 환경 변수로 주입된 Secret 확인
kubectl exec secret-demo -- env | grep DB_

# 예상 출력:
# DB_USER=admin
# DB_PASS=S3cur3P@ssw0rd!

# 볼륨 마운트된 Secret 파일 확인
kubectl exec secret-demo -- ls /etc/secrets
# 예상 출력:
# username
# password

kubectl exec secret-demo -- cat /etc/secrets/username
# 예상 출력: admin
```

### 5.5 정리

```bash
kubectl delete pod configmap-demo secret-demo
kubectl delete configmap app-config
kubectl delete secret db-secret
```

---

## 실습 6: PVC로 영속 스토리지 사용

### 6.1 PVC 생성

```bash
kubectl apply -f examples/pvc.yaml
```

```bash
# PVC 상태 확인 — STATUS가 Bound가 될 때까지 대기
kubectl get pvc my-pvc -w

# 예상 출력:
# NAME     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# my-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   1Gi        RWO            vsphere-csi    10s
```

> Ctrl+C로 watch를 중단합니다.

### 6.2 PVC를 마운트한 Pod 생성 및 데이터 쓰기

```bash
kubectl apply -f examples/pvc-pod.yaml
```

```bash
# Pod 상태 확인
kubectl get pod pvc-writer -w

# Running이 되면 Ctrl+C

# Pod 내부에서 데이터 쓰기
kubectl exec pvc-writer -- sh -c 'echo "Hello from PVC! Written at $(date)" > /data/message.txt'

# 데이터 확인
kubectl exec pvc-writer -- cat /data/message.txt

# 예상 출력:
# Hello from PVC! Written at Mon Apr  7 09:30:00 UTC 2026
```

### 6.3 Pod 삭제 후 데이터 영속성 확인

```bash
# Pod 삭제
kubectl delete pod pvc-writer

# Pod가 삭제되었는지 확인
kubectl get pods

# 같은 PVC를 마운트하는 새 Pod 생성
kubectl apply -f examples/pvc-pod.yaml

# Pod가 Ready 될 때까지 대기
kubectl get pod pvc-writer -w

# 데이터가 그대로 남아 있는지 확인
kubectl exec pvc-writer -- cat /data/message.txt

# 예상 출력:
# Hello from PVC! Written at Mon Apr  7 09:30:00 UTC 2026
```

> Pod를 삭제하고 다시 생성해도 PVC에 저장된 데이터는 그대로 유지됩니다!

### 6.4 정리

```bash
kubectl delete pod pvc-writer
kubectl delete pvc my-pvc
```

---

## 실습 7: HPA (Horizontal Pod Autoscaler) 체험

### 7.1 리소스 요청이 설정된 Deployment 생성

```bash
kubectl apply -f examples/hpa-deployment.yaml
```

```bash
# Deployment 확인
kubectl get deployment hpa-demo

# 예상 출력:
# NAME       READY   UP-TO-DATE   AVAILABLE   AGE
# hpa-demo   1/1     1            1           10s
```

### 7.2 HPA 생성

```bash
kubectl apply -f examples/hpa.yaml
```

```bash
# HPA 확인
kubectl get hpa

# 예상 출력 (처음에는 TARGETS가 <unknown>/50%일 수 있음, 잠시 대기):
# NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# hpa-demo   Deployment/hpa-demo   0%/50%    1         10        1          30s
```

### 7.3 부하 생성

새 터미널 또는 별도 탭에서 다음 명령어를 실행합니다:

```bash
# CPU 부하를 주는 Pod 생성
kubectl run load-generator --image=busybox:1.37 --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://hpa-demo-svc; done"
```

### 7.4 HPA 스케일링 관찰

원래 터미널에서 HPA를 지속적으로 관찰합니다:

```bash
# HPA 상태를 실시간으로 관찰 (1~2분 소요)
kubectl get hpa -w

# 예상 출력 (시간 경과에 따라):
# NAME       REFERENCE             TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
# hpa-demo   Deployment/hpa-demo   0%/50%     1         10        1          1m
# hpa-demo   Deployment/hpa-demo   68%/50%    1         10        1          2m
# hpa-demo   Deployment/hpa-demo   68%/50%    1         10        2          2m30s
# hpa-demo   Deployment/hpa-demo   45%/50%    1         10        3          3m
```

> CPU 사용률이 50%를 초과하면 HPA가 자동으로 Pod 수를 늘립니다!

```bash
# Pod 수가 증가하는 것을 확인
kubectl get pods -l app=hpa-demo
```

### 7.5 부하 제거 후 스케일 다운 관찰

```bash
# 부하 생성기 중지
kubectl delete pod load-generator

# HPA 관찰 (5~10분 후 스케일 다운)
kubectl get hpa -w

# 시간이 지나면 REPLICAS가 다시 1로 줄어듭니다
```

### 7.6 정리

```bash
kubectl delete hpa hpa-demo
kubectl delete deployment hpa-demo
kubectl delete svc hpa-demo-svc
kubectl delete pod load-generator 2>/dev/null
```

---

## 실습 완료 후 정리

모든 실습이 끝나면 본인 네임스페이스의 리소스를 정리합니다:

```bash
# 남아 있는 리소스 확인
kubectl get all
kubectl get pvc
kubectl get configmap
kubectl get secret

# 남은 리소스가 있다면 삭제
kubectl delete all --all
kubectl delete pvc --all
kubectl delete configmap --all
kubectl delete secret --all
```

> **주의**: `--all` 옵션은 해당 네임스페이스의 모든 리소스를 삭제합니다. 본인 네임스페이스에서만 실행하세요.

---

## 핵심 정리

| 실습 | 배운 내용 |
|------|-----------|
| **kubectl 기본** | 노드, Pod, 네임스페이스 조회 방법 |
| **Pod** | Pod의 생성, 조회, 로그, exec, 삭제 |
| **Deployment** | 선언적 배포, 스케일링, Rolling Update, Rollback |
| **Service** | ClusterIP로 Pod 접근, DNS 기반 서비스 디스커버리 |
| **ConfigMap/Secret** | 설정과 민감정보의 외부화 (환경 변수 + 볼륨) |
| **PVC** | 영속 스토리지로 데이터 보존 (Pod 삭제 후에도 유지) |
| **HPA** | CPU 기반 자동 스케일링으로 부하 대응 |
