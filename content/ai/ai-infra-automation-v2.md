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

다른 팀원이 인프라 요청을 하려면 특정 채널에서 특정 키워드를 포함해서 글을 써야 했다. "NAT IP 알려주세요"는 반응하지만 "외부에서 접근하려는데 IP가 뭐야?"는 키워드 필터에서 무시된다. 에이전트가 지금 뭘 하고 있는지도 Slack 스레드를 직접 따라가야 알 수 있었다.

개발자 입장에서도 부담이 있었다. 새 채널이나 에이전트를 추가하려면 `event_triggers.yaml`, `slack_listener.py`, `triage.py` 3개 파일을 건드려야 했고, 키워드 목록은 점점 길어졌다.

팀 도구가 되려면 두 가지가 필요했다.

1. **진입장벽을 낮추는 것** — 키워드를 외우지 않아도 되게
2. **투명성을 확보하는 것** — 누구나 에이전트 현황을 볼 수 있게

---

## 첫 번째: 진입장벽 낮추기

### 채널이 곧 의도다

핵심 인사이트는 단순하다. #t_인프라 채널에 글을 쓰는 사람은 이미 인프라 관련 이야기를 하려는 의도가 있다. 굳이 키워드로 분류할 필요가 없다.

1편에서 <span style="color:#1565c0; font-weight:bold">결정론적 작업과 비결정론적 작업을 분리</span>한 것처럼, 이번에는 라우팅을 LLM 판단에서 구조로 옮겼다. 채널 자체가 라우팅이 된다.

```mermaid
flowchart LR
    subgraph After[채널 라우팅]
        Msg2[메시지] --> CH{채널 등록?}
        CH -->|미등록| Drop3[무시]
        CH -->|등록| GK{Haiku 게이트키퍼}
        GK -->|무시| Drop4[무시]
        GK -->|처리| Agent[채널 전용 에이전트]
    end

    style After fill:#f0fdf4,stroke:#4ade80
```

Haiku의 역할도 바뀌었다. 이전에는 "분류 + 에이전트 선택"을 했지만, 지금은 <span style="color:#1565c0; font-weight:bold">"이 메시지를 처리해야 하는가?"</span> 한 가지만 판단한다. 에이전트 선택은 채널이 이미 결정했기 때문이다.

```
"서버 NAT IP 알려주세요" → 처리 (인프라 요청)
"점심 뭐 먹지"           → 무시 (일상 대화)
"참고로 공유합니다"       → 무시 (FYI)
```

@멘션의 경우 게이트키퍼를 거치지 않고 바로 실행된다. 명시적으로 에이전트를 호출했으니 판단할 필요가 없다.

결과적으로 사용자가 해야 하는 것이 줄었다.

| | Before | After |
|---|---|---|
| 요청 방법 | @멘션 + 키워드 포함해서 글쓰기 | 채널에 그냥 글쓰기 |
| 반응 조건 | 키워드가 매칭되어야 반응 | 채널에 등록되어 있으면 반응 |
| 형식 제약 | "NAT", "파라미터" 등 특정 단어 필요 | 자연어 그대로 |

### 설정도 단순해졌다

채널을 추가하는 데 필요한 설정도 바뀌었다. 기존에는 3개 파일에 흩어진 키워드와 트리거를 관리해야 했지만, 지금은 yaml에 항목 하나만 추가하면 된다.

```yaml
channel_triggers:
  channels:
    - channel_id: "C06PW1CTU5B"
      name: "t_인프라"
      default_agent: infra_request
    - channel_id: "C09BW8WJVD1"
      name: "웹훅-테스트"
      default_agent: infra_request
```

코드 수정이 필요 없다.

---

## 두 번째: 투명성 확보

Slack은 좋은 인터페이스지만 한계가 있다. 해당 채널에 없는 팀원은 에이전트 활동을 모르고, 전체 현황을 보려면 채널마다 스레드를 열어봐야 한다.

혼자 쓸 때는 괜찮았다. 하지만 팀 도구가 되면 "지금 진행 중인 요청이 몇 개인지, 누가 뭘 요청했는지" 한 곳에서 보고 싶어진다.

웹 콘솔을 만들었다. 모든 요청의 상태를 한눈에 보여주는 대시보드다.

![웹 콘솔 랜딩 페이지 - 요청 목록과 상태별 분류](/images/ai-infra-automation/webconsole-landing.png)

요청을 선택하면 Slack 스레드의 대화가 그대로 보인다. 에이전트 응답, 사용자 답글, 승인 요청까지 전부 웹에서 확인할 수 있다.

![웹 콘솔 상세 화면 - Slack 스레드 대화가 실시간으로 동기화된다](/images/ai-infra-automation/webconsole-detail.png)

### 실시간 동기화

웹 콘솔의 핵심은 Slack 스레드와의 실시간 동기화다. Slack에서 메시지가 오가면 WebSocket을 통해 웹에도 즉시 반영된다.

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
```

에이전트가 생각하는 동안은 Thinking 인디케이터가 표시된다. 채널에 없는 팀원도 웹 콘솔에서 진행 상황을 확인할 수 있다.

요청이 들어오면 즉시 Jira 티켓이 생성되고, 웹 콘솔에 카드가 추가된다. 에이전트 실행부터 승인 대기, 완료까지 전체 흐름이 웹에서 추적된다.

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
