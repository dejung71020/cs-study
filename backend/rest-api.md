# REST API & RESTful

## 내가 공부 전 아는 상식
* REST API는 GET, POST, PUT, DELETE 로 HTTP 통신을 하는 것.
* 각각 API 요청은 독립적이어야 한다.
* API 요청은 필요한 모든 정보들을 담고 있어야 한다.
* RESTFUL은 REST API의 원칙이 잘 지켜진 상태를 의미한다.
* 모든 정보를 담지 않으면 요청에 대해 다른 응답이 나올 수 있다.

## REST란
**RE**presentational **S**tate **T**ransfer — 자원을 URI로 표현하고 HTTP 메서드로 행위를 정의하는 아키텍처 스타일.

## REST 6가지 제약 조건

| 제약 조건 | 설명 |
|---|---|
| 클라이언트-서버 | 클라이언트와 서버 역할을 분리 |
| 무상태 (Stateless) | 각 요청은 독립적, 서버가 상태 저장 안 함 |
| 캐시 가능 | 응답에 캐시 가능 여부 명시 |
| 계층화 | 클라이언트는 중간 서버 존재를 몰라도 됨 |
| 균일한 인터페이스 | URI + HTTP 메서드로 일관된 방식 |
| 코드 온 디맨드 (선택) | 서버가 클라이언트에 코드 전송 가능 |

## HTTP 메서드와 역할

| 메서드 | 역할 | 예시 |
|---|---|---|
| GET | 조회 | `GET /users/1` |
| POST | 생성 | `POST /users` |
| PUT | 전체 수정 | `PUT /users/1` |
| PATCH | 일부 수정 | `PATCH /users/1` |
| DELETE | 삭제 | `DELETE /users/1` |

## REST vs RESTful

- **REST:** 아키텍처 스타일 (개념)
- **RESTful:** REST 제약 조건을 잘 지킨 API (구현)

모든 HTTP API가 RESTful한 것은 아님.

### RESTful하지 않은 예시
```
GET /getUserInfo      ← 동사 사용 (X)
POST /deleteUser      ← 메서드와 URI 역할 불일치 (X)
```

### RESTful한 예시
```
GET /users/1          ← 자원(명사) + 적절한 메서드 (O)
DELETE /users/1       ← 명확한 의도 (O)
```

## URI 설계 원칙
- 명사 사용, 동사 금지: `/users` (O), `/getUsers` (X)
- 소문자, 하이픈(-) 사용: `/user-profiles` (O), `/userProfiles` (X)
- 계층 구조로 자원 관계 표현: `/users/1/orders`
- 파일 확장자 포함 금지: `/users.json` (X)

## 상태 코드

| 코드 | 의미 |
|---|---|
| 200 OK | 성공 |
| 201 Created | 생성 성공 |
| 400 Bad Request | 잘못된 요청 |
| 401 Unauthorized | 인증 필요 |
| 403 Forbidden | 권한 없음 |
| 404 Not Found | 자원 없음 |
| 500 Internal Server Error | 서버 오류 |

## 면접 한 줄 답변
> "REST는 URI로 자원을 표현하고 HTTP 메서드로 행위를 정의하는 아키텍처 스타일이고, RESTful은 REST 제약 조건을 충실히 지킨 API를 말합니다. 가장 핵심은 무상태와 균일한 인터페이스입니다."

## 프로젝트 적용
- 팡팡팡 FastAPI: `/api/diagnosis`, `/api/weather` 등 명사 기반 URI 설계
- WorkB: `/api/slack/callback`, `/api/jira/issues` — 외부 서비스별 계층 구조로 분리
