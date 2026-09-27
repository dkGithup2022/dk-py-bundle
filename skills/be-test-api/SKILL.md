---
name: be-test-api
description: api 패키지의 Router 단위 테스트(unittest.mock)를 평가하고 작성한다.
---

# be-test-api

api 패키지의 Router 단위 테스트를 3단계로 수행한다.
라우터 함수 직접 호출 방식 (TestClient 사용하지 않음).

---

## 입력

```
대상 Router 모듈 이름 (예: setup_router, comment_router)
```

입력이 없으면 "어떤 Router를 테스트할까요?" 로 확인한다.

---

## Phase 0. 프로젝트 컨텍스트 해석

이 스킬은 `{pkg}`, `{packages.api}` 플레이스홀더를 사용한다.
실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md`의 절차로 해석한다:

1. 프로젝트 루트의 `dk-py-bundle.config.json` 확인
2. 없으면 `packages/model/src/` 아래 디렉터리에서 `{pkg}` 추출
3. `{packages.api}` 존재 확인
4. 탐지 실패 시 유저에게 질의

---

## Phase 1. assess-infra

`${CLAUDE_PLUGIN_ROOT}/references/be-test-refs/infra-checklist.md` §공통 + §api 참조.

| 항목 | 경로 | 확인 |
|------|------|------|
| pytest | 루트 `pyproject.toml` dev 그룹 | 있는지 |
| tests 디렉터리 | `{packages.api}/tests/` | 있는지 |

있으면 PASS, 없으면 추가.

**직접 호출 가능 여부 확인 (최초 1회)**: `route_class=DishkaSyncRoute` 가 함수 객체를 감싸지 않는지 확인한다.

```bash
uv run python -c "
from {pkg}.api.{domain}.{domain}_router import {handler}
import inspect; print(inspect.signature({handler}))
"
```

시그니처에 `FromDishka[...]` 파라미터가 그대로 보이면 직접 호출 가능 → Phase 3 기본 방식. 감싸져 있으면 api-patterns.md §M TestClient fallback 을 쓴다.

**불필요**: TestClient, dishka 컨테이너, DB, conftest — 컨텍스트 없음.

---

## Phase 2. assess-scope

### 2-1. 대상 분석

1. Router 소스를 Read로 읽는다
2. 각 핸들러 함수를 추출:
   - HTTP 메서드 + 경로 + 데코레이터의 `status_code`
   - 파라미터 (경로, 쿼리, 본문 `BaseModel`, `FromDishka[CurrentUser]`)
   - 반환 타입 (`{Domain}Response` / `Response`)
   - `FromDishka[...]` 로 받는 Reader/Usecase

### 2-2. 의존성 추출 + specs lookup

1. 핸들러의 `FromDishka[...]` 파라미터 전부 → `Mock(spec=...)` 후보
2. 각 의존성의 API 시그니처를 확인:
   - `specs/usecase.md` — Reader ABC 메서드 / Usecase `execute` 시그니처
   - `specs/api.md` — 엔드포인트별 요청/응답 스펙
3. 실제 ABC 소스도 Read로 확인 (specs와 차이 있을 수 있음)

### 2-3. 테스트 케이스 도출

각 핸들러에 대해 **정상 경로 + 예외 경로**를 도출한다.

#### 정상 경로 TC

| HTTP 메서드 | 반환 | TC |
|------------|------|-----|
| GET (단건) | `XxxResponse` | 반환 모델 필드 검증 + 의존 호출 인자 검증 |
| GET (목록) | `XxxListResponse` | data 길이 / total 검증 |
| POST (생성, 201) | `XxxCreateResponse` | id 검증 + Command 변환 검증 (`request_user_id`, `list → tuple`) + 라우트 `status_code == 201` |
| PUT (수정) | `XxxCreateResponse` | id 검증 |
| DELETE (204) | `Response` | `response.status_code == 204` + `assert_called_once_with` |

#### 예외 경로 TC — **Usecase 예외 전파**

| 소스 | 예외 타입 | TC |
|------|----------|-----|
| Usecase → None | `EntityNotFoundException` 하위 | 404 코드+메시지 검증 |
| Usecase → 소유권 불일치 | `ForbiddenException` | 403 코드+메시지 검증 |
| Usecase → 중복 | `ConflictException` | 409 코드+메시지 검증 |
| Usecase → 규칙 위반 | `InvalidInputException` | 400 코드+메시지 검증 |
| Pydantic 검증 실패 | `ValidationError` | Request DTO 생성 시점 검증 (라우터 함수 밖) |

라우터 내부 파싱 분기는 없다 (Pydantic 이 검증). Java 의 `InvalidInputException` 400 파싱 TC 는 Request DTO `ValidationError` TC 로 대체.

#### 예외 검증 규칙 (핵심)

모든 예외 TC에서 `http_status_code` / `http_error_message` 를 반드시 검증한다:

```python
with pytest.raises(XxxException) as ei:
    handler(...)
assert ei.value.http_status_code == expected_code
assert ei.value.http_error_message == expected_message
```

**`error_handlers.py` 가 이 두 속성을 그대로 사용하므로, 예외 객체에서 직접 검증하면 HTTP 응답 검증과 동일하다.**

#### 추가 TC 규칙

| 코드 패턴 | TC 추가 |
|----------|---------|
| 핸들러 내부 `if/else` (있다면) | 분기별 TC 각각 |
| `FromDishka[CurrentUser]` | `CurrentUser(user_id=42)` 로 user_id 전달 검증 |
| `list → tuple` 변환 | Command 의 `tuple` 확인 |
| None 반환 Usecase | `assert_called_once_with(...)` |

### 2-4. 산출물

```
## 분석 결과: {router 모듈}
### 호출 방식: 직접 호출 / TestClient
### Mock 의존성
| 의존성 | 역할 |
### 핸들러별 테스트 케이스
| # | 핸들러 | 테스트명 | 유형 | 검증 |
```

사용자 확인 후 Phase 3 진행.

---

## Phase 3. add-test

### 파일 위치

```
{packages.api}/tests/{domain}/
└── test_{domain}_router.py
```

### 모듈 템플릿

```python
from dataclasses import replace
from unittest.mock import Mock

import pytest
from fastapi import status

from {pkg}.api.auth import CurrentUser
from {pkg}.api.{domain}.{domain}_router import get_xxx, create_xxx, ...
from {pkg}.api.{domain}.dto.{domain}_create_request import XxxCreateRequest
from {pkg}.model.audit import utcnow
from {pkg}.model.{domain}.{domain}_identity import XxxIdentity
from {pkg}.model.{domain}.{domain}_read import XxxRead
from {pkg}.service.{domain}.{domain}_reader import XxxReader
from {pkg}.service.{domain}.usecase.{domain}_create_usecase import XxxCreateUsecase

current_user = CurrentUser(user_id=1)
user_id = 1


def _read(**overrides) -> XxxRead:
    base = dict(xxx_id=1, title="제목", ..., created_at=utcnow(), updated_at=utcnow())
    return XxxRead(**{**base, **overrides})
```

### Request DTO 생성

Pydantic 이라 kwargs 로 바로 만든다. 검증 실패 TC 는 `pytest.raises(ValidationError)`.

### 작성 규칙

1. **네이밍**: `test_should_예상결과_when_조건`
2. **구조**: `# given` → `# when` → `# then`
3. **정상 검증**: 반환 모델 필드 `assert response.id == ...`. `Response` 반환은 `response.status_code`
4. **선언 상태 코드 검증**: 201/204 는 라우트 메타 `router.routes` 에서 (api-patterns.md §D)
5. **예외 검증**: `pytest.raises` + `http_status_code` + `http_error_message` 동시 검증
6. **부수효과**: `mock.execute.assert_called_once_with(...)` / `assert_not_called()`
7. **Read 모델**: `_read()` 헬퍼로 직접 생성. `Mock` 으로 만들지 않는다 (frozen dataclass 는 싸다)
8. **컨텍스트 금지**: dishka 컨테이너, TestClient, DB 없음. Mock 만

### 예외 상태코드·메시지 매핑 참조

| 예외 클래스 | 기본 http_status_code | 기본 http_error_message |
|-----------|--------------------|--------------------|
| `EntityNotFoundException` 하위 | 404 | "해당 리소스를 찾을 수 없습니다" |
| `ForbiddenException` | 403 | "접근 권한이 없습니다" |
| `InvalidInputException` | 400 | "잘못된 요청입니다" |
| `ConflictException` | 409 | "이미 존재하는 리소스입니다" |
| `UnauthorizedException` | 401 | "인증이 필요합니다" |

커스텀 메시지로 생성된 예외도 있으므로, 실제 Usecase/Exception 소스를 확인하여 정확한 값으로 검증한다.

코드 예제는 `${CLAUDE_PLUGIN_ROOT}/references/be-test-refs/api-patterns.md`의 패턴 레터 참조.

### 실제 예시: test_bookmark_router.py

```python
from unittest.mock import Mock

import pytest
from fastapi import status

from {pkg}.api.auth import CurrentUser
from {pkg}.api.bookmark.bookmark_router import bookmark, unbookmark
from {pkg}.api.bookmark.dto.bookmark_request import BookmarkRequest
from {pkg}.exception import ConflictException, InvalidInputException
from {pkg}.exception.user.user_not_found_exception import UserNotFoundException
from {pkg}.service.bookmark.usecase.bookmark_usecase import BookmarkUsecase
from {pkg}.service.bookmark.usecase.unbookmark_usecase import UnbookmarkUsecase

current_user = CurrentUser(user_id=1)
user_id = 1
setup_id = 10


@pytest.fixture
def bookmark_usecase() -> Mock:
    return Mock(spec=BookmarkUsecase)


@pytest.fixture
def unbookmark_usecase() -> Mock:
    return Mock(spec=UnbookmarkUsecase)


# --- 정상: POST 201 ---
def test_should_return_created_when_bookmark_success(bookmark_usecase):
    # given
    request = BookmarkRequest(setup_id=setup_id)

    # when
    response = bookmark(request, current_user=current_user, bookmark_usecase=bookmark_usecase)

    # then
    assert response.status_code == status.HTTP_201_CREATED
    bookmark_usecase.execute.assert_called_once_with(user_id, setup_id)


# --- 예외: 404 + http_status_code/http_error_message 검증 ---
def test_should_raise_404_when_bookmark_user_not_found(bookmark_usecase):
    # given
    bookmark_usecase.execute.side_effect = UserNotFoundException("user not found: 999")

    # when & then
    with pytest.raises(UserNotFoundException) as ei:
        bookmark(BookmarkRequest(setup_id=setup_id), current_user=current_user, bookmark_usecase=bookmark_usecase)
    assert ei.value.http_status_code == 404
    assert ei.value.http_error_message == "사용자를 찾을 수 없습니다"


# --- 예외: 409 ---
def test_should_raise_409_when_already_bookmarked(bookmark_usecase):
    bookmark_usecase.execute.side_effect = ConflictException("already bookmarked")

    with pytest.raises(ConflictException) as ei:
        bookmark(BookmarkRequest(setup_id=setup_id), current_user=current_user, bookmark_usecase=bookmark_usecase)
    assert ei.value.http_status_code == 409
    assert ei.value.http_error_message == "이미 존재하는 리소스입니다"


# --- 정상: DELETE 204 ---
def test_should_return_no_content_when_unbookmark_success(unbookmark_usecase):
    response = unbookmark(setup_id, current_user=current_user, unbookmark_usecase=unbookmark_usecase)

    assert response.status_code == status.HTTP_204_NO_CONTENT
    unbookmark_usecase.execute.assert_called_once_with(user_id, setup_id)


# --- 예외: 400 ---
def test_should_raise_400_when_not_bookmarked(unbookmark_usecase):
    unbookmark_usecase.execute.side_effect = InvalidInputException("not bookmarked")

    with pytest.raises(InvalidInputException) as ei:
        unbookmark(setup_id, current_user=current_user, unbookmark_usecase=unbookmark_usecase)
    assert ei.value.http_status_code == 400
    assert ei.value.http_error_message == "잘못된 요청입니다"
```

> 참고: `UserNotFoundException`의 `http_error_message`는 기본값 `"해당 리소스를 찾을 수 없습니다"`가 아니라
> 커스텀 `"사용자를 찾을 수 없습니다"`이다. **반드시 실제 Exception 소스를 확인할 것.**

### 검증

```bash
uv run pytest {packages.api}/tests/{domain}/test_{domain}_router.py -v
```

---

## 공통 규칙

- Phase 1은 최초 1회만 실행
- Phase 2 산출물을 사용자에게 보여주고 확인 후 Phase 3 진행
- 테스트 코드 내에서 프로덕션 코드를 수정하지 않는다
- 기존 테스트 파일이 있으면 덮어쓰지 않고 추가/수정한다
