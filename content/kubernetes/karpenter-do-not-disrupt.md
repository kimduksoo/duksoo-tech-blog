---
title: "Karpenter do-not-disrupt로 Prod 안정성과 비용 절감 양립하기"
weight: 8
description: "Prod EKS에서 Karpenter consolidation을 꺼둘 수밖에 없었던 이유와, do-not-disrupt 어노테이션과 시간대 제한으로 안전하게 유휴 노드를 정리한 경험"
tags: ["Kubernetes", "Karpenter", "Cost Optimization", "do-not-disrupt", "Disruption"]
keywords: ["Karpenter do-not-disrupt", "consolidation", "disruption budgets", "schedule", "비용 최적화", "NodePool", "Pod 보호"]
---

Karpenter consolidation의 진짜 어려움은 "켜느냐 끄느냐"가 아니다. 클러스터 안의 워크로드가 모두 같은 수준의 내구성을 갖고 있지 않다는 점이다. 어떤 서비스는 재배치되어도 문제없지만, 어떤 서비스는 한 번의 재시작이 장애로 이어진다.

Prod 환경에서 유휴 노드 3대가 CPU 3~4%로 방치되고 있었지만, consolidation을 켤 수 없었다. 과거에 consolidation으로 민감한 서비스들이 동시에 재시작되면서 장애가 발생한 이력이 있었기 때문이다. `do-not-disrupt` 어노테이션으로 민감한 서비스만 선택적으로 보호하고, consolidation 시간대를 새벽으로 제한하여 안정성과 비용 절감을 양립한 과정을 공유한다.

## 이전 글과의 관계

[이전 글](/kubernetes/karpenter-cost-optimization-vs-stability/)에서는 Beta 환경에서 Spot + Consolidation + Drift가 복합적으로 작용하는 상황을 다뤘다. 이번 글은 **Prod 환경에서 On-Demand + Stable NodePool**이라는 다른 맥락이다.

| 항목 | 이전 글 (Beta) | 이번 글 (Prod) |
|------|---------------|---------------|
| 인스턴스 타입 | Spot | On-Demand |
| 문제 | 업무시간 중 잦은 Pod 재시작 | 유휴 노드 비용 낭비 |
| 해결 | Disruption Budgets Schedule | **do-not-disrupt** + Schedule |
| 핵심 | NodePool 수준 정책 | **Pod 수준 + NodePool 수준 조합** |

## 문제 상황

Prod EKS 클러스터의 stable NodePool에는 3대의 t3.xlarge 노드가 있었다.

```
stable 노드 A: CPU 3%
stable 노드 B: CPU 4%
stable 노드 C: CPU 2% (인프라 Pod만 잔류)
```

단순 계산으로는 3대를 1대로 줄일 수 있다. 월 $310 절감. 그런데 stable NodePool의 consolidation은 사실상 꺼져 있었다.

```yaml
# 당시 설정
disruption:
  consolidationPolicy: WhenEmpty    # 빈 노드만 정리 (Pod 1개라도 있으면 안 건드림)
  expireAfter: Never                # 노드 만료 없음
```

### consolidation을 끈 이유

클러스터에는 부팅 시 높은 CPU를 사용하거나, 재배치에 민감한 서비스들이 있었다. 과거에 consolidation으로 인해 이런 서비스들이 동시에 재시작되면서 장애가 발생한 이력이 있었고, 이후 stable NodePool의 consolidation을 꺼두게 되었다.

안전하지만, 유휴 노드가 24시간 방치되면서 비용을 태우는 구조였다.

### 딜레마

consolidation을 **켜면** 유휴 노드가 정리되지만, 민감한 서비스가 재배치되면서 장애 위험이 생긴다. **끄면** 안전하지만 비용이 낭비된다.

이 딜레마를 해결하려면, "전부 켜거나 전부 끄는" 이분법에서 벗어나야 했다.

## 가설: 선택적 보호

모든 서비스가 consolidation에 민감한 건 아니었다. PDB(Pod Disruption Budget)와 TopologySpread가 갖춰진 서비스는 재배치되어도 안전하다. 문제는 이런 안전 장치 없이 재배치에 민감한 일부 서비스였다.

**가설: 민감한 서비스만 consolidation에서 제외하고, 나머지 유휴 노드는 새벽에 정리하면 되지 않을까?**

Karpenter의 `do-not-disrupt` 어노테이션이 이 가설을 실현할 수 있는 기능이었다.

## do-not-disrupt 어노테이션

### 기본 동작

Pod에 `karpenter.sh/do-not-disrupt: "true"` 어노테이션을 추가하면, Karpenter는 **해당 Pod이 있는 노드 전체를 consolidation 대상에서 제외**한다.

```yaml
# Deployment의 Pod template에 추가
spec:
  template:
    metadata:
      annotations:
        karpenter.sh/do-not-disrupt: "true"
```

Pod만 보호되는 게 아니라, **노드 자체를 건드리지 않는다**는 점이 중요하다.

```
Karpenter: "이 노드 drain 할까?"
  → Pod 목록 확인
  → do-not-disrupt: "true" Pod 발견
  → 이 노드는 건너뜀 (drain 자체를 안 함)
```

같은 노드에 있는 다른 Pod들도 함께 보호된다.

### 보호 범위

do-not-disrupt가 보호하는 것과 못 막는 것을 구분해야 한다.

| 상황 | 보호 여부 | 분류 |
|------|----------|------|
| Consolidation (유휴 노드 통합) | ✅ 보호됨 | 자발적 disruption |
| Underutilized 정리 | ✅ 보호됨 | 자발적 disruption |
| Drift 감지 (AMI 변경 등) | ✅ 보호됨 | 자발적 disruption |
| expireAfter 만료 | ❌ 못 막음 | 강제적 disruption |
| kubectl delete node (수동) | ❌ 못 막음 | 수동 |
| EC2 Spot 중단 | ❌ 못 막음 | AWS 강제 |

핵심은 **자발적 disruption만 차단**한다는 점이다. expireAfter나 Spot 중단 같은 강제적 경로는 막지 못한다.

우리 환경에서는 stable NodePool이 `expireAfter: Never` + On-Demand 인스턴스이므로, 강제적 disruption 경로가 없다. do-not-disrupt로 완전한 보호가 가능한 조건이다.

### 적용 방법과 주의사항

do-not-disrupt를 적용하는 방법은 두 가지다.

| 방법 | 재시작 | 지속성 | ArgoCD 호환 |
|------|--------|--------|------------|
| `kubectl annotate pod` | ❌ 안 됨 | 임시 (Pod 재생성 시 사라짐) | ❌ 원복됨 |
| Helm values에서 podAnnotations 추가 | ✅ Rolling Update 발생 | 영구 | ✅ |

ArgoCD + Helm으로 관리하는 환경에서는 Helm values 수정이 올바른 방법이다. 다만 Pod template이 변경되므로 **Rolling Update가 발생**한다는 점을 미리 공지해야 한다.

```yaml
# values.prod.yaml
podAnnotations:
  karpenter.sh/do-not-disrupt: "true"
  # 기존 어노테이션 유지
  ad.datadoghq.com/web.check_names: '["openmetrics"]'
```

## Consolidation 시간대 제한

do-not-disrupt로 민감한 서비스를 보호한 뒤, NodePool의 consolidation 정책을 변경했다.

### NodePool 수준 설정

```yaml
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized  # 유휴 노드도 정리 대상
  consolidateAfter: 300s
  budgets:
    - nodes: "1"             # 동시에 1대만 drain
      schedule: "0 19 * * *" # UTC 19시 = KST 새벽 4시
      duration: 2h           # 4시~6시만 허용
    - nodes: "0"             # 나머지 시간 차단
```

budgets에 schedule과 duration을 지정하면, 특정 시간대에만 consolidation을 허용할 수 있다. 여러 budget이 있으면 가장 제한적인 조건이 적용된다.

```
00:00~03:59  → budget 매칭: nodes "0" → consolidation 차단
04:00~05:59  → budget 매칭: nodes "1" → 1대씩 유휴 노드 정리
06:00~23:59  → budget 매칭: nodes "0" → consolidation 차단
```

### Pod 수준 + NodePool 수준 조합

두 설정을 조합하면 이런 흐름이 된다.

```
새벽 4시: consolidation 창 열림
  ↓
Karpenter: "stable 노드 A를 drain할까?"
  → 민감한 서비스 Pod 발견 → do-not-disrupt: true
  → ❌ 이 노드는 건너뜀
  ↓
Karpenter: "stable 노드 C를 drain할까?"
  → 보호 대상 Pod 없음
  → ✅ drain 진행 → 유휴 노드 정리
  ↓
새벽 6시: consolidation 창 닫힘
```

민감한 서비스가 있는 노드는 24시간 보호되고, 민감한 서비스가 없는 유휴 노드만 새벽에 정리된다.

## Pod 수준 vs NodePool 수준 비교

do-not-disrupt는 Pod과 Node 양쪽에서 설정할 수 있고, NodePool의 disruption budgets와 역할이 다르다. 각각의 용도를 정리한다.

| 구분 | Pod 수준 (do-not-disrupt) | NodePool 수준 (budgets) |
|------|-------------------------|------------------------|
| 설정 위치 | Deployment annotation | NodePool spec.disruption.budgets |
| 범위 | 해당 Pod이 있는 노드만 보호 | NodePool 전체 노드에 적용 |
| 시간대 제어 | ❌ 불가 (항상 보호) | ✅ schedule + duration |
| 비율 제어 | ❌ 불가 | ✅ nodes: "10%", "1", "0" |
| 용도 | 특정 워크로드 선택적 보호 | 전체 정책 관리 |

이번 작업에서는 **둘 다 사용**했다.
- Pod 수준: 민감한 서비스만 선택적 보호 (어떤 시간이든 보호)
- NodePool 수준: 새벽 시간대만 consolidation 허용 (전체 정책)

## 적용 결과

| 항목 | 변경 전 | 변경 후 |
|------|--------|--------|
| stable 노드 수 | 3대 (고정) | 2~3대 (새벽 정리) |
| consolidation | 사실상 불가 (WhenEmpty) | 새벽 4~6시 동작 (WhenEmptyOrUnderutilized) |
| 민감 서비스 보호 | 없음 (정책으로 우회) | do-not-disrupt로 명시적 보호 |
| 업무시간 영향 | 없음 | 없음 (동일) |
| 예상 절감 | - | 월 ~$150 |

공격적으로 노드를 줄이진 않았지만, 안정성을 유지하면서 유휴 리소스를 정리하는 구조를 만들었다.

## 배운 것들

### 비용 절감은 안정성 분석이 먼저다

"request를 줄이고 노드를 합치면 된다"는 단순한 접근은 위험하다. 클러스터 안의 워크로드가 모두 같은 수준의 내구성을 갖고 있지 않기 때문이다. 어떤 서비스가 재배치에 민감한지 먼저 파악해야 한다.

### do-not-disrupt는 "선택적 보호"

모든 것을 보호하면 consolidation이 의미 없고, 아무것도 보호하지 않으면 장애 위험이 있다. 위험한 워크로드만 골라서 보호하는 것이 핵심이다. PDB, TopologySpread 등 안전 장치가 충분한 서비스는 보호 대상에서 제외해도 된다.

### 시간대 제한은 가장 안전한 첫 걸음

consolidation을 켜고 싶지만 확신이 없을 때, 새벽 시간대만 여는 것이 리스크가 가장 낮다. 문제가 생겨도 사용자 영향이 없는 시간이고, 모니터링 후 점진적으로 확대할 수 있다.

### 단계적 접근

한 번에 모든 최적화를 적용하지 않았다.

1. do-not-disrupt로 민감한 서비스 보호
2. consolidation 시간대 제한 적용
3. 주말 새벽 모니터링
4. 결과 확인 후 다음 단계

각 단계마다 안전 장치를 확인하고, 문제가 없음을 검증한 뒤 다음 단계로 넘어갔다.

## 참고

- [Karpenter 공식 문서 - Disruption](https://karpenter.sh/docs/concepts/disruption/)
- [Karpenter Disruption 정리 - cobain.me](https://cobain.me/2024/04/03/Karpenter-Disruption.html)
