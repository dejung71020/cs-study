# JWT와 OAuth 2.0

## JWT (JSON Web Token)

### 구조
```
Header.Payload.Signature

eyJhbGciOiJIUzI1NiJ9  ← Header (알고리즘)
.eyJ1c2VyX2lkIjoxfQ   ← Payload (데이터)
.SflKxwRJSMeKKF2QT4   ← Signature (검증용)
```

### Access Token vs Refresh Token

| | Access Token | Refresh Token |
|---|---|---|
| 유효기간 | 짧음 (30분~1시간) | 김 (7일~30일) |
| 용도 | API 요청 인증 | Access Token 재발급 |
| 저장 위치 | 메모리 or localStorage | httpOnly 쿠키 |

### FastAPI 구현
```python
def create_access_token(user_id: int) -> str:
    payload = {
        "user_id": user_id,
        "exp": datetime.utcnow() + timedelta(minutes=30)
    }
    return jwt.encode(payload, SECRET_KEY, algorithm="HS256")
```

---

## OAuth 2.0

### 인가 코드 흐름 (Authorization Code Flow)

```
1. 사용자 → "Slack으로 로그인" 클릭
2. 내 서버 → Slack 인가 서버로 리다이렉트
3. 사용자 → Slack에서 권한 허가
4. Slack → 내 서버로 인가 코드 전송
5. 내 서버 → 인가 코드로 Access Token 교환 (서버-서버)
6. 내 서버 → Access Token으로 Slack API 호출
```

### WorkB에서 실제 구현

- Slack / Jira / Google Calendar 3개 OAuth 2.0 직접 구현
- 워크스페이스별 토큰을 DB에 저장해 관리
- n8n은 동적 토큰 관리 불가 → FastAPI 직접 구현으로 전환한 이유
