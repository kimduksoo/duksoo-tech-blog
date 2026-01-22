---
title: "테스트 포스트 - 블로그 파이프라인 검증"
weight: 99
---

이 글은 블로그 배포 파이프라인 테스트용 포스트입니다.

## 테스트 항목

- [x] Markdown 렌더링
- [x] Front Matter 파싱
- [x] Mermaid 다이어그램

## 샘플 다이어그램

```mermaid
flowchart LR
    A[글 작성] --> B[Git Push]
    B --> C[Cloudflare Pages]
    C --> D[배포 완료]
```

## 샘플 코드 블록

```bash
echo "Hello, Blog!"
```

## 결론

테스트 완료.
