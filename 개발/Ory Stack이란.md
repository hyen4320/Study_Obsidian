# Ory Stack이란

> 2026-09-23 정리

## 결론

**Ory Stack은 인증·인가를 직접 만들지 않고 컨테이너로 띄워 쓰는 오픈소스 신원(Identity) 인프라이며, Kratos(로그인) · Keto(권한) · Hydra(OAuth2/OIDC) · Oathkeeper(게이트웨이) 4개 중 필요한 것만 골라 Docker Compose로 구축한다.**

- 로그인만 필요하면 **Kratos 하나로 충분**
- 서화처럼 FastAPI + Next.js 구조라면 **Kratos + Keto** 조합으로 시작하는 것이 적절
- 전부 Go로 작성된 단일 바이너리, 설정은 YAML, 저장소는 PostgreSQL/MySQL/CockroachDB

---

## 1. 구성 요소

| 구성 요소 | 역할 | 필요한 경우 | 기본 포트 (공개 / 관리) |
| --- | --- | --- | --- |
| **Kratos** | 회원가입·로그인·세션·MFA·비밀번호 재설정·이메일 인증 | 거의 항상 (핵심) | 4433 / 4434 |
| **Keto** | 권한 관리 (Google Zanzibar 모델) | 역할·리소스별 권한이 필요할 때 | 4466(read) / 4467(write) |
| **Hydra** | OAuth2 · OpenID Connect 인가 서버 | 다른 서비스에 SSO/토큰을 발급할 때 | 4444 / 4445 |
| **Oathkeeper** | 인증·인가를 검사하는 리버스 프록시 (Identity & Access Proxy) | 백엔드마다 인증 코드를 넣지 않고 게이트웨이에서 막을 때 | 4455(proxy) / 4456(api) |

### 한 줄 비유

- Kratos = **"너 누구야?"** (Authentication)
- Keto = **"너 이거 해도 돼?"** (Authorization)
- Hydra = **"다른 앱에 너 대신 출입증 발급"** (OAuth2 Provider)
- Oathkeeper = **"문 앞 경비원"** (위 둘을 요청마다 확인)

---

## 2. 요청 흐름

```mermaid
sequenceDiagram
    participant U as 브라우저
    participant F as 프론트(Next.js)
    participant K as Kratos
    participant B as 백엔드(FastAPI)
    participant P as Keto

    U->>F: 로그인 페이지 접속
    F->>K: 로그인 flow 생성
    U->>K: 아이디/비밀번호 제출
    K-->>U: 세션 쿠키(ory_kratos_session) 발급
    U->>B: API 요청 (쿠키 포함)
    B->>K: /sessions/whoami (쿠키 전달)
    K-->>B: identity.id 반환
    B->>P: check(blueprint:X#editor@user:A)
    P-->>B: allowed true/false
    B-->>U: 응답
```

---

## 3. 구축 순서

1. **DB 준비** — 구성 요소별 DB를 따로 생성 (`kratos`, `keto` …). 같은 PostgreSQL 서버여도 됨
2. **마이그레이션** — 각 구성 요소를 `migrate sql -e --yes`로 먼저 실행해 스키마 생성
3. **Kratos 설정**
   - `kratos.yml` : 공개/관리 URL, 쿠키 도메인, 셀프서비스 UI 경로, SMTP(courier)
   - `identity.schema.json` : 사용자 속성(email, name 등) 정의
4. **로그인 UI** — 공식 `oryd/kratos-selfservice-ui-node`로 동작 확인 → 이후 Next.js에서 `@ory/client`로 직접 구현
5. **백엔드 연동** — FastAPI 미들웨어에서 요청 쿠키를 그대로 Kratos `/sessions/whoami`에 전달, 응답의 `identity.id`를 사용자 ID로 사용
6. **(선택) Keto** — 관계 튜플 등록 후 API마다 `/relation-tuples/check` 호출

---

## 4. 최소 docker-compose (Kratos + Keto)

```yaml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_USER: ory
      POSTGRES_PASSWORD: secret
    volumes: [pgdata:/var/lib/postgresql/data]

  kratos-migrate:
    image: oryd/kratos:<버전>
    command: migrate sql -e --yes
    environment:
      DSN: postgres://ory:secret@postgres:5432/kratos?sslmode=disable
    depends_on: [postgres]

  kratos:
    image: oryd/kratos:<버전>
    command: serve -c /etc/config/kratos/kratos.yml --dev --watch-courier
    environment:
      DSN: postgres://ory:secret@postgres:5432/kratos?sslmode=disable
    volumes: [./kratos:/etc/config/kratos]
    ports: ["4433:4433", "4434:4434"]   # 4433 공개 / 4434 관리
    depends_on: [kratos-migrate]

  keto:
    image: oryd/keto:<버전>
    command: serve -c /home/ory/keto.yml
    environment:
      DSN: postgres://ory:secret@postgres:5432/keto?sslmode=disable
    volumes: [./keto/keto.yml:/home/ory/keto.yml]
    ports: ["4466:4466", "4467:4467"]   # 4466 조회 / 4467 쓰기

volumes:
  pgdata:
```

> `<버전>`은 공식 문서 기준으로 고정. `latest` 금지.

---

## 5. 백엔드 연동 예시 (FastAPI)

```python
import httpx
from fastapi import Request, HTTPException

KRATOS_PUBLIC = "http://kratos:4433"

async def current_user(request: Request) -> str:
    async with httpx.AsyncClient() as c:
        r = await c.get(
            f"{KRATOS_PUBLIC}/sessions/whoami",
            headers={"cookie": request.headers.get("cookie", "")},
        )
    if r.status_code != 200:
        raise HTTPException(401, "로그인 필요")
    return r.json()["identity"]["id"]
```

---

## 6. Keto 권한 모델 (Zanzibar)

**관계 튜플 형식**: `namespace:object#relation@subject`

| 예시 | 의미 |
| --- | --- |
| `blueprint:bp-01#owner@user:A` | 사용자 A는 청사진 bp-01의 소유자 |
| `blueprint:bp-01#viewer@group:dev#member` | dev 그룹 멤버는 bp-01 조회 가능 |
| `group:dev#member@user:B` | 사용자 B는 dev 그룹 멤버 |

→ `check(blueprint:bp-01#viewer@user:B)` = **true** (그룹 경유 상속)

---

## 7. 주의할 점

- **관리 포트(4434 · 4445 · 4467)는 외부 노출 금지** — 내부망에서만 접근
- **쿠키 도메인 일치** — 프론트와 Kratos가 같은 최상위 도메인 아래 있어야 세션 쿠키 공유. IP 직접 접속(예: `192.168.0.163`) 환경에서 쿠키 문제가 잦으므로 도메인 또는 리버스 프록시로 한 오리진에 묶는 것이 안전
- **`--dev` 플래그는 개발 전용** — 운영에서는 제거하고 HTTPS, `secrets.cookie`, `secrets.cipher` 설정
- **Self-hosted vs Ory Network** — 같은 API를 Ory가 호스팅하는 SaaS(Ory Network)로도 제공. 직접 운영 부담이 싫으면 SaaS 선택지 있음
- **Kratos는 UI를 제공하지 않음** — headless API만 있으므로 로그인 화면은 직접 만들어야 함

---

## 8. 용어

| 용어 | 뜻 |
| --- | --- |
| Identity | Kratos에 저장된 사용자 1명 (ID + traits) |
| Traits | identity schema로 정의한 사용자 속성 |
| Flow | 로그인·가입·복구 등 한 번의 셀프서비스 과정 (ID로 추적) |
| Session | 로그인 후 발급되는 인증 상태 (쿠키 또는 토큰) |
| Relation Tuple | Keto의 권한 한 줄 (`객체#관계@주체`) |
| Zanzibar | Google의 전역 권한 시스템 논문. Keto의 모델 기반 |

---

## 참고

- [Running Ory with Docker](https://www.ory.com/docs/integrates-with/containerization/docker)
- [Kratos Quickstart](https://www.ory.com/docs/kratos/quickstart)
- [Run Ory Hydra in Docker](https://www.ory.com/docs/hydra/self-hosted/configure-deploy)
- [Ory Hydra Quickstart](https://www.ory.com/docs/oel/hydra/quickstart)
- [ory-reference-compose (GitHub)](https://github.com/radekg/ory-reference-compose)
- [kratos-selfservice-ui-node (Docker Hub)](https://hub.docker.com/r/oryd/kratos-selfservice-ui-node)
