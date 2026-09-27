---
name: be-init
description: FastAPI + SQLAlchemy 2.0 + Python 3.12 + Hexagonal Architecture 10-패키지 uv workspace 백엔드 인프라를 스캐폴드한다. 도메인 코드는 만들지 않고(모든 도메인은 be-planner / add-* 스킬이 담당) 빌드·설정·감사·예외·트랜잭션 계약·로깅·DI 컨테이너·테스트 기반만 세팅한다. 생성 직후 `uv sync --all-groups && uv run python -c "import {pkg}.app.main as m; m.create_app()" && uv run ruff check . && uv run lint-imports && uv run mypy` 로 검증하고, 실패 시 원인 분류표 기반으로 최대 3회 자가 수정한다. 재실행(루트 pyproject.toml 에 `[tool.uv.workspace]` 존재) 시 즉시 abort.
---

# be-init — 백엔드 인프라 스캐폴더

## 1. 스킬 경계 (만드는 것 / 만들지 않는 것)

만드는 것:
- 10개 uv workspace 패키지 (`app`, `api`, `service`, `persistence`, `infrastructure`, `transaction`, `my_logging`, `model`, `schema`, `exception`) + 루트 `pyproject.toml` + `uv.lock`
- `app/main.py` (`create_app()`), `lifespan.py`, `settings.py`, `di.py`, `routers.py`, `error_handlers.py`, `error_response.py`, `middleware/request_logging.py`, `providers/__init__.py` (도메인 Provider 가 들어갈 빈 서브패키지)
- `transaction/manager.py`, `transaction/decorator.py` — `TransactionManager` ABC + `NoopTransactionManager` + `@transactional`
- `my_logging/config.py`, `context.py`, `render.py` — structlog 기반 JSON 한 줄 로깅과 요청 컨텍스트
- Auditing 컨트랙트: `AuditFields` Protocol (abstract BaseEntity 아님 — 이유는 §4.6)
- 트랜잭션 컨트랙트: `TransactionManager` ABC + `transactional` 데코레이터 + `NoopTransactionManager` (transaction), `DefaultTransactionManager` (persistence)
- 로깅 기반: `my_logging` (structlog 엔진, JSON 한 줄, 요청 컨텍스트 키) + `RequestLoggingMiddleware` (app)
- SQLAlchemy 기반: `Base`, `utcnow`, `entities.py` (Entity 레지스트리), `session.py` (엔진·세션 팩토리, `autobegin=False`)
- DI: dishka `AppProvider` (APP 스코프) + `RequestProvider` (REQUEST 스코프) + `DOMAIN_PROVIDERS` 레지스트리
- 예외 계층: `DomainException` + `HttpException` Protocol + 구체 5종 (`EntityNotFoundException` abstract / `ConflictException` / `ForbiddenException` / `InvalidInputException` / `UnauthorizedException`)
- 인증 stub: `CurrentUser` + `resolve_current_user` (local 환경에서만 `X-User-Id` 헤더 허용)
- Alembic: `alembic.ini`, `env.py` (FK 제약 제외 필터), 빈 `versions/`
- 테스트 기반: 루트 `conftest.py` (testcontainers Postgres, Alembic 적용, 세이브포인트 세션, truncate 픽스처)
- `docker-compose.yml` (Postgres), `.env.example`, `.gitignore`, `README.md` (최소)
- 각 패키지 `src/{pkg}/{layer}/__init__.py` + `tests/` 디렉터리

만들지 않는 것:
- 도메인 코드, Entity, Repository, Reader/Writer, Router → be-planner / add-* 담당
- 실제 인증 (OAuth, 세션, JWT) → stub 만 제공
- Elasticsearch, Redis, Kafka
- Dockerfile, CI 설정
- 샘플 migration (`versions/` 는 비어 있음)

## 2. 입력 파라미터

`${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차를 따른다. `{projectRoot}/dk-py-bundle.config.json`이 있으면 그것을 읽고, 없으면 사용자에게 묻는다.

| key | 예시 | 용도 |
| --- | --- | --- |
| `projectName` | `share-ai-setup-be` | 루트 `[project] name`, 패키지 배포명 접두어, FastAPI title |
| `pkg` | `share_ai_setup` | Python import 루트 (네임스페이스 패키지 이름) |
| `projectRoot` | `/Users/.../share-ai-setup-be` | 생성 위치 |

파생값:
- `dbName = projectName.replace('-', '_')` — Postgres DB 이름 (하이픈 불가)
- `{projectName}-{layer}` — 각 패키지 배포명 (예: `share-ai-setup-be-model`)

**사전 불변식 검사**: `pkg`가 유효한 Python 식별자(`str.isidentifier()`)이고 소문자·밑줄만 포함해야 파일 생성 시작. 깨지면 abort.

**재실행 가드**: `{projectRoot}/pyproject.toml`이 존재하고 `[tool.uv.workspace]` 섹션이 있으면 즉시 abort (도메인 덮어쓰기 방지).

## 3. 패키지 구조 & 의존성 그래프

```
app                    → api, service, persistence, infrastructure, schema, transaction, my_logging, exception, model
schema                 → persistence
api                    → service, my_logging, model, exception
service                → infrastructure, transaction, my_logging, model, exception
persistence            → infrastructure, transaction, my_logging, model, exception
infrastructure         → model, exception
transaction            → (pure)
my_logging             → structlog 만
model                  → (pure)
exception              → (pure)
```

핵심 판단:
- **infrastructure 는 port 패키지** — Repository ABC가 여기에 산다.
- **transaction 은 트랜잭션 계약 패키지** — `TransactionManager` ABC, `NoopTransactionManager`, `@transactional` 데코레이터. 순수 Python 이고 다른 패키지를 모른다. Java 에서 service 가 spring-tx 만 의존하던 자리.
- **my_logging 은 로깅 패키지** — 로그 형식(JSON 한 줄)과 요청 컨텍스트 키를 한 곳에서 관리한다. 엔진은 structlog 이고 도메인 코드는 `logging.getLogger(__name__)` 를 그대로 쓴다.
- `service → transaction`: Reader/Writer/Usecase가 `*Repository` port(infrastructure)와 `TransactionManager`(transaction)에 의존. 구현체는 모름.
- `persistence → infrastructure, transaction`: `Default{Domain}Repository`가 port를 구현, `DefaultTransactionManager`가 트랜잭션 계약을 구현.
- `app`이 service(포트 사용자) + persistence(port 구현체 `Default{Domain}Repository` + 쿼리 클래스 `{Domain}EntityQuery` / `{Domain}JoinQuery`)를 dishka provider로 묶는다. 전형적 port-adapter 패턴.
- `schema → persistence`: Alembic `env.py`가 `Base.metadata`를 읽어야 한다.
- **의존 방향 강제는 두 겹**: 각 패키지 `pyproject.toml`의 `dependencies`(설치 시점) + 루트 import-linter 계약(검증 시점). uv workspace는 venv 하나를 공유하므로 물리적으로는 어느 패키지든 import할 수 있어서, import-linter가 실제 강제 수단이다.
- import-linter 계약은 두 개다. `layers` 는 패키지 사이 방향을, `forbidden` 은 persistence / schema / app 밖에서 `sqlalchemy` / `psycopg` / `alembic` 을 import 하는 것을 막는다. 두 번째 계약이 없으면 service 가 `from sqlalchemy import ...` 를 해도 layers 계약은 통과한다 (실행으로 확인).
- HTTP 클라이언트·외부 IO 어댑터는 별도 패키지(예: `http_adapter`)를 추가할 수 있으나 be-init 범위 밖.

## 4. 파일 스펙 (모두 구체적 내용 포함)

### 4.1 루트 `pyproject.toml`

```toml
[project]
name = "{projectName}"
version = "0.1.0"
description = "FastAPI / SQLAlchemy 2.0 / Postgres Hexagonal multi-package backend"
requires-python = ">=3.12"
dependencies = [
    "{projectName}-app",
]

[tool.uv]
package = false

[tool.uv.workspace]
members = ["packages/*"]

[tool.uv.sources]
"{projectName}-model" = { workspace = true }
"{projectName}-exception" = { workspace = true }
"{projectName}-transaction" = { workspace = true }
"{projectName}-my-logging" = { workspace = true }
"{projectName}-infrastructure" = { workspace = true }
"{projectName}-persistence" = { workspace = true }
"{projectName}-service" = { workspace = true }
"{projectName}-api" = { workspace = true }
"{projectName}-schema" = { workspace = true }
"{projectName}-app" = { workspace = true }

[dependency-groups]
dev = [
    "pytest>=8.3",
    "httpx>=0.27",
    "testcontainers[postgres]>=4.8",
    "ruff>=0.6",
    "import-linter>=2.1",
    "mypy>=1.11",
]

[tool.ruff]
line-length = 100
target-version = "py312"
src = ["packages/*/src"]

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP"]

[tool.ruff.lint.isort]
known-first-party = ["{pkg}"]

[tool.mypy]
python_version = "3.12"
files = ["packages/*/src"]
mypy_path = "packages/model/src:packages/exception/src:packages/transaction/src:packages/my_logging/src:packages/infrastructure/src:packages/persistence/src:packages/service/src:packages/api/src:packages/schema/src:packages/app/src"
namespace_packages = true
explicit_package_bases = true
check_untyped_defs = true
warn_unused_ignores = true
ignore_missing_imports = true

[tool.pytest.ini_options]
testpaths = ["packages"]
python_files = ["test_*.py"]
addopts = "--import-mode=importlib"
filterwarnings = ["error::sqlalchemy.exc.SAWarning"]

[tool.importlinter]
root_package = "{pkg}"
include_external_packages = true

[[tool.importlinter.contracts]]
name = "hexagonal layers"
type = "layers"
layers = [
    "{pkg}.app",
    "{pkg}.schema",
    "{pkg}.api | {pkg}.persistence",
    "{pkg}.service",
    "{pkg}.infrastructure",
    "{pkg}.model | {pkg}.exception | {pkg}.transaction | {pkg}.my_logging",
]

[[tool.importlinter.contracts]]
name = "no db libraries outside persistence / schema / app"
type = "forbidden"
source_modules = [
    "{pkg}.model",
    "{pkg}.exception",
    "{pkg}.transaction",
    "{pkg}.my_logging",
    "{pkg}.infrastructure",
    "{pkg}.service",
    "{pkg}.api",
]
forbidden_modules = ["sqlalchemy", "psycopg", "alembic"]
```

설계 근거:
- `package = false`: 루트는 가상 프로젝트. 코드가 없고 workspace를 묶기만 한다. Java 루트 `build.gradle.kts`에 boot 플러그인을 `apply false`로 둔 것과 같은 의도.
- `[tool.uv.sources] { workspace = true }`: 멤버 간 의존을 PyPI가 아니라 workspace에서 해석. Java `project(":modules:model")`에 해당.
- dev 그룹에 테스트·린트·타입 도구를 모아 Java 루트 `subprojects { testImplementation(...) }` 역할.
- `filterwarnings = ["error::sqlalchemy.exc.SAWarning"]`: SQLAlchemy 경고(예: relationship 설정 오류, 세션 상태 경고)를 테스트 실패로 승격. 조용한 매핑 버그 예방.
- `addopts = "--import-mode=importlib"`: 패키지별 `tests/` 에 `__init__.py` 가 없어 같은 파일명(`conftest.py`, `test_xxx.py`)이 여러 패키지에 생기면 기본 `prepend` 모드가 모듈명 충돌을 낸다. importlib 모드는 경로 기반이라 충돌이 없다. 같은 디렉터리의 `conftest` 헬퍼는 `from .conftest import ...` 로 import 한다 (실행으로 확인).
- import-linter `layers` 계약의 `|`는 같은 층의 독립 패키지. `api`와 `persistence`는 서로 import 금지. 맨 아래 층의 `transaction` / `my_logging` 도 서로와 `model` / `exception` 을 import 하지 않는다.
- `forbidden` 계약은 외부 라이브러리를 대상으로 하므로 `include_external_packages = true` 가 있어야 한다. 없으면 계약이 조용히 아무것도 검사하지 않는다.
- mypy는 `check_untyped_defs`만 켠 완화 모드. 생성 코드에 과한 타입 부담을 주지 않으면서 dataclass/ABC 시그니처 불일치는 잡는다. (pyright와 비교 후 mypy 채택: 순수 Python이라 도구 하나 덜 필요)
- mypy는 **소스 경로 기반**(`files` + `mypy_path` + `explicit_package_bases`)으로 돈다. `packages = ["{pkg}"]` 처럼 설치 패키지로 지정하면 네임스페이스 패키지에 `py.typed` 마커를 둘 곳이 없어 "cannot be type checked due to missing py.typed marker" 로 실패한다 (실행으로 확인). 레이어별 `py.typed` 도 소용없다.

### 4.2 패키지 `pyproject.toml` 10개

공통 골격 (모든 패키지 동일, `dependencies`만 다름):

```toml
[project]
name = "{projectName}-{layer-dashed}"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [ ... ]

[build-system]
requires = ["hatchling>=1.25"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/{pkg}"]
```

`{layer-dashed}`는 `my_logging → my-logging`, 나머지는 그대로 (PyPI 배포명은 밑줄 대신 하이픈). import 이름은 `{pkg}.my_logging` 으로 밑줄 그대로다.

패키지별 `dependencies`:

`packages/model`:
```toml
dependencies = []
```

`packages/exception`:
```toml
dependencies = []
```

`packages/transaction`:
```toml
dependencies = []
```

`packages/my_logging`:
```toml
dependencies = ["structlog>=24.1"]
```

`packages/infrastructure`:
```toml
dependencies = [
    "{projectName}-model",
    "{projectName}-exception",
]
```

`packages/persistence`:
```toml
dependencies = [
    "{projectName}-infrastructure",
    "{projectName}-transaction",
    "{projectName}-my-logging",
    "{projectName}-model",
    "{projectName}-exception",
    "sqlalchemy>=2.0.30",
    "psycopg[binary]>=3.2",
]
```

`packages/service`:
```toml
dependencies = [
    "{projectName}-infrastructure",
    "{projectName}-transaction",
    "{projectName}-my-logging",
    "{projectName}-model",
    "{projectName}-exception",
]
```

`packages/api`:
```toml
dependencies = [
    "{projectName}-service",
    "{projectName}-my-logging",
    "{projectName}-model",
    "{projectName}-exception",
    "fastapi>=0.115",
    "pydantic>=2.9",
    "dishka>=1.4",
]
```

`packages/schema`:
```toml
dependencies = [
    "{projectName}-persistence",
    "alembic>=1.13",
    "python-dotenv>=1.0",
]
```

`packages/app`:
```toml
dependencies = [
    "{projectName}-api",
    "{projectName}-service",
    "{projectName}-persistence",
    "{projectName}-infrastructure",
    "{projectName}-transaction",
    "{projectName}-my-logging",
    "{projectName}-schema",
    "{projectName}-exception",
    "{projectName}-model",
    "fastapi>=0.115",
    "uvicorn[standard]>=0.30",
    "pydantic-settings>=2.5",
    "dishka>=1.4",
]
```

**네임스페이스 패키지**: 각 패키지의 `src/{pkg}/` 에는 `__init__.py`를 **두지 않는다** (PEP 420). `src/{pkg}/{layer}/__init__.py`만 둔다. 열 패키지가 하나의 `{pkg}` 이름 아래로 합쳐져 `{pkg}.model`, `{pkg}.service` 로 import된다. hatchling editable 설치가 각 `src/`를 `.pth`로 등록하므로 동작한다.

**Java `api(project(...))` vs `implementation(project(...))` 구분**: Python은 전이 import가 기본이라 구분이 없다. 대신 import-linter가 "허용된 층 아래만 import" 를 강제한다.

### 4.3 `app` 패키지

경로: `packages/app/src/{pkg}/app/`

#### `settings.py`

```python
from typing import Literal

from pydantic_settings import BaseSettings, SettingsConfigDict

from {pkg}.my_logging import LogFormat


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="APP_", env_file=".env", extra="ignore")

    env: Literal["local", "test", "prod"] = "local"
    database_url: str = "postgresql+psycopg://app:app@localhost:5432/{dbName}"
    db_pool_size: int = 20
    db_max_overflow: int = 20
    db_echo: bool = False
    log_level: str = "INFO"
    log_format: LogFormat | None = None  # None 이면 env 로 결정

    @property
    def effective_log_format(self) -> LogFormat:
        if self.log_format is not None:
            return self.log_format
        return "json-color" if self.env == "local" else "json"
```

`log_format` 을 비워 두면 `local` 은 `json-color`(터미널에서 키에 색), 나머지는 `json`(유효한 JSON 한 줄). `APP_LOG_FORMAT=json` 으로 언제든 덮어쓴다.

#### `error_response.py`

```python
from pydantic import BaseModel


class ErrorResponse(BaseModel):
    status: int
    error: str
    message: str
```

#### `error_handlers.py`

```python
import logging

from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

from {pkg}.exception.domain_exception import DomainException

from .error_response import ErrorResponse

logger = logging.getLogger(__name__)


def register_error_handlers(app: FastAPI) -> None:
    @app.exception_handler(RequestValidationError)
    def handle_validation(request: Request, exc: RequestValidationError) -> JSONResponse:
        message = ", ".join(
            f"{'.'.join(str(p) for p in err['loc'])}: {err['msg']}" for err in exc.errors()
        )
        logger.info("Validation failed: %s", message)
        body = ErrorResponse(status=400, error="ValidationFailed", message=message)
        return JSONResponse(status_code=400, content=body.model_dump())

    @app.exception_handler(DomainException)
    def handle_domain(request: Request, exc: DomainException) -> JSONResponse:
        status = getattr(exc, "http_status_code", None)
        client_message = getattr(exc, "http_error_message", None)
        if status is not None and client_message is not None:
            logger.warning(
                "Domain error [%s] - %s: %s", status, type(exc).__name__, exc.detail
            )
            body = ErrorResponse(status=status, error=type(exc).__name__, message=client_message)
            return JSONResponse(status_code=status, content=body.model_dump())
        logger.error("Unhandled domain error - %s: %s", type(exc).__name__, exc.detail, exc_info=exc)
        body = ErrorResponse(status=500, error=type(exc).__name__, message="Internal server error")
        return JSONResponse(status_code=500, content=body.model_dump())

    @app.exception_handler(Exception)
    def handle_unexpected(request: Request, exc: Exception) -> JSONResponse:
        logger.error("Unexpected error: %s", exc, exc_info=exc)
        body = ErrorResponse(status=500, error="InternalServerError", message="Internal server error")
        return JSONResponse(status_code=500, content=body.model_dump())
```

검증 실패를 **400**으로 내린다 (FastAPI 기본 422 아님). Java 번들과 API 계약을 같게 두기 위한 결정. 422를 원하면 `status=400` 두 곳만 바꾼다.

#### `di.py`

```python
from collections.abc import Iterator

from dishka import Container, Provider, Scope, from_context, make_container, provide
from fastapi import Request
from sqlalchemy import Engine
from sqlalchemy.orm import Session, sessionmaker

from {pkg}.api.auth import CurrentUser, resolve_current_user
from {pkg}.my_logging import set_context
from {pkg}.persistence.session import create_db_engine, create_session_factory
from {pkg}.persistence.transaction import DefaultTransactionManager
from {pkg}.transaction import TransactionManager

from .settings import Settings


class AppProvider(Provider):
    """앱 수명(싱글턴). Spring 싱글턴 빈에 해당하는 것만 둔다."""

    scope = Scope.APP

    @provide
    def settings(self) -> Settings:
        return Settings()

    @provide
    def engine(self, settings: Settings) -> Engine:
        return create_db_engine(
            settings.database_url,
            pool_size=settings.db_pool_size,
            max_overflow=settings.db_max_overflow,
            echo=settings.db_echo,
        )

    @provide
    def session_factory(self, engine: Engine) -> sessionmaker[Session]:
        return create_session_factory(engine)


class RequestProvider(Provider):
    """요청 수명. Session 과 TransactionManager 는 요청당 하나."""

    scope = Scope.REQUEST

    request = from_context(provides=Request)

    @provide
    def session(self, factory: sessionmaker[Session]) -> Iterator[Session]:
        with factory() as session:  # close() 가 미커밋 내용을 롤백
            yield session

    @provide
    def transaction_manager(self, session: Session) -> TransactionManager:
        return DefaultTransactionManager(session)

    @provide
    def current_user(self, request: Request, settings: Settings) -> CurrentUser:
        user = resolve_current_user(request, settings.env)
        set_context("user_id", user.user_id)  # 이후 이 요청의 모든 로그 줄에 붙는다
        return user


# add-usecase 스킬이 app/providers/{domain}_provider.py 를 만들고 여기에 등록한다 (C-10).
DOMAIN_PROVIDERS: list[Provider] = []


def build_container(*extra: Provider) -> Container:
    return make_container(AppProvider(), RequestProvider(), *DOMAIN_PROVIDERS, *extra)
```

설계 근거:
- APP 스코프 객체는 첫 요청 시 지연 생성된다. `create_app()` 만으로는 DB에 붙지 않으므로 검증 명령이 DB 없이 돈다.
- `Session` provider가 제너레이터라 요청 종료 시 `with` 블록이 닫히며 `close()`가 호출된다. 여기서 커밋하지 않는다. 커밋 경계는 `@transactional`.
- `DOMAIN_PROVIDERS` 는 Java 컴포넌트 스캔의 대체. 자동 스캔이 없으므로 add-* 스킬이 한 줄씩 등록한다.
- `build_container(*extra)` 의 `extra`는 테스트에서 provider 를 덮어쓸 때 쓴다 (dishka는 뒤에 온 provider 가 우선).

#### `routers.py`

```python
from fastapi import APIRouter, FastAPI

# add-api 스킬이 도메인별 router 를 여기에 등록한다.
ROUTERS: list[APIRouter] = []


def register_routers(app: FastAPI) -> None:
    for router in ROUTERS:
        app.include_router(router)
```

#### `lifespan.py`

```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from fastapi import FastAPI
from sqlalchemy import Engine


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    yield
    container = app.state.dishka_container
    container.get(Engine).dispose()  # 풀의 커넥션 정리 (컨테이너 close 는 dispose 를 부르지 않는다)
    container.close()
```

`lifespan`은 FastAPI 규약상 `async def`여야 한다. 앱 시작·종료 훅일 뿐 요청 처리 경로가 아니므로 `def + Session` 결정과 충돌하지 않는다.

`container.close()` 는 제너레이터 provider 의 finalizer 만 돌린다. `Engine` 은 일반 `@provide` 라 finalizer 가 없어서, 종료 시 풀에 남은 커넥션을 닫으려면 `engine.dispose()` 를 직접 불러야 한다. 아직 첫 요청이 없었으면 `container.get(Engine)` 이 이 시점에 Engine 을 만들고 바로 dispose 하므로 무해하다.

#### `middleware/request_logging.py`

```python
"""HTTP 요청 1회 = 로그 컨텍스트 1회.

순수 ASGI 미들웨어. 시작에 request_id / method / path 를 바인딩하고,
끝에 status / duration_ms 한 줄을 남기고 clear 한다.
send 를 감싸 http.response.start 에서 상태 코드를 잡는다.
잡히지 않은 예외는 500 으로 기록하고 재전파한다 (응답은 바깥 ServerErrorMiddleware 가 만든다).
"""

import time
import uuid
from collections.abc import Awaitable, Callable
from typing import Any

from {pkg}.my_logging import clear, get_logger, set_context

Receive = Callable[[], Awaitable[Any]]
Send = Callable[[Any], Awaitable[None]]
ASGIApp = Callable[[dict[str, Any], Receive, Send], Awaitable[None]]

REQUEST_ID_HEADER = b"x-request-id"


class RequestLoggingMiddleware:
    def __init__(self, app: ASGIApp) -> None:
        self.app = app

    async def __call__(
        self,
        scope: dict[str, Any],
        receive: Receive,
        send: Send,
    ) -> None:
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        headers = dict(scope.get("headers") or [])
        request_id = headers.get(REQUEST_ID_HEADER, b"").decode() or uuid.uuid4().hex[:16]
        method, path = scope["method"], scope["path"]

        clear()
        set_context("request_id", request_id)
        set_context("method", method)
        set_context("path", path)

        status: dict[str, int | None] = {"code": None}
        started = time.perf_counter()

        async def send_wrapped(message: Any) -> None:
            if message["type"] == "http.response.start":
                status["code"] = message["status"]
                out = list(message.get("headers", []))
                out.append((REQUEST_ID_HEADER, request_id.encode()))
                message["headers"] = out
            await send(message)

        try:
            await self.app(scope, receive, send_wrapped)
        except Exception:
            status["code"] = 500
            raise
        finally:
            code = status["code"] or 500
            duration_ms = round((time.perf_counter() - started) * 1000, 1)
            log = get_logger("http")
            emit = log.error if code >= 500 else log.info
            emit("http.request", status=code, duration_ms=duration_ms)
            clear()
```

`app/middleware/__init__.py` 는 빈 파일.

설계 근거:
- `BaseHTTPMiddleware` 대신 순수 ASGI. `BaseHTTPMiddleware` 는 요청 본문을 스트림으로 한 번 더 감싸고 contextvars 전파에 알려진 문제가 있다.
- 미들웨어가 `async def` 인 것은 ASGI 규약이다. `def` 핸들러는 anyio 워커 스레드에서 돌지만 contextvars 는 그 스레드로 복사되어 따라가므로 핸들러·Usecase·Repository 의 로그 줄에 `request_id` 가 그대로 붙는다 (실행으로 확인).
- 응답 헤더에 `x-request-id` 를 돌려주므로 클라이언트가 같은 값으로 문의할 수 있다. 요청에 이미 있으면 그 값을 쓴다.
- uvicorn 의 액세스 로그는 `configure_logging` 이 끈다. 이 미들웨어의 `http.request` 한 줄이 그 자리를 대신한다.

#### `main.py`

```python
from dishka.integrations.fastapi import setup_dishka
from fastapi import FastAPI

from {pkg}.my_logging import configure_logging

from .di import build_container
from .error_handlers import register_error_handlers
from .lifespan import lifespan
from .middleware.request_logging import RequestLoggingMiddleware
from .routers import register_routers
from .settings import Settings


def create_app() -> FastAPI:
    settings = Settings()
    configure_logging(settings.log_level, settings.effective_log_format)  # 가장 먼저

    app = FastAPI(title="{projectName}", lifespan=lifespan)
    app.add_middleware(RequestLoggingMiddleware)
    register_error_handlers(app)
    register_routers(app)
    setup_dishka(build_container(), app)
    return app
```

`configure_logging` 은 `create_app()` 의 첫 줄이다. 그 뒤에 만들어지는 모든 로거(uvicorn 포함)가 같은 핸들러를 타야 하기 때문이다. `Settings()` 를 여기서 한 번 더 만드는 것은 DI 컨테이너가 아직 없어서이며, APP 스코프의 `Settings` provider 와 값은 같다.

실행: `uv run uvicorn {pkg}.app.main:create_app --factory --reload`

### 4.4 `model` 패키지 — `AuditFields` Protocol

경로: `packages/model/src/{pkg}/model/audit.py`

```python
from datetime import UTC, datetime
from typing import Protocol


def utcnow() -> datetime:
    """도메인·Entity 가 공통으로 쓰는 현재 시각. tz-aware UTC."""
    return datetime.now(UTC)


class AuditFields(Protocol):
    created_at: datetime
    updated_at: datetime
```

`utcnow()`를 model 에 두는 이유: 도메인 팩토리(`create()`, `with_xxx()`)가 `created_at`/`updated_at`을 채워야 하는데 model 은 순수 패키지라 다른 곳에서 가져올 수 없다. Entity(`persistence/base.py`)도 같은 함수를 import 해 시각 소스를 하나로 맞춘다.

**기술적 근거**: 도메인 모델은 `frozen dataclass`, Entity는 SQLAlchemy 매핑 클래스라 공통 부모 클래스를 둘 수 없다. 구조적 타입(Protocol)으로 "두 필드가 있다"는 계약만 건다. Java에서 Spring Data JDBC가 `@MappedSuperclass`를 지원하지 않아 인터페이스로 간 것과 같은 결론이다.

다운스트림 사용 예 (예시만, 생성하지 않음):
```python
@dataclass(frozen=True, slots=True)
class Article:            # AuditFields 를 명시 상속하지 않아도 구조로 만족
    article_id: int | None
    title: str
    created_at: datetime
    updated_at: datetime
```

### 4.5 `exception` 패키지 (2단 구조)

경로: `packages/exception/src/{pkg}/exception/`

`domain_exception.py`:
```python
class DomainException(Exception):
    """모든 도메인 예외의 부모. detail 은 로그용 내부 메시지."""

    def __init__(self, detail: str) -> None:
        super().__init__(detail)
        self.detail = detail
```

`http_exception.py`:
```python
from typing import Protocol


class HttpException(Protocol):
    """HTTP 로 노출될 예외가 갖는 속성. error_handlers 가 이 두 값을 그대로 쓴다."""

    http_status_code: int
    http_error_message: str
```

**구체 예외 5종** (모두 `DomainException`을 상속하고 `HttpException` 구조를 만족):

| 클래스 | 기본 코드 | 기본 메시지 | abstract |
|---|---|---|---|
| `EntityNotFoundException` | 404 | 해당 리소스를 찾을 수 없습니다 | 예. 도메인별 하위 클래스로만 사용 |
| `ConflictException` | 409 | 이미 존재하는 리소스입니다 | 아니오 |
| `ForbiddenException` | 403 | 접근 권한이 없습니다 | 아니오 |
| `InvalidInputException` | 400 | 잘못된 요청입니다 | 아니오 |
| `UnauthorizedException` | 401 | 인증이 필요합니다 | 아니오 |

`entity_not_found_exception.py`:
```python
from .domain_exception import DomainException


class EntityNotFoundException(DomainException):
    """도메인별 하위 클래스(예: ArticleNotFoundException)로만 사용한다. 직접 생성 금지."""

    def __init__(
        self,
        detail: str,
        status_code: int = 404,
        client_message: str = "해당 리소스를 찾을 수 없습니다",
    ) -> None:
        if type(self) is EntityNotFoundException:
            raise TypeError("EntityNotFoundException 은 도메인별 하위 클래스로만 사용한다")
        super().__init__(detail)
        self.http_status_code = status_code
        self.http_error_message = client_message
```

`conflict_exception.py` (`ForbiddenException`, `InvalidInputException`, `UnauthorizedException` 은 코드·메시지만 다른 동일 패턴):
```python
from .domain_exception import DomainException


class ConflictException(DomainException):
    def __init__(
        self,
        detail: str,
        status_code: int = 409,
        client_message: str = "이미 존재하는 리소스입니다",
    ) -> None:
        super().__init__(detail)
        self.http_status_code = status_code
        self.http_error_message = client_message
```

`__init__.py`에서 6개를 re-export 한다:
```python
from .conflict_exception import ConflictException
from .domain_exception import DomainException
from .entity_not_found_exception import EntityNotFoundException
from .forbidden_exception import ForbiddenException
from .http_exception import HttpException
from .invalid_input_exception import InvalidInputException
from .unauthorized_exception import UnauthorizedException

__all__ = [
    "ConflictException",
    "DomainException",
    "EntityNotFoundException",
    "ForbiddenException",
    "HttpException",
    "InvalidInputException",
    "UnauthorizedException",
]
```

**배치 이유**: `error_handlers.py`와 `ErrorResponse`는 `app`에 둔다. `exception` 패키지에 두면 FastAPI 의존이 생겨 순수 패키지가 아니게 된다. `app`에 두면 조립 경계에서 예외↔HTTP 매핑을 하나로 관리할 수 있다. Java와 같은 배치.

`UnauthorizedException`은 Java 번들의 4종에 없던 것이다. §4.8 인증 stub이 401을 던져야 하는데 `fastapi.HTTPException`을 쓰면 `ErrorResponse` 형식을 벗어나므로 추가했다.

### 4.6 `transaction` 패키지 — 트랜잭션 계약

경로: `packages/transaction/src/{pkg}/transaction/`

`manager.py`:
```python
from abc import ABC, abstractmethod
from contextlib import AbstractContextManager, nullcontext
from typing import Any


class TransactionManager(ABC):
    """Spring PlatformTransactionManager 에 해당하는 port. 구현은 persistence."""

    @abstractmethod
    def in_transaction(self) -> bool: ...

    @abstractmethod
    def begin(self) -> AbstractContextManager[Any]:
        """새 트랜잭션. 정상 종료 시 commit, 예외 시 rollback 후 재전파."""

    @abstractmethod
    def begin_nested(self) -> AbstractContextManager[Any]:
        """세이브포인트. Spring REQUIRES_NEW 의 근사치 — 바깥이 롤백되면 같이 롤백된다."""


class NoopTransactionManager(TransactionManager):
    """단위 테스트용. 아무것도 열지 않는다."""

    def in_transaction(self) -> bool:
        return False

    def begin(self) -> AbstractContextManager[Any]:
        return nullcontext()

    def begin_nested(self) -> AbstractContextManager[Any]:
        return nullcontext()
```

`decorator.py`:
```python
from collections.abc import Callable
from functools import wraps
from typing import Any

from .manager import TransactionManager


def transactional[R](method: Callable[..., R]) -> Callable[..., R]:
    """Spring @Transactional(propagation=REQUIRED) 대응.

    - 대상 객체는 self.tx (TransactionManager) 를 가져야 한다.
    - 이미 열린 트랜잭션이 있으면 합류하고, 없으면 새로 열어 메서드 종료 시 commit 한다.
    - 모든 예외에서 rollback 후 재전파한다 (Spring 과 달리 checked/unchecked 구분 없음).
    """

    @wraps(method)
    def wrapper(self: Any, *args: Any, **kwargs: Any) -> R:
        tx = getattr(self, "tx", None)
        if not isinstance(tx, TransactionManager):
            raise AttributeError(
                f"{type(self).__name__}.tx 가 TransactionManager 가 아니다. "
                "생성자에서 self.tx 를 설정해야 @transactional 을 쓸 수 있다."
            )
        if tx.in_transaction():
            return method(self, *args, **kwargs)
        with tx.begin():
            return method(self, *args, **kwargs)

    return wrapper
```

`__init__.py`:
```python
from .decorator import transactional
from .manager import NoopTransactionManager, TransactionManager

__all__ = ["NoopTransactionManager", "TransactionManager", "transactional"]
```

사용하는 쪽은 항상 `from {pkg}.transaction import TransactionManager, transactional` 로 패키지 루트에서 가져온다.

**설계 근거**:
- 계약을 순수 Python 인 별도 패키지 `transaction` 에 두어 `service`가 SQLAlchemy를 모르게 한다. Java에서 service가 spring-tx만 의존하던 것과 같다.
- `infrastructure` 가 아니라 별도 패키지인 이유: `infrastructure` 는 Repository port 가 사는 곳이고 도메인마다 파일이 늘어난다. 트랜잭션 계약은 도메인과 무관한 프레임워크 성격의 코드라 섞이지 않게 뗀다. 배치 · 파이프라인처럼 Repository 없이 트랜잭션만 필요한 소비자도 이 패키지 하나만 의존하면 된다.
- 데코레이터가 `self.tx`를 찾는다. 세션을 전역(스레드 로컬)에서 꺼내는 방식 대신 생성자 주입을 유지한다.
- 데코레이터라 Spring AOP 프록시의 "같은 클래스 내부 호출은 트랜잭션이 안 걸리는" 함정이 없다.
- `def transactional[R](...)` 는 PEP 695 문법. ruff `UP047` 이 `TypeVar` 방식을 거부한다 (`target-version = "py312"`).

### 4.6b `my_logging` 패키지 — JSON 한 줄 로깅과 요청 컨텍스트

경로: `packages/my_logging/src/{pkg}/my_logging/`

역할: 로그 **형식** 과 **컨텍스트 키** 를 한 곳에서 정한다. 엔진은 structlog 이지만 도메인 코드는 structlog 를 import 하지 않는다. `logging.getLogger(__name__)` 를 그대로 쓰면 같은 JSON 형식으로 나간다.

| 함수 | 뜻 |
| --- | --- |
| `configure_logging(level, fmt)` | 표준 logging 루트 핸들러에 structlog 포매터를 건다. `create_app()` 첫 줄에서 한 번 |
| `set_context(key, value)` | **HTTP 요청 1회** 범위 키. 같은 키는 덮어쓴다 (contextvars) |
| `remove(key)` / `clear()` | 요청 범위 키 제거 / 전부 제거 |
| `set_global_context(key, value)` / `remove_global(key)` | 프로세스 전역 키. service · version · env 같은 것 |
| `keys()` | `(전역 키 목록, 요청 범위 키 목록)` |
| `get_logger(name)` | structlog 로거. `logger.info("event", k=v)` 처럼 필드를 키워드로 넣을 때 |
| `log(message, **fields)` | 일반 출력. `print` 대신 이걸 쓴다 (level info) |
| `error(message, **fields)` | 에러 출력. `exc_info=True` 를 주면 traceback 이 `exception` 필드로 붙는다 |
| `info / warning(message, **fields)` | 레벨을 명시하고 싶을 때 |

같은 키가 전역과 요청 범위 양쪽에 있으면 요청 키가 이긴다.

`context.py`:
```python
"""로그 컨텍스트. 요청 범위(contextvars)와 프로세스 전역(dict) 두 층.

- 요청 범위: HTTP 요청 1회. 미들웨어가 시작에 set_context, 끝에 clear.
  contextvars 라서 요청마다 독립이고, anyio 가 def 핸들러를 워커 스레드로 넘길 때 복사되어 따라간다.
- 전역: 프로세스 수명. service / version / env 같은 것.
- 같은 키가 양쪽에 있으면 요청 키가 이긴다.
"""

import threading
from typing import Any

import structlog
from structlog.typing import EventDict, WrappedLogger

_global: dict[str, Any] = {}
_global_lock = threading.Lock()


def set_context(key: str, value: Any) -> None:
    """요청 범위 키. 같은 키는 덮어쓴다."""
    structlog.contextvars.bind_contextvars(**{key: value})


def remove(key: str) -> None:
    """요청 범위 키 제거. 없어도 조용히 지나간다."""
    structlog.contextvars.unbind_contextvars(key)


def clear() -> None:
    """요청 범위 키 전부 제거. 요청 종료 시 미들웨어가 부른다."""
    structlog.contextvars.clear_contextvars()


def set_global_context(key: str, value: Any) -> None:
    """프로세스 전역 키. 모든 로그 줄에 붙는다."""
    with _global_lock:
        _global[key] = value


def remove_global(key: str) -> None:
    with _global_lock:
        _global.pop(key, None)


def keys() -> tuple[list[str], list[str]]:
    """(전역 키 목록, 요청 범위 키 목록)."""
    with _global_lock:
        g = list(_global)
    return g, list(structlog.contextvars.get_contextvars())


def merge_global_context(_: WrappedLogger, __: str, event_dict: EventDict) -> EventDict:
    """structlog processor. 전역 키를 합치되 요청 키를 덮어쓰지 않는다."""
    with _global_lock:
        for k, v in _global.items():
            event_dict.setdefault(k, v)
    return event_dict
```

`render.py`:
```python
"""렌더러.

json 은 유효한 한 줄 JSON. json-color 는 같은 모양에 ANSI 색 (터미널 전용, 유효 JSON 아님).
"""

import json
from typing import Any

from structlog.typing import EventDict, WrappedLogger

_FIRST = ("timestamp", "level", "logger", "event")
_RESET = "\033[0m"
_KEY = "\033[36m"  # cyan
_LEVEL = {
    "debug": "\033[2m",  # dim
    "info": "\033[32m",  # green
    "warning": "\033[33m",  # yellow
    "error": "\033[31m",  # red
    "critical": "\033[35m",  # magenta
}


def order_keys(_: WrappedLogger, __: str, event_dict: EventDict) -> EventDict:
    """timestamp, level, logger, event 를 앞으로. 나머지는 삽입 순서."""
    ordered: EventDict = {k: event_dict[k] for k in _FIRST if k in event_dict}
    for k, v in event_dict.items():
        ordered.setdefault(k, v)
    return ordered


def _dump(value: Any) -> str:
    return json.dumps(value, ensure_ascii=False, default=str)


class ColorJSONRenderer:
    """한 줄 JSON 모양. 키는 cyan, level 값은 심각도 색."""

    def __call__(self, _: WrappedLogger, __: str, event_dict: EventDict) -> str:
        parts = []
        for k, v in event_dict.items():
            val = _dump(v)
            if k == "level":
                val = f"{_LEVEL.get(str(v), '')}{val}{_RESET}"
            parts.append(f'{_KEY}"{k}"{_RESET}:{val}')
        return "{" + ",".join(parts) + "}"
```

`config.py`:
```python
"""configure_logging — 표준 logging 루트 핸들러에 structlog 포매터를 건다.

그래서 logging.getLogger(__name__) 로 찍는 기존 코드, uvicorn 로거, structlog 로거가
전부 같은 형식으로 나간다.
uvicorn 의 액세스 로그는 끈다 (app 의 RequestLoggingMiddleware 가 대체).
"""

import json
import logging
import sys
from typing import Literal

import structlog

from .context import merge_global_context
from .render import ColorJSONRenderer, order_keys

LogFormat = Literal["json", "json-color", "console"]


def configure_logging(level: str = "INFO", fmt: LogFormat = "json") -> None:
    shared: list[structlog.typing.Processor] = [
        structlog.contextvars.merge_contextvars,  # 요청 범위 키
        merge_global_context,  # 전역 키 (요청 키를 덮지 않음)
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso", utc=True),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,  # exc_info -> "exception" 문자열
        order_keys,
    ]

    renderer: structlog.typing.Processor
    if fmt == "json":
        renderer = structlog.processors.JSONRenderer(
            serializer=json.dumps, ensure_ascii=False, separators=(",", ":"), default=str
        )
    elif fmt == "json-color":
        renderer = ColorJSONRenderer()
    else:
        renderer = structlog.dev.ConsoleRenderer()

    structlog.configure(
        processors=[*shared, structlog.stdlib.ProcessorFormatter.wrap_for_formatter],
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )

    formatter = structlog.stdlib.ProcessorFormatter(
        foreign_pre_chain=shared,  # 표준 logging 으로 들어온 레코드에도 같은 체인
        processors=[structlog.stdlib.ProcessorFormatter.remove_processors_meta, renderer],
    )
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(formatter)

    root = logging.getLogger()
    root.handlers.clear()
    root.addHandler(handler)
    root.setLevel(level.upper())

    for name in ("uvicorn", "uvicorn.error"):
        lg = logging.getLogger(name)
        lg.handlers.clear()
        lg.propagate = True
    logging.getLogger("uvicorn.access").disabled = True
```

`__init__.py`:
```python
"""my_logging — 로그 형식과 컨텍스트를 한 곳에서.

도메인 코드는 logging.getLogger(__name__) 를 그대로 써도 같은 형식으로 나간다.
요청 컨텍스트에 키를 더할 때만 set_context 를 쓴다.
"""

from typing import Any

import structlog

from .config import LogFormat, configure_logging
from .context import (
    clear,
    keys,
    remove,
    remove_global,
    set_context,
    set_global_context,
)


def get_logger(name: str | None = None) -> structlog.stdlib.BoundLogger:
    return structlog.stdlib.get_logger(name)


def log(message: str, **fields: Any) -> None:
    """일반 출력. print 대신 이걸 쓴다. level 은 info."""
    get_logger().info(message, **fields)


def info(message: str, **fields: Any) -> None:
    get_logger().info(message, **fields)


def warning(message: str, **fields: Any) -> None:
    get_logger().warning(message, **fields)


def error(message: str, **fields: Any) -> None:
    get_logger().error(message, **fields)


__all__ = [
    "LogFormat",
    "clear",
    "configure_logging",
    "error",
    "get_logger",
    "info",
    "keys",
    "log",
    "remove",
    "remove_global",
    "set_context",
    "set_global_context",
    "warning",
]
```

출력 예 (`json`, 한 요청의 세 줄. `request_id` 와 `user_id` 가 Usecase 줄 · 라우터 줄 · 미들웨어 줄에 같은 값으로 붙는다):
```
{"timestamp":"2026-09-26T10:35:57.576021Z","level":"info","logger":"{pkg}.service.article.impl.default_article_writer","event":"Article created: id=7","request_id":"demo-req-2","user_id":1,"method":"POST","path":"/api/articles"}
{"timestamp":"2026-09-26T10:35:57.580833Z","level":"info","logger":"{pkg}.api.article.article_router","event":"Article created: id=7","request_id":"demo-req-2","user_id":1,"method":"POST","path":"/api/articles"}
{"timestamp":"2026-09-26T10:35:57.581371Z","level":"info","logger":"http","event":"http.request","status":201,"duration_ms":141.3,"request_id":"demo-req-2","user_id":1,"method":"POST","path":"/api/articles"}
```

**설계 근거**:
- structlog 를 직접 노출하지 않고 `my_logging` 이라는 얇은 facade 를 둔다. 형식이나 엔진을 바꿔도 도메인 코드는 그대로다.
- `ProcessorFormatter` 를 표준 logging 루트 핸들러에 걸기 때문에 structlog 로거와 `logging.getLogger` 로거, uvicorn 로거가 한 형식으로 나간다. 도메인 코드에 `logging.getLogger(__name__)` 를 유지하는 이유다.
- `json-color` 는 유효한 JSON 이 아니다. 터미널에서 눈으로 볼 때만 쓰고, 파일이나 수집기로 보내는 환경은 `json` 이어야 한다. 그래서 `local` 만 기본이 `json-color` 다.
- 요청 범위 키는 contextvars 라서 요청마다 독립이고 `def + Session` 핸들러가 도는 anyio 워커 스레드에도 복사되어 따라간다. 전역 키는 프로세스 dict 라 Lock 으로 보호한다.
- `default=str` 이라 `datetime` · `Decimal` · dataclass 같은 값을 넣어도 직렬화에 실패하지 않는다.

### 4.7 `persistence` 패키지 — 기반 파일

경로: `packages/persistence/src/{pkg}/persistence/`

`base.py`:
```python
from sqlalchemy.orm import DeclarativeBase

from {pkg}.model.audit import utcnow  # noqa: F401  Entity 모듈이 여기서 re-import 한다


class Base(DeclarativeBase):
    """모든 Entity 의 부모. 공통 컬럼은 두지 않는다 — 각 Entity 가 4필드를 직접 선언 (C-3)."""
```

`entities.py` (Entity 레지스트리):
```python
"""Alembic 과 테스트가 Base.metadata 를 완성하기 위해 import 하는 레지스트리.

add-entity-specs 스킬이 도메인 Entity 모듈을 한 줄씩 append 한다.
예: from .article.article_entity import ArticleEntity  # noqa: F401
"""
```

`session.py`:
```python
from sqlalchemy import Engine, create_engine
from sqlalchemy.orm import Session, sessionmaker


def create_db_engine(
    url: str, *, pool_size: int = 20, max_overflow: int = 20, echo: bool = False
) -> Engine:
    return create_engine(
        url, pool_size=pool_size, max_overflow=max_overflow, pool_pre_ping=True, echo=echo
    )


def create_session_factory(engine: Engine) -> sessionmaker[Session]:
    return sessionmaker(
        bind=engine,
        autobegin=False,        # 트랜잭션은 @transactional 이 명시적으로 연다
        expire_on_commit=False,  # commit 후에도 객체를 재조회 없이 읽는다
        autoflush=True,
    )
```

`transaction.py`:
```python
from contextlib import AbstractContextManager
from typing import Any

from sqlalchemy.orm import Session

from {pkg}.transaction import TransactionManager


class DefaultTransactionManager(TransactionManager):
    def __init__(self, session: Session) -> None:
        self._session = session

    def in_transaction(self) -> bool:
        return self._session.in_transaction()

    def begin(self) -> AbstractContextManager[Any]:
        return self._session.begin()

    def begin_nested(self) -> AbstractContextManager[Any]:
        return self._session.begin_nested()
```

**`autobegin=False` 근거**: SQLAlchemy 기본값은 첫 쿼리에서 트랜잭션이 조용히 시작된다. 그러면 Reader가 먼저 select한 요청에서 Writer의 `@transactional`이 "이미 열려 있음"으로 판단해 합류만 하고 아무도 commit하지 않는다. `autobegin=False`면 `@transactional` 없이 DB에 손대는 순간 `InvalidRequestError`가 나서, 조용한 버그가 소리 나는 버그로 바뀐다.

**풀 크기 근거**: `def` 라우터는 anyio 스레드풀(기본 40)에서 돈다. `pool_size + max_overflow >= 40` 이어야 스레드가 커넥션을 기다리지 않는다. 기본값 20 + 20.

### 4.8 `api` 패키지 — 인증 stub

경로: `packages/api/src/{pkg}/api/auth.py`

```python
from dataclasses import dataclass

from fastapi import Request

from {pkg}.exception import UnauthorizedException

DEV_USER_HEADER = "X-User-Id"


@dataclass(frozen=True, slots=True)
class CurrentUser:
    user_id: int


def resolve_current_user(request: Request, env: str) -> CurrentUser:
    """개발용 stub. env == "local" 에서만 X-User-Id 헤더를 믿는다.

    그 외 환경에서는 무조건 401 — 헤더 신뢰가 프로덕션에 남는 것을 막는 가드.
    실제 인증은 별도 스킬이 이 함수와 RequestProvider.current_user 를 교체한다.
    """
    if env != "local":
        raise UnauthorizedException("authentication is not configured for this environment")
    raw = request.headers.get(DEV_USER_HEADER)
    if raw is None or not raw.isdigit():
        raise UnauthorizedException(f"missing or invalid {DEV_USER_HEADER} header")
    return CurrentUser(user_id=int(raw))
```

라우터는 `current_user: FromDishka[CurrentUser]` 로 받는다 (Java `@AuthenticationPrincipal SessionUser`). 인증이 필요 없는 공개 엔드포인트는 파라미터를 선언하지 않으면 provider가 호출되지 않는다.

### 4.9 `schema` 패키지 — Alembic

경로: `packages/schema/`

```
packages/schema/
├── pyproject.toml
├── alembic.ini
├── alembic/
│   ├── env.py
│   ├── script.py.mako          # 아래 현대식 템플릿 (기본값은 ruff UP035/UP007/I001 에 걸린다)
│   └── versions/
│       └── .gitkeep
└── src/{pkg}/schema/__init__.py   # 빈 파일. 패키지 대칭성 유지
```

`alembic.ini`:
```ini
[alembic]
script_location = %(here)s/alembic
file_template = %%(year)d%%(month).2d%%(day).2d_%%(rev)s_%%(slug)s
sqlalchemy.url =

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
```

`alembic/script.py.mako` (Alembic 기본 템플릿은 `typing.Union`/`Sequence` 를 써서 생성 파일이 ruff 에 걸린다. 현대식으로 교체):
```mako
"""${message}

Revision ID: ${up_revision}
Revises: ${down_revision | comma,n}
Create Date: ${create_date}

"""
from collections.abc import Sequence

import sqlalchemy as sa
from alembic import op
${imports if imports else ""}

# revision identifiers, used by Alembic.
revision: str = ${repr(up_revision)}
down_revision: str | None = ${repr(down_revision)}
branch_labels: str | Sequence[str] | None = ${repr(branch_labels)}
depends_on: str | Sequence[str] | None = ${repr(depends_on)}


def upgrade() -> None:
    ${upgrades if upgrades else "pass"}


def downgrade() -> None:
    ${downgrades if downgrades else "pass"}
```

`alembic/env.py`:
```python
import os

from alembic import context
from dotenv import load_dotenv
from sqlalchemy import engine_from_config, pool

from {pkg}.persistence import entities  # noqa: F401  metadata 완성
from {pkg}.persistence.base import Base

load_dotenv()

config = context.config
if not config.get_main_option("sqlalchemy.url"):
    config.set_main_option("sqlalchemy.url", os.environ["APP_DATABASE_URL"])

target_metadata = Base.metadata


def include_object(obj, name, type_, reflected, compare_to) -> bool:  # noqa: ANN001
    """DB FK 제약은 두지 않는다 (C-5). Entity 의 ForeignKey() 는 ORM 조인 조건 선언용."""
    if type_ == "foreign_key_constraint":
        return False
    return True


def run_migrations_offline() -> None:
    context.configure(
        url=config.get_main_option("sqlalchemy.url"),
        target_metadata=target_metadata,
        include_object=include_object,
        literal_binds=True,
        compare_type=True,
    )
    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    connectable = engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            include_object=include_object,
            compare_type=True,
        )
        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

**설계 근거**:
- URL은 `sqlalchemy.url`(테스트가 주입) → `APP_DATABASE_URL` 환경변수(`.env`) 순. `app.settings`를 import하면 `schema → app` 의존이 생기므로 환경변수를 직접 읽는다.
- `include_object`가 FK 제약을 걸러내므로 Entity에 `ForeignKey(...)`를 써도 migration에 `create_foreign_key`가 나오지 않는다.
- `compare_type=True`: 컬럼 타입 변경도 autogenerate가 감지.
- 도메인별 migration 1개 (`add-entity-specs`가 `alembic revision --autogenerate -m "{domain}"` 실행 후 검토). Java의 `-- === {Domain} ===` 블록 append에 해당.

실행 (프로젝트 루트에서): `uv run alembic -c packages/schema/alembic.ini upgrade head`

### 4.10 테스트 기반 — 루트 `conftest.py`

경로: `{projectRoot}/conftest.py`

```python
from collections.abc import Iterator
from pathlib import Path

import pytest
from alembic import command
from alembic.config import Config
from sqlalchemy import Engine, create_engine, text
from sqlalchemy.orm import Session, sessionmaker
from testcontainers.community.postgres import PostgresContainer

from {pkg}.persistence import entities  # noqa: F401  metadata 완성
from {pkg}.persistence.base import Base
from {pkg}.persistence.session import create_session_factory

ROOT = Path(__file__).parent


@pytest.fixture(scope="session")
def database_url() -> Iterator[str]:
    """testcontainers Postgres. DB 가 필요한 테스트가 처음 요청할 때만 기동된다."""
    with PostgresContainer("postgres:16-alpine", driver="psycopg") as pg:
        yield pg.get_connection_url()


@pytest.fixture(scope="session")
def engine(database_url: str) -> Iterator[Engine]:
    cfg = Config(str(ROOT / "packages" / "schema" / "alembic.ini"))
    cfg.set_main_option("sqlalchemy.url", database_url)
    command.upgrade(cfg, "head")
    engine = create_engine(database_url, pool_pre_ping=True)
    yield engine
    engine.dispose()


@pytest.fixture(scope="session")
def session_factory(engine: Engine) -> sessionmaker[Session]:
    return create_session_factory(engine)


@pytest.fixture
def session(engine: Engine) -> Iterator[Session]:
    """외부 트랜잭션 + 세이브포인트 (Java 테스트 클래스 @Transactional 대응).

    테스트 안의 commit 은 전부 세이브포인트 해제라서, 끝나면 통째로 롤백된다.
    """
    with engine.connect() as conn:
        outer = conn.begin()
        s = Session(
            bind=conn,
            autobegin=False,
            expire_on_commit=False,
            join_transaction_mode="create_savepoint",
        )
        try:
            yield s
        finally:
            s.close()
            outer.rollback()


# migration 이 시드를 넣는 참조 테이블 (예: categories). 비우면 다음 테스트가 시드 없이 돈다.
# 참조 테이블을 만드는 add-entity-specs 가 여기에 이름을 추가한다.
SEED_TABLES: set[str] = set()


@pytest.fixture
def truncate_all(engine: Engine) -> Iterator[None]:
    """동시성 테스트 전용. 실제 commit 을 허용하고 끝나면 시드 테이블을 뺀 전 테이블을 비운다."""
    yield
    with engine.begin() as conn:
        for table in reversed(Base.metadata.sorted_tables):
            if table.name not in SEED_TABLES:
                conn.execute(text(f'TRUNCATE TABLE "{table.name}" CASCADE'))
```

**근거**:
- 루트 `conftest.py` 하나가 Java의 `IntegrationTestBase` × 2, `TestApplication`, `IntegrationConfig` 네 파일을 대체한다. pytest는 rootdir의 `conftest.py`를 `packages/*/tests` 전부에 적용한다.
- Mockito 단위 테스트(model, service, api)는 이 픽스처를 요청하지 않으므로 Docker 없이 돈다.
- `truncate_all`이 `Base.metadata.sorted_tables`를 쓰므로 테이블 목록 하드코딩이 없다.

각 패키지에 빈 `tests/__init__.py`는 두지 않는다 (rootdir 기반 import). 대신 `tests/.gitkeep`.

### 4.11 기타

`docker-compose.yml`:
```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: {dbName}
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

`.env.example`:
```
APP_ENV=local
APP_DATABASE_URL=postgresql+psycopg://app:app@localhost:5432/{dbName}
```

`.gitignore`:
```
.venv/
__pycache__/
*.pyc
.pytest_cache/
.mypy_cache/
.ruff_cache/
dist/
build/
*.egg-info/
.env
.idea/
.vscode/
.DS_Store
*.log
```

`README.md` (최소):
```markdown
# {projectName}

FastAPI / SQLAlchemy 2.0 / Postgres 기반 Hexagonal uv workspace 백엔드.

## 실행

    docker compose up -d db
    cp .env.example .env
    uv sync --all-groups
    uv run alembic -c packages/schema/alembic.ini upgrade head
    uv run uvicorn {pkg}.app.main:create_app --factory --reload

- OpenAPI: http://localhost:8000/docs

## 패키지

- app — 실행 진입점, DI 바인딩, 예외 핸들러
- api — APIRouter / Pydantic DTO
- service — Reader / Writer / Usecase
- infrastructure — Repository port
- persistence — Entity, EntityQuery(ORM), JoinQuery(SQL), Default{Domain}Repository(port 구현체)
- transaction — TransactionManager 계약 + @transactional
- my_logging — JSON 한 줄 로깅, 요청 컨텍스트 키
- model — 도메인 모델 + AuditFields
- schema — Alembic migration
- exception — 공통 예외
```

각 패키지의 `src/{pkg}/{layer}/__init__.py` (빈 파일. `transaction` 과 `my_logging` 은 §4.6 의 re-export 내용) + `tests/.gitkeep`. `app` 은 추가로 `src/{pkg}/app/providers/__init__.py` 와 `src/{pkg}/app/middleware/__init__.py` (빈 파일).

## 5. 실행 순서 & 자가 수정 루프

### 5.1 실행 순서

1. **입력 파라미터 로딩**: `{projectRoot}/dk-py-bundle.config.json` 우선, 없으면 대화로 3개 값을 받는다. 기본값 제시 금지(특히 `pkg`는 사용자 고유).
2. **사전 불변식 검사**: `pkg.isidentifier()` 이고 소문자·밑줄만. 깨지면 abort.
3. **재실행 가드**: `{projectRoot}/pyproject.toml`에 `[tool.uv.workspace]`가 있으면 abort.
4. **uv 존재 확인**: `uv --version`. 없으면 설치 안내 후 중단.
5. **디렉터리 생성**: 10개 `packages/{layer}/src/{pkg}/{layer}/` 와 `tests/`.
6. **파일 작성**: §4의 각 파일을 플레이스홀더(`{projectName}`, `{pkg}`, `{dbName}`) 치환 후 Write.
7. **Python 확보**: `uv python install 3.12` (이미 있으면 no-op).
8. **검증 실행**: §5.2 명령 실행.
9. 실패 시 §5.3 원인 분류표로 판정 → 수정 → 재시도 (최대 3회). 표에 없는 에러는 **추측 없이 escalate**.
10. (선택) 기동 스모크: Docker 가 떠 있으면 `docker compose up -d db` 후 `timeout 20 uv run uvicorn {pkg}.app.main:create_app --factory` 로 `Application startup complete` 확인. Docker 없으면 건너뛰고 스킬은 성공으로 보고(검증 통과가 기준).
11. 완료 리포트 (§6).

### 5.2 검증 명령

```bash
uv sync --all-groups && \
uv run python -c "import {pkg}.app.main as m; m.create_app()" && \
uv run ruff check . && \
uv run lint-imports && \
uv run mypy
```

**이 조합 근거**:
- `uv sync`: workspace 8개가 서로를 해석하고 설치되는지 (Java `settings.gradle.kts` + 의존 그래프 검증).
- `python -c "create_app()"`: 8개 패키지 import + dishka 컨테이너 조립 + 라우터·핸들러 등록까지. DB 연결은 지연 생성이라 DB 없이 통과해야 한다 (Java `compileJava` + 테스트 소스 컴파일).
- `ruff check`: 문법·미사용 import·import 순서.
- `lint-imports`: 의존 방향 계약. **이것이 통과해야 헥사고날이 지켜진 것**.
- `mypy`: ABC 구현 누락, dataclass 시그니처 불일치.
- `pytest`는 돌리지 않는다. 스캐폴드 시점엔 테스트가 없고, DB 픽스처는 Docker가 필요하다.

### 5.3 원인 분류표

| 에러 패턴 | 진단 | 조치 |
| --- | --- | --- |
| `No interpreter found for Python 3.12` | Python 3.12 미설치 | `uv python install 3.12` 실행 후 재시도 |
| `No solution found when resolving dependencies` | 버전 하한 충돌 | 충돌 패키지 하한 완화. `dishka`/`fastapi`/`sqlalchemy` 최신 안정판 확인 |
| `ModuleNotFoundError: No module named '{pkg}.<layer>'` | 해당 패키지가 `members`에 없거나 소비 패키지 `dependencies` 누락, 또는 `src/{pkg}/__init__.py`가 실수로 생성됨 | §4.1 members·sources, §4.2 dependencies 대조. `src/{pkg}/__init__.py` 있으면 삭제 |
| `ImportError: cannot import name ... (most likely due to a circular import)` | 순환 import | §3 그래프와 대조, 역방향 import 제거 |
| `lint-imports` → `Broken contract` | 의존 방향 위반 | 출력된 import 체인을 보고 상위 층 import 제거 |
| `ruff` `F401 ... imported but unused` in `entities.py` / `env.py` / `conftest.py` | 레지스트리 import | `# noqa: F401` 확인 |
| `ruff` `I001` | import 정렬 | `uv run ruff check --fix .` |
| `mypy` `Cannot instantiate abstract class` | ABC 메서드 누락 | 누락 메서드 구현 |
| `psycopg` 관련 `ImportError: no pq wrapper available` | binary extra 누락 | `psycopg[binary]` 확인 |
| `dishka` `NoFactoryError` (create_app 시) | provider 누락 또는 스코프 불일치 | `AppProvider`/`RequestProvider` 의 `provides` 타입과 소비 타입 대조 |
| 요청 시 500, 로그에 `TypeError: object X can't be used in 'await' expression` | 라우터가 `DishkaRoute`(async) 를 씀 | `route_class=DishkaSyncRoute` 로 교체 (C-11) |
| `ruff` `UP047 Generic function ... should use type parameters` | `TypeVar` 방식 제네릭 | `def f[R](...)` PEP 695 문법으로 교체 |
| `mypy` `Package '{pkg}' cannot be type checked due to missing py.typed marker` | `[tool.mypy] packages = [...]` 로 설치 패키지를 지정함 | §4.1 의 `files` + `mypy_path` + `explicit_package_bases` 설정으로 교체 |
| `ModuleNotFoundError: No module named 'structlog'` | `my_logging` 패키지 `dependencies` 에 `structlog` 누락, 또는 `uv sync` 전 | §4.2 확인 후 `uv sync --all-groups` |
| `lint-imports` → `forbidden` 계약이 항상 KEPT 인데 service 에 `import sqlalchemy` 가 있음 | `include_external_packages = true` 누락 | §4.1 `[tool.importlinter]` 에 추가 |
| 로그가 JSON 이 아니라 uvicorn 기본 형식 | `configure_logging` 이 `create_app()` 첫 줄이 아님 | §4.3 `main.py` 순서 확인 |
| 위 표에 없는 에러 | 모름 | **추측 금지 — stderr 전문 포함해 escalate** |

### 5.4 자가 수정 루프 의사코드

```
MAX_RETRY = 3
for attempt in 1..MAX_RETRY:
    result = run(verifyCmd)
    if result.ok: break
    cause = classify(result.stderr, table=§5.3)
    if cause is None:
        report(result.stderr); abort
    apply(cause.fix)
else:
    report("3회 재시도 소진, 파일은 보존"); abort
```

**MAX_RETRY=3 근거**: 1회차는 타이포, 2회차는 의존 누락, 3회차는 버전 이슈가 일반적. 4회 이상은 구조적 문제로 사람이 개입해야 하므로 시도 가치가 낮음.

## 6. 완료 후 상태

- `uv sync` → `create_app()` → `ruff` → `lint-imports` → `mypy` 전부 green.
- Docker Postgres 기동 + `alembic upgrade head` (migration 0개, no-op) + `uvicorn` 기동 시 `/docs` 접근 가능.
- 도메인 테이블은 없다. `versions/`는 비어 있다.
- 각 패키지 `src/{pkg}/{layer}/__init__.py`와 `tests/.gitkeep`.

완료 리포트 형식:
```
be-init 완료

- 생성 위치: {projectRoot}
- 패키지: 10개 (app 포함)
- 검증: OK (retry {N}회)
- 다음 단계: docker compose up -d db 후 be-planner로 첫 도메인 추가
  └ 예: /dk-py-bundle:be-planner "POST /api/articles"
```

## 7. Downstream 컨트랙트 (C-1 ~ C-13)

be-planner 및 add-* 스킬이 의존하는 공식 계약.

| # | 내용 |
| --- | --- |
| C-1 | **workspace members는 be-init이 고정.** downstream은 루트 `pyproject.toml`의 `members`와 `sources`, 각 패키지의 `dependencies`를 수정하지 않는다. 외부 라이브러리 추가가 필요하면 유저 확인 후 해당 패키지 `dependencies`에만 추가. |
| C-2 | import 루트는 `{pkg}`. downstream은 `{pkg}.{layer}.{domain}` 하위만 추가한다. `src/{pkg}/__init__.py`는 절대 만들지 않는다. |
| C-3 | 감사 필드명은 `created_at` / `updated_at`, 타입 tz-aware `datetime`. Entity는 `mapped_column(DateTime(timezone=True), default=utcnow)` / `(..., default=utcnow, onupdate=utcnow)` 를 **직접** 선언. 도메인 모델도 같은 두 필드를 가진다 (`AuditFields` Protocol 구조 만족). |
| C-4 | 모든 Entity는 `is_deleted`, `deleted_at`, `created_at`, `updated_at` 4필드를 직접 선언. Stats 테이블과 aggregate 자식 컬렉션 테이블만 예외. |
| C-5 | **DB FK 제약은 두지 않는다.** Entity의 `ForeignKey(...)`는 ORM 조인 조건 선언용이며 `env.py`의 `include_object`가 migration에서 제외한다. 참조 무결성은 애플리케이션이 관리. |
| C-6 | 스키마는 Alembic. 도메인당 migration 1개, `add-entity-specs`가 `alembic revision --autogenerate -m "{domain}"` 후 검토. Entity 모듈은 `persistence/entities.py`에 한 줄 append. |
| C-7 | 도메인 예외는 `DomainException`을 상속하고 `http_status_code` / `http_error_message` 속성을 가진다. 404 계열은 `EntityNotFoundException`을 상속한 도메인별 하위 클래스로 정의. |
| C-8 | `error_handlers.py`는 be-init 소유. 추가 핸들러는 별도 함수로 분리해 `create_app()`에서 등록. |
| C-9 | **트랜잭션**: 계약은 `{pkg}.transaction` (`from {pkg}.transaction import TransactionManager, transactional`). persistence 는 `commit()`을 부르지 않는다. Writer / Usecase / `Default{Domain}Repository`(port 구현체)의 public 메서드에 `@transactional`. 대상 클래스는 생성자에서 `self.tx: TransactionManager`를 설정. Reader 와 쿼리 클래스(`{Domain}EntityQuery`, `{Domain}JoinQuery`)에는 붙이지 않는다. |
| C-10 | **DI**: 도메인별 `Provider`(REQUEST 스코프)를 `app/providers/{domain}_provider.py`에 만들고 `app/di.py`의 `DOMAIN_PROVIDERS`에 append (add-usecase). 라우터는 `app/routers.py`의 `ROUTERS`에 append (add-api). 컴포넌트 스캔이 없으므로 이 두 줄이 빠지면 조립되지 않는다. |
| C-11 | 라우터는 `APIRouter(route_class=DishkaSyncRoute)`, 의존성은 `FromDishka[Type]` 파라미터로 받는다. **`DishkaRoute`(async 용)는 동기 컨테이너와 조합하면 500** — 반드시 `DishkaSyncRoute`. `api`는 `service`의 ABC만 import한다. |
| C-12 | 테스트 베이스는 루트 `conftest.py`의 픽스처. DB 통합 테스트는 `session`(세이브포인트), 동시성 테스트는 `session_factory` + `truncate_all`. 단위 테스트는 `NoopTransactionManager`(`{pkg}.transaction`) + `Mock(spec=...)`. |
| C-13 | **로깅**: 도메인 코드는 모듈 상단 `logger = logging.getLogger(__name__)` 를 그대로 쓴다. 형식은 `my_logging` 이 정하므로 핸들러·포매터를 직접 만들지 않는다. 요청 진입 로그(`"GET /api/..."`)는 찍지 않는다. `RequestLoggingMiddleware` 의 `http.request` 한 줄이 method / path / status / duration_ms 를 남긴다. 요청 범위 키를 더할 때만 `from {pkg}.my_logging import set_context` 를 쓴다 (예: `di.py` 의 `user_id`). `structlog` 직접 import 는 `my_logging` 안에서만. |

## 8. 실패 모드

| 모드 | 증상 | 대응 |
| --- | --- | --- |
| uv 미설치 | `uv: command not found` | `curl -LsSf https://astral.sh/uv/install.sh \| sh` 또는 `brew install uv` 안내 후 중단 |
| Python 3.12 미설치 | `No interpreter found` | `uv python install 3.12` 실행 후 재시도 |
| `pkg`가 식별자 아님 | 파일 생성 전 즉시 중단 | 파라미터 재입력 요청 |
| 네임스페이스 깨짐 | `ModuleNotFoundError: {pkg}.service` 인데 패키지는 설치됨 | `src/{pkg}/__init__.py` 존재 여부 확인 후 삭제 |
| 라이브러리 버전 충돌 | `No solution found` | 하한 완화, 파일은 보존 |
| 3회 재시도 소진 | 분류표 미해당 에러 반복 | **파일은 보존**, 전체 로그 포함해 escalate |

## 9. 스킬 경계 (재강조)

- **One-shot**. `[tool.uv.workspace]`가 이미 있으면 abort.
- 도메인 생성은 be-planner 및 add-model / add-infrastructure / add-entity-specs / add-query / add-usecase / add-api 스킬의 책임.
- DB 교체 (Postgres → MySQL)는 범위 밖. SQLAlchemy URL과 driver만 바꾸면 되지만 별도 스킬로 분리 예정.
- 실제 인증 / ES / Docker 이미지 / CI 도 범위 밖. 각각 별도 스킬 또는 수동 작업.
