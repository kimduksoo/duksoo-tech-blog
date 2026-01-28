---
title: "Claude Agent SDK vs n8n: 업무 자동화 도구 비교"
weight: 1
draft: true
description: "슬랙 자동화를 위한 Claude Agent SDK와 n8n 비교. 개발자 관점에서 어떤 도구가 더 적합한지 정리."
tags: ["자동화", "Claude", "n8n", "AI", "슬랙"]
keywords: ["Claude Agent SDK", "n8n", "workflow automation", "AI agent", "slack bot"]
---

슬랙 채널 모니터링하고 자동으로 답변 다는 시스템을 만들려고 한다. 두 가지 선택지가 있다.

1. **Claude Agent SDK** - Anthropic의 AI 에이전트 프레임워크
2. **n8n** - 오픈소스 워크플로우 자동화 도구

둘 다 써보고 비교한 내용을 정리한다.

## 요구사항

| 기능 | 설명 |
|------|------|
| 슬랙 채널 모니터링 | 특정 채널 메시지 감지 |
| AI 분석/처리 | 메시지 내용 이해 및 분석 |
| 자동 답변 | 스레드에 댓글 작성 |
| Git 관리 | 버전 관리, 코드 리뷰, 롤백 |

## 기본 비교

| 항목 | Claude Agent SDK | n8n |
|------|-----------------|-----|
| 유형 | 코드 기반 프레임워크 | GUI 기반 워크플로우 |
| 언어 | Python | JavaScript (노드) |
| AI 통합 | Claude 네이티브 | 플러그인 (Claude, GPT 등) |
| 러닝커브 | 코드 작성 필요 | 낮음 (드래그앤드롭) |
| 유연성 | 매우 높음 | 중간 |

## 비용 비교

### Claude Agent SDK

| 환경 | 비용 |
|------|------|
| 로컬 (데스크탑/노트북) | **구독 포함** (Pro/Max) |
| 서버 (헤드리스) | API 과금 |

로컬에서 Claude Code 로그인 상태로 실행하면 구독비에 포함된다. 서버 배포 시에는 `ANTHROPIC_API_KEY`가 필요하고 사용량에 따라 비용이 발생한다.

### n8n

| 환경 | 비용 |
|------|------|
| 셀프호스팅 | **무료** |
| 클라우드 | 유료 플랜 |
| AI 노드 (Claude) | API 과금 별도 |

n8n 자체는 셀프호스팅하면 무료다. 단, Claude AI 노드를 사용하면 Anthropic API 비용이 발생한다.

## Git 버전 관리

### Claude Agent SDK

```
agent-scheduler/
├── src/*.py           # 코드 → git ✅
├── config/
│   ├── agents.yaml    # 에이전트 정의 → git ✅
│   └── schedules.yaml # 스케줄 정의 → git ✅
└── .env               # 시크릿 → .gitignore
```

코드 기반이라 자연스럽게 git 관리 가능. PR, 코드 리뷰, diff 확인이 쉽다.

### n8n

| 플랜 | Git 지원 |
|------|---------|
| Self-hosted (무료) | ❌ 기본 없음 |
| Business (€667/월) | ✅ 내장 |

무료 버전에서는 워크플로우 JSON을 export해서 수동 커밋하거나, 별도 툴(n8n2git 등)을 사용해야 한다.

## 개발 경험

### Claude Agent SDK

```yaml
# config/agents.yaml
agents:
  slack_bot:
    name: "슬랙 자동 답변 봇"
    mcp_servers:
      - slack
    allowed_tools:
      - "mcp__slack__slack_get_channel_history"
      - "mcp__slack__slack_reply_to_thread"
    max_turns: 20
```

YAML로 에이전트 정의하고, Python으로 로직 작성. 개발자에게 익숙한 방식.

### n8n

GUI에서 노드 연결:

```
[Slack Trigger] → [Claude AI] → [Slack Send Message]
```

비개발자도 쉽게 이해할 수 있다. 하지만 복잡한 로직은 코드 노드가 필요.

## 장단점 정리

### Claude Agent SDK

**장점:**
- 코드로 세밀한 제어 가능
- Git 관리 자연스러움
- MCP 서버 연동 강력
- 로컬 실행 시 구독 포함

**단점:**
- 코딩 필요
- 서버 배포 시 API 비용
- 비개발자 협업 어려움

### n8n

**장점:**
- GUI로 빠른 프로토타이핑
- 비개발자 협업 쉬움
- 셀프호스팅 무료
- 다양한 통합 노드

**단점:**
- 복잡한 로직은 한계
- Git 관리 불편 (무료 버전)
- AI 노드 사용 시 별도 API 비용

## 어떤 걸 선택할까

| 상황 | 추천 |
|------|------|
| 개발자, 코드 선호 | **Claude Agent SDK** |
| 빠른 프로토타입 | **n8n** |
| 비개발자 협업 | **n8n** |
| Git 관리 중요 | **Claude Agent SDK** |
| 서버 24시간 운영 + 비용 민감 | **n8n** (셀프호스팅) |
| 로컬에서 가끔 실행 | **Claude Agent SDK** |

## 내 결론

(TODO: 직접 써보고 결론 작성)

현재 상황:
- 이미 Claude Agent SDK로 프로젝트 구축해둠
- 개발자라 코드가 편함
- Git 관리 원함
- 로컬에서 실행하면 비용 없음

n8n을 직접 써보고 비교한 뒤 최종 결론을 내릴 예정.

## 참고

- [Claude Agent SDK 문서](https://platform.claude.com/docs/en/agent-sdk/overview)
- [n8n 공식 문서](https://docs.n8n.io/)
- [n8n Source Control](https://docs.n8n.io/source-control-environments/)
