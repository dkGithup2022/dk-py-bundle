# API References

`be-add-api` 스킬이 참고하는 APIRouter + Request/Response DTO 작성 레퍼런스.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 패키지)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

---

## 공통 요소

### Router 선언

```python
import logging

from dishka.integrations.fastapi import DishkaSyncRoute, FromDishka
from fastapi import APIRouter, Response, status

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/api/foos", tags=["Foo"], route_class=DishkaSyncRoute)
```

- `/api/` prefix 필수
- 도메인은 **복수형** (`/api/setups`, `/api/comments`, `/api/bookmarks`)
- `tags` — OpenAPI 그룹 표시
- **`route_class=DishkaSyncRoute`** 필수 — 이게 있어야 `FromDishka[...]` 파라미터가 주입된다 (be-init C-11). 핸들러에 `@inject` 를 붙이지 않는다
- `DishkaRoute`(async 용) 를 쓰면 안 된다. be-init 의 컨테이너는 `make_container`(동기)라서 `DishkaRoute` 는 `await container.get()` 에서 `TypeError: object X can't be used in 'await' expression` 으로 500 이 난다 (실행으로 확인)
- 핸들러는 **`def`** (`async def` 아님). `def + Session` 결정. FastAPI 가 스레드풀에서 돌린다

### 의존성 주입

Java 의 생성자 주입 대신 **핸들러 파라미터**로 받는다:

```python
@router.get("/{id}")
def get_foo(id: int, reader: FromDishka[FooReader]) -> FooResponse: ...
```

- `FromDishka[Type]` 파라미터는 경로·쿼리·본문 파라미터와 섞어 쓸 수 있다
- 라우터가 아는 타입은 `service` 의 ABC (`FooReader`) 와 Usecase 클래스, 그리고 `CurrentUser` 뿐. 구현체(`DefaultFooReader`)나 `Session` 은 모른다
- **Reader + Usecase 중심.** Writer 를 핸들러에 직접 주입하지 않는다 — 선검증·부수효과가 필요한 동작은 Usecase 를 거친다
- 인증 필요 없는 공개 API 는 `CurrentUser` 파라미터를 선언하지 않는다

### 메서드 데코레이터 셋

- `@router.get("/{id}", summary="...")` / `.post("", status_code=201)` / `.put("/{id}")` / `.delete("/{id}", status_code=204)`
- `summary` — OpenAPI 요약 (Java `@Operation(summary)`)
- 파라미터:
  - 경로: `id: int`
  - 쿼리: `page: int = 0`, `sort: str = "recent"`, 리스트는 `categories: list[FooCategory] = Query(default=[])`
  - 본문: `request: FooCreateRequest`
  - 인증: `current_user: FromDishka[CurrentUser]`

### HTTP 상태 코드와 반환 관례

| 상황 | 반환 |
|---|---|
| 조회 성공 | `return FooResponse.from_read(read)` — 200 |
| 생성 성공 | 데코레이터 `status_code=status.HTTP_201_CREATED` + `return FooCreateResponse(id=...)` — 201 |
| 삭제 성공 (body 없음) | 데코레이터 `status_code=204, response_class=Response` + `return Response(status_code=204)` |
| 단순 완료 (body 없음) | 데코레이터 `status_code=200, response_class=Response` + `return Response(status_code=200)` |

반환 타입 힌트가 곧 `response_model` 이다 (FastAPI 가 함수 반환 타입을 읽는다).

### 로깅 관례

엔드포인트 진입 로그는 찍지 않는다. app 의 `RequestLoggingMiddleware` 가 요청마다 `http.request` 한 줄(method / path / status / duration_ms / request_id)을 남기고, 핸들러 안에서 찍는 모든 줄에 같은 `request_id` 와 `user_id` 가 자동으로 붙는다 (be-init C-13). 핸들러는 결과만 한 번:

```python
# ... execute ...
logger.info("Setup created: id=%s", identity.setup_id)
```

### 입력 검증 — Pydantic 이 한다

Java 의 `parseCategories` 같은 enum 파싱 helper 가 없다. Request 필드를 `category: FooCategory` (StrEnum) 로 선언하면 Pydantic 이 검증하고, 실패하면 be-init 의 `RequestValidationError` 핸들러가 **400** `ValidationFailed` 로 응답한다.

콤마 구분 쿼리(`?categories=a,b`)는 Pydantic 기본이 아니다. 반복 파라미터(`?categories=a&categories=b`)를 쓴다.

### Request DTO 관례

```python
from pydantic import BaseModel, Field


class FooCreateRequest(BaseModel):
    title: str = Field(min_length=1, max_length=255)
    body: str = Field(min_length=1)
    categories: list[FooCategory]
    link: str | None = None    # optional
```

- Pydantic `BaseModel`. 필수 필드는 기본값 없음 (Java `@NotNull`), 빈 문자열 금지는 `min_length=1` (Java `@NotBlank`)
- optional 은 `| None = None`
- 컬렉션은 `list` (JSON 배열). Command 로 넘길 때 `tuple(request.categories)`
- 파일 위치: `api/{domain}/dto/{domain}_{action}_request.py`

### Response DTO 관례

```python
from typing import Self

from pydantic import BaseModel, Field


class FooResponse(BaseModel):
    id: int = Field(description="ID", examples=[1])
    title: str = Field(description="제목")
    author: str = Field(description="작성자 닉네임")
    categories: list[FooCategory] = Field(description="카테고리")
    created_at: datetime = Field(description="작성 시각")

    @classmethod
    def from_read(cls, read: FooRead) -> Self:
        return cls(
            id=read.foo_id,
            title=read.title,
            author=read.author,
            categories=list(read.categories),
            created_at=read.created_at,
        )

    class RelatedVO(BaseModel):
        id: int = Field(description="ID")
        name: str = Field(description="이름")
```

- Pydantic `BaseModel`, `Field(description=, examples=)` 로 OpenAPI 설명 (Java `@Schema`)
- **`from_read({Domain}Read)` classmethod** — Read 모델 → Response 변환. 이 팩토리가 핸들러 본문을 간결하게 유지
- `read.foo_id` 는 `int` (Read 모델은 None 아님, model-references)
- StrEnum 은 그대로 문자열로 직렬화된다
- `tuple → list` 변환 (JSON 배열)
- Inner class 로 연관 VO 표현 가능

### FE 계약이 있으면 그 모양이 우선

프로젝트에 FE 가 준 API 초안(핸드오프 문서 등)이 있으면 **목록 모양, JSON 키 표기, 오류 모양은 그 초안을 따른다.** 아래 번들 기본값은 초안이 없을 때만 쓴다. 번들 관례를 이유로 FE 계약을 바꾸면 비용은 FE 가 진다 (2026-09-27 testbed 에서 실제로 발생: `{ data, total }` · snake_case 로 FE 화면 대부분이 깨지고 페이지 번호가 없어 "더 보기" 가 동작하지 않았다).

- camelCase 계약이면 DTO 공통 부모에 `ConfigDict(alias_generator=to_camel, validate_by_name=True, validate_by_alias=True, serialize_by_alias=True)`. 코드 안 필드명은 snake_case 그대로. snake_case 가 아닌 query 이름은 `Query(alias="categoryGroup")`.
- 페이지 응답 계약이 있으면 제네릭 `PageResponse[T]` 하나를 두고 목록 DTO 가 상속한다 (`class FooListResponse(PageResponse[FooResponse])`, 라우터는 `FooListResponse.of(items, total, page, size)`).
- 오류 모양은 be-init 의 `ErrorResponse` 를 초안에 맞게 바꾼다. 분기용 예외 이름 필드는 따로 둔다.

### List + total envelope (FE 계약이 없을 때의 기본값)

```python
class FooListResponse(BaseModel):
    data: list[FooResponse] = Field(description="결과 목록")
    total: int = Field(description="총 건수")
```

### 라우터 등록 (app 패키지, be-init C-10)

`app/routers.py` 의 `ROUTERS` 리스트에 추가:

```python
from {pkg}.api.foo.foo_router import router as foo_router

ROUTERS: list[APIRouter] = [
    foo_router,
]
```

---

## Pattern Foo — 표준 RESTful Router (리소스형 CRUD + 인증)

**언제 쓰나**: 리소스 명사형, 인증 필요, 전체 CRUD. 가장 흔한 경우.

```python
router = APIRouter(prefix="/api/foos", tags=["Foo"], route_class=DishkaSyncRoute)


@router.get("", summary="Foo 목록 조회")
def get_foos(
    reader: FromDishka[FooReader],
    sort: str = "recent",
    q: str = "",
    page: int = 0,
    size: int = 20,
) -> FooListResponse:
    reads = reader.get_all(sort, q, page, size)
    total = reader.count_all(q)
    return FooListResponse(data=[FooResponse.from_read(r) for r in reads], total=total)


@router.get("/{id}", summary="Foo 상세 조회")
def get_foo(id: int, detail_usecase: FromDishka[FooDetailUsecase]) -> FooDetailResponse:
    read = detail_usecase.execute(FooIdentity(id))
    return FooDetailResponse.from_read(read)


@router.post("", summary="Foo 생성", status_code=status.HTTP_201_CREATED)
def create_foo(
    request: FooCreateRequest,
    current_user: FromDishka[CurrentUser],
    create_usecase: FromDishka[FooCreateUsecase],
) -> FooCreateResponse:
    user_id = current_user.user_id

    command = FooCreateCommand(
        request_user_id=user_id,
        title=request.title,
        body=request.body,
        categories=tuple(request.categories),
    )
    identity = create_usecase.execute(user_id, command)

    logger.info("Foo created: id=%s", identity.foo_id)
    return FooCreateResponse(id=identity.foo_id)


@router.put("/{id}", summary="Foo 수정")
def update_foo(
    id: int,
    request: FooUpdateRequest,
    current_user: FromDishka[CurrentUser],
    update_usecase: FromDishka[FooUpdateUsecase],
) -> FooCreateResponse:
    user_id = current_user.user_id

    command = FooUpdateCommand(
        request_user_id=user_id,
        foo_id=id,
        title=request.title,
        body=request.body,
        categories=tuple(request.categories),
    )
    updated = update_usecase.execute(user_id, command)
    assert updated.foo_id is not None
    return FooCreateResponse(id=updated.foo_id)


@router.delete("/{id}", summary="Foo 삭제", status_code=status.HTTP_204_NO_CONTENT, response_class=Response)
def delete_foo(
    id: int,
    current_user: FromDishka[CurrentUser],
    delete_usecase: FromDishka[FooDeleteUsecase],
) -> Response:
    user_id = current_user.user_id
    delete_usecase.execute(user_id, FooIdentity(id))
    return Response(status_code=status.HTTP_204_NO_CONTENT)
```

**특징**:
- 엔드포인트 5개 (list/detail/create/update/delete) + 선택적 2개 (me/view 등)
- Reader 1개 + Usecase N개 (create/update/delete/detail), 핸들러마다 필요한 것만 파라미터로
- Request DTO 2~3개 (Create/Update, 필요 시 Query)
- Response DTO 3~4개 (Response/DetailResponse/ListResponse/CreateResponse)
- enum 파싱 helper 없음 — `FooCreateRequest.categories: list[FooCategory]` 가 검증

---

## Pattern Bar — Action Router (동작형 리소스)

**언제 쓰나**: "행동" 이 리소스인 경우. bookmark/rating 같이 **상태 전이** 중심.

```python
router = APIRouter(prefix="/api/bookmarks", tags=["Bookmark"], route_class=DishkaSyncRoute)


@router.post("", summary="북마크 추가", status_code=status.HTTP_201_CREATED, response_class=Response)
def bookmark(
    request: BookmarkRequest,
    current_user: FromDishka[CurrentUser],
    bookmark_usecase: FromDishka[BookmarkUsecase],
) -> Response:
    user_id = current_user.user_id
    bookmark_usecase.execute(user_id, request.setup_id)
    return Response(status_code=status.HTTP_201_CREATED)


@router.delete("/{setup_id}", summary="북마크 취소", status_code=status.HTTP_204_NO_CONTENT, response_class=Response)
def unbookmark(
    setup_id: int,
    current_user: FromDishka[CurrentUser],
    unbookmark_usecase: FromDishka[UnbookmarkUsecase],
) -> Response:
    user_id = current_user.user_id
    unbookmark_usecase.execute(user_id, setup_id)
    return Response(status_code=status.HTTP_204_NO_CONTENT)


@router.get("", summary="내 북마크 목록")
def get_my_bookmarks(
    current_user: FromDishka[CurrentUser],
    saved_post_reader: FromDishka[SavedPostReader],
    page: int = 0,
    size: int = 20,
) -> BookmarkListResponse:
    reads = saved_post_reader.get_by_user_id(current_user.user_id, page, size)
    return BookmarkListResponse.from_reads(reads)
```

**특징**:
- CRUD 풀세트가 아니라 **주요 동작 2~4개**
- Usecase 위주 — 동작 하나당 Usecase 하나
- Path 파라미터는 대상 리소스의 ID (여기선 setup_id) — bookmark 자체 id 가 아님

---

## Pattern Baz — Read-only Public Router (인증 없음)

**언제 쓰나**: 정적·reference 데이터. `CurrentUser` 파라미터 없음.

```python
router = APIRouter(prefix="/api/ai-doc-series", tags=["AiDocSeries"], route_class=DishkaSyncRoute)


@router.get("", summary="시리즈 목록")
def get_all(reader: FromDishka[AiDocSeriesReader]) -> AiDocSeriesListResponse:
    reads = reader.get_all()
    return AiDocSeriesListResponse(data=[AiDocSeriesResponse.from_read(r) for r in reads])


@router.get("/{id}", summary="시리즈 상세")
def get_by_id(id: int, reader: FromDishka[AiDocSeriesReader]) -> AiDocSeriesResponse:
    read = reader.get_by_id(AiDocSeriesIdentity(id))
    return AiDocSeriesResponse.from_read(read)
```

**특징**:
- GET 만
- `CurrentUser` 없음 — dishka 가 provider 를 호출하지 않으므로 헤더가 없어도 401 이 나지 않는다
- Reader 만 의존

---

## Pattern Qux — 집계·마이페이지 Router (Usecase 중심)

**언제 쓰나**: 여러 도메인을 한 엔드포인트로 조합 반환. 대시보드·마이페이지.

```python
router = APIRouter(prefix="/api/mypage", tags=["Mypage"], route_class=DishkaSyncRoute)


@router.get("", summary="마이페이지 대시보드")
def get_mypage(
    current_user: FromDishka[CurrentUser], mypage_usecase: FromDishka[MypageUsecase]
) -> MypageResponse:
    read = mypage_usecase.execute(current_user.user_id)
    return MypageResponse.from_read(read)


@router.get("/setups", summary="내 글 목록")
def get_my_setups(
    current_user: FromDishka[CurrentUser],
    mypage_setup_usecase: FromDishka[MypageSetupUsecase],
    page: int = 0,
    size: int = 20,
) -> SetupListResponse:
    result = mypage_setup_usecase.execute(current_user.user_id, page, size)
    return SetupListResponse.from_list(result)
```

**특징**:
- Reader 미주입 (있어도 되지만 주력 로직은 Usecase)
- `CurrentUser` 필수 (내 데이터이므로)
- Usecase 가 여러 도메인 Repository 를 직접 조립 (service-references §Qux 참조)

---

## 패턴 선택 가이드

| 엔드포인트 특성 | Pattern |
|---|---|
| 리소스 명사형, 인증, 전체 CRUD | **Foo** (주력) |
| 동작형 리소스 (bookmark/rating) | **Bar** |
| 공개 조회 전용 | **Baz** |
| 여러 도메인 집계 (마이페이지/대시보드) | **Qux** |

---

## 💡 팁 — 위 패턴들로 풀리지 않을 때

1. **파일 분리 먼저 의심** — 한 Router 에 핸들러가 7~8개 이상이면 도메인 경계 재검토. Foo + Bar 혼합인 경우 분리.
2. **핸들러에 로직 들어가지 않게** — 핸들러 본문이 request → command → usecase.execute → response 의 4줄 이상이면 Usecase 로 밀어내기.
3. **Response DTO 는 Read 모델과 1:N 관계일 수 있음** — 같은 Read 로 DetailResponse/ListResponse/MeResponse 등 여러 뷰 투영이 자연스러움.
4. 그래도 표현이 어색하면 **유저에게 질의** — GraphQL / BFF / 특수 endpoint 스펙 등 구조적 선택지.

---

## 파일 구성 요약

```
packages/api/src/{pkg}/api/{domain}/
├── __init__.py
├── {domain}_router.py                  ← Pattern Foo / Bar / Baz / Qux
└── dto/
    ├── __init__.py
    ├── {domain}_create_request.py      ← BaseModel + Field 검증
    ├── {domain}_update_request.py
    ├── {domain}_response.py            ← BaseModel + from_read(...)
    ├── {domain}_detail_response.py
    ├── {domain}_list_response.py       ← List + total envelope
    ├── {domain}_create_response.py     ← id 만 담는 생성 응답
    └── {domain}_me_response.py         ← 인증된 사용자 상태 (bookmark/my_rating 등, 선택)

packages/app/src/{pkg}/app/routers.py   ← ROUTERS 에 등록 (add-api 가 추가)
```

---

## 공통 금지 사항

- `route_class=DishkaSyncRoute` 누락 금지 — `FromDishka` 파라미터가 주입되지 않는다
- 핸들러 `async def` 금지 — `def + Session` 결정. 스레드풀 실행
- `Depends(...)` 로 service 객체 받기 금지 — dishka 로 통일
- Response DTO 에 도메인 모델 (`{Domain}Read`) 직접 노출 금지 — 항상 Response 로 투영
- Writer 를 핸들러에 직접 주입 금지 — Usecase 경유
- 핸들러에서 `@transactional` / `TransactionManager` 사용 금지 — 트랜잭션 경계는 service
- Request DTO 에 비즈니스 로직 메서드 금지 — 순수 데이터 홀더
- Response DTO 에 `from_domain({Domain})` (Write 모델) 금지 — `from_read({Domain}Read)` 만
- `{pkg}.app` import 금지 — 의존 방향 역전 (import-linter)
- 헤더·쿠키 직접 읽어 인증 금지 — `FromDishka[CurrentUser]` 로 받음
