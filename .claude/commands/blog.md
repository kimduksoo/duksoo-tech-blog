# 블로그 글 작성

사용자가 제공한 주제로 블로그 글을 작성한다.

## 입력
$ARGUMENTS - 블로그 글 주제 (예: "EKS 1.29 업그레이드 경험담")

## 작성 규칙

1. **카테고리 선택**: 주제에 맞는 카테고리 폴더에 파일 생성
   - `content/kubernetes/` - EKS, K8s 관련
   - `content/aws/` - AWS 아키텍처, 인프라
   - `content/observability/` - Datadog, LGTM, 모니터링

2. **파일명**: 영문 kebab-case (예: `eks-129-upgrade.md`)

3. **Front Matter** (YAML 형식):
```yaml
---
title: "제목"
weight: 10
---
```

4. **본문 구조**:
   - 개요/배경
   - 본문 (단계별 또는 섹션별)
   - 코드 블록은 언어 명시 (```bash, ```yaml 등)
   - Mermaid 다이어그램 활용 (```mermaid)
   - 결론/정리

5. **작성 완료 후**:
   - git add, commit, push 실행
   - 배포 URL 안내: https://duksoo-tech-blog.pages.dev

## 예시

주제: "EKS 1.29 업그레이드 경험담"
→ 파일: `content/kubernetes/eks-129-upgrade.md`
→ 카테고리: Kubernetes
