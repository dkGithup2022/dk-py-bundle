# persistence 테스트 패턴 레퍼런스

카테고리별 테스트 작성 패턴. `be-test-repository` 스킬의 Phase 3에서 참조한다.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 패키지)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

공통 전제:

```python
import pytest
from sqlalchemy.orm import Session

from {pkg}.persistence.transaction import DefaultTransactionManager
from {pkg}.persistence.baz.baz_entity_query import BazEntityQuery
from {pkg}.persistence.baz.baz_join_query import BazJoinQuery
from {pkg}.persistence.baz.baz_repository import DefaultBazRepository
from {pkg}.persistence.baz.baz_stats_entity_query import BazStatsEntityQuery
from {pkg}.persistence.baz.baz_stats_repository import DefaultBazStatsRepository
from {pkg}.persistence.user.user_entity_query import UserEntityQuery
from {pkg}.persistence.user.user_repository import DefaultUserRepository


@pytest.fixture
def tx(session: Session) -> DefaultTransactionManager:
    return DefaultTransactionManager(session)


@pytest.fixture
def entities(session: Session) -> BazEntityQuery:
    return BazEntityQuery(session)


@pytest.fixture
def joins(session: Session) -> BazJoinQuery:
    return BazJoinQuery(session)


@pytest.fixture
def repository(tx, entities, joins) -> DefaultBazRepository:
    return DefaultBazRepository(tx=tx, entities=entities, joins=joins)


@pytest.fixture
def stats_repository(session: Session, tx) -> DefaultBazStatsRepository:
    return DefaultBazStatsRepository(tx=tx, entities=BazStatsEntityQuery(session))


@pytest.fixture
def test_user_id(session: Session, tx) -> int:
    user_repository = DefaultUserRepository(tx=tx, entities=UserEntityQuery(session))
    user = user_repository.save(User.create("TestUser", "test@test.com", "google123", ""))
    assert user.user_id is not None
    return user.user_id
```

- `session` 은 루트 conftest 의 세이브포인트 세션. 테스트 끝에 전부 롤백된다
- 구현체는 `tx` + 쿼리 클래스를 같은 `session` 으로 조립해 직접 생성. 같은 `session` 을 쓰므로 선행 데이터가 보인다
- 구현체의 `@transactional` 은 세이브포인트 위에서 정상 동작한다 (`join_transaction_mode="create_savepoint"`)
- 쿼리 클래스를 직접 호출하는 테스트는 `with session.begin():` 안에서 호출한다 (쿼리 클래스에는 `@transactional` 이 없다)
- 선행 데이터는 반드시 구현체 API 로 넣는다. raw SQL INSERT 금지

---

## A. Entity↔Domain 변환 (구현체)

save → find_write_by_id → 모든 필드 왕복 검증. **모든 구현체에 필수**.

```python
def test_should_return_all_fields_when_saved_and_found_by_id(repository, test_user_id):
    # given
    to_save = Baz.create(test_user_id, BazType.ARTICLE, "제목", "본문",
                         (BazCategory.VIBE_CODING,), (BazTag.CLAUDE,), BazSource.SELF)

    # when
    saved = repository.save(to_save)
    found = repository.find_write_by_id(BazIdentity(saved.baz_id))

    # then
    assert found is not None
    assert found.baz_id == saved.baz_id
    assert found.user_id == test_user_id
    assert found.title == "제목"
    assert found.type == BazType.ARTICLE
    assert found.created_at is not None
    assert found.updated_at is not None
```

---

## B. 자식 컬렉션 (relationship) 왕복 (EntityQuery + 구현체)

```python
def test_should_load_children_when_found_by_entity_query(session, entities, test_user_id):
    # given — EntityQuery 레벨 왕복
    with session.begin():
        saved = entities.add(BazEntity.from_domain(
            Baz.create(test_user_id, BazType.ARTICLE, "제목", "본문",
                       (BazCategory.VIBE_CODING, BazCategory.DEV_EFFICIENCY), (), BazSource.SELF)))
        # when
        found = entities.find_by_id_and_active(saved.id)

    # then
    assert found is not None
    assert {c.category for c in found.categories} == {BazCategory.VIBE_CODING, BazCategory.DEV_EFFICIENCY}


def test_should_replace_collections_when_updated(repository, test_user_id):
    # given — 구현체 레벨. 자식 교체 시 orphan 이 지워지는지
    saved = repository.save(Baz.create(test_user_id, BazType.ARTICLE, "제목", "본문",
                                       (BazCategory.VIBE_CODING,), (), BazSource.SELF))

    # when
    repository.save(saved.with_update("제목", "본문", (BazCategory.DEV_EFFICIENCY,), (), BazSource.SELF))
    found = repository.find_write_by_id(BazIdentity(saved.baz_id))

    # then
    assert found is not None
    assert found.categories == (BazCategory.DEV_EFFICIENCY,)
```

순서가 의미 있는 자식(`order_by`)은 `set` 대신 튜플 그대로 비교한다.

---

## C. JOIN 결과 검증 (JoinQuery — 메서드마다 필수)

JoinQuery 메서드를 **직접** 호출해 SQL 결과 행을 검증한다. JOIN 대상 도메인의 선행 데이터 필요. 상대 행이 **없는 경우** 도 반드시 1개.

```python
def test_should_return_author_nickname_when_joined(session, joins, repository, stats_repository, test_user_id):
    # given — User 선행 생성 (test_user_id 픽스처), Stats 선행 생성
    saved = repository.save(Baz.create(test_user_id, BazType.ARTICLE, "제목", "본문", (), (), BazSource.SELF))
    stats_repository.save(BazStats.init(saved.baz_id))

    # when
    with session.begin():
        row = joins.find_by_id_with_details(saved.baz_id)

    # then
    assert row is not None
    assert row.author_nickname == "TestUser"
    assert row.view_count == 0


def test_should_coalesce_zero_when_stats_row_missing(session, joins, repository, test_user_id):
    # given — LEFT JOIN + COALESCE 확인 (stats 없음)
    saved = repository.save(Baz.create(test_user_id, BazType.ARTICLE, "제목", "본문", (), (), BazSource.SELF))

    # when
    with session.begin():
        row = joins.find_by_id_with_details(saved.baz_id)

    # then
    assert row is not None
    assert row.view_count == 0
    assert row.rating_count == 0


def test_should_exclude_deleted_when_joined(session, joins, repository, test_user_id):
    saved = repository.save(Baz.create(test_user_id, BazType.ARTICLE, "제목", "본문", (), (), BazSource.SELF))
    repository.delete_by_id(BazIdentity(saved.baz_id))

    with session.begin():
        row = joins.find_by_id_with_details(saved.baz_id)

    assert row is None
```

구현체의 `_to_read` 조립은 별도로 1개:

```python
def test_should_assemble_read_when_found_by_id(repository, stats_repository, test_user_id):
    saved = repository.save(Baz.create(test_user_id, BazType.ARTICLE, "제목", "본문", (BazCategory.VIBE_CODING,), (), BazSource.SELF))
    stats_repository.save(BazStats.init(saved.baz_id))

    found = repository.find_by_id(BazIdentity(saved.baz_id))

    assert found is not None
    assert found.author == "TestUser"
    assert found.categories == (BazCategory.VIBE_CODING,)   # 자식은 EntityQuery 에서
```

---

## D. `update()` 문 (EntityQuery)

실행 → 재조회 → 변경값 확인.

```python
def test_should_increment_view_count_when_increase_view_count_called(stats_repository, baz_id):
    # given
    stats_repository.save(BazStats.init(baz_id))

    # when
    stats_repository.increase_view_count(BazIdentity(baz_id), 5)

    # then
    found = stats_repository.find_by_baz_id(BazIdentity(baz_id))
    assert found is not None
    assert found.view_count == 5


def test_should_accumulate_when_increase_view_count_called_multiple_times(stats_repository, baz_id):
    stats_repository.save(BazStats.init(baz_id))

    stats_repository.increase_view_count(BazIdentity(baz_id), 3)
    stats_repository.increase_view_count(BazIdentity(baz_id), 7)

    found = stats_repository.find_by_baz_id(BazIdentity(baz_id))
    assert found is not None
    assert found.view_count == 10


def test_should_increment_only_matching_rows_when_increment_sort_numbers_above_called(qux_repository):
    # given — qux_order=1 인 댓글 3개 (sort_number: 1, 2, 3)
    ids = [qux_repository.save(_qux(qux_order=1, sort_number=n)).qux_id for n in (1, 2, 3)]

    # when — sort_number > 1 인 것만 +1
    qux_repository.increment_sort_numbers_above(article_id, 1, 1)

    # then
    found = [qux_repository.find_write_by_id(QuxIdentity(i)) for i in ids]
    assert [q.sort_number for q in found] == [1, 3, 4]
```

`update()` 문 후 같은 세션에서 `find_*` 로 재조회할 때 identity map 의 stale 객체가 돌아오면, EntityQuery 의 `execute()` 에 `execution_options={"synchronize_session": "fetch"}` 를 추가한다 (query-references Bar 참조).

---

## E. 동적 조건 (검색 필터·정렬) (EntityQuery)

조건부 `.where()` 분기별 테스트.

```python
def test_should_return_all_when_no_filters(repository, test_user_id):
    _save_bazs(repository, test_user_id, 3)

    result = repository.find_all("recent", "", [], 0, 20)

    assert len(result) == 3


def test_should_filter_by_category_when_category_provided(repository, test_user_id):
    _save_baz(repository, test_user_id, categories=(BazCategory.VIBE_CODING,))
    _save_baz(repository, test_user_id, categories=(BazCategory.DEV_EFFICIENCY,))

    result = repository.find_all("recent", "", [BazCategory.VIBE_CODING], 0, 20)

    assert len(result) == 1
    assert BazCategory.VIBE_CODING in result[0].categories


def test_should_filter_by_keyword_when_keyword_provided(repository, test_user_id):
    _save_baz(repository, test_user_id, title="FastAPI 가이드")
    _save_baz(repository, test_user_id, title="React 입문")

    result = repository.find_all("recent", "FastAPI", [], 0, 20)

    assert len(result) == 1


def test_should_count_filtered_when_category_provided(repository, test_user_id):
    _save_baz(repository, test_user_id, categories=(BazCategory.VIBE_CODING,))
    _save_baz(repository, test_user_id, categories=(BazCategory.VIBE_CODING,))
    _save_baz(repository, test_user_id, categories=(BazCategory.DEV_EFFICIENCY,))

    count = repository.count_all("", [BazCategory.VIBE_CODING])

    assert count == 2
```

---

## F. 페이징

```python
def test_should_return_paged_results_when_page_and_size_provided(repository, test_user_id):
    _save_bazs(repository, test_user_id, 5)

    page1 = repository.find_by_user_id(test_user_id, 0, 3)
    page2 = repository.find_by_user_id(test_user_id, 1, 3)

    assert len(page1) == 3
    assert len(page2) == 2
```

---

## G. 집계 함수

```python
def test_should_return_max_order_when_quxs_exist(qux_repository):
    for order in (3, 7, 5):
        qux_repository.save(_qux(qux_order=order))

    assert qux_repository.get_max_comment_order(article_id) == 7


def test_should_return_zero_when_no_quxs(qux_repository):
    assert qux_repository.get_max_comment_order(999) == 0
```

---

## H. 단건 조회 (존재 / 미존재)

```python
def test_should_return_present_when_existing_id(rating_repository):
    saved = rating_repository.save(sample_rating)

    found = rating_repository.find_by_setup_id_and_user_id(saved.setup_id, saved.user_id)

    assert found is not None


def test_should_return_none_when_non_existing_id(rating_repository):
    found = rating_repository.find_by_setup_id_and_user_id(999, 999)
    assert found is None
```

---

## I. 삭제 (soft delete)

모든 Entity 가 `is_deleted`/`deleted_at` 필드를 가지므로 정석 soft-delete 를 쓰는 도메인에서는 아래 패턴을 따른다 (hide/show 류·upsert-only 류는 해당 없음).

```python
def test_should_not_find_by_id_when_deleted(repository, test_user_id):
    saved = repository.save(_baz(test_user_id))
    identity = BazIdentity(saved.baz_id)

    repository.delete_by_id(identity)

    assert repository.find_write_by_id(identity) is None
    assert repository.find_by_id(identity) is None


def test_should_be_noop_when_deleting_twice(repository, test_user_id):
    saved = repository.save(_baz(test_user_id))
    identity = BazIdentity(saved.baz_id)

    repository.delete_by_id(identity)
    repository.delete_by_id(identity)   # 예외 없어야 함

    assert repository.find_write_by_id(identity) is None
```

EntityQuery 의 `find_*_and_active` 가 삭제 행을 제외하는지도 1개:

```python
def test_should_exclude_deleted_when_find_active(session, entities, repository, test_user_id):
    saved = repository.save(_baz(test_user_id))
    repository.delete_by_id(BazIdentity(saved.baz_id))

    with session.begin():
        found = entities.find_by_id_and_active(saved.baz_id)
        raw = entities.get(saved.baz_id)

    assert found is None
    assert raw is not None and raw.is_deleted is True
```

---

## J. 숨김 처리 (Qux 전용)

```python
def test_should_mask_text_when_qux_is_hidden(qux_repository):
    saved = qux_repository.save(_qux(text="원본 텍스트"))
    qux_repository.save(saved.hide())

    found = qux_repository.find_by_id(QuxIdentity(saved.qux_id))

    assert found is not None
    assert found.is_hidden is True
    assert found.text != "원본 텍스트"   # 구현체 _to_read 가 마스킹하는 경우
```

---

## K. 감사 필드 자동 채움

`default=utcnow` / `onupdate=utcnow` 가 도는지 한 번 확인한다.

```python
def test_should_set_updated_at_on_update(repository, test_user_id):
    saved = repository.save(_baz(test_user_id))
    before = saved.updated_at

    updated = repository.save(saved.with_update("수정", "본문", (), (), BazSource.SELF))

    assert updated.updated_at >= before
    assert updated.created_at == saved.created_at
```
