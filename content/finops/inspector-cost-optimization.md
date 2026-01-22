---
title: "Inspector 비용, Dev가 Prod보다 10배 높았던 이유"
weight: 3
description: "AWS Inspector와 ECR Enhanced Scanning의 비용 구조를 이해하고, 환경별 적절한 보안 스캔 전략으로 비용을 최적화한 경험."
tags: ["AWS", "Inspector", "ECR", "보안", "비용최적화", "FinOps"]
keywords: ["AWS Inspector 비용", "ECR Enhanced Scanning", "컨테이너 보안 스캔", "Inspector 비용 최적화"]
---

보안은 공짜가 아니다. 하지만 환경별로 필요한 보안 수준은 다르다. Dev 환경에 Prod 수준의 보안 모니터링을 적용하면 비용만 낭비된다.

AWS Inspector 비용 리포트를 보다가 이상한 점을 발견했다. Dev 환경이 Prod보다 거의 10배 비용이 나오고 있었다. 원인을 추적하다 보니 ECR Enhanced Scanning의 동작 방식을 제대로 이해하게 되었다.

## 문제 상황

월간 비용 리포트에서 Inspector 비용이 눈에 띄었다.

| 환경 | Inspector 비용 (월) |
|------|---------------------|
| Dev | **$207** |
| Prod | $10 |

상식적으로 Prod가 더 높아야 하지 않나? 트래픽도 많고, 인스턴스도 많은데. Dev가 Prod의 **20배**라니.

## Amazon Inspector란

Amazon Inspector는 AWS 워크로드의 보안 취약점을 자동으로 스캔하는 서비스다.

**스캔 대상:**
- EC2 인스턴스
- ECR 컨테이너 이미지
- Lambda 함수

**주요 기능:**
- CVE(Common Vulnerabilities and Exposures) 기반 취약점 탐지
- 네트워크 도달성 분석
- 취약점 심각도 분류 및 우선순위 제공

Inspector 자체는 좋은 서비스다. 문제는 **어떻게 사용하느냐**에 있었다.

## ECR 스캔 방식: Basic vs Enhanced

ECR에서 컨테이너 이미지를 스캔하는 방식은 두 가지가 있다.

| 구분 | Basic Scanning | Enhanced Scanning |
|------|----------------|-------------------|
| 제공자 | ECR 자체 (Clair 기반) | Amazon Inspector 통합 |
| 스캔 시점 | 푸시 시 또는 수동 | **지속적** (새 CVE 발견 시 자동 재스캔) |
| 비용 | 무료 | 유료 (이미지당 스캔 횟수 과금) |
| 커버리지 | OS 패키지만 | OS + 언어 패키지 (npm, pip 등) |

### Basic Scanning

이미지를 ECR에 푸시할 때 한 번 스캔한다. 또는 수동으로 스캔을 트리거할 수 있다. **무료**다.

### Enhanced Scanning

Inspector와 통합되어 **지속적으로** 이미지를 스캔한다. 핵심은 "지속적"이라는 점이다.

새로운 CVE가 발표되면, Inspector가 기존에 스캔했던 모든 이미지를 다시 스캔해서 해당 취약점이 있는지 확인한다. 어제까지 안전했던 이미지가 오늘 새 CVE 발표로 취약해질 수 있기 때문이다.

이 기능은 Prod 환경에서는 의미가 있다. 운영 중인 이미지의 보안 상태를 실시간으로 파악해야 하니까.

## 원인: Dev에서 Enhanced Scanning

문제는 Dev에서도 Enhanced Scanning이 켜져 있었다는 것이다.

Dev 환경의 특성:
- 빌드가 빈번하다 (하루에도 수십 번)
- 이미지가 계속 쌓인다
- 대부분의 이미지는 테스트 후 버려진다

Enhanced Scanning이 켜져 있으면:
- 모든 이미지가 지속적 스캔 대상
- 새 CVE 발표될 때마다 전체 이미지 재스캔
- 이미지 수 × 스캔 횟수 = 비용 폭증

Dev는 Prod보다 이미지 수가 훨씬 많았다. 빌드할 때마다 새 이미지가 쌓이니까. 그리고 그 모든 이미지가 계속 재스캔되고 있었다.

## 해결: 환경별 스캔 전략

Dev와 Prod의 보안 요구사항은 다르다.

| 환경 | 필요한 스캔 | 이유 |
|------|------------|------|
| Dev | 빌드 시점 1회 | 개발 중 취약점 확인용, 이미지는 곧 교체됨 |
| Prod | 지속적 스캔 | 운영 중인 이미지의 실시간 보안 상태 파악 |

**적용한 변경:**
- Dev: Enhanced Scanning → Basic Scanning으로 전환
- Prod: Enhanced Scanning 유지

ECR 콘솔에서 리포지토리별로 스캔 설정을 변경할 수 있다. 또는 Inspector 설정에서 특정 리포지토리를 스캔 대상에서 제외할 수 있다.

## 결과

| 환경 | Before | After |
|------|--------|-------|
| Dev | $207/월 | **$0/월** |
| Prod | $10/월 | $10/월 (유지) |

Dev 환경의 Inspector 비용이 **$207 → $0**으로 떨어졌다. Basic Scanning은 무료이기 때문이다.

보안 수준이 낮아진 것은 아니다. 빌드 시점에 취약점을 확인하는 것은 동일하다. 다만 빌드 후 지속적 모니터링을 하지 않을 뿐이다. Dev 이미지는 어차피 금방 교체되니까 문제없다.

Prod는 월 $10 정도로 유지된다. 운영 중인 이미지만 스캔하면 되니까 이미지 수가 적고, 비용도 적다.

## 추가 최적화 포인트

### 이미지 Lifecycle Policy

Enhanced Scanning 비용을 더 줄이려면 이미지 보존 정책을 타이트하게 설정해야 한다.

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep only 10 images",
      "selection": {
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

오래된 이미지를 자동으로 삭제하면 스캔 대상 이미지 수가 줄어들고, 비용도 줄어든다.

### Inspector 스캔 제외 설정

특정 리포지토리나 태그를 Inspector 스캔 대상에서 제외할 수 있다. 테스트용 이미지나 임시 이미지는 스캔할 필요가 없다.

## 정리

Inspector 비용 최적화의 핵심은 **환경별 보안 요구사항을 구분**하는 것이다.

1. Enhanced Scanning의 동작 방식 이해: 지속적 재스캔
2. Dev vs Prod 보안 요구사항 구분
3. Dev: Basic Scanning으로 충분
4. Prod: Enhanced Scanning으로 실시간 모니터링
5. 이미지 Lifecycle Policy로 스캔 대상 최소화

보안과 비용은 트레이드오프가 아니다. 환경에 맞는 적절한 수준을 선택하면 된다.
