---
title: "Grafana 대시보드 구축"
weight: 2
---

LGTM 스택 구축 후, 운영에 필요한 대시보드를 직접 만들었다.
이 글에서는 EKS 인프라 대시보드와 APM 대시보드 구성 과정을 공유한다.

## 배경

Grafana에는 커뮤니티 대시보드가 많지만, 우리 환경에 딱 맞는 건 없었다. cAdvisor 메트릭 기반으로 필요한 지표만 골라서 직접 구성했다.

## EKS Infrastructure 대시보드

클러스터 전체 상태를 한눈에 파악하기 위한 대시보드다.

![EKS Infrastructure Dashboard](/images/grafana-dashboard/eks-infrastructure.png)

### 구성 섹션

| 섹션 | 내용 |
|------|------|
| Cluster Overview | Ready Nodes, Running Pods, CPU/Memory 총량 및 트렌드 |
| Node Health | 노드별 스펙, CPU/Memory 사용률 |
| Pod Status | Pod 상태별 현황 |
| Deployment Status | Deployment 가용성 |
| Resource Usage | 컨테이너별 리소스 사용량 |
| Network & Disk | 네트워크 I/O, 디스크 사용량 |

### 주요 PromQL 쿼리

**Ready Nodes:**
```promql
sum(kube_node_status_condition{condition="Ready", status="true"})
```

**Running Pods:**
```promql
sum(kube_pod_status_phase{phase="Running", namespace=~"$namespace"})
```

**CPU Usage %:**
```promql
sum(rate(container_cpu_usage_seconds_total{namespace=~"$namespace"}[5m]))
/ sum(kube_node_status_allocatable{resource="cpu"}) * 100
```

## APM 대시보드

서비스별 요청량, 에러율, 레이턴시를 모니터링하는 대시보드다.

![APM Dashboard](/images/grafana-dashboard/apm-dashboard.png)

### 구성 섹션

| 섹션 | 내용 |
|------|------|
| Service Summary | 요청 수, 에러 수, 레이턴시 분포, 버전 타임라인 |
| Resources | Top 5 요청 엔드포인트, Top 5 레이턴시, Top 5 에러 |

### 주요 PromQL 쿼리

**Requests per Second:**
```promql
sum(rate(http_server_requests_seconds_count{job="$job"}[1m]))
```

**Error Rate:**
```promql
sum(rate(http_server_requests_seconds_count{job="$job", status=~"4..|5.."}[1m]))
```

**p95 Latency:**
```promql
histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket{job="$job"}[5m])) by (le))
```

## 대시보드 배포 방식

대시보드 JSON 파일을 Git으로 관리하고, Grafana ConfigMap으로 자동 프로비저닝한다.

```
observability/grafana/dashboard/
├── eks-infrastructure-dashboard.json
├── ajdcar-api-dashboard.json
└── observability-dashboard.json
```

Grafana values에서 대시보드 프로바이더를 설정하면 Pod 재시작 시 자동으로 대시보드가 로드된다.

## 결과

인프라 상태와 서비스 상태를 실시간으로 확인할 수 있게 되었다. 알림과 연계하여 문제 발생 시 대시보드에서 바로 원인을 파악할 수 있다.
