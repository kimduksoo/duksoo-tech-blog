# duksoo.dev

Cloud Engineer의 기술 블로그 소스. [duksoo-tech-blog.pages.dev](https://duksoo-tech-blog.pages.dev/)로 배포된다.

## Stack

- **SSG**: Hugo
- **Theme**: [hugo-geekdoc](https://github.com/thegeeklab/hugo-geekdoc) (submodule)
- **Hosting**: Cloudflare Pages (main 브랜치 push 시 자동 배포)

## 구조

| 경로 | 용도 |
|------|------|
| `content/` | 글 원본 (카테고리별 디렉토리) |
| `static/` | 이미지/아이콘 등 정적 파일 |
| `layouts/` | Hugo 템플릿 커스텀 |
| `hugo.toml` | Hugo 설정 |
| `themes/hugo-geekdoc/` | 테마 (submodule) |

### 카테고리

- `ai/` — AI/자동화
- `aws/` — AWS 서비스
- `backend/` — 백엔드 이슈/경험
- `finops/` — 비용 최적화
- `kubernetes/` — K8s/EKS
- `observability/` — 모니터링/관측성
- `security/` — 보안

## 로컬 실행

```bash
# 테마 submodule 포함해서 clone
git clone --recurse-submodules https://github.com/kimduksoo/duksoo-tech-blog.git
cd duksoo-tech-blog

# 개발 서버
hugo server -D

# 빌드
hugo
```

## 배포

main 브랜치에 push하면 Cloudflare Pages가 자동으로 빌드 후 배포한다.

```bash
git add .
git commit -m "post: <글 제목>"
git push origin main
```

## 새 글 작성

```bash
hugo new content/kubernetes/my-new-post.md
```
