# Docker와 컨테이너

## 1. Docker란?

Docker는 애플리케이션을 **컨테이너(Container)** 단위로 패키징해 어디서든 동일하게 실행할 수 있게 해주는 플랫폼이다.

핵심 가치: **"내 로컬에서는 됐는데 서버에서 안 돼요"** 문제 해결  
→ 실행 환경(OS, 라이브러리, 설정)까지 이미지에 포함시켜 어디서나 동일하게 동작

---

## 2. 컨테이너 vs 가상머신(VM)

| 구분 | 가상머신(VM) | 컨테이너 |
|---|---|---|
| 가상화 대상 | 하드웨어 + OS 전체 | 프로세스 (OS 커널 공유) |
| 크기 | GB 단위 | MB 단위 |
| 시작 속도 | 분 단위 | 초 단위 |
| 격리 수준 | 강함 | 중간 |
| 대표 도구 | VMware, VirtualBox | Docker |

컨테이너는 호스트 OS의 커널을 공유하면서 프로세스만 격리 → 가볍고 빠름

---

## 3. 핵심 개념

| 개념 | 설명 |
|---|---|
| **Image** | 컨테이너 실행을 위한 읽기 전용 패키지 (코드 + 라이브러리 + 설정) |
| **Container** | 이미지의 실행 인스턴스. 이미지 하나로 컨테이너 여러 개 생성 가능 |
| **Registry** | 이미지 저장소. 공개: Docker Hub, 사설: AWS ECR, GitHub Container Registry |
| **Dockerfile** | 이미지를 만드는 레시피 파일 |
| **Docker Compose** | 여러 컨테이너를 하나의 파일로 정의하고 함께 실행하는 도구 |

---

## 4. Dockerfile 기본 문법

```dockerfile
FROM python:3.12-slim          # 베이스 이미지

WORKDIR /app                   # 작업 디렉터리 설정

COPY requirements.txt .        # 파일 복사
RUN pip install -r requirements.txt  # 명령 실행 (이미지에 반영)

COPY . .                       # 소스 코드 복사

ENV PORT=8000                  # 환경변수 설정

EXPOSE 8000                    # 열어둘 포트 (문서용, 실제 오픈은 run 시)

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**주요 명령어 정리**

| 명령 | 용도 |
|---|---|
| `FROM` | 베이스 이미지 지정 |
| `RUN` | 빌드 시 실행 (레이어 생성) |
| `COPY` | 호스트 파일 → 이미지로 복사 |
| `ENV` | 환경변수 설정 |
| `EXPOSE` | 컨테이너가 사용할 포트 명시 |
| `CMD` | 컨테이너 시작 시 기본 실행 명령 |
| `ENTRYPOINT` | CMD와 유사하지만 덮어쓰기 불가 (고정 명령) |

---

## 5. 주요 CLI 명령어

```bash
# 이미지 빌드
docker build -t my-app:latest .

# 컨테이너 실행
docker run -d -p 8000:8000 --name my-app my-app:latest
# -d: 백그라운드 실행
# -p 호스트포트:컨테이너포트

# 실행 중인 컨테이너 목록
docker ps
docker ps -a    # 멈춘 컨테이너 포함

# 컨테이너 로그
docker logs my-app
docker logs -f my-app    # 실시간 스트리밍

# 컨테이너 진입
docker exec -it my-app bash

# 컨테이너 중지 / 삭제
docker stop my-app
docker rm my-app

# 이미지 목록 / 삭제
docker images
docker rmi my-app:latest
```

---

## 6. Docker Compose

여러 컨테이너를 하나의 `docker-compose.yml` 파일로 정의하고 함께 관리한다.

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
    depends_on:
      - db
    restart: always

  db:
    image: postgres:16
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=pass

volumes:
  db_data:
```

```bash
docker compose up -d        # 모든 서비스 백그라운드 실행
docker compose down         # 모든 서비스 중지 + 컨테이너 삭제
docker compose logs -f api  # 특정 서비스 로그
docker compose ps           # 서비스 상태 확인
docker compose build        # 이미지 재빌드
```

---

## 7. 볼륨 (Volume)

컨테이너는 재시작하면 내부 데이터가 사라진다. 볼륨으로 데이터를 호스트에 영속화한다.

| 종류 | 형태 | 용도 |
|---|---|---|
| **Named Volume** | `volume_name:/path` | DB 데이터 등 Docker가 관리하는 영속 저장소 |
| **Bind Mount** | `/host/path:/container/path` | 개발 중 소스 코드 실시간 반영 |

```yaml
volumes:
  - db_data:/var/lib/postgresql/data   # named volume
  - ./src:/app/src                      # bind mount (개발용)
```

---

## 8. 네트워크

같은 `docker-compose.yml`에 정의된 서비스들은 **자동으로 같은 네트워크**에 배치된다.  
서비스 이름으로 서로를 호출할 수 있다.

```python
# api 컨테이너에서 db 컨테이너 접근
DATABASE_URL = "postgresql://user:pass@db:5432/mydb"
#                                              ↑ 서비스 이름이 호스트명
```

| 네트워크 모드 | 설명 |
|---|---|
| `bridge` | 기본값. 격리된 가상 네트워크 |
| `host` | 호스트 네트워크 직접 사용 (포트 매핑 불필요) |
| `none` | 네트워크 없음 |

---

## 9. 실무 패턴

### 멀티스테이지 빌드 (이미지 크기 최소화)

```dockerfile
# 1단계: 빌드
FROM python:3.12 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# 2단계: 실행 (빌드 결과물만 복사)
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0"]
```

### .dockerignore

```
__pycache__/
*.pyc
.env
.git
node_modules/
```

### GPU 지원 (NVIDIA Container Toolkit)

야몬 DFAS처럼 GPU가 필요한 컨테이너는 `runtime: nvidia`를 추가한다.

```yaml
services:
  dfas-api:
    image: dfas-api:latest
    runtime: nvidia    # NVIDIA GPU 사용
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
```

### restart 정책

```yaml
restart: always      # 항상 재시작 (서버 재부팅 후에도)
restart: on-failure  # 실패 시만 재시작
restart: unless-stopped  # 수동으로 멈추기 전까지 재시작
```

---

## 10. 요약

| 개념 | 한 줄 정리 |
|---|---|
| Docker | 컨테이너 기반 실행 환경 통일 플랫폼 |
| Image | 실행 가능한 패키지 (불변, 레이어 구조) |
| Container | 이미지의 실행 인스턴스 |
| Dockerfile | 이미지 빌드 레시피 |
| Docker Compose | 멀티 컨테이너 오케스트레이션 (로컬/소규모) |
| Volume | 컨테이너 데이터 영속화 |

**언제 쓰나?**
- 개발 환경 통일 ("나는 됐는데 너는 안 돼" 방지)
- 서비스 배포 자동화 (CI/CD 파이프라인)
- 마이크로서비스 각 컴포넌트 격리 (DB, API, 큐 등)

---

## 11. 내 생각

야몬에서 첫날 인수인계 자료를 보니 DFAS 프로젝트가 `dfas-api`, `dfas-qdrant`, `dfas-tika` 세 컨테이너로 구성되어 있고, GPU 런타임까지 붙어있었다. Docker를 대충은 알고 있다고 생각했는데, 막상 실제 프로젝트 `docker-compose.yml`을 보니 볼륨 마운트나 서비스 간 의존 관계 같은 부분이 낯설었다.

특히 `restart: always`가 왜 붙어있는지, 컨테이너끼리 서비스 이름으로 통신하는 원리가 뭔지 지금은 이해됐다. 앞으로 API 서버 리팩토링 작업을 하면서 Docker 환경에서 직접 실행하고 로그도 봐야 할 텐데, 명령어를 미리 정리해둔 게 도움이 될 것 같다.

컨테이너가 VM보다 가볍고 빠른 이유가 OS 커널을 공유하기 때문이라는 것, 그리고 이미지 레이어 캐싱 때문에 `COPY requirements.txt`를 먼저 올려야 빌드가 빠르다는 것이 제일 인상적이었다.

---

## 12. 추가 공부

### Docker 이미지 레이어 캐싱

Dockerfile의 각 명령어(`RUN`, `COPY` 등)는 레이어를 생성한다. 레이어는 캐시되므로 **변경이 잦은 코드는 아래에, 변경이 드문 의존성은 위에** 두어야 빌드가 빠르다.

```dockerfile
# 나쁜 예 — 코드 한 줄 바꿔도 pip install부터 다시 실행
COPY . .
RUN pip install -r requirements.txt

# 좋은 예 — requirements.txt 변경 없으면 pip install 캐시 재사용
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

### Docker Compose vs Kubernetes

| | Docker Compose | Kubernetes (k8s) |
|---|---|---|
| 규모 | 단일 서버, 소규모 | 다중 서버, 대규모 클러스터 |
| 학습 곡선 | 낮음 | 높음 |
| 오토스케일링 | 없음 | 있음 |
| 롤링 업데이트 | 수동 | 자동 |
| 용도 | 로컬 개발, 소형 서비스 | 프로덕션 대규모 서비스 |

야몬의 DFAS처럼 물리 서버 1~2대 구성은 Docker Compose로 충분하다.  
서버가 10대 이상이거나 오토스케일링이 필요하면 K8s를 고려한다.

### 헬스체크 (Healthcheck)

```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

`depends_on`과 함께 쓰면 의존 서비스가 **실제로 준비됐을 때** 다음 컨테이너를 시작할 수 있다.

```yaml
depends_on:
  db:
    condition: service_healthy
```
