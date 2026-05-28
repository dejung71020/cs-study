# HTTP vs HTTPS

## HTTP

데이터를 평문으로 전송. 중간에 누구든 내용을 볼 수 있음.

## HTTPS

HTTP + TLS/SSL 암호화. 데이터를 암호화해서 전송.

## TLS 핸드셰이크 과정

```
클라이언트 → 서버: "이런 암호화 방식 지원해요" (ClientHello)
서버 → 클라이언트: SSL 인증서 전달
클라이언트: 인증서 유효성 검증 (CA 서명 확인)
클라이언트 → 서버: 세션 키 교환
이후: 세션 키로 대칭 암호화 통신
```

## SSL 인증서 종류

| 종류 | 설명 | 사용 |
|---|---|---|
| Self-signed | 직접 서명 | 개발/테스트용, 브라우저 경고 뜸 |
| Let's Encrypt | 무료 CA 발급 | 실제 서비스 (certbot으로 발급) |
| 유료 CA | Comodo, DigiCert 등 | 기업 서비스 |

## 팡팡팡 프로젝트 적용

- EC2 + nginx + certbot으로 Let's Encrypt 인증서 발급
- nginx에서 SSL Termination: HTTPS → HTTP로 변환 후 내부 전달
- 인증서 90일 만료 → cron으로 자동 갱신 (`certbot renew`)
