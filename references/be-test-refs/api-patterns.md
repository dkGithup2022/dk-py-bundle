# API Router 단위 테스트 패턴 레퍼런스

SKILL.md의 패턴 레터에서 참조하는 코드 예제 모음. 1 레터 = 1 예제.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 패키지)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

공통 전제: 라우터 함수를 **직접 호출**한다. `FromDishka[...]` 파라미터는 kwargs 로 Mock 을 넘긴다. Spring 컨텍스트도 dishka 컨테이너도 띄우지 않는다.

```python
from unittest.mock import Mock

import pytest
from fastapi import status

from {pkg}.api.auth import CurrentUser
from {pkg}.api.foo import foo_router
from {pkg}.api.foo.foo_router import create_foo, delete_foo, get_foo, get_foos, update_foo
from {pkg}.exception import ConflictException, ForbiddenException, InvalidInputException
from {pkg}.exception.foo.foo_not_found_exception import FooNotFoundException
from {pkg}.model.foo.foo_identity import FooIdentity
from {pkg}.model.foo.foo_read import FooRead
from {pkg}.service.foo.foo_reader import FooReader
from {pkg}.service.foo.usecase.foo_create_usecase import FooCreateUsecase

current_user = CurrentUser(user_id=1)
user_id = 1
```

---

## A. GET — 단건 정상 (200)

```python
def test_should_return_response_when_foo_exists():
    # given
    read = FooRead(foo_id=1, title="제목", author="tester", categories=(), created_at=now, updated_at=now)
    detail_usecase = Mock(spec=FooDetailUsecase)
    detail_usecase.execute.return_value = read

    # when
    response = get_foo(1, detail_usecase=detail_usecase)

    # then
    assert response.id == 1
    assert response.title == "제목"
    detail_usecase.execute.assert_called_once_with(FooIdentity(1))
```

Read 모델은 frozen dataclass 라 팩토리로 직접 만드는 것이 Mock 보다 짧다. 필드가 많으면 테스트 모듈 상단에 `_read(**overrides)` 헬퍼를 둔다.

## B. GET — 단건 미존재 (404 검증)

```python
def test_should_raise_404_when_foo_not_found():
    # given
    detail_usecase = Mock(spec=FooDetailUsecase)
    detail_usecase.execute.side_effect = FooNotFoundException("foo not found: 999")

    # when & then
    with pytest.raises(FooNotFoundException) as ei:
        get_foo(999, detail_usecase=detail_usecase)
    assert ei.value.http_status_code == 404
    assert ei.value.http_error_message == "해당 리소스를 찾을 수 없습니다"
```

`error_handlers.py` 가 `http_status_code` / `http_error_message` 를 그대로 쓰므로, 예외 객체에서 직접 검증하면 HTTP 응답 검증과 동일하다.

## C. GET — 목록 정상 (200)

```python
def test_should_return_list_when_valid_params():
    # given
    reader = Mock(spec=FooReader)
    reader.get_all.return_value = [_read(foo_id=1), _read(foo_id=2)]
    reader.count_all.return_value = 2

    # when
    response = get_foos(reader=reader, sort="popular", q="", page=0, size=20)

    # then
    assert len(response.data) == 2
    assert response.total == 2
    reader.get_all.assert_called_once_with("popular", "", 0, 20)
```

## D. POST — 생성 정상 (201)

```python
def test_should_return_created_when_valid_request():
    # given
    request = FooCreateRequest(title="제목", body="본문", categories=[FooCategory.VIBE_CODING])
    create_usecase = Mock(spec=FooCreateUsecase)
    create_usecase.execute.return_value = FooIdentity(100)

    # when
    response = create_foo(request, current_user=current_user, create_usecase=create_usecase)

    # then
    assert response.id == 100
    command = create_usecase.execute.call_args.args[1]
    assert command.request_user_id == user_id
    assert command.categories == (FooCategory.VIBE_CODING,)   # list → tuple 변환 확인
```

상태 코드 201 은 데코레이터에 선언되어 있다. 검증하려면 라우트 메타를 본다:

```python
def test_create_route_declares_201():
    route = next(r for r in foo_router.router.routes if r.path == "/api/foos" and "POST" in r.methods)
    assert route.status_code == status.HTTP_201_CREATED
```

## E. PUT — 수정 정상 (200)

```python
def test_should_return_ok_when_update_valid():
    # given
    request = FooUpdateRequest(title="수정", body="본문", categories=[])
    update_usecase = Mock(spec=FooUpdateUsecase)
    update_usecase.execute.return_value = Foo.create(user_id, "수정", "본문", ()) .__class__(
        foo_id=1, user_id=user_id, title="수정", body="본문", categories=(), created_at=now, updated_at=now
    )

    # when
    response = update_foo(1, request, current_user=current_user, update_usecase=update_usecase)

    # then
    assert response.id == 1
```

Write 모델을 만들 때는 팩토리(`create`)가 `foo_id=None` 을 주므로, id 가 필요한 경우 `dataclasses.replace(Foo.create(...), foo_id=1)` 로 채운다.

## F. DELETE — 삭제 정상 (204)

```python
def test_should_return_no_content_when_delete_success():
    # given
    delete_usecase = Mock(spec=FooDeleteUsecase)

    # when
    response = delete_foo(1, current_user=current_user, delete_usecase=delete_usecase)

    # then
    assert response.status_code == status.HTTP_204_NO_CONTENT
    delete_usecase.execute.assert_called_once_with(user_id, FooIdentity(1))
```

## G. 예외 — 소유권 불일치 (403)

```python
def test_should_raise_403_when_not_owner():
    # given
    update_usecase = Mock(spec=FooUpdateUsecase)
    update_usecase.execute.side_effect = ForbiddenException("not owner of foo: 1")

    # when & then
    with pytest.raises(ForbiddenException) as ei:
        update_foo(1, request, current_user=current_user, update_usecase=update_usecase)
    assert ei.value.http_status_code == 403
    assert ei.value.http_error_message == "접근 권한이 없습니다"
```

## H. 예외 — 잘못된 입력 (400)

Pydantic 검증 실패는 라우터 함수에 도달하기 전에 일어나므로 **직접 호출 테스트로는 재현되지 않는다**. Request DTO 생성 시점에 검증한다:

```python
def test_should_reject_empty_title():
    with pytest.raises(ValidationError):
        FooCreateRequest(title="", body="본문", categories=[])
```

Usecase 가 던지는 비즈니스 규칙 위반은 B/G 와 같은 패턴:

```python
def test_should_raise_400_when_invalid_input():
    usecase = Mock(spec=RatingUsecase)
    usecase.execute.side_effect = InvalidInputException("score out of range")

    with pytest.raises(InvalidInputException) as ei:
        rate(1, request, current_user=current_user, rating_usecase=usecase)
    assert ei.value.http_status_code == 400
    assert ei.value.http_error_message == "잘못된 요청입니다"
```

## I. 예외 — 충돌 (409)

```python
def test_should_raise_409_when_already_bookmarked():
    bookmark_usecase = Mock(spec=BookmarkUsecase)
    bookmark_usecase.execute.side_effect = ConflictException("already bookmarked: setup_id=1")

    with pytest.raises(ConflictException) as ei:
        bookmark(BookmarkRequest(setup_id=1), current_user=current_user, bookmark_usecase=bookmark_usecase)
    assert ei.value.http_status_code == 409
    assert ei.value.http_error_message == "이미 존재하는 리소스입니다"
```

## J. CurrentUser 전달 — user_id 추출

```python
def test_should_pass_user_id_when_authenticated():
    # given
    create_usecase = Mock(spec=FooCreateUsecase)
    create_usecase.execute.return_value = FooIdentity(1)

    # when
    create_foo(request, current_user=CurrentUser(user_id=42), create_usecase=create_usecase)

    # then
    called_user_id, command = create_usecase.execute.call_args.args
    assert called_user_id == 42
    assert command.request_user_id == 42
```

## K. Request DTO 생성

Pydantic 이라 kwargs 로 바로 만든다. Java 의 Reflection 헬퍼는 필요 없다.

```python
request = FooCreateRequest(title="제목", body="본문", categories=[FooCategory.VIBE_CODING])
request = FooCreateRequest.model_validate({"title": "제목", "body": "본문", "categories": ["VIBE_CODING"]})  # JSON 형태로
```

## L. None 반환 Usecase — assert_called 패턴

```python
def test_should_call_usecase_when_valid_request():
    # given
    usecase = Mock(spec=UnbookmarkUsecase)

    # when
    response = unbookmark(1, current_user=current_user, unbookmark_usecase=usecase)

    # then
    assert response.status_code == status.HTTP_204_NO_CONTENT
    usecase.execute.assert_called_once_with(user_id, 1)
```

## M. 직접 호출이 안 될 때 — TestClient fallback

`route_class=DishkaSyncRoute` 가 함수 객체를 감싸서 직접 호출이 실패하면 (Phase 1 에서 확인), HTTP 로 검증한다:

```python
from dishka import Provider, Scope, make_container, provide
from dishka.integrations.fastapi import setup_dishka
from fastapi.testclient import TestClient


def _client(**mocks) -> TestClient:
    class TestProvider(Provider):
        scope = Scope.REQUEST
        for name, mock in mocks.items():
            locals()[name] = provide(lambda m=mock: m, provides=type(mock)._spec_class)
    app = create_app()   # 실제 Provider 위에 TestProvider 가 덮어씀
    setup_dishka(make_container(TestProvider()), app)
    return TestClient(app)


def test_should_return_200_via_http():
    reader = Mock(spec=FooReader)
    reader.get_by_id.return_value = _read(foo_id=1)
    client = _client(reader=reader)

    res = client.get("/api/foos/1", headers={"X-User-Id": "1"})

    assert res.status_code == 200
    assert res.json()["id"] == 1
```

이 방식은 be-init 의 `RequestProvider` 도 그대로 타므로 `Session` 이 필요해질 수 있다. 직접 호출이 되는 한 쓰지 않는다.
