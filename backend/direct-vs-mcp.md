# 직접 연동 vs MCP 연동

## 1. 정의

**직접 연동 (Direct Integration)**
LLM 애플리케이션에서 외부 도구(API, DB, 파일 시스템 등)를 직접 코드로 연결하는 방식. 각 도구마다 별도의 함수와 API 호출 코드를 작성한다.

**MCP 연동 (Model Context Protocol Integration)**
Anthropic이 2024년 11월 공개한 오픈 프로토콜. LLM과 외부 도구 사이의 표준 통신 규격을 정의하여, MCP 서버를 한 번 구현하면 어떤 LLM 클라이언트에서도 동일하게 사용할 수 있다.

---

## 2. 등장 배경

LLM 애플리케이션이 늘어나면서 외부 도구 연동 방식이 제각각이었다:
- OpenAI Function Calling
- LangChain Tools
- 각 회사별 커스텀 구현

동일한 기능(예: 파일 읽기, DB 조회)을 플랫폼마다 다시 구현해야 하는 **파편화 문제**가 심각해졌다. MCP는 이를 USB처럼 표준화된 연결 규격으로 해결하려 등장했다.

---

## 3. 해결하는 문제

| 문제 | 직접 연동 | MCP |
|---|---|---|
| 플랫폼 종속 | Claude → 재작성, GPT → 재작성 | 서버 1개로 모든 클라이언트 지원 |
| 코드 중복 | 도구마다 개별 구현 | 서버 구현 후 재사용 |
| 유지보수 | API 변경 시 전체 코드 수정 | 서버만 수정 |
| 보안 | 앱 코드에 API 키 혼재 | 서버에서 독립적으로 관리 |

---

## 4. 내부 동작 원리

### 직접 연동 흐름
```
사용자 입력 → LLM → 함수 호출 결정 → 앱 코드에서 API 직접 호출 → 결과 반환 → LLM 최종 답변
```

### MCP 연동 흐름
```
사용자 입력 → LLM (MCP Client) → MCP 프로토콜로 요청 → MCP Server → 외부 도구 실행 → 결과 반환 → LLM 최종 답변
```

**MCP 구성 요소:**
- **MCP Client**: LLM 호스트 (Claude Desktop, Cursor 등)
- **MCP Server**: 도구를 노출하는 경량 서버
- **Transport**: stdio (로컬), HTTP + SSE (원격)

**MCP 3대 프리미티브:**
- **Tools**: LLM이 호출할 수 있는 함수 (예: `search_web`, `read_file`)
- **Resources**: LLM이 읽을 수 있는 데이터 (예: DB 스키마, 파일 내용)
- **Prompts**: 재사용 가능한 프롬프트 템플릿

---

## 5. 관련 기술

- **Function Calling**: OpenAI, Anthropic 등이 지원하는 LLM 도구 호출 기능. MCP의 Tools가 이를 추상화
- **LangChain Tools**: MCP 이전의 에이전트 도구 연동 방식
- **JSON-RPC 2.0**: MCP가 내부적으로 사용하는 통신 프로토콜
- **SSE (Server-Sent Events)**: HTTP 기반 MCP 원격 연결에 사용
- **stdio**: 로컬 MCP 서버 통신 방식 (표준 입출력)

---

## 6. 실제 서비스 사례

- **Claude Desktop**: MCP 서버를 설정 파일(`claude_desktop_config.json`)로 연결, 로컬 파일·DB 조회 가능
- **Cursor IDE**: MCP를 통해 코드베이스 외 외부 API 연동
- **GitHub MCP Server**: GitHub API를 MCP로 래핑 → 어떤 MCP 클라이언트에서도 GitHub 조작 가능
- **Notion MCP Server**: Notion DB를 LLM에 연결해 문서 검색·생성 자동화

---

## 7. 구현 예시

### 직접 연동 (Python)
```python
import anthropic
import requests

def search_web(query: str) -> str:
    response = requests.get(f"https://api.example.com/search?q={query}")
    return response.json()["result"]

tools = [{
    "name": "search_web",
    "description": "웹 검색",
    "input_schema": {
        "type": "object",
        "properties": {"query": {"type": "string"}},
        "required": ["query"]
    }
}]

client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-opus-4-7",
    tools=tools,
    messages=[{"role": "user", "content": "파이썬 최신 버전 검색해줘"}]
)

# 도구 호출 결과를 앱 코드에서 직접 처리
if response.stop_reason == "tool_use":
    result = search_web(response.content[0].input["query"])
```

### MCP 서버 구현 (Python)
```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
import requests

app = Server("my-search-server")

@app.tool()
async def search_web(query: str) -> str:
    """웹 검색"""
    response = requests.get(f"https://api.example.com/search?q={query}")
    return response.json()["result"]

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())
```

```json
// claude_desktop_config.json — 설정만 바꾸면 어떤 클라이언트에서도 동작
{
  "mcpServers": {
    "my-search": {
      "command": "python",
      "args": ["my_search_server.py"]
    }
  }
}
```

---

## 8. 장단점

### 직접 연동
| 장점 | 단점 |
|---|---|
| 구현 단순, 빠른 프로토타이핑 | 플랫폼 종속 |
| 특정 모델에 최적화 가능 | 재사용 불가 |
| 추가 인프라 불필요 | API 변경 시 전체 수정 필요 |

### MCP 연동
| 장점 | 단점 |
|---|---|
| 플랫폼 무관 재사용 | 초기 설정 복잡 |
| 도구 추가·교체 용이 | 로컬 stdio 방식은 원격 배포 어려움 |
| 보안 분리 (서버 독립적 관리) | 에코시스템 아직 성숙 중 |

---

## 9. 면접 질문

**Q. MCP와 일반 Function Calling의 차이는?**
A. Function Calling은 특정 LLM 플랫폼에 종속된 도구 호출 방식입니다. MCP는 이를 추상화한 표준 프로토콜로, 동일한 MCP 서버를 Claude, GPT 등 어떤 클라이언트에서도 사용할 수 있습니다.

**Q. MCP를 사용하면 얻는 가장 큰 이점은?**
A. 도구 재사용성과 유지보수 용이성입니다. 한 번 MCP 서버를 구현하면 모델이 바뀌어도 서버 코드 변경 없이 그대로 사용할 수 있습니다.

**Q. MCP의 3대 프리미티브를 설명하세요.**
A. Tools(LLM이 호출하는 함수), Resources(LLM이 읽는 데이터), Prompts(재사용 프롬프트 템플릿)입니다.

---

## 10. 나의 프로젝트 적용 아이디어

QUAIL 프로젝트에서 Gemini API를 직접 연동했는데, MCP 서버로 분리하면 나중에 Claude나 GPT로 교체할 때 service.py 코드 변경 없이 MCP 설정만 바꾸면 됩니다. 또한 진단 도구들(이미지 분석, ChromaDB 검색, 리포트 생성)을 각각 MCP Tool로 분리하면 독립적인 테스트와 재사용이 가능합니다. 야몬 같은 AI 오케스트레이션 회사라면 여러 AI 서비스를 MCP 서버로 묶는 구조가 핵심 아키텍처가 될 수 있습니다.

---

## 11. 나의 생각
- 이번엔 추가적인 cs-study가 더 필요하다. 요즘 추세에는 MCP를 백엔드 개발자는 필수적으로 알아야 하는 개념인 것 같다.
- 추후 확실하게 개념을 정해야 한다.
- 내가 이해하기로는 MCP 는 LLM 에서 외부서비스 같은 것을 툴로 만들어 LLM 이 호출할 수 있게하는 기본 규격이다. 라고 이해했다.
