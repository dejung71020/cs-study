# 동기 vs 비동기 (Sync vs Async)

## 내가 공부 전 아는 상식
* 동기는 순서대로 실행되고, 비동기는 기다리지 않고 다음으로 넘어간다.
* FastAPI는 async를 쓴다.
* async는 대용량 작업에 효율적이다.

## 핵심 차이

| | 동기 (Sync) | 비동기 (Async) |
|---|---|---|
| 실행 방식 | 작업이 끝날 때까지 대기 | 기다리지 않고 다음 작업 진행 |
| 스레드 | 대기 중에도 스레드 점유 | 대기 중에 스레드 반환 |
| 적합한 상황 | CPU 연산이 많은 작업 | I/O 대기가 많은 작업 (DB, HTTP) |
| 코드 복잡도 | 단순 | 상대적으로 복잡 |

## Python 비동기 기본 구조

```python
import asyncio

async def fetch_data():
    await asyncio.sleep(1)  # I/O 대기 (블로킹 없이)
    return "data"

async def main():
    result = await fetch_data()
    print(result)
```

- `async def`: 비동기 함수 선언
- `await`: 여기서 잠깐 양보, 다른 작업 처리 후 돌아옴

## FastAPI에서 동기 vs 비동기

```python
# 동기 (간단하지만 요청마다 스레드 점유)
@app.get("/sync")
def sync_endpoint():
    time.sleep(1)
    return {"result": "ok"}

# 비동기 (DB·외부 API 호출 시 스레드 효율적으로 사용)
@app.get("/async")
async def async_endpoint():
    await asyncio.sleep(1)
    return {"result": "ok"}
```

FastAPI는 둘 다 지원하지만, DB 조회나 외부 API 호출처럼 I/O 대기가 있으면 `async def`가 성능상 유리하다.

## 주의: CPU 바운드 작업은 비동기가 오히려 독

비동기는 I/O 대기 시간을 활용하는 것.
CPU를 계속 써야 하는 연산(이미지 처리, 머신러닝 추론)은 비동기로 해도 의미 없음.
→ 이 경우엔 멀티프로세싱(concurrent.futures) 사용

## 면접 한 줄 답변
> "동기는 작업이 끝날 때까지 기다리고, 비동기는 I/O 대기 중에 다른 작업을 처리합니다. FastAPI에서 DB 조회나 외부 API 호출은 async/await를 써서 스레드를 효율적으로 사용합니다."

## 프로젝트 적용
- 팡팡팡: SQLAlchemy 2.0 AsyncSession + async def 라우터 → DB 조회 중 다른 요청 처리 가능
- WorkB: 외부 API(Slack, Jira, Calendar) 호출을 asyncio로 처리
