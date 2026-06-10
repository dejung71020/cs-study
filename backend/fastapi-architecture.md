# FastAPI 아키텍처 패턴

## 1. 왜 아키텍처가 중요한가

FastAPI는 구조를 강제하지 않는다. 자유도가 높은 만큼 설계가 나쁘면 빠르게 스파게티 코드가 된다.

좋은 아키텍처의 목표:

- **가독성**: 새로운 개발자가 코드를 빠르게 파악 가능
- **재사용성**: 공통 로직을 반복하지 않음
- **테스트 용이성**: 비즈니스 로직을 분리해 단위 테스트 가능
- **유지보수**: 변경이 다른 부분에 영향을 최소화

---

## 2. Flat Structure (소규모 / 프로토타입)

**적합한 경우**: 1인 개발, 프로토타입, 도메인이 2~3개인 소규모 API

```
app/
├── main.py
├── database.py
├── models.py        # 모든 ORM 모델
├── schemas.py       # 모든 Pydantic 모델
├── routers/
│   ├── users.py
│   └── items.py
└── dependencies.py
```

**특징**:

- 파일 수가 적어 빠르게 시작 가능
- 프로젝트가 커지면 `models.py`, `schemas.py`가 비대해짐
- 도메인 간 의존성이 뒤엉키기 쉬움

---

## 3. Domain-based Modular Structure (중규모 / 현업 표준)

**적합한 경우**: 팀 개발, 도메인이 5개 이상, 장기 유지보수

```
app/
├── main.py
├── core/
│   ├── config.py        # 환경변수
│   ├── database.py      # DB 연결
│   └── security.py      # JWT 등
├── domains/
│   ├── user/
│   │   ├── router.py
│   │   ├── service.py
│   │   ├── models.py
│   │   └── schemas.py
│   └── item/
│       ├── router.py
│       ├── service.py
│       ├── models.py
│       └── schemas.py
└── utils/
    └── pagination.py
```

**특징**:

- 도메인별로 완전히 분리 → 팀원이 충돌 없이 작업 가능
- 새 도메인 추가 시 폴더 하나만 추가하면 됨
- 현업에서 가장 보편적으로 사용하는 구조

---

## 4. Clean Architecture (대규모 / 테스트 중심)

**적합한 경우**: 대형 서비스, 테스트 커버리지가 중요한 프로젝트

```
app/
├── api/                        # Presentation Layer (라우터)
│   └── v1/
│       └── endpoints/
│           └── users.py
├── core/                       # Application Layer (Use Cases)
│   └── use_cases/
│       └── create_user.py
├── domain/                     # Domain Layer (Entity, 인터페이스)
│   ├── entities/
│   │   └── user.py
│   └── repositories/
│       └── user_repository.py  # 추상 인터페이스
├── infrastructure/             # Infrastructure Layer (실제 구현)
│   ├── db/
│   │   └── user_repository.py  # 구체 구현
│   └── external/
│       └── email_service.py
└── main.py
```

**핵심 원칙**: 안쪽 레이어(Domain)는 바깥쪽(Infrastructure)을 모른다.

```python
# domain/repositories/user_repository.py — 추상 인터페이스
from abc import ABC, abstractmethod

class UserRepository(ABC):
    @abstractmethod
    async def find_by_id(self, user_id: int) -> User:
        pass

# infrastructure/db/user_repository.py — 구체 구현
class SQLUserRepository(UserRepository):
    async def find_by_id(self, user_id: int) -> User:
        # 실제 DB 쿼리
        pass
```

DB를 PostgreSQL → MongoDB로 바꿔도 Domain Layer는 수정 없음.

---

## 5. Repository Pattern

Service가 DB를 직접 호출하면 테스트가 어렵다. Repository를 두면 Service는 인터페이스만 알면 된다.

```python
# 나쁜 예 — Service가 DB에 직접 의존
class UserService:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_user(self, user_id: int):
        result = await self.db.execute(select(User).where(User.id == user_id))
        return result.scalar_one_or_none()

# 좋은 예 — Repository를 통해 분리
class UserService:
    def __init__(self, repo: UserRepository):
        self.repo = repo

    async def get_user(self, user_id: int):
        return await self.repo.find_by_id(user_id)
```

테스트 시 `UserRepository`의 Mock을 주입하면 DB 없이 Service 테스트 가능.

---

## 6. Generic CRUD Base (재사용성 극대화)

도메인마다 CRUD를 반복 구현하는 대신 Generic Base를 만든다.

```python
from typing import Generic, TypeVar, Type
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

ModelType = TypeVar("ModelType")
CreateSchemaType = TypeVar("CreateSchemaType")

class CRUDBase(Generic[ModelType, CreateSchemaType]):
    def __init__(self, model: Type[ModelType]):
        self.model = model

    async def get(self, db: AsyncSession, id: int):
        result = await db.execute(select(self.model).where(self.model.id == id))
        return result.scalar_one_or_none()

    async def create(self, db: AsyncSession, obj_in: CreateSchemaType):
        obj = self.model(**obj_in.model_dump())
        db.add(obj)
        await db.commit()
        return obj

# 사용 — 기본 CRUD는 상속, 커스텀만 추가
class CRUDUser(CRUDBase[User, UserCreate]):
    async def get_by_email(self, db: AsyncSession, email: str):
        result = await db.execute(select(User).where(User.email == email))
        return result.scalar_one_or_none()

user_crud = CRUDUser(User)
```

---

## 7. 표준 응답 모델

API 응답 형식을 통일하면 프론트엔드와 협업이 쉬워지고 에러 처리가 일관된다.

```python
from typing import Generic, TypeVar, Optional
from pydantic import BaseModel

T = TypeVar("T")

class BaseResponse(BaseModel, Generic[T]):
    success: bool
    message: str
    data: Optional[T] = None

# 사용
@router.get("/users/{user_id}", response_model=BaseResponse[UserSchema])
async def get_user(user_id: int):
    user = await user_service.get(user_id)
    return BaseResponse(success=True, message="OK", data=user)

# 에러 시
return BaseResponse(success=False, message="유저를 찾을 수 없습니다.")
```

---

## 8. 전역 예외 처리

각 라우터에서 try/except를 반복하는 대신 전역에서 처리한다.

```python
# 커스텀 예외 정의
class NotFoundException(Exception):
    def __init__(self, message: str):
        self.message = message

class UnauthorizedException(Exception):
    def __init__(self, message: str = "인증이 필요합니다."):
        self.message = message

# main.py에 전역 등록
@app.exception_handler(NotFoundException)
async def not_found_handler(request: Request, exc: NotFoundException):
    return JSONResponse(status_code=404, content={"success": False, "message": exc.message})

@app.exception_handler(UnauthorizedException)
async def unauthorized_handler(request: Request, exc: UnauthorizedException):
    return JSONResponse(status_code=401, content={"success": False, "message": exc.message})

# 라우터에서 사용
async def get_user(user_id: int):
    user = await user_repo.find_by_id(user_id)
    if not user:
        raise NotFoundException(f"유저 {user_id}를 찾을 수 없습니다.")
    return user
```

---

## 9. 의존성 주입 (Depends) 패턴

FastAPI의 `Depends`를 활용해 재사용 가능한 의존성을 만든다.

```python
# 현재 유저 추출 → 모든 인증 라우터에서 재사용
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)
) -> User:
    payload = verify_token(token)
    user = await user_repo.find_by_id(db, payload["user_id"])
    if not user:
        raise UnauthorizedException()
    return user

# 라우터에서 주입
@router.get("/me")
async def get_me(current_user: User = Depends(get_current_user)):
    return current_user

@router.delete("/me")
async def delete_me(current_user: User = Depends(get_current_user)):
    await user_service.delete(current_user.id)
```

---

## 10. 비교 요약

| 구조 | 규모 | 재사용성 | 학습 비용 | 현업 사용 |
|---|---|---|---|---|
| Flat | 소규모 | 낮음 | 낮음 | 프로토타입 |
| Domain-based | 중~대규모 | 높음 | 중간 | **가장 일반적** |
| Clean Architecture | 대규모 | 매우 높음 | 높음 | 대형 서비스 |

**현업에서 가장 많이 쓰는 조합**

> Domain-based + Repository Pattern + Generic CRUD + 표준 응답 모델 + 전역 예외 처리

---

## 11. 내 생각

- 앞으로의 FastAPI는 도메인 기반 레포 패턴 + CRUD 추상화 + 표준 응답 + 전역 예외 처리를 적용해보아야겠다.
- 회사에서 FastAPI가 하나의 파일인 경우 리팩토링하는 방법을 내일 찾아봐야겠다.

