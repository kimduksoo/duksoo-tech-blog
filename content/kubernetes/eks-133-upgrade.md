---
title: "EKS 1.32 → 1.33 업그레이드 실전 가이드"
weight: 8
description: "dev에서 삽질하고 prod에서 깔끔하게 마친 EKS 업그레이드 경험. VPC CNI 설정 불일치, Karpenter CRD 문제, DD_HOSTNAME 누락까지."
tags: ["Kubernetes", "EKS", "업그레이드", "Karpenter", "VPC-CNI", "구축도입기"]
keywords: ["EKS 1.33 업그레이드", "EKS upgrade", "Karpenter 업그레이드", "VPC CNI", "Self-managed addon"]
---

EKS 1.33이 릴리스됐다. 보통 마이너 버전 업그레이드는 "Control Plane 올리고, Addon 올리고, Node 올리면 끝"이라고 생각하기 쉽다. 나도 그렇게 생각했다.

dev 환경에서 먼저 작업하면서 예상치 못한 문제들을 만났다. VPC CNI 설정 불일치로 Pod IP 할당이 안 되고, Karpenter CRD 버전 문제로 노드 프로비저닝이 막히고, Datadog DD_HOSTNAME 누락으로 모니터링이 깨졌다.

dev에서 삽질한 덕분에 prod는 40분 만에 깔끔하게 끝냈다. 그 과정을 공유한다.

## 환경 구성

| 환경 | 노드 구성 | 특이사항 |
|------|----------|---------|
| dev | MNG + Karpenter | Self-managed Addons |
| prod | MNG + Karpenter | Self-managed Addons |

두 환경 모두 **Self-managed Addon** 방식이다. EKS managed addon이 아니라 직접 이미지 버전을 관리한다. 이게 업그레이드 시 추가 작업이 필요한 이유다.

## dev 환경 업그레이드

### Step 1: Control Plane 업그레이드

```bash
aws eks update-cluster-version --name eks-blue --kubernetes-version 1.33
```

약 8분 소요. 여기까지는 순조로웠다.

### Step 2: Core Addon 업그레이드 - 첫 번째 삽질

Self-managed addon이라 직접 이미지를 업데이트해야 한다.

```bash
# ECR 베이스 URL
ECR=602401143452.dkr.ecr.ap-northeast-2.amazonaws.com

# VPC CNI
kubectl set image daemonset/aws-node -n kube-system \
  aws-node=${ECR}/amazon-k8s-cni:v1.21.1

# CoreDNS
kubectl set image deployment/coredns -n kube-system \
  coredns=${ECR}/eks/coredns:v1.12.4-eksbuild.1

# kube-proxy
kubectl set image daemonset/kube-proxy -n kube-system \
  kube-proxy=${ECR}/eks/kube-proxy:v1.33.0-minimal-eksbuild.2
```

VPC CNI 업데이트 후 테스트 Pod를 띄워봤다.

```bash
kubectl run test-pod --image=busybox --restart=Never --command -- sleep 30
kubectl get pod test-pod -o wide
```

**Pod가 Pending 상태에서 멈췄다.** IP 할당이 안 된다.

```
NAME       READY   STATUS    RESTARTS   AGE   IP       NODE
test-pod   0/1     Pending   0          30s   <none>   <none>
```

### VPC CNI 설정 불일치 문제

원인을 파헤쳐보니 **Network Policy 설정이 불일치**했다.

```bash
# ConfigMap 설정
kubectl get configmap amazon-vpc-cni -n kube-system -o yaml
# enable-network-policy-controller: "false"

# DaemonSet 환경변수
kubectl get daemonset aws-node -n kube-system -o yaml | grep NETWORK_POLICY
# NETWORK_POLICY_ENFORCING_MODE: standard
```

ConfigMap에서는 Network Policy를 끄라고 하고, DaemonSet env에서는 standard 모드로 켜라고 한다. VPC CNI가 혼란에 빠진 것이다.

**해결:**

```bash
# DaemonSet에서 충돌하는 env 제거
kubectl set env daemonset/aws-node -n kube-system NETWORK_POLICY_ENFORCING_MODE-
```

이후 aws-node Pod가 재시작되면서 정상화됐다.

```bash
kubectl run test-pod2 --image=busybox --restart=Never --command -- sleep 30
kubectl get pod test-pod2 -o wide
# IP 할당 성공!
```

### Step 3: MNG 노드 업그레이드 - 두 번째 삽질

```bash
aws eks update-nodegroup-version \
  --cluster-name eks-blue \
  --nodegroup-name on-demand \
  --kubernetes-version 1.33
```

에러가 났다.

```
An error occurred (InvalidParameterException) when calling the UpdateNodegroupVersion operation:
Nodegroup can't be upgraded because maxSize (2) is equal to desiredSize (2).
```

**Surge 업그레이드 방식이 막힌 것이다.** EKS는 노드 그룹 업그레이드 시 새 노드를 먼저 띄우고 기존 노드를 드레인하는 Surge 방식을 쓴다. 그런데 maxSize가 desiredSize와 같으면 새 노드를 띄울 공간이 없다.

**해결:**

```bash
# maxSize 임시 증가
aws eks update-nodegroup-config \
  --cluster-name eks-blue \
  --nodegroup-name on-demand \
  --scaling-config minSize=1,maxSize=4,desiredSize=2

# 이후 업그레이드 재시도
aws eks update-nodegroup-version \
  --cluster-name eks-blue \
  --nodegroup-name on-demand \
  --kubernetes-version 1.33
```

### Step 4: Karpenter 노드 업그레이드 - 세 번째 삽질

Karpenter 노드는 EC2NodeClass의 AMI Selector를 변경하면 Drift가 감지되어 자동으로 교체된다.

```bash
kubectl patch ec2nodeclass karpenter --type=merge \
  -p '{"spec":{"amiSelectorTerms":[{"name":"amazon-eks-node-al2023-x86_64-standard-1.33*"}]}}'
```

그런데 노드가 안 뜬다. Karpenter 로그를 확인했다.

```bash
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --tail=100
```

```
ERROR controller.provisioner unable to determine if instance type
"t3.medium" is allowed by the NodePool, error: unsupported nodeClassRef
kind, no corresponding implementation found, expected one of
[]schema.GroupVersionKind{schema.GroupVersionKind{...}}
```

**Karpenter CRD 버전 문제였다.** Karpenter v1.0.5에서 v1.5.0으로 업그레이드하면서 CRD가 변경됐는데, Helm 업그레이드 시 CRD가 자동으로 업데이트되지 않았다.

**해결:**

```bash
# CRD 수동 업데이트
KARPENTER_CRD="https://raw.githubusercontent.com/aws/karpenter/v1.5.0/pkg/apis/crds"

kubectl apply -f ${KARPENTER_CRD}/karpenter.sh_nodepools.yaml
kubectl apply -f ${KARPENTER_CRD}/karpenter.k8s.aws_ec2nodeclasses.yaml
kubectl apply -f ${KARPENTER_CRD}/karpenter.sh_nodeclaims.yaml

# Karpenter 재시작
kubectl rollout restart deployment/karpenter -n karpenter
```

이후 AMI Selector 변경이 정상 반영되고 노드가 교체됐다.

### Step 5: DD_HOSTNAME 누락 발견

업그레이드 후 Datadog 대시보드를 확인하는데 일부 노드에서 호스트 메트릭이 누락됐다.

원인: Datadog Agent가 `DD_HOSTNAME` 환경변수 없이 띄워졌다. 기존 노드에서는 설정되어 있었는데, 새 노드에서 Agent가 재시작되면서 값이 누락된 것이다.

Helm values를 확인해보니 `DD_HOSTNAME`이 `spec.nodeName`에서 가져오도록 설정되어 있지 않았다.

**해결 (Terraform에서 Helm values 수정):**

```yaml
agents:
  containers:
    agent:
      env:
        - name: DD_HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
```

Helm upgrade 후 정상화됐다.

### dev 업그레이드 총 소요 시간

| 단계 | 예상 | 실제 | 비고 |
|------|------|------|------|
| Control Plane | 10분 | 8분 | 정상 |
| Core Addons | 5분 | 25분 | VPC CNI 설정 불일치 |
| MNG Node Group | 10분 | 15분 | maxSize 문제 |
| Karpenter Nodes | 15분 | 25분 | CRD 버전 문제 |
| 검증 | 5분 | 15분 | DD_HOSTNAME 발견 |
| **총** | **45분** | **~90분** | |

## prod 환경 업그레이드

dev에서 삽질한 내용을 바탕으로 prod 업그레이드 전에 사전 작업을 진행했다.

### 사전 작업 (전날)

| 작업 | 내용 |
|------|------|
| DD_HOSTNAME 설정 | Datadog Helm values 수정 |
| Replicas HA | 핵심 워크로드 replicas 2 이상 확인 |
| PDB 설정 | 핵심 워크로드 PDB 확인 |
| MNG maxSize | maxSize > desiredSize 확보 |
| Karpenter 업그레이드 | v1.0.5 → v1.5.0, CRD 선행 적용 |

### prod 업그레이드 타임라인

| 단계 | 소요 시간 | 누적 |
|------|----------|------|
| Pre-check | 1분 | 1분 |
| Control Plane 업그레이드 | 8분 | 9분 |
| VPC CNI env 제거 + Addon 업그레이드 | 5분 | 14분 |
| Pod IP 할당 테스트 | 1분 | 15분 |
| MNG 노드 업그레이드 (Surge) | 9분 | 24분 |
| Karpenter 노드 교체 | 12분 | 36분 |
| 전체 검증 | 5분 | 41분 |

**총 41분.** dev에서 2시간 30분 걸린 것과 비교하면 극적인 차이다.

### 핵심 차이점

| 항목 | dev | prod |
|------|-----|------|
| VPC CNI env 제거 | 문제 발생 후 조치 | 업그레이드 전 선제 조치 |
| MNG maxSize | 변경 필요 (2→4) | 이미 충분 (5>3) |
| Karpenter CRD | 문제 발생 후 조치 | 사전에 적용 완료 |
| DD_HOSTNAME | 업그레이드 후 발견 | 사전에 적용 완료 |

## 업그레이드 체크리스트

dev 경험을 바탕으로 정리한 체크리스트다.

### Pre-flight Checks

```bash
# 1. 클러스터 상태
aws eks describe-cluster --name ${CLUSTER} \
  --query 'cluster.{version:version,status:status}'

# 2. 노드 상태
kubectl get nodes -o wide

# 3. 핵심 워크로드 replicas 확인
kubectl get deploy -A \
  -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,REPLICAS:.spec.replicas \
  | grep -v "^kube-"

# 4. PDB 확인
kubectl get pdb -A

# 5. MNG maxSize 확인 (Surge 업그레이드용)
aws eks describe-nodegroup \
  --cluster-name ${CLUSTER} \
  --nodegroup-name ${NG} \
  --query 'nodegroup.scalingConfig'
# maxSize > desiredSize 필요!

# 6. VPC CNI Network Policy 설정 확인
kubectl get cm amazon-vpc-cni -n kube-system -o yaml \
  | grep enable-network-policy
kubectl get ds aws-node -n kube-system -o yaml \
  | grep -A1 NETWORK_POLICY_ENFORCING_MODE
# 불일치하면 env 제거 필요
```

### Control Plane 업그레이드

```bash
aws eks update-cluster-version --name ${CLUSTER} --kubernetes-version 1.33

# 상태 확인 (UPDATING → ACTIVE)
watch "aws eks describe-cluster --name ${CLUSTER} --query 'cluster.status'"
```

### Core Addon 업그레이드 (Self-managed)

```bash
# ECR 베이스 URL
ECR=602401143452.dkr.ecr.ap-northeast-2.amazonaws.com

# 1. VPC CNI 충돌 env 제거 (필수!)
kubectl set env daemonset/aws-node -n kube-system NETWORK_POLICY_ENFORCING_MODE-

# 2. VPC CNI
kubectl set image daemonset/aws-node -n kube-system \
  aws-node=${ECR}/amazon-k8s-cni:v1.21.1

# 3. CoreDNS
kubectl set image deployment/coredns -n kube-system \
  coredns=${ECR}/eks/coredns:v1.12.4-eksbuild.1

# 4. kube-proxy
kubectl set image daemonset/kube-proxy -n kube-system \
  kube-proxy=${ECR}/eks/kube-proxy:v1.33.0-minimal-eksbuild.2

# 5. Pod IP 할당 테스트
kubectl run test-pod --image=busybox --restart=Never --command -- sleep 30
kubectl get pod test-pod -o wide  # IP 할당 확인
kubectl delete pod test-pod
```

### MNG 노드 업그레이드

```bash
aws eks update-nodegroup-version \
  --cluster-name ${CLUSTER} \
  --nodegroup-name ${NG} \
  --kubernetes-version 1.33

# 상태 확인
watch "kubectl get nodes -o wide"
```

### Karpenter 노드 업그레이드

```bash
# AMI Selector 변경으로 Drift 트리거
kubectl patch ec2nodeclass karpenter --type=merge \
  -p '{"spec":{"amiSelectorTerms":[{"name":"amazon-eks-node-al2023-x86_64-standard-1.33*"}]}}'

# Drift 감지 및 노드 교체 확인
watch "kubectl get nodes -l karpenter.sh/nodepool=default -o wide"
```

### Post-upgrade 검증

```bash
# 1. 모든 노드 버전 확인
kubectl get nodes -o wide

# 2. Non-running Pod 확인
kubectl get pods -A | grep -v Running | grep -v Completed

# 3. 핵심 서비스 헬스체크
curl -s https://api.example.com/health

# 4. Datadog 에러율 확인 (업그레이드 전후 비교)
# - APM 대시보드에서 에러율 급증 여부 확인
# - 로그에서 새로운 에러 패턴 확인
```

## 교훈

### 1. dev에서 충분히 삽질하라

prod에서 40분 만에 끝낼 수 있었던 건 dev에서 모든 문제를 미리 겪었기 때문이다. dev 업그레이드를 "빨리 끝내야 할 작업"이 아니라 "문제를 발견하는 기회"로 봐야 한다.

### 2. Self-managed Addon은 추가 주의

EKS managed addon이면 버전 호환성을 AWS가 관리해준다. Self-managed는 직접 이미지 버전을 맞춰야 하고, 설정 충돌도 직접 해결해야 한다.

### 3. 사전 작업이 본 작업보다 중요

| 사전 작업 | 이유 |
|----------|------|
| MNG maxSize 확보 | Surge 업그레이드 가능하게 |
| VPC CNI env 정리 | 설정 충돌 방지 |
| Karpenter CRD 업데이트 | NodeClass 인식 문제 방지 |
| DD_HOSTNAME 설정 | 모니터링 연속성 |
| Replicas/PDB 확인 | 무중단 보장 |

### 4. 시뮬레이션 사고방식

체크리스트를 따라가기만 하면 안 된다. 각 단계에서 "여기서 막히면?"을 시뮬레이션해야 한다.

- MNG 업그레이드 → maxSize가 모자라면?
- VPC CNI 업그레이드 → Pod IP 할당 안 되면?
- Karpenter AMI 변경 → 노드가 안 뜨면?

미리 시뮬레이션하면 문제를 예방할 수 있다.

### 5. AI 활용 - 대시보드 & 런북 자동 생성

이번 업그레이드에서 Claude Code를 활용해 **사전 점검 대시보드**와 **실행 런북**을 자동 생성했다.

#### EKS Upgrade Readiness Dashboard

dev에서 발견한 이슈들을 prod 사전 점검에 반영한 대시보드다. 한눈에 BLOCKER/WARNING/INSIGHTS를 파악할 수 있다.

| 섹션 | 내용 |
|------|------|
| BLOCKERS | Karpenter 버전, VPC CNI env 불일치 등 필수 조치 |
| WARNINGS | 주의 필요 항목 |
| INSIGHTS | EKS Upgrade Insights API 결과 |
| Dev 경험 반영 | dev에서 발견한 이슈 → prod 체크 항목으로 |

**핵심 기능:**
- 클러스터 상태 자동 조회 (AWS CLI, kubectl)
- dev 이슈 → prod BLOCKER로 자동 연결
- 해결 여부 실시간 표시 (검증됨 ✓)

#### EKS Upgrade Runbook

실제 작업 시 따라갈 수 있는 단계별 런북이다. 작업 개요, BLOCKER 사전 작업, 단계별 체크리스트, 롤백 절차가 포함된다.

| 섹션 | 내용 |
|------|------|
| 작업 개요 | 날짜, 예상 소요시간, 서비스 영향, Jira 링크 |
| BLOCKER 사전 작업 | dev 경험 기반 필수 선행 작업 |
| 단계별 체크리스트 | Control Plane → Addons → MNG → Karpenter |
| 롤백 절차 | 문제 발생 시 복구 방법 |

**활용 방식:**
1. AI가 클러스터 상태 조회 → 대시보드 생성
2. dev 이슈 분석 → BLOCKER 항목 도출
3. 런북 생성 → 팀 공유 및 작업 시 참조
4. 작업 완료 후 → 블로그 글 자동 생성

prod 업그레이드가 41분 만에 끝난 건 이런 사전 준비 덕분이다. AI를 "코드 작성 도구"가 아니라 **"운영 지식 정리 도구"**로 활용한 사례다.

## 버전 정보

최종 버전 (prod):

| 구성요소 | 이전 | 이후 |
|---------|------|------|
| Control Plane | 1.32 | 1.33 |
| VPC CNI | v1.19.2 | v1.21.1 |
| CoreDNS | v1.11.4 | v1.12.4 |
| kube-proxy | v1.32.0 | v1.33.0 |
| Node (MNG) | v1.32.3 | v1.33.5 |
| Node (Karpenter) | v1.32.3 | v1.33.5 |
| Karpenter | v1.0.5 | v1.5.0 |

## 마무리

EKS 업그레이드는 단순해 보이지만, Self-managed 환경에서는 숨은 복잡성이 많다. VPC CNI 설정 충돌, Karpenter CRD 호환성, 모니터링 연속성 등 예상치 못한 곳에서 문제가 터진다.

dev에서 충분히 삽질하고, 발견한 문제를 사전 작업으로 정리하면 prod는 순조롭게 진행할 수 있다. 이번 업그레이드에서 가장 큰 교훈이다.
