# Flask vs FastAPI vs Node.js

## 내가 공부 전 아는 상식
* FastAPI를 직접 써봤고, Flask보다 빠르다고 알고 있다.
* Node.js는 JavaScript 기반이고 비동기에 강하다고 들었다.
* FastAPI는 Swagger 문서가 자동으로 생성된다.
* Flask는 FastAPI보다 더 소규모에 적합하다고 들었다.

## 핵심 비교

| | Flask | FastAPI | Node.js |
|---|---|---|---|
| 언어 | Python | Python | JavaScript |
| 동기/비동기 | 동기 (기본) | 비동기 지원 (async/await) | 비동기 (기본, 이벤트 루프) |
| 성능 | 보통 | 빠름 (ASGI 기반) | 빠름 (논블로킹 I/O) |
| 타입 검사 | 없음 | 자동 (Pydantic) | 없음 (TypeScript 사용 시 가능) |
| API 문서 자동화 | 없음 | Swagger 자동 생성 (/docs) | 없음 |
| 적합한 상황 | 소규모, 간단한 서버 | 빠른 API, ML 연동 | 실시간 서비스, 채팅 |

## Flask

- 최소한의 기능만 제공하는 마이크로 프레임워크
- ORM, 인증 등 필요한 건 직접 라이브러리 추가
- 동기 방식 → 요청 하나 끝나야 다음 처리

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello"
```

## FastAPI

- 타입 힌트 기반 → Pydantic이 자동으로 요청 데이터 검증
- `async def` 지원 → DB·외부 API 호출 시 성능 유리
- `/docs` 접속하면 Swagger UI 자동 생성

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: int

@app.post("/items")
async def create_item(item: Item):
    return item
```

## Node.js

- JavaScript 런타임 (프레임워크가 아님 — Express, NestJS 등 위에서 동작)
- 이벤트 루프 기반 논블로킹 I/O → 동시 요청 처리에 강함
- CPU 집약적 작업(ML, 이미지 처리)에는 부적합
- 프론트(React, Vue)와 언어를 JS로 통일 가능

```javascript
const express = require('express')
const app = express()

app.get('/', (req, res) => res.send('Hello'))
app.listen(3000)
```

## 면접 한 줄 답변
> "Flask는 가볍고 자유도가 높은 동기 프레임워크, FastAPI는 비동기 지원과 자동 타입 검증으로 성능과 생산성이 높은 프레임워크, Node.js는 이벤트 루프 기반으로 I/O 처리에 강한 JS 런타임입니다."

## 프로젝트 적용
- 애완용품샵: 동기 FastAPI + SQLAlchemy 1.x 사용 → Flask와 유사한 구조, 단순 CRUD에 적합
- QUAIL: 비동기 FastAPI + SQLAlchemy 2.0 AsyncSession → AI 파이프라인과 외부 API 동시 처리에 유리
