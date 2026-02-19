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
   - **Mermaid 다이어그램 적극 활용** (```mermaid) — 아키텍처, 흐름도, 비교 등 시각화 가능한 내용은 Mermaid로 표현
   - 결론/정리

5. **회사 정보 비식별화**:
   - 회사명, AWS 계정 ID, 내부 도메인, Jira 티켓 번호 등 회사 식별 가능한 정보는 블로그에 노출하지 않음
   - 예: `047719655696` → 제거, `ajdcorp.com` → `example.corp`, 팀명/서비스명은 일반화

5. **작성 완료 후**:
   - git add, commit, push 실행
   - 배포 URL 안내: https://duksoo-tech-blog.pages.dev

## 예시

주제: "EKS 1.29 업그레이드 경험담"
→ 파일: `content/kubernetes/eks-129-upgrade.md`
→ 카테고리: Kubernetes
