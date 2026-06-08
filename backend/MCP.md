# MCP (Model Context Protocol)

## 1. MCP란?

**MCP(Model Context Protocol)**는 AI 모델(LLM)과 외부 도구(Tool), 데이터 소스(Data Source)를 연결하기 위한 표준 통신 프로토콜이다.

쉽게 말해, AI가 데이터베이스, 파일 시스템, API, 사내 시스템 등 외부 자원과 통신할 때 사용하는 **공통 규격**이다.

Anthropic이 2024년 11월 공개했으며, AI 애플리케이션에서 USB와 같은 역할을 목표로 한다.

---

## 2. MCP가 등장한 이유

과거에는 AI 모델마다 외부 도구를 연결하는 방식이 달랐다.

예를 들어:

* OpenAI용 DB 연동 코드
* Claude용 DB 연동 코드
* Gemini용 DB 연동 코드

를 각각 따로 작성해야 했다.

이 방식은 다음과 같은 문제가 있었다.

* 개발 비용 증가
* 유지보수 어려움
* AI 모델 변경 시 재개발 필요

MCP는 이러한 문제를 해결하기 위해 등장했다.

---

## 3. MCP 구조

```text
┌──────────────────────────┐
│ MCP Client               │
│ (Claude Desktop / Cursor)│
│                          │
│  ┌────────────────────┐  │
│  │  LLM (Claude)      │  │
│  └────────────────────┘  │
└──────────┬───────────────┘
           │ MCP 프로토콜
           ▼
┌─────────────┐
│ MCP Server  │
└──────┬──────┘
       │
       ▼
┌────────────────────┐
│ DB / API / Files   │
│ Slack / GitHub     │
└────────────────────┘
```

### MCP Client

AI 애플리케이션이 사용하는 클라이언트

예시:

* Claude Desktop
* Cursor
* Windsurf
* VS Code Extension
* Codex CLI (OpenAI)

### MCP Server

외부 시스템과 실제 통신하는 서버

예시:

* GitHub MCP Server
* PostgreSQL MCP Server
* Slack MCP Server
* Notion MCP Server

---

## 4. MCP 동작 과정

예시 질문:

> "GitHub에서 최근 Pull Request 보여줘"

### 1단계

사용자가 질문한다.

### 2단계

LLM이 필요한 도구를 판단한다.

```text
GitHub 데이터 필요
```

### 3단계

MCP Client가 MCP Server 호출

```json
{
  "tool": "list_pull_requests",
  "repository": "project-a"
}
```

### 4단계

MCP Server가 GitHub API 호출

### 5단계

결과 반환

```json
{
  "prs": [
    {
      "title": "Fix Login Bug"
    }
  ]
}
```

### 6단계

LLM이 자연어로 응답 생성

```text
최근 Pull Request는
Fix Login Bug 입니다.
```

---

## 5. MCP의 장점

### 표준화

한 번 구현하면 여러 LLM에서 재사용 가능

### 유지보수 용이

도구 변경 시 MCP Server만 수정

### 생산성 향상

새로운 AI 모델이 나와도 기존 서버 재사용 가능

### 확장성

새로운 시스템 연동이 쉬움

---

## 6. 기존 방식과 비교

| 항목    | 기존 방식  | MCP     |
| ----- | ------ | ------- |
| 연동 방식 | 모델별 구현 | 표준 프로토콜 |
| 유지보수  | 어려움    | 쉬움      |
| 재사용성  | 낮음     | 높음      |
| 확장성   | 낮음     | 높음      |

---

## 7. 실무 활용 사례

### 개발

* GitHub 조회
* 코드 분석
* PR 생성

### 데이터베이스

* PostgreSQL 조회
* MySQL 조회
* 데이터 분석

### 협업 툴

* Slack 메시지 조회
* Notion 문서 검색
* Jira 티켓 조회

### 파일 시스템

* 로컬 파일 읽기
* 로그 분석
* 문서 검색

---

## 8. MCP와 Function Calling의 차이

### Function Calling

특정 AI 플랫폼 내부 기능

```text
OpenAI ↔ Function
```

플랫폼이 바뀌면 다시 구현해야 한다.

### MCP

플랫폼 독립적 표준

```text
OpenAI
Claude
Gemini
Cursor
Windsurf
      │
      ▼
MCP Server
```

하나의 서버를 여러 AI가 공유할 수 있다.

---

## 9. 한 줄 정리

MCP(Model Context Protocol)는 AI 모델과 외부 도구를 연결하기 위한 표준 프로토콜로, "AI 시대의 USB"라고 불리며 한 번 구현한 도구를 여러 LLM에서 재사용할 수 있게 해준다.
