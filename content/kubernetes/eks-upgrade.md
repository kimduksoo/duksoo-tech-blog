---
title: "EKS 1.28 → 1.29 업그레이드 후기"
weight: 1
---

프로덕션 EKS 클러스터를 1.28에서 1.29로 업그레이드한 경험을 공유합니다.

## 업그레이드 프로세스

```mermaid
flowchart LR
    A[사전 점검] --> B[Add-on 호환성 확인]
    B --> C[Control Plane 업그레이드]
    C --> D[Node Group 업그레이드]
    D --> E[Add-on 업그레이드]
    E --> F[검증]
```

## 사전 점검 사항

### 1. Deprecated API 확인

```bash
# pluto로 deprecated API 스캔
pluto detect-helm -o wide
pluto detect-files -d manifests/
```

### 2. Add-on 호환성 매트릭스

| Add-on | 현재 버전 | 권장 버전 |
|--------|----------|----------|
| VPC CNI | v1.15.1 | v1.16.0 |
| CoreDNS | v1.10.1 | v1.11.1 |
| kube-proxy | v1.28.2 | v1.29.0 |
| EBS CSI | v1.25.0 | v1.28.0 |

## 업그레이드 실행

### Control Plane

```bash
aws eks update-cluster-version \
  --name my-cluster \
  --kubernetes-version 1.29
```

### Node Group (Blue-Green 방식)

```mermaid
sequenceDiagram
    participant Old as Old NodeGroup (1.28)
    participant New as New NodeGroup (1.29)
    participant Pods as Workloads

    Note over New: 새 노드그룹 생성 (1.29)
    Pods->>New: Pod 스케줄링
    Note over Old: Cordon & Drain
    Pods-->>Old: Pod 이전
    Note over Old: 기존 노드그룹 삭제
```

## 교훈

1. **Blue-Green NodeGroup**: 롤백이 쉽고 안전함
2. **Add-on 순서**: metrics-server → CoreDNS → kube-proxy → CNI
3. **충분한 테스트**: staging 환경에서 최소 1주일 검증
