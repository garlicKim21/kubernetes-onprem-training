# Ch.12 모니터링과 관측성: Prometheus & Grafana

## 학습 목표

- Prometheus의 아키텍처와 스크래핑 모델을 이해한다
- kube-prometheus-stack의 구성 요소를 이해한다
- ServiceMonitor / PodMonitor의 개념을 이해한다
- Grafana 대시보드를 탐색하고 클러스터 상태를 모니터링한다
- HPA 데모와 연관하여 메트릭 변화를 관찰한다

---

## 1. Prometheus 아키텍처

### Prometheus란?

Prometheus는 **CNCF 졸업 프로젝트**로, 쿠버네티스 환경에서 가장 널리 사용되는 **오픈소스 모니터링 시스템**입니다.

### Pull 기반 스크래핑 모델

Prometheus는 다른 모니터링 시스템과 달리 **Pull 방식**으로 메트릭을 수집합니다:

```
  ┌─────────────────────────────────────────────────────┐
  │                   Prometheus Server                  │
  │                                                      │
  │  ┌──────────┐  ┌──────────┐  ┌───────────────────┐ │
  │  │ Scraper  │  │ TSDB     │  │ PromQL Engine     │ │
  │  │ (수집기)  │  │ (시계열DB)│  │ (쿼리 엔진)       │ │
  │  └────┬─────┘  └──────────┘  └───────────────────┘ │
  │       │                              │               │
  └───────┼──────────────────────────────┼───────────────┘
          │ HTTP GET /metrics             │ 쿼리
          │ (주기적 스크래핑)              │
    ┌─────┼─────┐                   ┌─────┼─────┐
    ▼     ▼     ▼                   ▼           ▼
  ┌───┐ ┌───┐ ┌───┐           ┌────────┐  ┌──────────┐
  │앱1│ │앱2│ │앱3│           │Grafana │  │Alertmgr │
  └───┘ └───┘ └───┘           └────────┘  └──────────┘
  /metrics 엔드포인트 노출      시각화        알림 전송
```

### 핵심 특징

| 특징 | 설명 |
|------|------|
| **Pull 모델** | Prometheus가 대상에게 HTTP로 메트릭을 요청 |
| **시계열 데이터베이스 (TSDB)** | 타임스탬프 기반의 효율적인 데이터 저장 |
| **PromQL** | 강력한 쿼리 언어로 메트릭 분석 |
| **서비스 디스커버리** | 쿠버네티스 API를 통해 모니터링 대상 자동 발견 |
| **다차원 데이터 모델** | 레이블(Label) 기반으로 메트릭 분류 |

---

## 2. kube-prometheus-stack 구성 요소

우리 클러스터에는 **kube-prometheus-stack**이 설치되어 있습니다. 이 스택에는 다음 구성 요소가 포함됩니다:

| 구성 요소 | 역할 |
|-----------|------|
| **Prometheus** | 메트릭 수집 및 저장 |
| **Grafana** | 시각화 대시보드 |
| **Alertmanager** | 알림 라우팅 및 전송 (Slack, 이메일 등) |
| **node-exporter** | 노드(서버)의 시스템 메트릭 수집 (CPU, 메모리, 디스크, 네트워크) |
| **kube-state-metrics** | 쿠버네티스 오브젝트 상태 메트릭 (Pod 수, Deployment 상태 등) |
| **Prometheus Operator** | CRD를 통한 Prometheus 설정 관리 |

```
  ┌─────────────────────────────────────────────────────────────┐
  │                    kube-prometheus-stack                      │
  │                                                              │
  │  ┌──────────────┐  ┌──────────┐  ┌──────────────────────┐  │
  │  │  Prometheus   │  │ Grafana  │  │    Alertmanager      │  │
  │  │  Server       │  │          │  │                      │  │
  │  └──────┬────────┘  └────┬─────┘  └──────────────────────┘  │
  │         │                │                                    │
  │  ┌──────┴────────┐      │   시각화                           │
  │  │ 메트릭 수집    │      │                                    │
  │  │               │      │                                    │
  │  ▼               ▼      │                                    │
  │  ┌────────────┐ ┌────────────────┐ ┌─────────────────────┐  │
  │  │ node-      │ │ kube-state-    │ │ Prometheus         │  │
  │  │ exporter   │ │ metrics        │ │ Operator (CRD)     │  │
  │  └────────────┘ └────────────────┘ └─────────────────────┘  │
  └─────────────────────────────────────────────────────────────┘
```

---

## 3. ServiceMonitor / PodMonitor

### ServiceMonitor

ServiceMonitor는 **Prometheus Operator의 CRD(Custom Resource Definition)**로, Prometheus가 어떤 Service의 메트릭을 수집할지 선언적으로 정의합니다.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-app                    # 이 레이블을 가진 Service 대상
  endpoints:
    - port: metrics                  # Service의 "metrics" 포트에서 스크래핑
      interval: 30s                  # 30초마다 수집
      path: /metrics                 # 메트릭 엔드포인트 경로
```

### PodMonitor

PodMonitor는 Service 없이 **Pod에서 직접** 메트릭을 수집합니다.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-pod-monitor
spec:
  selector:
    matchLabels:
      app: my-app
  podMetricsEndpoints:
    - port: metrics
      interval: 15s
```

### ServiceMonitor vs PodMonitor

| 항목 | ServiceMonitor | PodMonitor |
|------|---------------|------------|
| 대상 | Service를 통해 Pod 발견 | Pod 직접 선택 |
| 사용 시나리오 | Service가 있는 일반적인 경우 | DaemonSet, 독립 Pod 등 |

---

> 🎓 **강사 데모** — 이 섹션은 강사가 시연합니다. 수강생들은 Headlamp이나 Grafana에서 결과를 확인할 수 있습니다.

## 4. Grafana 대시보드 투어

### 4.1 Grafana 접속

1. 웹 브라우저에서 **https://grafana.basphere.dev** 접속
2. 로그인 정보:
   - **수강생용**: 사용자명 `student`, 비밀번호 `k8s-training` (읽기 전용)
   - **관리자용**: 사용자명 `admin`, 비밀번호 `Basphere2026!` (전체 권한)

### 4.2 대시보드 탐색: 클러스터 개요

1. 왼쪽 메뉴에서 **Dashboards** (대시보드 아이콘) 클릭
2. **Kubernetes / Compute Resources / Cluster** 선택

이 대시보드에서 확인할 수 있는 내용:

| 패널 | 설명 |
|------|------|
| CPU Usage | 클러스터 전체 CPU 사용량 |
| CPU Quota | 네임스페이스별 CPU 요청/제한 |
| Memory Usage | 클러스터 전체 메모리 사용량 |
| Memory Quota | 네임스페이스별 메모리 요청/제한 |

### 4.3 노드 메트릭 확인

1. **Kubernetes / Compute Resources / Node (Pods)** 대시보드 선택
2. 상단에서 노드 선택 (예: `wrk-0`)

확인 가능한 메트릭:
- 노드별 CPU/메모리 사용률
- Pod별 리소스 사용량
- 네트워크 I/O
- 파일시스템 사용량

### 4.4 Pod 메트릭 확인

1. **Kubernetes / Compute Resources / Namespace (Pods)** 대시보드 선택
2. 네임스페이스를 `load-tester`로 변경
3. 개별 Pod의 CPU/메모리 사용량 그래프를 확인

### 4.5 Cilium 대시보드 (ServiceMonitor가 활성화된 경우)

Cilium이 ServiceMonitor를 통해 메트릭을 노출하고 있다면:

1. **Cilium / Overview** 또는 **Cilium / Operator** 대시보드 선택
2. 확인 가능한 내용:
   - 네트워크 정책 적용 현황
   - Pod 간 트래픽 흐름
   - DNS 쿼리 통계
   - eBPF 프로그램 상태

### 4.6 HPA 데모와 연계 관찰

Ch.11에서 실행한 HPA 부하 테스트 결과를 Grafana에서 확인합니다:

1. **Kubernetes / Compute Resources / Namespace (Pods)** 대시보드
2. Namespace: `load-tester`
3. 관찰 포인트:
   - **CPU 사용량 그래프**: 부하 시작 시 급격한 상승, Pod 추가 후 분산
   - **Pod 수 변화**: Pod가 1개 → 여러 개로 증가하는 것을 확인
   - **부하 중지 후**: CPU 사용량 감소, 약 5분 후 Pod 수 감소

```
  CPU 사용률 그래프 예시:

  100% │     ╱╲
       │    ╱  ╲     ← 부하 시작, HPA가 Pod 추가
   50% │───╱────╲──────────────────── 목표선
       │  ╱      ╲    ← Pod 추가로 부하 분산
       │ ╱        ╲_______________
    0% │╱                          ╲___ ← 부하 중지
       └──────────────────────────────── 시간
           ↑           ↑            ↑
       부하시작     Pod증가      부하중지
```

---

## 5. Alertmanager 개요

### Alertmanager란?

Alertmanager는 Prometheus가 발생시킨 **알림(Alert)을 관리하고 전송**하는 구성 요소입니다.

```
  Prometheus                    Alertmanager                   수신처
  ┌──────────┐    Alert     ┌──────────────┐    알림 전송    ┌──────────┐
  │ Alert    │──────────▶  │ 그룹핑       │──────────────▶ │ Slack    │
  │ Rules    │              │ 중복 제거     │               │ Email    │
  │          │              │ 라우팅       │               │ PagerDuty│
  └──────────┘              │ 묵음(Silence)│               │ Webhook  │
                            └──────────────┘               └──────────┘
```

### Alertmanager 주요 기능

| 기능 | 설명 |
|------|------|
| **그룹핑 (Grouping)** | 유사한 알림을 묶어서 하나의 알림으로 전송 |
| **억제 (Inhibition)** | 특정 알림이 발생하면 관련된 다른 알림을 억제 |
| **묵음 (Silence)** | 특정 기간 동안 알림을 무시 |
| **라우팅 (Routing)** | 알림 종류에 따라 다른 수신처로 전송 |

### 알림 규칙 예시

```yaml
# PrometheusRule CRD
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-alerts
spec:
  groups:
    - name: pod.rules
      rules:
        - alert: PodCrashLooping
          expr: rate(kube_pod_container_status_restarts_total[5m]) > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }}가 반복적으로 재시작되고 있습니다"
```

---

## 핵심 요약

| 개념 | 설명 |
|------|------|
| **Prometheus** | Pull 기반 메트릭 수집, 시계열 DB 저장, PromQL 쿼리 |
| **Grafana** | 메트릭 시각화 대시보드 |
| **Alertmanager** | 알림 관리 및 전송 |
| **node-exporter** | 노드 시스템 메트릭 수집 |
| **kube-state-metrics** | 쿠버네티스 오브젝트 상태 메트릭 |
| **ServiceMonitor** | Prometheus가 수집할 대상을 선언적으로 정의 |

---

> **다음 챕터**: [Ch.13 종합 데모: 배포부터 오토스케일링까지](../ch13-comprehensive-demo/README.md)
