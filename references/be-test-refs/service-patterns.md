# Service 단위 테스트 패턴 레퍼런스

SKILL.md의 패턴 레터에서 참조하는 코드 예제 모음. 1 레터 = 1 예제.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 패키지)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

공통 전제:

```python
from dataclasses import replace
from unittest.mock import ANY, Mock

import pytest

from {pkg}.transaction import NoopTransactionManager

# sut 는 픽스처나 각 테스트에서 명시 생성. @InjectMocks 대응 없음.
@pytest.fixture
def repository() -> Mock:
    return Mock(spec=FooRepository)


@pytest.fixture
def sut(repository: Mock) -> DefaultFooWriter:
    return DefaultFooWriter(tx=NoopTransactionManager(), repository=repository)
```

- `Mock(spec=ABC)` — spec 을 주면 없는 메서드 호출이 `AttributeError` 로 잡힌다
- `NoopTransactionManager` — `@transactional` 이 그냥 통과. 트랜잭션 동작 자체는 통합 테스트 영역
- 도메인 객체는 팩토리로 직접 생성. id 가 필요하면 `replace(Foo.create(...), foo_id=1)`

---

## A. Usecase — happy path

```python
def test_should_create_foo_when_valid_request(sut, user_repository, foo_writer, user_stats_repository):
    # given
    user_repository.find_by_id.return_value = _user(user_id=1)
    foo_writer.create.return_value = FooIdentity(100)

    # when
    result = sut.execute(1, command)

    # then
    assert result.foo_id == 100
    user_stats_repository.increase_post_count.assert_called_once_with(UserIdentity(1))
```

## B. Usecase — 유저 미존재

```python
def test_should_raise_user_not_found_when_user_not_exists(sut, user_repository, foo_writer):
    # given
    user_repository.find_by_id.return_value = None

    # when & then
    with pytest.raises(UserNotFoundException):
        sut.execute(999, command)
    foo_writer.create.assert_not_called()
```

## C. Usecase — 엔티티 미존재

```python
def test_should_raise_foo_not_found_when_foo_not_exists(sut, user_repository, foo_repository):
    # given
    user_repository.find_by_id.return_value = _user(user_id=1)
    foo_repository.find_write_by_id.return_value = None

    # when & then
    with pytest.raises(FooNotFoundException):
        sut.execute(1, command)
```

## D. Usecase — 소유권 실패

```python
def test_should_raise_forbidden_when_not_owner(sut, user_repository, foo_repository):
    # given
    user_repository.find_by_id.return_value = _user(user_id=1)
    foo_repository.find_write_by_id.return_value = _foo(foo_id=1, user_id=999)

    # when & then
    with pytest.raises(ForbiddenException):
        sut.execute(1, command)
```

## E. Usecase — 비즈니스 규칙 위반

```python
def test_should_raise_conflict_when_already_bookmarked(sut, user_repository, saved_post_reader):
    # given
    user_repository.find_by_id.return_value = _user(user_id=1)
    saved_post_reader.is_bookmarked.return_value = True

    # when & then
    with pytest.raises(ConflictException):
        sut.execute(1, 10)
```

## F. Reader — 단건 정상

```python
def test_should_return_foo_read_when_exists(sut, repository):
    # given
    expected = _read(foo_id=1)
    repository.find_by_id.return_value = expected

    # when & then
    assert sut.get_by_id(FooIdentity(1)) is expected
```

## G. Reader — 단건 미존재

```python
def test_should_raise_not_found_when_foo_not_exists(sut, repository):
    # given
    repository.find_by_id.return_value = None

    # when & then
    with pytest.raises(FooNotFoundException):
        sut.get_by_id(FooIdentity(999))
```

## H. Reader — 목록

```python
def test_should_return_list_when_data_exists(sut, repository):
    repository.find_by_user_id.return_value = [_read(foo_id=1), _read(foo_id=2)]
    assert len(sut.get_by_user_id(1, 0, 10)) == 2


def test_should_return_empty_list_when_no_data(sut, repository):
    repository.find_by_user_id.return_value = []
    assert sut.get_by_user_id(999, 0, 10) == []
```

## I. Reader — 카운트

```python
def test_should_return_count_when_called(sut, repository):
    repository.count_by_user_id.return_value = 5
    assert sut.count_by_user_id(1) == 5
```

## J. Reader — `X | None` 반환

```python
def test_should_return_score_when_rating_exists(sut, rating_repository):
    rating_repository.find_by_setup_id_and_user_id.return_value = _rating(score=4)
    assert sut.get_score(1, 2) == 4


def test_should_return_none_when_no_rating(sut, rating_repository):
    rating_repository.find_by_setup_id_and_user_id.return_value = None
    assert sut.get_score(1, 2) is None
```

## K. Writer — 생성 + 부수효과

```python
def test_should_save_and_init_stats_when_create(sut, repository, stats_repository):
    # given
    repository.save.side_effect = lambda foo: replace(foo, foo_id=100)

    # when
    result = sut.create(command)

    # then
    assert result.foo_id == 100
    stats_repository.save.assert_called_once()
    saved_stats = stats_repository.save.call_args.args[0]
    assert saved_stats.foo_id == 100
```

`save` 가 받은 도메인에 id 만 붙여 돌려주는 `side_effect` 가 Java 의 `thenAnswer(inv -> inv.getArgument(0))` 대응.

## L. Writer — 수정 (조회→변환→저장)

```python
def test_should_update_foo_when_valid_command(sut, repository):
    # given
    existing = _foo(foo_id=1, title="기존")
    repository.find_write_by_id.return_value = existing
    repository.save.side_effect = lambda foo: foo

    # when
    result = sut.update(command)

    # then
    saved = repository.save.call_args.args[0]
    assert saved.title == command.title
    assert saved.foo_id == 1
    assert result.title == command.title
```

## M. Writer — 미존재 예외

```python
def test_should_raise_not_found_when_update_missing(sut, repository):
    repository.find_write_by_id.return_value = None

    with pytest.raises(FooNotFoundException):
        sut.update(command)
    repository.save.assert_not_called()
```

## N. Writer — 상태 전이 (hide/show)

```python
def test_should_hide_qux_when_not_hidden(sut, repository):
    qux = Qux.create_root(1, QuxArticleType.SETUP, 1, "내용", 1)
    repository.find_write_by_id.return_value = qux
    repository.save.side_effect = lambda q: q

    sut.hide(QuxIdentity(1))

    saved = repository.save.call_args.args[0]
    assert saved.is_hidden is True


def test_should_raise_wrong_qux_when_already_hidden(sut, repository):
    qux = Qux.create_root(1, QuxArticleType.SETUP, 1, "내용", 1).hide()
    repository.find_write_by_id.return_value = qux

    with pytest.raises(WrongQuxException):
        sut.hide(QuxIdentity(1))
```

## O. Writer — 분기 로직 (루트 vs 답글)

```python
def test_should_create_root_qux_when_no_parent(sut, repository):
    cmd = QuxCreateCommand(request_user_id=1, article_id=1, article_type=QuxArticleType.SETUP, text="내용", parent_id=None)
    repository.get_max_comment_order.return_value = 3
    repository.save.side_effect = lambda q: replace(q, qux_id=10)

    sut.create(cmd)

    saved = repository.save.call_args.args[0]
    assert saved.qux_order == 4
    assert saved.level == 0
```

## P. Usecase — 분기별 TC (신규 vs 수정)

```python
def test_should_add_rating_when_first_time(sut, rating_reader, user_stats_repository):
    rating_reader.get_score.return_value = None
    # ... 나머지 stub 생략
    sut.execute(1, command)
    user_stats_repository.add_rating.assert_called_once()
    user_stats_repository.update_rating.assert_not_called()


def test_should_update_rating_when_already_rated(sut, rating_reader, user_stats_repository):
    rating_reader.get_score.return_value = 3
    sut.execute(1, command)
    user_stats_repository.update_rating.assert_called_once_with(ANY, 3, ANY)
    user_stats_repository.add_rating.assert_not_called()
```

## Q. Usecase — 다중 메서드

```python
def test_should_return_my_posts_when_valid_user(sut, user_repository, foo_reader):
    user_repository.find_by_id.return_value = _user(user_id=1)
    foo_reader.get_by_user_id.return_value = [_read(foo_id=1)]
    assert len(sut.get_my_posts(1, 0, 10)) == 1
```

## R. `@transactional` 부착 확인 (선택)

Writer/Usecase 의 public 메서드에 데코레이터가 붙었는지 정적으로 확인한다. `NoopTransactionManager` 로 도는 단위 테스트에서는 트랜잭션 누락이 드러나지 않으므로, 한 번 짚어둔다:

```python
def test_writer_public_methods_are_transactional():
    for name in ("create", "update", "delete"):
        method = getattr(DefaultFooWriter, name)
        assert getattr(method, "__wrapped__", None) is not None, f"{name} 에 @transactional 없음"
```

`functools.wraps` 가 `__wrapped__` 를 남기므로 데코레이터 유무를 알 수 있다.
