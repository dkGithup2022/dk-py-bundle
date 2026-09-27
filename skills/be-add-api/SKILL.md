---
name: be-add-api
description: api 패키지에 APIRouter와 Pydantic Request/Response DTO를 생성/변경하고, app 패키지의 ROUTERS에 등록한다.
---

# add-api

api 패키지에 Router 와 Request/Response DTO 를 생성/변경한다. 생성한 router 를 `app/routers.py` 에 등록한다.

## 입력

### 지시
- assess-api 산출물 (API 스펙, 호출 대상 Reader/Usecase) 에 따라 작업한다.
- 개별 API 추가 요청에도 아래의 공통 규칙을 따르시오.

### Phase 0. 프로젝트 컨텍스트 해석
`{pkg}`, `{packages.api}`, `{packages.app}` 플레이스홀더 사용. 실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.

### 참고 문서 (읽어올 것)
- `${CLAUDE_PLUGIN_ROOT}/references/be-refs/api-references.md` — 4 패턴 (Foo/Bar/Baz/Qux) + 공통 요소 (DI, Request/Response, 인증, 로깅, 라우터 등록)
- specs/api.md — 기존 API 엔드포인트 정보 (프로젝트 루트 기준, 없으면 소스에서 추론)
- specs/usecase.md — 호출 대상 ABC 시그니처 + Command 스펙 (프로젝트 루트 기준)
- specs/models.md — 도메인 모델 (Response 필드 참조) (프로젝트 루트 기준)

## 패턴 선택 (먼저 결정)

api-references.md §"패턴 선택 가이드" 로 엔드포인트 특성에 맞는 패턴을 먼저 고른다:

| 엔드포인트 특성 | Pattern |
|---|---|
| 리소스 명사형, 인증, 전체 CRUD | **Foo** (주력) |
| 동작형 리소스 (bookmark/rating) | **Bar** |
| 공개 조회 전용 | **Baz** |
| 여러 도메인 집계 (마이페이지/대시보드) | **Qux** |

## 작업 순서

1. **참고 문서를 읽는다.**
2. **개발 가능 여부를 판단한다.**
   - 요구 스펙을 현재 Reader/Usecase 와 모델로 구현할 수 있는지 확인한다.
   - 쓰기 엔드포인트인데 해당 Usecase 가 없으면 → add-usecase 가 먼저 필요하다고 보고한다 (Writer 직접 주입으로 우회하지 않는다).
3. **불가하거나 스펙 변화가 필요한 경우:**
   - 최소 변화로 스펙을 달성할 수 있는 방안을 유저에게 제시하고 확인받는다.
4. **가능하다면 제공된 데이터 스펙과 인터페이스에 맞춰 개발한다.**
5. **`app/routers.py` 에 등록한다.**

## 생성 결과물 구조

### 기본

```
{packages.api}/src/{pkg}/api/{domain}/
├── __init__.py
├── {domain}_router.py
└── dto/
    ├── __init__.py
    ├── {domain}_create_request.py
    └── {domain}_response.py
```

### 복수 DTO

```
{packages.api}/src/{pkg}/api/{domain}/dto/
├── {domain}_create_request.py
├── {domain}_update_request.py
├── {domain}_response.py
├── {domain}_list_response.py           ← 목록 + 페이징
└── {domain}_{action}_response.py       ← 특수 응답
```

### 등록 (app 패키지, be-init C-10)

```
{packages.app}/src/{pkg}/app/routers.py   ← ROUTERS 에 router 추가 + import
```

## 생성 규칙

### 1. Router

```python
import logging

from dishka.integrations.fastapi import DishkaSyncRoute, FromDishka
from fastapi import APIRouter, Response, status

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/api/{domain-plural}", tags=["{Domain}"], route_class=DishkaSyncRoute)
```

- `route_class=DishkaSyncRoute` 필수 (be-init C-11)
- 핸들러는 `def` (`async def` 금지)
- 의존성은 핸들러 파라미터 `FromDishka[{Domain}Reader]`, `FromDishka[{Domain}CreateUsecase]`, `FromDishka[CurrentUser]`
- Reader 와 Usecase 만 의존. Writer 직접 주입 금지
- 비즈니스 로직 없음 — 변환과 위임만

### 2. 엔드포인트 패턴

**조회 (GET):**

```python
@router.get("/{id}", summary="...")
def get_{domain}(id: int, reader: FromDishka[{Domain}Reader]) -> {Domain}Response:
    read = reader.get_by_id({Domain}Identity(id))
    return {Domain}Response.from_read(read)
```

**생성 (POST):**

```python
@router.post("", summary="...", status_code=status.HTTP_201_CREATED)
def create_{domain}(
    request: {Domain}CreateRequest,
    current_user: FromDishka[CurrentUser],
    create_usecase: FromDishka[{Domain}CreateUsecase],
) -> {Domain}CreateResponse:
    command = {Domain}CreateCommand(
        request_user_id=current_user.user_id,
        title=request.title,
        categories=tuple(request.categories),   # list → tuple
    )
    identity = create_usecase.execute(current_user.user_id, command)
    return {Domain}CreateResponse(id=identity.{domain}_id)
```

**삭제 (DELETE):**

```python
@router.delete("/{id}", summary="...", status_code=status.HTTP_204_NO_CONTENT, response_class=Response)
def delete_{domain}(
    id: int,
    current_user: FromDishka[CurrentUser],
    delete_usecase: FromDishka[{Domain}DeleteUsecase],
) -> Response:
    delete_usecase.execute(current_user.user_id, {Domain}Identity(id))
    return Response(status_code=status.HTTP_204_NO_CONTENT)
```

### 3. Request DTO

```python
class {Domain}CreateRequest(BaseModel):
    required_field: {Type}
    title: str = Field(min_length=1, max_length=255)
    optional_field: str | None = None
```

- Pydantic `BaseModel`
- 필수 필드는 기본값 없음, 문자열 빈값 금지는 `min_length=1`
- optional 은 `| None = None`
- enum 은 StrEnum 타입으로 선언 (Pydantic 이 검증, 실패 시 400)
- 컬렉션은 `list`. Command 로 넘길 때 `tuple(...)`

### 4. Response DTO

- **FE 가 준 API 초안이 있으면 목록 모양 · JSON 키 표기 · 오류 모양은 초안을 따른다** (api-references "FE 계약이 있으면 그 모양이 우선"). 번들 기본값(`{ data, total }`, snake_case)은 초안이 없을 때만. 초안과 다르게 가야 하면 구현 전에 사용자에게 묻는다.

```python
class {Domain}Response(BaseModel):
    id: int = Field(description="ID", examples=[1])
    # ...

    @classmethod
    def from_read(cls, read: {Domain}Read) -> Self:
        return cls(id=read.{domain}_id, ...)
```

- Pydantic `BaseModel`, `Field(description=, examples=)`
- **`from_read()` classmethod** 로 Read → Response 변환
- 모델의 `tuple` 은 `list(...)` 로

### 5. 로깅

- 요청 진입 로그(`"{METHOD} {path} - params..."`)는 **찍지 않는다**. app 의 `RequestLoggingMiddleware` 가 `http.request` 한 줄에 method / path / status / duration_ms / request_id 를 남긴다 (be-init C-13)
- 결과가 있는 핸들러만: `logger.info("결과 요약")` (예: `"Foo created: id=%s"`)
- `logger = logging.getLogger(__name__)` 그대로. 형식은 `my_logging` 이 정한다

### 6. 라우터 등록

`{packages.app}/src/{pkg}/app/routers.py`:

```python
from {pkg}.api.{domain}.{domain}_router import router as {domain}_router

ROUTERS: list[APIRouter] = [
    ...,
    {domain}_router,
]
```

등록 누락 시 엔드포인트가 404. 검증에서 `/openapi.json` 에 경로가 있는지 확인한다.

## 기존 변경 시

- as-is/to-be 를 유저에게 제시하고 확인받는다
- 엔드포인트 추가는 하위 호환
- Request/Response 스펙 변경은 유저 확인 필수
- 기존 엔드포인트와 유사한 요청이 오면 병합 여부를 유저에게 문의

## 검증

```bash
uv run ruff check {packages.api} {packages.app} && uv run lint-imports && uv run mypy && \
uv run python -c "
import {pkg}.app.main as m
app = m.create_app()
paths = app.openapi()['paths']
assert '/api/{domain-plural}' in paths or any(p.startswith('/api/{domain-plural}') for p in paths), paths.keys()
print('registered:', [p for p in paths if p.startswith('/api/{domain-plural}')])
"
```

마지막 명령이 라우터 등록과 Pydantic 스키마 생성까지 검증한다. `lint-imports` 가 `api → app` 역방향 import 를 잡는다.

## 산출물

- 생성/변경된 Router, Request, Response 파일 목록
- `routers.py` 에 추가된 줄
- `specs/api.md` 업데이트 (없으면 `specs/` 와 함께 생성) — 엔드포인트별 메서드·경로·Request/Response 필드·인증 여부·호출 Usecase
- 추가/변경된 스펙 정리본 출력
