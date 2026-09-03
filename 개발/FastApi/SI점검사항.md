---
tags:
  - FastAPI
  - SI
  - 체크리스트
  - 온보딩
created: 2026-09-03
관점: Spring/Java 개발자 시각
---
# FastAPI SI 프로젝트 투입 점검사항

> Spring/Java 배경으로 FastAPI 프로젝트에 처음 들어갈 때, 코드를 읽기 전에 먼저 확인해야 할 것들.
> 순서대로 훑으면 대략 2~3일 안에 프로젝트 전체 그림이 잡힌다.

---

## 0. 먼저 머리에 박아둘 개념 대응표

| Spring / Java | FastAPI / Python | 비고 |
|---|---|---|
| Tomcat (서블릿, 스레드풀) | Uvicorn (ASGI, 이벤트 루프) | **가장 큰 차이. 여기서 사고 남** |
| `@RestController` | `APIRouter` | |
| `@RequestMapping` | `@router.get/post(...)` | |
| DTO + Lombok + `@Valid` | Pydantic `BaseModel` | 검증이 모델에 내장 |
| `@Autowired` / 생성자 주입 | `Depends()` | 요청 스코프 주입에 가까움 |
| Spring Container (싱글턴 Bean) | **없음** — 모듈 전역 변수 + `Depends` | DI 컨테이너 개념이 약함 |
| JPA / Hibernate | SQLAlchemy | |
| `@Transactional` | 세션 컨텍스트 수동 관리 | **AOP 트랜잭션 없음. 직접 확인 필수** |
| Flyway / Liquibase | Alembic | |
| `application.yml` + Profile | `pydantic-settings` + `.env` | |
| `@ControllerAdvice` | `@app.exception_handler` | |
| Spring Security | `Depends`로 직접 구현하거나 fastapi-users 등 | 표준이 없음 |
| Maven / Gradle | pip + requirements.txt / Poetry / uv | |
| Checkstyle / SpotBugs | ruff, black, mypy | |
| JUnit + MockMvc | pytest + `TestClient` | |
| Swagger 설정 | **자동 생성** (`/docs`, `/redoc`) | 공짜로 나옴 |

---

## 1. 프로젝트 실행부터 시켜본다

가장 먼저 할 일은 코드 읽기가 아니라 **로컬에서 띄우는 것**이다.

- [ ] Python 버전 확인 (`.python-version`, `pyproject.toml`, Dockerfile) — 3.9와 3.11은 문법·성능 차이 있음
- [ ] 의존성 관리 도구가 무엇인가?
  - `requirements.txt` (pip) / `pyproject.toml` + `poetry.lock` / `uv.lock` / `Pipfile`
  - **버전이 고정(lock)되어 있는가?** 안 되어 있으면 환경마다 다르게 도는 지뢰
- [ ] 가상환경 방식 (venv, conda, Docker only)
- [ ] 실행 커맨드 확인 (`uvicorn app.main:app --reload` 형태)
- [ ] `/docs` 접속해서 API 전체 목록 훑기 ← **여기가 프로젝트 파악의 지름길**
- [ ] 로컬 실행에 필요한 외부 의존(DB, Redis, 외부 API)이 뭔지, 로컬 대체제가 있는지

---

## 2. 디렉토리 구조 / 아키텍처

Spring처럼 정해진 규약이 없다. **팀이 정한 구조를 파악해야 한다.**

- [ ] 진입점 파일 위치 (`main.py`, `app/main.py`)
- [ ] 어떤 구조를 따르는가?
  - 계층형: `routers/` `services/` `repositories/` `models/` `schemas/` (Spring과 유사, 가장 흔함)
  - 도메인형: `domains/user/`, `domains/order/` 안에 router·service·model
  - 평면형: 라우터에 로직 다 들어있음 (레거시 SI에서 자주 봄)
- [ ] `schemas`(Pydantic, DTO)와 `models`(SQLAlchemy, Entity)가 분리되어 있는가?
  - 안 되어 있으면 ORM 객체가 그대로 응답으로 나가는 구조 → 리팩터링 위험 구역
- [ ] `APIRouter`가 어디서 `include_router`로 등록되는지 추적
- [ ] 공통 유틸/상수/예외가 모여있는 위치 (`core/`, `common/`, `utils/`)
- [ ] 순환 import 이슈가 있는지 (Python 특유의 문제)

---

## 3. 설정 / 환경 분리

- [ ] 설정 로딩 방식: `pydantic-settings`(`BaseSettings`) / `os.environ` 직접 접근 / config.py 하드코딩
- [ ] 환경 분리(dev/stg/prod)를 어떻게 하는가? `.env.dev`, `ENV` 변수 분기 등
- [ ] `.env`가 git에 올라가 있지는 않은가? (SI에서 자주 발견됨)
- [ ] **DB 접속정보, 외부 API 키가 소스에 하드코딩되어 있는지** — 있으면 즉시 이슈 제기
- [ ] 운영 환경 설정값은 누가 관리하는가 (고객사? 우리 팀? 배포 스크립트?)

---

## 4. DB / ORM — 자바 개발자가 가장 많이 데이는 구간

- [ ] ORM 사용 여부: SQLAlchemy / SQLModel / Tortoise / 생 SQL / MyBatis 유사물
- [ ] SQLAlchemy면 **1.x 스타일인가 2.0 스타일인가** (문법이 꽤 다름)
- [ ] 동기(`Session`)인가 비동기(`AsyncSession`)인가 — 섞여 있으면 위험 신호
- [ ] **세션 생명주기 관리**: `Depends(get_db)`로 요청당 세션을 열고 닫는가?
- [ ] **트랜잭션 경계가 어디인가?**
  - `@Transactional` 같은 게 없으므로 `commit()` / `rollback()` 위치를 직접 찾아야 함
  - 서비스 계층에서 commit 하는지, 라우터에서 하는지 확인
  - 여러 테이블 갱신이 한 트랜잭션으로 묶이는지 반드시 확인 ← **SI 하자 1순위**
- [ ] 마이그레이션 도구: Alembic 쓰는가, 아니면 DDL을 손으로 날리는가?
  - `alembic/versions/` 이력이 실제 DB 스키마와 일치하는지
- [ ] N+1 문제 대응 (`selectinload`, `joinedload`) 사용 여부
- [ ] 커넥션 풀 설정 (`pool_size`, `max_overflow`, `pool_recycle`) — 운영 장애 단골

---

## 5. 비동기(async) 사용 실태 — **최우선 점검**

FastAPI에서 자바 개발자가 가장 크게 사고 치는 부분.

- [ ] 엔드포인트가 `async def`인가 `def`인가, 기준이 일관적인가?
  - `def` → 스레드풀에서 실행됨 (블로킹 OK)
  - `async def` → 이벤트 루프에서 실행됨 (**블로킹 코드가 있으면 서버 전체가 멈춤**)
- [ ] `async def` 안에서 블로킹 호출을 하고 있지는 않은가?
  - `requests.get()` (→ `httpx.AsyncClient` 써야 함)
  - `time.sleep()` (→ `asyncio.sleep()`)
  - 동기 SQLAlchemy 세션
  - 파일 I/O, `subprocess`
- [ ] 무거운 CPU 작업이 있다면 어떻게 처리하는가 (`run_in_executor`, Celery, 별도 워커)
- [ ] 백그라운드 작업: `BackgroundTasks` / Celery / APScheduler 중 무엇인가
- [ ] 이 프로젝트가 **비동기가 정말 필요한 프로젝트인지** 판단
  - 대부분의 SI CRUD는 동기(`def`)로 짜는 게 더 안전하고 유지보수 쉬움

---

## 6. 요청/응답 스펙

- [ ] Pydantic 버전 확인 (**v1 vs v2 문법이 크게 다름**: `orm_mode` → `from_attributes`, `@validator` → `@field_validator`)
- [ ] 요청 검증 규칙이 Pydantic 모델에 잘 정의되어 있는가, 아니면 서비스에서 if문으로 하는가
- [ ] `response_model` 지정 여부 — 없으면 내부 필드가 그대로 노출될 수 있음
- [ ] 공통 응답 포맷(`{code, message, data}`)이 정의되어 있는가
- [ ] 페이징 규약 (page/size, offset/limit, 커서)
- [ ] 날짜/시간 포맷과 **타임존 정책** (UTC 저장 여부, KST 변환 지점)
- [ ] `camelCase` ↔ `snake_case` 변환 정책 (프론트와 합의된 규칙)

---

## 7. 예외 처리 / 에러 응답

- [ ] 전역 예외 핸들러(`@app.exception_handler`) 등록 여부
- [ ] 커스텀 예외 클래스 체계가 있는가
- [ ] `HTTPException`을 어디서 던지는가 (서비스에서 던지면 계층 침범)
- [ ] 에러 응답 포맷이 일관적인가
- [ ] **스택트레이스가 운영 환경 응답에 노출되지 않는가**

---

## 8. 인증 / 인가 / 보안

Spring Security 같은 표준이 없으므로 **직접 짠 코드를 읽어야 한다.**

- [ ] 인증 방식: JWT / 세션 / OAuth2 / 고객사 SSO 연동
- [ ] 토큰 검증 로직 위치와 `Depends`로 어떻게 주입되는지
- [ ] 권한(Role) 체크 방식 — 라우터마다 수동인가, 공통 의존성인가
- [ ] 인증 누락된 엔드포인트가 있는지 `/docs`에서 전수 확인
- [ ] CORS 설정 (`allow_origins=["*"]`로 열려있지 않은지)
- [ ] 비밀번호 해싱 (bcrypt/argon2 사용 여부)
- [ ] SQL Injection 위험 (문자열 포맷팅으로 쿼리 만드는 곳 있는지 grep)
- [ ] 개인정보 처리 항목과 마스킹/암호화 정책 (SI 계약서에 명시된 경우 많음)

---

## 9. 로깅 / 모니터링

- [ ] 로깅 설정 위치 (`logging.config`, loguru, structlog)
- [ ] Uvicorn 액세스 로그와 앱 로그가 분리되어 있는가
- [ ] 로그 레벨과 로그 파일 로테이션 정책
- [ ] 요청 추적 ID(Correlation ID) 미들웨어 존재 여부
- [ ] **로그에 개인정보/토큰이 찍히지 않는지**
- [ ] APM/모니터링 도구 (Sentry, Prometheus, 고객사 지정 툴)
- [ ] 헬스체크 엔드포인트 (`/health`) 존재 여부 — L4/K8s가 요구할 수 있음

---

## 10. 테스트

- [ ] 테스트 코드가 존재하는가 (SI에서는 없는 경우가 많음)
- [ ] pytest 설정 (`conftest.py`, fixture 구조)
- [ ] 테스트 DB 전략 (SQLite 대체 / 별도 스키마 / testcontainers)
- [ ] `TestClient` 기반 통합 테스트가 있는가
- [ ] 커버리지 기준이 계약상 요구사항에 있는지 확인

---

## 11. 빌드 / 배포 / 인프라

- [ ] Dockerfile 존재 여부와 베이스 이미지
- [ ] 운영 실행 방식: `uvicorn` 단독 / `gunicorn -k uvicorn.workers.UvicornWorker` / K8s
- [ ] **워커 수 설정** — 워커별로 메모리·커넥션풀이 별도임을 인지
- [ ] 리버스 프록시(Nginx, Apache) 설정과 타임아웃 값
- [ ] CI/CD 파이프라인 (Jenkins, GitHub Actions, 고객사 내부 시스템)
- [ ] 배포 승인 절차 — SI는 대부분 고객사 승인 필요
- [ ] 폐쇄망 여부: **인터넷이 안 되면 pip install이 안 된다.** 사내 PyPI 미러나 wheel 반입 절차 확인 ← 공공 SI 필수 확인

---

## 12. 코드 품질 / 협업 규칙

- [ ] 포매터·린터 설정 (`ruff`, `black`, `isort`, `flake8`) 및 pre-commit 훅
- [ ] 타입 힌트 사용 정도, `mypy` 적용 여부
- [ ] 브랜치 전략과 커밋 컨벤션
- [ ] 코드 리뷰 프로세스 존재 여부
- [ ] 네이밍 컨벤션 (Python은 `snake_case`가 표준 — 자바 습관 주의)

---

## 13. SI 프로젝트 고유 점검사항

기술 외적인 부분이지만 **여기서 프로젝트가 망한다.**

- [ ] **요구사항 정의서 / 기능명세서** 최신본 위치와 버전
- [ ] 현재 프로젝트 단계 (분석/설계/개발/테스트/안정화)와 남은 일정
- [ ] 산출물 목록과 제출 기한 (화면설계서, 테이블정의서, 인터페이스정의서, 단위테스트결과서 등)
- [ ] 내가 맡을 범위(모듈/화면)와 인수인계자
- [ ] 이슈 트래커 (Jira, Redmine, 고객사 지정 시스템)
- [ ] **외부 시스템 연동 목록** — 대내외 인터페이스, 규격서, 테스트 계정, 방화벽 신청 여부
- [ ] 고객사 개발 환경 접속 방법 (VPN, VDI, 망분리)
- [ ] 변경 요청(CR) 처리 절차 — 구두 요청을 그냥 반영하면 안 됨
- [ ] 검수 기준과 하자보수 범위
- [ ] 성능 요구사항(TPS, 응답시간)이 계약서에 명시되어 있는지

---

## 14. 자바 개발자가 자주 밟는 지뢰

1. **`async def` 안에서 블로킹 코드 호출** → 서버 전체 성능 붕괴. 1순위.
2. **`@Transactional`이 없다는 걸 잊음** → 중간에 실패해도 앞의 INSERT가 커밋되어 데이터 깨짐.
3. **DI 컨테이너가 없음** → Spring처럼 싱글턴 Bean을 기대하면 안 됨. 모듈 로드 시점 부작용 주의.
4. **Pydantic v1/v2 혼동** → 인터넷 예제 그대로 붙이면 안 돌아감. 프로젝트 버전 먼저 확인.
5. **mutable default argument** (`def f(x=[])`) — Python 고전 함정.
6. **타입 힌트는 런타임에 강제되지 않음** → `int`라고 써도 str이 들어올 수 있음. Pydantic이 검증하는 구간에서만 안전.
7. **패키지 버전 미고정** → 로컬은 되는데 운영에서 안 됨.
8. **N+1 쿼리** → JPA 습관대로 lazy 접근하면 SQLAlchemy에서 `DetachedInstanceError`까지 터짐.
9. **동시성 모델 오해** → 워커 4개면 인스턴스도 4벌. 전역 변수로 상태 공유 불가.

---

## 15. 투입 첫 주 액션 플랜

| 일차 | 할 일 |
|---|---|
| 1일차 | 로컬 실행 성공 + `/docs`로 전체 API 목록 파악 + 요구사항 정의서 훑기 |
| 2일차 | 디렉토리 구조 / 라우터 등록 흐름 / 설정 로딩 방식 파악 |
| 3일차 | DB 스키마 + ORM 모델 + **트랜잭션 경계** 확인 |
| 4일차 | 대표 기능 1개를 라우터→서비스→리포지토리→DB까지 끝까지 추적 (`Ctrl+클릭` 타고 내려가기) |
| 5일차 | 인증/인가 흐름 + 외부 연동 목록 정리, 발견한 리스크를 PL에게 보고 |

---

## 16. 파악하며 기록할 것

- [ ] API 목록과 담당 모듈 매핑표
- [ ] DB 테이블 관계도 (간단히라도)
- [ ] 외부 연동 인터페이스 목록
- [ ] 발견한 기술 부채 / 리스크 목록 (**투입 초기에 문서화해두면 나중에 책임 소재에서 나를 보호함**)
- [ ] 모르는 것 질문 리스트 (한 번에 모아서 물어보기)

---

## 참고

- FastAPI 공식 문서: https://fastapi.tiangolo.com
- SQLAlchemy 2.0 튜토리얼: https://docs.sqlalchemy.org/en/20/tutorial/
- Pydantic v2 마이그레이션 가이드: https://docs.pydantic.dev/latest/migration/
