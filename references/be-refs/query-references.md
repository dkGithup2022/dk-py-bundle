# Query References

`be-add-query` 스킬이 참고하는 Default{Domain}Repository / EntityQuery / JoinQuery 작성 레퍼런스.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 패키지)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

---

## 공통 요소

### 클래스 3종과 네이밍

| 역할 | 이름 | 위치 | 다루는 것 |
|---|---|---|---|
| infra port | `{Domain}Repository` | `infrastructure/.../{domain}/{domain}_repository.py` | ABC |
| infra 구현체 | `Default{Domain}Repository` | `persistence/.../{domain}/{domain}_repository.py` | 도메인 모델 in/out. port 상속 |
| Entity 쿼리 | `{Domain}EntityQuery` | `persistence/.../{domain}/{domain}_entity_query.py` | Entity 자체 (같은 aggregate). SQLAlchemy `select` |
| JOIN 쿼리 | `{Domain}JoinQuery` | `persistence/.../{domain}/{domain}_join_query.py` | 다른 aggregate 와의 JOIN. **SQL 직접** (`text()`), 행 dataclass 반환 |

- 구현체는 port ABC 를 상속하고, 생성자로 `tx: TransactionManager`, `entities: {Domain}EntityQuery`, (있으면) `joins: {Domain}JoinQuery` 를 받는다
- 쿼리 클래스 둘은 `session: Session` 하나만 받는다. 도메인 모델을 모른다
- **경계 규칙 하나**: 같은 aggregate 안이면 EntityQuery, aggregate 를 넘으면 JoinQuery

### 주요 import

```python
from sqlalchemy import func, select, text, update
from sqlalchemy.orm import Session

from {pkg}.transaction import TransactionManager, transactional
```

### 트랜잭션

- **구현체의 모든 public 메서드에 `@transactional`** (be-init C-9). 바깥 트랜잭션이 있으면 합류, 없으면 자기가 열고 닫는다
- 쿼리 클래스에는 붙이지 않는다. 구현체 안에서 실행된다
- **`commit()` 호출 금지.** flush 까지만 (EntityQuery 의 `add` / `flush`)
- `autobegin=False` 세션이라 `@transactional` 없이 쿼리를 실행하면 `InvalidRequestError: Autobegin is disabled` 로 즉시 실패한다 — 의도된 안전장치

### soft-delete 필터링

- EntityQuery: `select(...).where(Entity.is_deleted.is_(False))` — `== False` 대신 `.is_(False)` (ruff E712)
- JoinQuery SQL: `WHERE ... AND t.is_deleted = false`
- 메서드 이름에 `_and_active` 를 붙여 필터가 있음을 드러낸다. 삭제 행까지 보는 것은 `get` (PK) 뿐

### delete_by_id 패턴

```python
@transactional
def delete_by_id(self, identity: FooIdentity) -> None:
    entity = self._entities.get(identity.foo_id)
    if entity is None or entity.is_deleted:
        return
    entity.soft_delete()
    self._entities.flush()
```

- 반환 타입 `None`
- 이미 삭제된 row 는 조용히 no-op
- 404 의미는 service 레이어에서 `find_write_by_id() → None → raise NotFoundException` 선검증으로 확보

### insert / update 구분

- **insert**: `entity = self._entities.add(FooEntity.from_domain(domain))` — `add` 가 flush 까지 해서 `id` 가 채워진다
- **update**: `self._entities.get(id)` 로 기존 Entity 를 가져와 **필드를 제자리 갱신**, `flush`. `from_domain` 으로 새 Entity 를 만들어 `merge` 하지 않는다 (자식 컬렉션 orphan 처리가 꼬이고 `updated_at` 이 도메인 값으로 덮인다)
- `save()` 하나로 둘 다 처리: `domain.foo_id is None` 이면 insert, 아니면 update

### 행 dataclass 배치

JoinQuery 의 결과를 담는 dataclass 는 `{Domain}With{What}Row` 네이밍, **JoinQuery 와 같은 파일**에 둔다. `dto/` 서브디렉토리 만들지 않음.

---

## Pattern Foo — 단순 CRUD (EntityQuery 위임)

**언제 쓰나**: 단건 조회·저장·존재 확인만 필요, JOIN/집계 없음. JoinQuery 파일 없음.

### infra port

```python
class FooRepository(ABC):
    def find_by_id(self, identity: FooIdentity) -> Foo | None: ...
    def find_by_email(self, email: str) -> Foo | None: ...
    def exists_by_nickname(self, nickname: str) -> bool: ...
    def save(self, foo: Foo) -> Foo: ...
```

### EntityQuery

```python
from sqlalchemy import exists, select
from sqlalchemy.orm import Session

from .foo_entity import FooEntity


class FooEntityQuery:
    def __init__(self, session: Session) -> None:
        self._session = session

    def find_by_id_and_active(self, id: int) -> FooEntity | None:
        stmt = select(FooEntity).where(FooEntity.id == id, FooEntity.is_deleted.is_(False))
        return self._session.scalar(stmt)

    def find_by_email_and_active(self, email: str) -> FooEntity | None:
        stmt = select(FooEntity).where(FooEntity.email == email, FooEntity.is_deleted.is_(False))
        return self._session.scalar(stmt)

    def exists_by_nickname_and_active(self, nickname: str) -> bool:
        stmt = select(exists().where(FooEntity.nickname == nickname, FooEntity.is_deleted.is_(False)))
        return bool(self._session.scalar(stmt))

    def get(self, id: int) -> FooEntity | None:
        return self._session.get(FooEntity, id)

    def add(self, entity: FooEntity) -> FooEntity:
        self._session.add(entity)
        self._session.flush()
        return entity

    def flush(self) -> None:
        self._session.flush()
```

### 구현체

```python
from {pkg}.exception.foo.foo_not_found_exception import FooNotFoundException
from {pkg}.infrastructure.foo.foo_repository import FooRepository
from {pkg}.transaction import TransactionManager, transactional
from {pkg}.model.foo.foo import Foo
from {pkg}.model.foo.foo_identity import FooIdentity

from .foo_entity import FooEntity
from .foo_entity_query import FooEntityQuery


class DefaultFooRepository(FooRepository):
    def __init__(self, tx: TransactionManager, entities: FooEntityQuery) -> None:
        self.tx = tx
        self._entities = entities

    @transactional
    def find_by_id(self, identity: FooIdentity) -> Foo | None:
        entity = self._entities.find_by_id_and_active(identity.foo_id)
        return entity.to_domain() if entity is not None else None

    @transactional
    def find_by_email(self, email: str) -> Foo | None:
        entity = self._entities.find_by_email_and_active(email)
        return entity.to_domain() if entity is not None else None

    @transactional
    def exists_by_nickname(self, nickname: str) -> bool:
        return self._entities.exists_by_nickname_and_active(nickname)

    @transactional
    def save(self, foo: Foo) -> Foo:
        if foo.foo_id is None:
            entity = self._entities.add(FooEntity.from_domain(foo))
        else:
            found = self._entities.get(foo.foo_id)
            if found is None:
                raise FooNotFoundException(f"foo not found: {foo.foo_id}")
            entity = found
            entity.nickname = foo.nickname
            entity.email = foo.email
            entity.bio = foo.bio
            entity.updated_at = foo.updated_at
            self._entities.flush()
        return entity.to_domain()
```

**특징**:
- Java 의 derived query 이름(`findByIdAndIsDeletedFalse`)이 EntityQuery 의 `find_by_id_and_active` 로 온다. 이름이 곧 쿼리 의도
- 구현체는 EntityQuery 를 부르고 `to_domain()` 만 한다

---

## Pattern Bar — Stats 집계 UPDATE (update 문)

**언제 쓰나**: 통계 테이블의 카운터 증감. 전체 엔티티 save 대신 부분 UPDATE 로 쓰기 경쟁 최소화. 별도 port 이므로 EntityQuery 와 구현체도 별도 파일.

### infra port

```python
class FooStatsRepository(ABC):
    def find_by_foo_id(self, identity: FooIdentity) -> FooStats | None: ...
    def save(self, stats: FooStats) -> FooStats: ...
    def increase_view_count(self, identity: FooIdentity, increment: int) -> None: ...
    def add_rating(self, identity: FooIdentity, score: int) -> FooStats: ...
```

### EntityQuery (`foo_stats_entity_query.py`)

```python
class FooStatsEntityQuery:
    def __init__(self, session: Session) -> None:
        self._session = session

    def find_by_foo_id(self, foo_id: int) -> FooStatsEntity | None:
        stmt = select(FooStatsEntity).where(FooStatsEntity.foo_id == foo_id)
        return self._session.scalar(stmt)

    def add(self, entity: FooStatsEntity) -> FooStatsEntity:
        self._session.add(entity)
        self._session.flush()
        return entity

    def increase_view_count(self, foo_id: int, increment: int) -> None:
        stmt = (
            update(FooStatsEntity)
            .where(FooStatsEntity.foo_id == foo_id)
            .values(view_count=FooStatsEntity.view_count + increment)
        )
        self._session.execute(stmt)

    def add_rating(self, foo_id: int, score: int) -> None:
        stmt = (
            update(FooStatsEntity)
            .where(FooStatsEntity.foo_id == foo_id)
            .values(
                rating_sum=FooStatsEntity.rating_sum + score,
                rating_count=FooStatsEntity.rating_count + 1,
            )
        )
        self._session.execute(stmt, execution_options={"synchronize_session": "fetch"})
```

### 구현체 (`foo_stats_repository.py`)

```python
class DefaultFooStatsRepository(FooStatsRepository):
    def __init__(self, tx: TransactionManager, entities: FooStatsEntityQuery) -> None:
        self.tx = tx
        self._entities = entities

    @transactional
    def find_by_foo_id(self, identity: FooIdentity) -> FooStats | None:
        entity = self._entities.find_by_foo_id(identity.foo_id)
        return entity.to_domain() if entity is not None else None

    @transactional
    def save(self, stats: FooStats) -> FooStats:
        return self._entities.add(FooStatsEntity.from_domain(stats)).to_domain()

    @transactional
    def increase_view_count(self, identity: FooIdentity, increment: int) -> None:
        self._entities.increase_view_count(identity.foo_id, increment)

    @transactional
    def add_rating(self, identity: FooIdentity, score: int) -> FooStats:
        self._entities.add_rating(identity.foo_id, score)
        found = self.find_by_foo_id(identity)   # 같은 트랜잭션에 합류
        if found is None:
            raise FooNotFoundException(f"foo stats not found: {identity.foo_id}")
        return found
```

**특징**:
- `update().values(col=col + n)` 이 SQL 레벨 원자 증감 (`SET view_count = view_count + :inc`)
- 변경 후 최신 상태가 필요하면 구현체 안에서 추가 조회 — `@transactional` 이 합류하므로 한 트랜잭션
- `update()` 문은 세션의 Entity 객체를 갱신하지 않는다. 같은 트랜잭션에서 그 Entity 를 다시 읽을 때 stale 하지 않게 `execution_options={"synchronize_session": "fetch"}` 를 준다
- Stats 테이블은 audit 4필드 미적용이라 `updated_at` 신경 안 씀. 감사 컬럼이 있는 테이블에 `update()` 를 쓰면 `.values(..., updated_at=utcnow())` 필수

---

## Pattern Baz — JOIN + 행 dataclass (JoinQuery)

**언제 쓰나**: 조회 시 다른 테이블의 필드(작성자명·집계)를 함께 가져와 Read 모델을 합성. 자식 컬렉션은 relationship 이 EntityQuery 쪽에서 로드한다.

### infra port

```python
class BazRepository(ABC):
    def find_by_id(self, identity: BazIdentity) -> BazRead | None: ...
    def find_by_user_id(self, user_id: int, page: int, size: int) -> list[BazRead]: ...
    def count_by_user_id(self, user_id: int) -> int: ...
    def find_write_by_id(self, identity: BazIdentity) -> Baz | None: ...
    def save(self, baz: Baz) -> Baz: ...
    def delete_by_id(self, identity: BazIdentity) -> None: ...
```

### JoinQuery (`baz_join_query.py`)

```python
from dataclasses import dataclass
from datetime import datetime

from sqlalchemy import text
from sqlalchemy.orm import Session


@dataclass(frozen=True, slots=True)
class BazWithDetailsRow:
    id: int
    user_id: int
    title: str
    body: str
    created_at: datetime
    updated_at: datetime
    # JOIN 필드
    author_nickname: str | None
    view_count: int
    rating_sum: int
    rating_count: int


class BazJoinQuery:
    _SELECT = """
        SELECT b.id, b.user_id, b.title, b.body, b.created_at, b.updated_at,
               u.nickname AS author_nickname,
               COALESCE(s.view_count, 0)   AS view_count,
               COALESCE(s.rating_sum, 0)   AS rating_sum,
               COALESCE(s.rating_count, 0) AS rating_count
        FROM bazs b
        LEFT JOIN users u     ON u.id = b.user_id
        LEFT JOIN baz_stats s ON s.baz_id = b.id
    """
    _BY_ID = text(_SELECT + " WHERE b.id = :id AND b.is_deleted = false")
    _BY_USER_ID = text(
        _SELECT
        + " WHERE b.user_id = :user_id AND b.is_deleted = false"
        + " ORDER BY b.created_at DESC LIMIT :limit OFFSET :offset"
    )

    def __init__(self, session: Session) -> None:
        self._session = session

    def find_by_id_with_details(self, id: int) -> BazWithDetailsRow | None:
        row = self._session.execute(self._BY_ID, {"id": id}).mappings().one_or_none()
        return BazWithDetailsRow(**row) if row is not None else None

    def find_by_user_id_with_details(self, user_id: int, limit: int, offset: int) -> list[BazWithDetailsRow]:
        rows = self._session.execute(
            self._BY_USER_ID, {"user_id": user_id, "limit": limit, "offset": offset}
        ).mappings().all()
        return [BazWithDetailsRow(**r) for r in rows]
```

### EntityQuery (`baz_entity_query.py`) — 자식 컬렉션 포함

```python
class BazEntityQuery:
    def __init__(self, session: Session) -> None:
        self._session = session

    def find_by_id_and_active(self, id: int) -> BazEntity | None:
        stmt = select(BazEntity).where(BazEntity.id == id, BazEntity.is_deleted.is_(False))
        return self._session.scalar(stmt)          # categories/tags 는 selectin 으로 함께 로드

    def find_by_id_in(self, ids: list[int]) -> list[BazEntity]:
        if not ids:
            return []
        stmt = select(BazEntity).where(BazEntity.id.in_(ids))
        return list(self._session.scalars(stmt).all())

    def count_by_user_id_and_active(self, user_id: int) -> int:
        stmt = select(func.count()).select_from(BazEntity).where(
            BazEntity.user_id == user_id, BazEntity.is_deleted.is_(False)
        )
        return int(self._session.scalar(stmt) or 0)

    def get(self, id: int) -> BazEntity | None: ...
    def add(self, entity: BazEntity) -> BazEntity: ...
    def flush(self) -> None: ...
```

### 구현체

```python
class DefaultBazRepository(BazRepository):
    def __init__(self, tx: TransactionManager, entities: BazEntityQuery, joins: BazJoinQuery) -> None:
        self.tx = tx
        self._entities = entities
        self._joins = joins

    @transactional
    def find_by_id(self, identity: BazIdentity) -> BazRead | None:
        row = self._joins.find_by_id_with_details(identity.baz_id)
        if row is None:
            return None
        entity = self._entities.find_by_id_and_active(row.id)   # 자식 컬렉션용
        return self._to_read(row, entity) if entity is not None else None

    @transactional
    def find_by_user_id(self, user_id: int, page: int, size: int) -> list[BazRead]:
        rows = self._joins.find_by_user_id_with_details(user_id, size, page * size)
        if not rows:
            return []
        # N+1 회피: 자식 컬렉션은 ID 배치 1회로
        entities = {e.id: e for e in self._entities.find_by_id_in([r.id for r in rows])}
        return [self._to_read(row, entities[row.id]) for row in rows]

    @transactional
    def count_by_user_id(self, user_id: int) -> int:
        return self._entities.count_by_user_id_and_active(user_id)

    @transactional
    def find_write_by_id(self, identity: BazIdentity) -> Baz | None:
        entity = self._entities.find_by_id_and_active(identity.baz_id)
        return entity.to_domain() if entity is not None else None

    @transactional
    def save(self, baz: Baz) -> Baz:
        if baz.baz_id is None:
            entity = self._entities.add(BazEntity.from_domain(baz))
        else:
            found = self._entities.get(baz.baz_id)
            if found is None:
                raise BazNotFoundException(f"baz not found: {baz.baz_id}")
            entity = found
            entity.title = baz.title
            entity.body = baz.body
            entity.categories = [BazCategoryEntity(category=c) for c in baz.categories]  # 통째 교체 → orphan delete
            entity.tags = [BazTagEntity(tag=t) for t in baz.tags]
            entity.updated_at = baz.updated_at
            self._entities.flush()
        return entity.to_domain()

    @transactional
    def delete_by_id(self, identity: BazIdentity) -> None:
        entity = self._entities.get(identity.baz_id)
        if entity is None or entity.is_deleted:
            return
        entity.soft_delete()
        self._entities.flush()

    def _to_read(self, row: BazWithDetailsRow, entity: BazEntity) -> BazRead:
        stats = BazStats(row.id, row.view_count, row.rating_sum, row.rating_count, 0)
        return entity.to_domain().to_read(
            author_nickname=row.author_nickname or "", stats=stats, links=()
        )
```

**특징**:
- JOIN 은 SQL 로 한눈에 보인다. 공통 SELECT 는 문자열 상수로 두고 WHERE 만 다른 SQL 을 만든다 (Java "동일 DTO 재사용")
- `LEFT JOIN` + `COALESCE` 로 NULL 처리. `LIMIT :limit OFFSET :offset` 은 바인드 파라미터
- 자식 컬렉션이 필요한 Read 는 JoinQuery 행 + EntityQuery Entity 두 단계. 목록은 `find_by_id_in` 배치 1회 (N+1 없음)
- `BazRead.rating` 처럼 DB 에 없는 파생값은 도메인 `to_read()` 가 계산
- **JoinQuery 의 두 메서드 모두 be-test-repository 테스트 필수**

---

## Pattern Qux — 트리·순서 연산 (update 문)

**언제 쓰나**: 댓글·메뉴·카테고리 같은 트리 구조에서 sort_number·child_count 재배치. 같은 테이블 안이므로 EntityQuery.

### infra port

```python
class QuxRepository(ABC):
    def increment_sort_numbers_above(self, article_id: int, comment_order: int, sort_number: int) -> None: ...
    def increment_child_count(self, parent_id: QuxIdentity) -> None: ...
    def get_max_comment_order(self, article_id: int) -> int: ...
```

### EntityQuery

```python
class QuxEntityQuery:
    def __init__(self, session: Session) -> None:
        self._session = session

    def increment_sort_numbers_above(self, article_id: int, comment_order: int, sort_number: int) -> None:
        stmt = (
            update(QuxEntity)
            .where(
                QuxEntity.article_id == article_id,
                QuxEntity.qux_order == comment_order,
                QuxEntity.sort_number > sort_number,
                QuxEntity.is_deleted.is_(False),
            )
            .values(sort_number=QuxEntity.sort_number + 1, updated_at=utcnow())
        )
        self._session.execute(stmt)

    def increment_child_count(self, id: int) -> None:
        stmt = (
            update(QuxEntity)
            .where(QuxEntity.id == id)
            .values(child_count=QuxEntity.child_count + 1, updated_at=utcnow())
        )
        self._session.execute(stmt)

    def find_max_comment_order(self, article_id: int) -> int:
        stmt = select(func.coalesce(func.max(QuxEntity.qux_order), 0)).where(
            QuxEntity.article_id == article_id, QuxEntity.is_deleted.is_(False)
        )
        return int(self._session.scalar(stmt) or 0)
```

### 구현체

```python
class DefaultQuxRepository(QuxRepository):
    def __init__(self, tx: TransactionManager, entities: QuxEntityQuery) -> None:
        self.tx = tx
        self._entities = entities

    @transactional
    def increment_sort_numbers_above(self, article_id: int, comment_order: int, sort_number: int) -> None:
        self._entities.increment_sort_numbers_above(article_id, comment_order, sort_number)

    @transactional
    def increment_child_count(self, parent_id: QuxIdentity) -> None:
        self._entities.increment_child_count(parent_id.qux_id)

    @transactional
    def get_max_comment_order(self, article_id: int) -> int:
        return self._entities.find_max_comment_order(article_id)
```

**특징**:
- 트리 재배치는 여러 row 에 걸친 UPDATE — `update()` 문
- 감사 컬럼이 있는 테이블이므로 `updated_at=utcnow()` 를 `.values()` 에 명시
- 순서 연산은 **service 레이어의 `@transactional`** 안에서 보호 (동시성). 구현체의 `@transactional` 은 합류만 한다

---

## 동적 조건 (검색 필터·정렬 분기)

WHERE 조건이 입력에 따라 있거나 없는 경우.

**EntityQuery** 에서는 `if` 로 `.where()` 를 조건부로 붙이면 된다. Java 에서 JdbcTemplate 도입 여부를 묻던 문제는 없다.

```python
def find_all_and_active(self, sort: str, keyword: str, categories: list[BazCategory], page: int, size: int) -> list[BazEntity]:
    stmt = select(BazEntity).where(BazEntity.is_deleted.is_(False))
    if keyword:
        stmt = stmt.where(BazEntity.title.ilike(f"%{keyword}%"))
    if categories:
        stmt = stmt.where(
            BazEntity.id.in_(select(BazCategoryEntity.baz_id).where(BazCategoryEntity.category.in_(categories)))
        )
    order = BazEntity.created_at.desc() if sort == "recent" else BazEntity.id.desc()
    return list(self._session.scalars(stmt.order_by(order).limit(size).offset(page * size)).all())
```

**JoinQuery** 에서는 SQL 문자열을 조립하지 않는다. JOIN 이 필요한데 조건이 동적이면 그 메서드만 `select()` 빌더로 쓰는 것을 예외로 허용하고, docstring 에 "동적 조건이라 빌더 사용" 을 적는다.

**유저에게 문의할 것**:
- `ilike` 검색어 필터가 성능 문제가 될 규모이거나 전문 검색이 필요할 때 — 검색 엔진 이관 후보
- Postgres 특화 문법(윈도우 함수, 재귀 CTE)이 필요할 때

---

## 패턴 선택 가이드

| 도메인 특성 | Pattern |
|---|---|
| 단순 CRUD (EntityQuery 로 충분) | **Foo** |
| 통계 카운터 증감 UPDATE | **Bar** |
| 조회 시 JOIN/집계 필요 | **Baz** |
| 트리·순서 재배치 UPDATE | **Qux** |

대부분 도메인은 Foo. 집계가 있으면 Bar 를 추가. 상세 조회에 다른 aggregate 가 붙으면 Baz. 트리는 Qux.

---

## 💡 팁 — 위 4개 패턴으로 풀리지 않을 때

1. **Baz + 보조 쿼리 조합을 먼저 재검토** — 대부분 JoinQuery SQL 하나 + EntityQuery 배치 하나 + Python 레벨 `dict` 합성으로 표현 가능.
2. **Python 레이어에서 여러 쿼리 조합** — 구현체 메서드 하나 안에서 EntityQuery / JoinQuery 를 순서대로 부르고 결과를 합성. `@transactional` 이 하나로 묶어준다.
3. **유저에게 질의** — 위 방식으로도 표현이 어색하면 중단하고 유저에게 물어본다. 검색 엔진 이관 / 스키마 재설계 중 어느 방향을 택할지 결정이 필요하다.

---

## 파일 구성 요약

```
persistence/src/{pkg}/persistence/{domain}/
├── __init__.py
├── {domain}_entity.py               ← be-add-entity-specs 가 생성
├── {child}_entity.py
├── {domain}_entity_query.py         ← {Domain}EntityQuery
├── {domain}_join_query.py           ← {Domain}JoinQuery + 행 dataclass (Baz 필요 시)
├── {domain}_repository.py           ← Default{Domain}Repository
├── {domain}_stats_entity_query.py   ← Stats (Bar 필요 시)
└── {domain}_stats_repository.py
```

---

## 공통 금지 사항

- 구현체 public 메서드에 `@transactional` 누락 금지 — `autobegin=False` 라 실행 시 실패한다
- 쿼리 클래스에 `@transactional` 붙이기 금지 — 경계는 구현체
- `commit()` / `rollback()` 호출 금지 — 경계는 `@transactional` 과 service
- `session.query(...)` (1.x 스타일) 금지 — `select()` 만
- `== True` / `== False` 금지 — `.is_(True)` / `.is_(False)`
- EntityQuery 에 `text()` 금지, JoinQuery 에 `select()` 금지 (동적 조건 예외 제외)
- JoinQuery SQL 문자열 조립 금지 — 상수 `text()` + 바인드 파라미터
- 쿼리 클래스가 도메인 모델을 import 하는 것 금지 — Entity 와 행 dataclass 만
- 구현체에서 Entity 를 도메인 외부로 누출 금지 — 반환 타입은 항상 도메인 모델
- `merge()` 로 수정 금지 — 조회 후 제자리 갱신
- Entity 에 없는 relationship 을 `joinedload` 로 우회 금지 — cross-aggregate 는 JoinQuery SQL
