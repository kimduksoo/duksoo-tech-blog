---
title: "Grafana 대시보드 구축"
weight: 2
---

LGTM 스택을 구축하고 나니, 다음 질문이 생겼다. "뭘 봐야 하지?"

메트릭은 쌓이고 있는데, 대시보드가 없으니 의미가 없었다. 커뮤니티 대시보드를 몇 개 import 해봤지만, 우리 환경과 맞지 않는 부분이 많았다. 결국 직접 만들기로 했다.

## 어떤 대시보드가 필요한가

운영 관점에서 세 가지가 필요했다.

1. **인프라 대시보드**: 클러스터 전체 상태, 노드 헬스, 리소스 사용량
2. **APM 대시보드**: 서비스별 요청량, 에러율, 레이턴시
3. **파이프라인 대시보드**: LGTM 스택 자체의 헬스 체크

Datadog을 쓸 때는 기본 제공되던 것들이다. LGTM 스택에서는 직접 구성해야 한다.

## EKS Infrastructure 대시보드

클러스터 전체 상태를 한눈에 파악하기 위한 대시보드다.

![EKS Infrastructure Dashboard](/images/grafana-dashboard/eks-infrastructure.png)

### 설계 의도

대시보드를 열었을 때 3초 안에 "지금 클러스터가 정상인가?"를 판단할 수 있어야 했다.

상단에는 핵심 숫자를 배치했다:
- Ready Nodes / Running Pods → 클러스터가 살아있는가
- Total CPU / Memory → 전체 리소스 현황
- Trend 그래프 → 급격한 변화가 있는가

그 아래로 드릴다운 할 수 있도록 섹션을 구성했다:

| 섹션 | 확인 포인트 |
|------|------------|
| Node Health | 특정 노드에 문제가 있는가 |
| Pod Status | Pending, Failed Pod가 있는가 |
| Deployment Status | Replica가 부족한 Deployment가 있는가 |
| Resource Usage | 리소스를 과도하게 사용하는 컨테이너가 있는가 |

### 핵심 쿼리

노드 상태 확인:
```promql
sum(kube_node_status_condition{condition="Ready", status="true"})
```

CPU 사용률 (노드별):
```promql
sum(rate(container_cpu_usage_seconds_total[5m])) by (node)
/ on(node) kube_node_status_allocatable{resource="cpu"} * 100
```

## APM 대시보드

서비스별 트래픽과 성능을 모니터링하는 대시보드다.

![APM Dashboard](/images/grafana-dashboard/apm-dashboard.png)

### 설계 의도

"지금 서비스에 문제가 있는가?"를 빠르게 파악하는 것이 목적이다.

RED 메트릭(Rate, Errors, Duration)을 기준으로 구성했다:
- **Rate**: 요청량 추이, 평소 대비 급증/급감 확인
- **Errors**: 에러 발생 현황, 4xx/5xx 구분
- **Duration**: 레이턴시 분포, p50/p95/p99 확인

Version Timeline 패널은 배포 시점을 표시한다. 성능 저하가 발생했을 때 "최근 배포 때문인가?"를 바로 확인할 수 있다.

### 핵심 쿼리

요청량:
```promql
sum(rate(http_server_requests_seconds_count{job="$job"}[1m]))
```

p95 레이턴시:
```promql
histogram_quantile(0.95,
  sum(rate(http_server_requests_seconds_bucket{job="$job"}[5m])) by (le)
)
```

에러율:
```promql
sum(rate(http_server_requests_seconds_count{job="$job", status=~"5.."}[1m]))
/ sum(rate(http_server_requests_seconds_count{job="$job"}[1m])) * 100
```

## Observability Pipeline 대시보드

LGTM 스택 자체를 모니터링하는 대시보드다. 수집기(Alloy)와 저장소(Loki, Mimir, Tempo) 컴포넌트의 헬스를 확인한다.

![Observability Pipeline Dashboard](/images/grafana-dashboard/observability-pipeline.png)

### 설계 의도

모니터링 시스템이 죽으면 알림도 오지 않는다. LGTM 스택 자체의 상태를 별도로 감시해야 한다.

주요 확인 포인트:
- **Pod CPU/Memory Usage**: 각 컴포넌트의 리소스 사용량
- **CPU Throttling**: 리소스 limit에 걸려서 throttling 되는지
- **OOM Events**: 메모리 부족으로 재시작되는 Pod가 있는지

Mimir Ingester나 Loki가 OOM으로 재시작되면 메트릭/로그 유실이 발생할 수 있다. 이 대시보드로 사전에 감지한다.

## 설계 시 고민

### 로딩 성능

대시보드에 패널이 많아지면 로딩이 느려진다. 처음엔 Row를 기본 접힘(collapsed) 상태로 설정해서 필요한 섹션만 펼쳐보는 방식을 고민했다.

찾아보니 Grafana는 기본적으로 lazy loading을 지원한다. 뷰포트에 보이는 패널만 쿼리를 실행하고, 스크롤하면 그때 로딩한다. 덕분에 Row를 굳이 접어두지 않아도 초기 로딩이 무겁지 않았다.

다만 상단 Overview 섹션은 항상 펼쳐두고, 세부 섹션은 필요에 따라 Row collapse를 활용했다.

### GitOps 방식 관리

대시보드를 UI에서만 관리하면 변경 이력 추적이 안 된다. 누가 언제 뭘 바꿨는지 알 수 없고, 롤백도 어렵다.

대시보드 JSON을 Git에 저장하고, Grafana ConfigMap provisioning으로 자동 배포하는 방식을 선택했다.

```yaml
# grafana/values-common.yaml
dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
      - name: 'default'
        folder: 'ajdcar'
        type: file
        options:
          path: /var/lib/grafana/dashboards
```

워크플로우:
1. Grafana UI에서 대시보드 수정
2. JSON export
3. Git commit & push
4. Helm upgrade 또는 Pod 재시작 시 자동 반영

이렇게 하면 대시보드도 코드처럼 리뷰하고 관리할 수 있다.

## 대시보드 파일 구조

대시보드 JSON 파일은 Git으로 관리한다.

```
observability/grafana/dashboard/
├── eks-infrastructure-dashboard.json
├── ajdcar-api-dashboard.json
└── observability-dashboard.json
```

ConfigMap 프로비저닝을 설정해두면 Grafana Pod 재시작 시 자동으로 로드된다. 대시보드가 코드로 관리되니 변경 이력 추적도 가능하다.

## 결과

대시보드를 만들고 나니 모니터링이 실체화되었다. 알림이 오면 대시보드를 열어 상황을 파악하고, 어디를 더 봐야 하는지 판단할 수 있게 되었다.

아직 부족한 부분도 있다. 로그와 트레이스 연동, 더 세분화된 서비스별 대시보드는 다음 단계로 남겨두었다.
