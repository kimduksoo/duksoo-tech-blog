---
title: "AI 인프라 자동화 #2 혼자 쓰던 도구를 팀 도구로"
weight: 2
description: "채널 기반 라우팅으로 구조를 단순화하고, 웹 콘솔을 추가해서 팀 전체가 에이전트 현황을 볼 수 있게 만든 이야기."
tags: ["AI", "Claude", "Agent SDK", "MCP", "인프라자동화", "Slack"]
keywords: ["AI 인프라 자동화", "채널 기반 라우팅", "웹 콘솔", "Slack 자동화"]
---

[1편](/ai/ai-infra-automation/)에서는 Claude Agent SDK와 MCP로 Slack 기반 인프라 자동화를 만들었다. 비용 분석, 파라미터 스토어 등록 같은 반복 업무를 AI 에이전트에 맡기는 구조였다.

잘 돌아갔다. 그런데 한 가지 문제가 있었다.

<span style="color:#1565c0; font-weight:bold">나만 쓸 수 있었다.</span>

---

## 혼자 돌리기엔 충분했다

1편의 시스템 구조를 다시 보면 이렇다.

```mermaid
flowchart TB
    subgraph 트리거
        Cron[스케줄러<br/>APScheduler]
        Event[Slack 이벤트<br/>Socket Mode]
    end

    subgraph 코어
        subgraph 트리아지
            KW[키워드 필터] --> Haiku[Haiku 3.5<br/>분류 + 에이전트 선택]
        end
        Pre[Python 전처리] --> Agent[에이전트<br/>Claude Agent SDK]
        Haiku --> Agent
    end

    subgraph 에이전트들
        A1[infra_request]
        A2[param_store]
        A3[k8s_service]
        A4[general_chat]
    end

    subgraph 외부도구[" "]
        MCP[MCP 서버<br/>Slack · Jira · Datadog]
        CLI[AWS CLI · kubectl]
    end

    Cron --> Pre
    Event --> KW
    Agent --> 에이전트들
    Agent --> MCP
    Agent --> CLI

    style 트리거 fill:#e8f4fd,stroke:#4a90d9
    style 코어 fill:#e8f8e8,stroke:#5ba85b
    style 트리아지 fill:#fef3c7,stroke:#f59e0b
    style 에이전트들 fill:#e8d5f5,stroke:#7b2fbe
    style 외부도구 fill:none,stroke:none
```

메시지가 들어오면 키워드 필터를 통과하고, Haiku가 분류해서 적절한 에이전트를 선택한다. 4가지 트리거(report, mention, triage, usergroup)가 각각 다른 코드 경로를 탄다.

동작하는 데는 문제가 없었다. 문제는 다른 곳에 있었다.

### 사용자 입장

다른 팀원이 인프라 요청을 하려면:

- **어디에 써야 하는지** 알아야 한다 — 특정 채널에서만 동작
- **어떻게 써야 하는지** 알아야 한다 — 키워드가 없으면 필터에서 무시됨
- **뭘 하고 있는지** 알 수 없다 — 에이전트가 작업 중인지, 승인 대기인지, 완료인지 Slack 스레드를 직접 따라가야 확인 가능

### 개발자 입장

새 채널이나 에이전트를 추가하려면:

| 파일 | 수정 내용 |
|------|----------|
| `event_triggers.yaml` | 트리거 설정 추가 |
| `slack_listener.py` | 라우팅 분기 추가 |
| `triage.py` | 키워드 목록 + Haiku 프롬프트 수정 |

3개 파일을 건드려야 했다. 키워드 목록은 점점 길어지고, 엣지케이스마다 예외를 추가해야 했다. 동작하지만 유지보수가 부담스러운 구조였다.

팀 도구가 되려면 두 가지가 필요했다.

1. **진입장벽을 낮추는 것** — 사용자가 형식을 신경 쓰지 않아도 되게
2. **투명성을 확보하는 것** — 누구나 에이전트 현황을 볼 수 있게

---

## 첫 번째: 채널 기반 라우팅

### 채널이 곧 의도다

#t_인프라 채널에 글을 쓰는 사람은 이미 인프라 관련 이야기를 하려는 의도가 있다. 굳이 키워드로 분류할 필요가 없다.

1편에서 <span style="color:#1565c0; font-weight:bold">"계산은 코드가 해야 한다"</span>는 교훈을 얻었다면, 이번에 얻은 교훈은 <span style="color:#1565c0; font-weight:bold">"라우팅은 구조가 해야 한다"</span>는 것이다.

키워드 필터와 Haiku 분류기가 하던 일을 채널 구조 자체가 대신한다.

```mermaid
flowchart LR
    Msg[Slack 메시지] --> CH{채널 등록?}
    CH -->|미등록| Drop[무시]
    CH -->|등록| Type{멘션?}
    Type -->|@멘션| Agent[채널 전용 에이전트]
    Type -->|일반| GK{Haiku 게이트키퍼}
    GK -->|처리| Agent
    GK -->|무시| Drop2[무시]

    style Msg fill:#dbeafe,stroke:#3b82f6,color:#1e3a5f
    style CH fill:#fef3c7,stroke:#f59e0b,color:#78350f
    style Type fill:#fef3c7,stroke:#f59e0b,color:#78350f
    style GK fill:#fef3c7,stroke:#f59e0b,color:#78350f
    style Agent fill:#d1fae5,stroke:#10b981,color:#064e3b
    style Drop fill:#f3f4f6,stroke:#9ca3af,color:#4b5563
    style Drop2 fill:#f3f4f6,stroke:#9ca3af,color:#4b5563
```

@멘션은 게이트키퍼를 거치지 않고 바로 실행된다. 명시적으로 에이전트를 호출했으니 판단할 필요가 없다. 일반 메시지만 Haiku가 "이걸 처리해야 하는가?"를 판단한다.

### 설정 변화

기존에는 키워드 목록, 트리아지 규칙, 유저그룹 패턴 등 여러 설정이 흩어져 있었다.

```yaml
# Before: 흩어진 설정들
mention_triggers:
  keywords: ["NAT", "파라미터", "배포", ...]
  haiku_prompt: "분류하고 에이전트를 선택하세요..."

triage_triggers:
  channels: [...]
  keywords: [...]

usergroup_triggers:
  patterns: [...]
```

지금은 이것이 전부다.

```yaml
# After: 채널 기반 설정
channel_triggers:
  enabled: true
  haiku_model: "claude-haiku-4-5-20251001"
  channels:
    - channel_id: "C06PW1CTU5B"
      name: "t_인프라"
      default_agent: infra_request
    - channel_id: "C09BW8WJVD1"
      name: "웹훅-테스트"
      default_agent: infra_request
```

채널을 추가하고 싶으면 `channels` 배열에 항목 하나를 추가하면 된다. 코드 수정이 필요 없다.

### Haiku의 역할 변화

Haiku의 역할이 축소됐다. 이전에는 메시지를 분류하고 에이전트까지 선택했지만, 이제는 <span style="color:#1565c0; font-weight:bold">"이 메시지를 처리해야 하는가?"</span> 한 가지만 판단한다.

```
"서버 NAT IP 알려주세요" → 처리 (인프라 요청)
"점심 뭐 먹지"           → 무시 (일상 대화)
"참고로 공유합니다"       → 무시 (FYI)
```

에이전트 선택이 사라진 이유는 채널이 이미 에이전트를 결정하기 때문이다. #t_인프라에 올라온 메시지는 infra_request 에이전트가 처리한다. Haiku가 고민할 필요가 없다.

### 삭제한 코드, 추가한 코드

| 삭제 | 역할 |
|------|------|
| `TriageClassifier` | 키워드 매칭 + Haiku 2단계 분류 |
| `UsergroupJudge` | 유저그룹 멘션 감지 + 판단 |
| `_route_to_agent()` | 키워드 기반 에이전트 라우팅 |
| `_try_triage()` | 트리아지 채널 처리 |
| `_detect_usergroup_mention()` | 유저그룹 패턴 감지 |

| 추가 | 역할 |
|------|------|
| `ChannelGatekeeper` | 처리 여부 판단 (단일 클래스) |
| `_try_channel_route()` | 채널 라우팅 (단일 메서드) |

5개 삭제하고 2개 추가했다. 코드가 줄었고, 각 구성요소의 책임이 명확해졌다.

### 승인 게이트 변경

1편에서는 @infra 그룹에 승인을 요청했다. 그룹 멘션은 누가 봤는지 알 수 없고, 책임이 분산된다.

지금은 개별 승인자에게 직접 멘션한다. 요청이 들어오면 즉시 Jira 티켓도 생성된다.

```mermaid
sequenceDiagram
    participant U as 사용자
    participant A as 에이전트
    participant J as Jira
    participant Ap as 승인자 (개인 멘션)

    U->>A: 변경 요청
    A->>J: 즉시 티켓 생성
    A->>Ap: 개별 멘션으로 승인 요청
    Note over A: 에이전트 정지 (대기)
    Ap->>A: "승인"
    A->>A: 작업 실행
    A->>J: 결과 댓글
    A->>U: 완료 보고

    %%{init: {'theme': 'default'}}%%
```

조회(R)는 승인 없이 바로 실행되고, 생성/수정/삭제(CUD)만 승인을 거친다.

---

## 두 번째: 웹 콘솔

### Slack만으로는 부족한 이유

Slack은 좋은 인터페이스지만 한계가 있다.

- **채널에 있어야 볼 수 있다** — 해당 채널에 없는 팀원은 에이전트 활동을 모른다
- **스레드를 따라가야 한다** — 에이전트가 작업 중인지, 승인 대기인지, 완료인지 스레드를 열어봐야 안다
- **전체 현황 파악이 어렵다** — 지금 진행 중인 요청이 몇 개인지, 누가 뭘 요청했는지 한눈에 안 보인다

에이전트가 나 혼자 쓰는 도구였을 때는 괜찮았다. 내가 요청하고, 내가 스레드를 확인하면 된다. 하지만 팀 도구가 되면 이야기가 달라진다. 인프라팀 3명이 각각 다른 채널에서 요청을 하고 있을 때, 전체 현황을 한 곳에서 보고 싶다.

웹 콘솔은 이 문제를 해결한다.

```mermaid
flowchart LR
    subgraph Slack
        S1[#t_인프라]
        S2[#웹훅-테스트]
    end

    subgraph 상태관리[StateManager]
        SM[요청 상태]
        WS[WebSocket]
    end

    subgraph 웹콘솔[웹 콘솔]
        List[요청 목록]
        Detail[상세 대화]
        Status[상태 표시]
    end

    S1 --> SM
    S2 --> SM
    SM --> WS
    WS --> List
    WS --> Detail
    WS --> Status

    style Slack fill:#e8f4fd,stroke:#4a90d9
    style 상태관리 fill:#e8f8e8,stroke:#5ba85b
    style 웹콘솔 fill:#e8d5f5,stroke:#7b2fbe
```

### Slack 스레드와 실시간 동기화

웹 콘솔의 핵심은 Slack 스레드와의 실시간 동기화다. Slack에서 대화가 오가면 웹에도 그대로 반영된다.

```mermaid
sequenceDiagram
    participant S as Slack 스레드
    participant SM as StateManager
    participant WS as WebSocket
    participant W as 웹 콘솔

    S->>SM: 사용자 메시지
    SM->>WS: broadcast(new_message)
    WS->>W: 메시지 표시

    SM->>WS: typing_start
    WS->>W: "Thinking..." 인디케이터
    Note over SM: 에이전트 실행 중
    SM->>WS: typing_stop
    WS->>W: 인디케이터 해제

    SM->>WS: broadcast(new_message)
    WS->>W: 에이전트 응답 표시

    S->>SM: 사용자 후속 답글
    SM->>WS: broadcast(new_message)
    WS->>W: 후속 답글 표시
```

에이전트가 생각하는 동안은 Thinking 인디케이터가 표시된다. 채널에 없는 팀원도 웹 콘솔에서 진행 상황을 확인하고, 직접 메시지를 보낼 수도 있다.

### 요청 생명주기

모든 요청은 웹 콘솔에서 상태별로 관리된다.

```mermaid
stateDiagram-v2
    [*] --> 접수: Slack 메시지 수신
    접수 --> 처리중: 에이전트 실행
    처리중 --> 승인대기: 변경 작업 감지
    승인대기 --> 처리중: 승인
    처리중 --> 완료: 작업 완료
    처리중 --> 대화중: 추가 정보 필요
    대화중 --> 처리중: 답변 수신
    대화중 --> 완료: 완료 확인
```

요청이 들어오면 즉시 Jira 티켓이 생성되고, 웹 콘솔에 카드가 추가된다. 에이전트 실행, 승인 대기, 추가 대화, 완료까지 전체 흐름이 웹에서 추적된다. Slack 채널에 없어도 "지금 어떤 요청이 처리되고 있는지"를 알 수 있다.

---

## 변경 전후 비교

```mermaid
flowchart TB
    subgraph Before[기존 구조]
        direction TB
        E1[Slack 이벤트] --> KW1[키워드 필터]
        KW1 --> H1[Haiku<br/>분류 + 선택]
        H1 --> AG1[infra_request]
        H1 --> AG2[param_store]
        H1 --> AG3[k8s_service]
    end

    subgraph After[현재 구조]
        direction TB
        E2[Slack 이벤트] --> CH2{채널 매핑}
        CH2 --> GK2[Haiku<br/>게이트키퍼]
        GK2 --> AG4[채널 전용 에이전트]
        AG4 --> J2[Jira 연동]
        AG4 --> WS2[웹 콘솔 동기화]
    end

    style Before fill:#fff5f5,stroke:#f87171
    style After fill:#f0fdf4,stroke:#4ade80
```

| | Before | After |
|---|---|---|
| 메시지 라우팅 | 4가지 트리거, 키워드 필터 | 채널 1:1 매핑 |
| Haiku 역할 | 분류 + 에이전트 선택 | 게이트키퍼 (처리 여부만) |
| 에이전트 | 기능별 3~4개 | 채널당 1개 |
| 승인 | @infra 그룹 | 개별 승인자 멘션 |
| Jira | 수동 | 요청 시 즉시 자동 생성 |
| 가시성 | Slack 스레드만 | Slack + 웹 콘솔 |
| 채널 추가 | 3개 파일 수정 | yaml 3줄 추가 |

---

## 아직 남은 것들

### 승인 게이트의 한계

현재 승인 게이트는 프롬프트에 의존한다. "변경 작업 전에 승인을 받아라"라고 프롬프트에 적어놓은 것이다. 대부분 잘 동작하지만, LLM이 판단을 잘못하면 승인 없이 실행될 가능성이 있다.

다음 단계는 SDK 레벨에서 승인을 강제하는 것이다. 특정 도구(AWS CLI write, kubectl apply 등)를 호출하기 전에 코드 레벨에서 승인 상태를 확인하는 구조다. 프롬프트가 아니라 코드가 게이트를 잡아야 한다.

### 에이전트 프롬프트 품질

채널을 추가하는 것은 yaml 3줄이면 된다. 하지만 그 채널의 에이전트가 좋은 품질로 동작하려면 프롬프트를 잘 작성해야 한다. 현재 infra_request 에이전트의 프롬프트는 수십 번의 테스트를 거쳐 다듬어진 것이다. 새 에이전트를 추가할 때마다 같은 반복이 필요하다.

### 권한 체계

현재는 허용된 사용자 목록으로 접근을 제어한다. 역할별로 요청 가능한 범위를 제한하는 것도 고려하고 있다. 개발자는 조회만, 인프라팀은 변경 요청 가능한 구조다.

---

## 마무리

1편이 "AI 에이전트가 일을 할 수 있는가?"에 대한 답이었다면, 이번 글은 "팀이 함께 쓸 수 있는가?"에 대한 답이다.

기술적으로 어려운 일은 아니었다. 채널 매핑은 dictionary lookup이고, 웹 콘솔은 WebSocket broadcast다. 어려운 것은 <span style="color:#1565c0; font-weight:bold">"어디를 단순화해야 사용자가 편해지는가"</span>를 찾는 일이었다.

복잡한 라우팅을 걷어내니 채널 추가가 쉬워졌고, 눈에 보이게 만드니 다른 사람도 쓸 수 있게 됐다.
