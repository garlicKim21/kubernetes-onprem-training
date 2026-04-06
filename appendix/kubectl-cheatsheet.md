# kubectl 명령어 치트시트

## 클러스터 정보 확인

```bash
# 클러스터 정보
kubectl cluster-info

# 클러스터 버전
kubectl version

# API 리소스 목록 (약어 확인)
kubectl api-resources

# API 버전 목록
kubectl api-versions
```

---

## 노드 관련

```bash
# 노드 목록
kubectl get nodes

# 노드 상세 정보
kubectl describe node <노드이름>

# 노드 리소스 사용량
kubectl top nodes

# 노드에 레이블 부여
kubectl label node <노드이름> <키>=<값>
```

---

## 네임스페이스

```bash
# 네임스페이스 목록
kubectl get namespaces

# 네임스페이스 생성
kubectl create namespace <이름>

# 네임스페이스 삭제 (내부 리소스 모두 삭제)
kubectl delete namespace <이름>

# 기본 네임스페이스 변경
kubectl config set-context --current --namespace=<이름>
```

---

## Pod

```bash
# Pod 목록 (현재 네임스페이스)
kubectl get pods

# Pod 목록 (모든 네임스페이스)
kubectl get pods -A

# Pod 목록 (특정 네임스페이스)
kubectl get pods -n <네임스페이스>

# Pod 상세 정보
kubectl describe pod <Pod이름> -n <네임스페이스>

# Pod 로그 확인
kubectl logs <Pod이름> -n <네임스페이스>

# Pod 로그 실시간 확인 (follow)
kubectl logs -f <Pod이름> -n <네임스페이스>

# 멀티 컨테이너 Pod의 특정 컨테이너 로그
kubectl logs <Pod이름> -c <컨테이너이름> -n <네임스페이스>

# Pod 리소스 사용량
kubectl top pods -n <네임스페이스>

# Pod 안에서 명령어 실행
kubectl exec -it <Pod이름> -n <네임스페이스> -- <명령어>

# Pod 안에서 셸 접속
kubectl exec -it <Pod이름> -n <네임스페이스> -- /bin/sh
kubectl exec -it <Pod이름> -n <네임스페이스> -- /bin/bash

# Pod 삭제
kubectl delete pod <Pod이름> -n <네임스페이스>
```

---

## Deployment

```bash
# Deployment 목록
kubectl get deployments -n <네임스페이스>

# Deployment 상세 정보
kubectl describe deployment <이름> -n <네임스페이스>

# Deployment 생성 (명령형)
kubectl create deployment <이름> --image=<이미지> --replicas=3 -n <네임스페이스>

# Deployment 스케일링
kubectl scale deployment <이름> --replicas=5 -n <네임스페이스>

# Deployment 이미지 업데이트 (롤링 업데이트)
kubectl set image deployment/<이름> <컨테이너이름>=<새이미지> -n <네임스페이스>

# 롤아웃 상태 확인
kubectl rollout status deployment/<이름> -n <네임스페이스>

# 롤아웃 히스토리
kubectl rollout history deployment/<이름> -n <네임스페이스>

# 롤백
kubectl rollout undo deployment/<이름> -n <네임스페이스>

# 특정 리비전으로 롤백
kubectl rollout undo deployment/<이름> --to-revision=2 -n <네임스페이스>
```

---

## Service

```bash
# Service 목록
kubectl get services -n <네임스페이스>
kubectl get svc -n <네임스페이스>

# Service 상세 정보
kubectl describe svc <이름> -n <네임스페이스>

# Service의 Endpoints 확인
kubectl get endpoints <이름> -n <네임스페이스>

# Service 생성 (Deployment 노출)
kubectl expose deployment <이름> --port=80 --target-port=8080 --type=ClusterIP -n <네임스페이스>
```

---

## ConfigMap / Secret

```bash
# ConfigMap 목록
kubectl get configmap -n <네임스페이스>

# ConfigMap 생성 (리터럴 값)
kubectl create configmap <이름> --from-literal=key1=value1 --from-literal=key2=value2 -n <네임스페이스>

# ConfigMap 생성 (파일에서)
kubectl create configmap <이름> --from-file=<파일경로> -n <네임스페이스>

# Secret 목록
kubectl get secret -n <네임스페이스>

# Secret 생성
kubectl create secret generic <이름> --from-literal=password=mypassword -n <네임스페이스>

# Secret 값 확인 (base64 디코딩)
kubectl get secret <이름> -n <네임스페이스> -o jsonpath='{.data.<키>}' | base64 -d
```

---

## 스토리지

```bash
# StorageClass 목록
kubectl get storageclass
kubectl get sc

# PersistentVolume 목록
kubectl get pv

# PersistentVolumeClaim 목록
kubectl get pvc -n <네임스페이스>

# PV 상세 정보
kubectl describe pv <이름>

# PVC 상세 정보
kubectl describe pvc <이름> -n <네임스페이스>
```

---

## HPA (Horizontal Pod Autoscaler)

```bash
# HPA 목록
kubectl get hpa -n <네임스페이스>

# HPA 상세 정보
kubectl describe hpa <이름> -n <네임스페이스>

# HPA 실시간 모니터링
kubectl get hpa -n <네임스페이스> -w
```

---

## 이벤트 및 디버깅

```bash
# 네임스페이스의 이벤트 확인 (최신순)
kubectl get events -n <네임스페이스> --sort-by='.lastTimestamp'

# 특정 리소스의 이벤트 확인
kubectl describe <리소스유형> <이름> -n <네임스페이스>
# → Events 섹션에서 확인 가능

# Pod가 왜 실패했는지 확인
kubectl describe pod <Pod이름> -n <네임스페이스>
kubectl logs <Pod이름> -n <네임스페이스> --previous
```

---

## 리소스 YAML 출력 및 관리

```bash
# 리소스를 YAML로 출력
kubectl get <리소스유형> <이름> -n <네임스페이스> -o yaml

# 리소스를 JSON으로 출력
kubectl get <리소스유형> <이름> -n <네임스페이스> -o json

# 특정 필드만 추출 (jsonpath)
kubectl get pod <이름> -o jsonpath='{.status.podIP}'

# wide 출력 (추가 정보 포함)
kubectl get pods -n <네임스페이스> -o wide

# YAML 파일 적용
kubectl apply -f <파일>.yaml

# YAML 파일 삭제
kubectl delete -f <파일>.yaml

# dry-run으로 YAML 생성 (실제 적용하지 않음)
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml
```

---

## 레이블과 셀렉터

```bash
# 레이블 기반 필터링
kubectl get pods -l app=nginx -n <네임스페이스>

# 여러 레이블 조건
kubectl get pods -l 'app=nginx,version=v2' -n <네임스페이스>

# 레이블 표시
kubectl get pods --show-labels -n <네임스페이스>

# 레이블 추가/변경
kubectl label pod <이름> env=prod -n <네임스페이스>

# 레이블 삭제
kubectl label pod <이름> env- -n <네임스페이스>
```

---

## 실시간 모니터링 (watch)

```bash
# Pod 변화 실시간 감시
kubectl get pods -n <네임스페이스> -w

# 모든 리소스 감시
kubectl get all -n <네임스페이스> -w

# 특정 레이블의 Pod 감시
kubectl get pods -l app=nginx -n <네임스페이스> -w
```

---

## 자주 사용하는 리소스 약어

| 전체 이름 | 약어 |
|-----------|------|
| namespaces | ns |
| nodes | no |
| pods | po |
| services | svc |
| deployments | deploy |
| replicasets | rs |
| statefulsets | sts |
| daemonsets | ds |
| configmaps | cm |
| persistentvolumes | pv |
| persistentvolumeclaims | pvc |
| storageclasses | sc |
| horizontalpodautoscalers | hpa |
| serviceaccounts | sa |
| endpoints | ep |
| ingresses | ing |

---

## 유용한 팁

```bash
# 모든 리소스 한 번에 확인
kubectl get all -n <네임스페이스>

# kubectl 자동 완성 설정 (bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc

# kubectl 자동 완성 설정 (zsh)
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
source ~/.zshrc

# kubectl 별칭 설정
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc

# 이후 'k'로 사용 가능
k get pods -A
```
