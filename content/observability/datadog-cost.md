---
title: "Datadog 비용 최적화 경험담"
weight: 1
---

Datadog 비용이 예상보다 빠르게 증가해서 최적화를 진행한 경험을 공유합니다.

## 비용 구조 분석

```mermaid
pie title Datadog 비용 비중
    "Logs" : 45
    "APM" : 30
    "Infra" : 15
    "RUM" : 10
```

## 문제 발견

| 월 | Logs (GB/day) | 비용 |
|----|---------------|------|
| 10월 | 50 | $2,500 |
| 11월 | 80 | $4,000 |
| 12월 | 120 | $6,000 |

## 원인 분석

```mermaid
flowchart TB
    A[Logs 급증] --> B{원인 분석}
    B --> C[Debug 로그 ON]
    B --> D[중복 로그]
    B --> E[불필요한 서비스]

    C --> F[Log Level 조정]
    D --> G[로그 필터링]
    E --> H[Exclusion Filter]
```

## 결과

| 항목 | Before | After | 절감 |
|------|--------|-------|------|
| Logs/day | 120GB | 40GB | -67% |
| APM Spans | 100M | 70M | -30% |
| **월 비용** | $6,000 | $4,200 | **-30%** |
