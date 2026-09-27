---
name: be-add-query
description: infra 인터페이스의 구현체(Default{Domain}Repository)와 쿼리(EntityQuery / JoinQuery)를 생성/변경한다.
---

# add-query

infra 인터페이스의 구현체와 쿼리를 생성/변경한다.

## 입력

### 지시
- 기본적으로 모든 infra 인터페이스를 구현한다.
- 개별 인터페이스 구현 요청이 오는 경우에도 아래의 공통 규칙을 따르시오.

### Phase 0. 프로젝트 컨텍스트 해석
`{pkg}`, `{packages.persistence}` 플레이스홀더 사용. 실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.

### 참고 문서 (읽어올 것)
- ${CLAUDE_PLUGIN_ROOT}/references/be-refs/query-references.md — 구현 스타일 레퍼런스 (이 스타일을 따를 것)
- ${CLAUDE_PLUGIN_ROOT}/references/be-refs/entity-references.md — Entity 변환 패턴 (to_domain/from_domain)
- specs/repository.md — 기존 쿼리 정보 (프로젝트 루트 기준)
- specs/infrastructure.md — 구현 대상 인터페이스 (프로젝트 루트 기준)
- specs/models.md — 도메인 모델 (반환 타입, VO 구조 참조) (프로젝트 루트 기준)

## 생성 결과물 구조

```
persistence/src/{pkg}/persistence/{domain}/
├── {domain}_entity_query.py       ← {Domain}EntityQuery (Entity 자체에 대한 쿼리)
├── {domain}_join_query.py         ← {Domain}JoinQuery (JOIN 쿼리, 필요 시) + 행 dataclass
└── {domain}_repository.py         ← Default{Domain}Repository (infra 구현체)
```

## 기본 원칙

1. EntityQuery 에서 **SQLAlchemy select** 를 이용해 Entity 에 대한 쿼리를 작성한다. JoinQuery 에서 **SQL 을 직접** 작성한다.
2. Default{Domain}Repository 에서 이를 **wrapping 하거나 간단한 조작**을 통해 외부로 제공하는 것이 목표이다.
3. 가능하면 **최소 조건, 최대한 간단하게** 각 단계의 쿼리를 작성 & 구현하시오.
4. 직접 SQL(JoinQuery)을 작성하기보다 **EntityQuery 를 우선** 사용하시오. JoinQuery 는 다른 aggregate 와의 JOIN 이 있을 때만.

## 생성 규칙

### 1. EntityQuery

```python
class {Domain}EntityQuery:
    def __init__(self, session: Session) -> None:
        self._session = session
```

Entity 클래스 자체를 접근하는 쿼리 구현체. 도메인 모델을 모른다.
Java 의 derived query 에 해당하는 것을 메서드 이름으로 직접 짓는다.

**A. 조회 쿼리 (기본)**

단일 테이블 조회의 기본 수단.

**조회 쿼리에는 반드시 `is_deleted.is_(False)` 조건을 포함한다.** soft delete된 데이터가 조회되지 않도록 한다.

```python
def find_by_user_id_and_active(self, user_id: int) -> list[{Domain}Entity]:
    stmt = select({Domain}Entity).where(
        {Domain}Entity.user_id == user_id, {Domain}Entity.is_deleted.is_(False)
    )
    return list(self._session.scalars(stmt).all())

def find_by_id_and_active(self, id: int) -> {Domain}Entity | None:
    stmt = select({Domain}Entity).where(
        {Domain}Entity.id == id, {Domain}Entity.is_deleted.is_(False)
    )
    return self._session.scalar(stmt)

def count_by_user_id_and_active(self, user_id: int) -> int:
    stmt = select(func.count()).select_from({Domain}Entity).where(
        {Domain}Entity.user_id == user_id, {Domain}Entity.is_deleted.is_(False)
    )
    return int(self._session.scalar(stmt) or 0)
```

**B. 원자적 UPDATE (update 문)**

카운터 증감 등 특정 컬럼만 변경하는 경우.

```python
def increase_{column}(self, id: int, increment: int) -> None:
    stmt = (
        update({Domain}Entity)
        .where({Domain}Entity.id == id)
        .values({column}={Domain}Entity.{column} + increment)
    )
    self._session.execute(stmt)
```

**C. Bulk 조회**

```python
def find_by_id_in(self, ids: list[int]) -> list[{Domain}Entity]:
    stmt = select({Domain}Entity).where({Domain}Entity.id.in_(ids))
    return list(self._session.scalars(stmt).all())
```

**D. 저장**

```python
def get(self, id: int) -> {Domain}Entity | None:
    return self._session.get({Domain}Entity, id)

def add(self, entity: {Domain}Entity) -> {Domain}Entity:
    self._session.add(entity)
    self._session.flush()
    return entity

def flush(self) -> None:
    self._session.flush()
```

### 2. Default{Domain}Repository

```python
class Default{Domain}Repository({Domain}Repository):
    def __init__(self, tx: TransactionManager, entities: {Domain}EntityQuery) -> None:
        self.tx = tx
        self._entities = entities
```

infra 인터페이스를 상속하고, EntityQuery 에 위임한다.
비즈니스 로직을 넣지 않는다. **변환과 위임만.**
모든 public 메서드에 `@transactional` (be-init C-9). `commit()` 은 부르지 않는다.

**A. Write 메서드 (Domain 반환)**

```python
@transactional
def save(self, domain: {Domain}) -> {Domain}:
    if domain.{domain}_id is None:
        entity = self._entities.add({Domain}Entity.from_domain(domain))
    else:
        entity = self._entities.get(domain.{domain}_id)
        if entity is None:
            raise {Domain}NotFoundException(f"{domain} not found: {domain.{domain}_id}")
        entity.title = domain.title          # 변경 가능한 필드를 제자리 갱신
        entity.updated_at = domain.updated_at
        self._entities.flush()
    return entity.to_domain()
```

**B. Read 메서드 — 단일 Entity (Domain 반환)**

soft delete 필터가 적용된 EntityQuery 메서드를 사용한다.

```python
@transactional
def find_by_id(self, identity: {Domain}Identity) -> {Domain} | None:
    entity = self._entities.find_by_id_and_active(identity.{domain}_id)
    return entity.to_domain() if entity is not None else None
```

**C. 삭제 메서드 (Soft Delete)**

delete_by_id 는 실제 DELETE 가 아닌 soft delete 로 구현한다.
Entity 의 soft_delete() 메서드로 is_deleted/deleted_at 을 설정하고 flush 한다.

```python
@transactional
def delete_by_id(self, identity: {Domain}Identity) -> None:
    entity = self._entities.get(identity.{domain}_id)
    if entity is None or entity.is_deleted:
        return
    entity.soft_delete()
    self._entities.flush()
```

**D. 원자적 연산**

Identity VO → int unwrap 후 EntityQuery 에 직접 위임.

```python
@transactional
def increase_{column}(self, identity: {Domain}Identity, increment: int) -> None:
    self._entities.increase_{column}(identity.{domain}_id, increment)
```

### 3. JOIN 쿼리 (여러 도메인에 걸친 조회)

Read 모델에 다른 테이블 필드(nickname 등)가 포함된 경우.
JOIN 관계를 나타내는 쿼리 구현체 `{Domain}JoinQuery` 를 별도로 두고, Default{Domain}Repository 가 사용한다.

**유저에게 아래 계획을 허락받을 것:**
1. 응답 행 dataclass (`{Domain}WithUserRow`)를 만든다
2. JoinQuery 에 LEFT JOIN SQL 메서드를 추가한다
3. Default{Domain}Repository 에서 행 → Read 변환 메서드를 추가한다

**3-1. WithUserRow**

```python
@dataclass(frozen=True, slots=True)
class {Domain}WithUserRow:
    # Entity 컬럼 전부 (VO 는 개별 필드로 분해)
    id: int
    user_id: int
    # ...

    # JOIN 필드
    nickname: str | None
```

**3-2. SQL 규칙**

- SELECT 컬럼을 **명시**한다 (SELECT * 금지)
- **LEFT JOIN** 사용 (nullable 허용)
- WHERE 만 다른 쿼리는 **동일 행 dataclass 재사용**
- SQL 은 클래스 상수 `text()` 로 고정. 문자열 이어붙이기 금지
- 바인드 파라미터는 `:name`
- **JoinQuery 의 모든 메서드는 be-test-repository 의 테스트가 있어야 완료다.** SQL 문자열은 mypy 가 못 잡는다

```python
class {Domain}JoinQuery:
    # 단건 조회
    _BY_ID_WITH_USER = text("""
        SELECT t.id, t.user_id, ..., u.nickname
        FROM {table} t
        LEFT JOIN users u ON t.user_id = u.id
        WHERE t.id = :id AND t.is_deleted = false
    """)

    # 목록 조회 (고정 WHERE + 페이징)
    _BY_USER_ID_WITH_USER = text("""
        SELECT t.id, t.user_id, ..., u.nickname
        FROM {table} t
        LEFT JOIN users u ON t.user_id = u.id
        WHERE t.user_id = :user_id AND t.is_deleted = false
        ORDER BY t.created_at DESC
        LIMIT :limit OFFSET :offset
    """)

    def __init__(self, session: Session) -> None:
        self._session = session

    def find_by_id_with_user(self, id: int) -> {Domain}WithUserRow | None:
        row = self._session.execute(self._BY_ID_WITH_USER, {"id": id}).mappings().one_or_none()
        return {Domain}WithUserRow(**row) if row is not None else None

    def find_by_user_id_with_user(self, user_id: int, limit: int, offset: int) -> list[{Domain}WithUserRow]:
        rows = self._session.execute(
            self._BY_USER_ID_WITH_USER, {"user_id": user_id, "limit": limit, "offset": offset}
        ).mappings().all()
        return [{Domain}WithUserRow(**r) for r in rows]
```

**3-3. to_read 변환 (Default{Domain}Repository 내부)**

```python
def _to_read(self, row: {Domain}WithUserRow) -> {Domain}Read:
    # VO 재구성: 행의 개별 컬럼을 VO 로 조립
    # JOIN 필드 포함
    ...
```

**Read 반환 메서드 구현:**

```python
def __init__(self, tx: TransactionManager, entities: {Domain}EntityQuery, joins: {Domain}JoinQuery) -> None:
    ...
    self._joins = joins

@transactional
def find_by_id(self, identity: {Domain}Identity) -> {Domain}Read | None:
    row = self._joins.find_by_id_with_user(identity.{domain}_id)
    return self._to_read(row) if row is not None else None
```

### 4. 자식 컬렉션 (1:N) 이 있는 경우

Entity 에 `relationship(lazy="selectin")` 자식 컬렉션이 있는 도메인 (entity-references.md §Baz).

- **조회**: EntityQuery 의 조회 메서드가 돌려주는 부모 Entity 에 자식이 이미 들어 있다. 자식 전용 조회 메서드는 두지 않는다.
- **저장**: Default{Domain}Repository 의 `save` update 분기에서 `entity.categories = [...]` 처럼 리스트를 **통째로 교체**한다. cascade 가 orphan 을 지운다. 개별 append/remove 금지.
- **자식 값으로 부모 필터** ("카테고리가 X 인 글"): 같은 aggregate 이므로 **EntityQuery** 에 서브쿼리로 둔다. JoinQuery 로 보내지 않는다.

```python
def find_by_category_and_active(self, category: {Domain}Category) -> list[{Domain}Entity]:
    child_ids = select({Domain}CategoryEntity.{domain}_id).where(
        {Domain}CategoryEntity.category == category
    )
    stmt = select({Domain}Entity).where(
        {Domain}Entity.id.in_(child_ids), {Domain}Entity.is_deleted.is_(False)
    )
    return list(self._session.scalars(stmt).all())
```

- **JOIN 목록 + 자식 컬렉션** 이 둘 다 필요한 경우: JoinQuery 로 행 목록을 받고, 그 id 들로 EntityQuery `find_by_id_in(ids)` 를 **한 번** 호출해 자식이 딸린 Entity 를 받아 `dict[id, Entity]` 로 합친다. 행마다 조회하는 N+1 은 금지.
- Stats 분리 테이블은 별도 인터페이스(`{Domain}StatsRepository`)이므로 `{Domain}StatsEntityQuery` + `Default{Domain}StatsRepository` 를 별도 파일로 둔다. 본 도메인 Read 에 Stats 값을 합치는 것은 본 도메인 JoinQuery 의 LEFT JOIN 이다.

### 5. 동적 SQL 이 필요한 경우

WHERE 절의 조건 자체가 입력에 따라 있거나 없거나 하는 경우 (검색어 필터, 태그 필터, 정렬 분기 등).

- **EntityQuery** 에서는 `if` 로 `.where()` / `.order_by()` 를 조건부로 붙이면 된다. 별도 도구 불필요.
- **JoinQuery** 에서는 SQL 문자열을 조립하지 않는다. JOIN 이 필요한데 조건이 동적이면 그 메서드만 `select()` 빌더로 쓰는 것을 예외로 허용하고, docstring 에 "동적 조건이라 빌더 사용" 을 적는다.

**다음 경우는 자체 판단으로 도입하지 말고, 유저에게 문의한다.**
- `ilike` 검색어 필터가 성능 문제가 될 규모이거나 전문 검색이 필요할 때 (검색 엔진 이관 후보)
- Postgres 특화 문법(윈도우 함수, 재귀 CTE)이 필요할 때

문의 시 포함할 것:
- 어떤 메서드가 동적 SQL 을 필요로 하는지
- 어떤 조건이 동적인지 (optional WHERE, 정렬 분기 등)

> 참고: WHERE 조건이 고정이고 파라미터만 바뀌는 경우는 동적 SQL 이 **아니다.**
> `WHERE user_id = :user_id`, `LIMIT :limit OFFSET :offset` 등은 JoinQuery 의 고정 SQL 로 처리한다.

---

## 기존 변경 시

- as-is/to-be 를 유저에게 제시하고 확인받는다
- infra 에 메서드 추가되면 Default{Domain}Repository 에도 구현 추가 (누락 시 인스턴스 생성이 `TypeError` 로 실패한다)
- Entity 필드 변경되면 `_to_read` 변환 메서드와 **JoinQuery 의 SQL 문자열** 도 함께 수정. SQL 은 mypy 가 못 잡으므로 컬럼명을 grep 으로 전수 확인
- 기존 쿼리와 유사한 요청이 오면 병합 여부를 유저에게 문의

## 검증

```bash
uv run ruff check {packages.persistence} && uv run lint-imports && uv run mypy
```

JoinQuery 의 SQL 은 여기서 검증되지 않는다. be-test-repository 를 이어서 실행한다.

## 산출물

- 생성/변경된 EntityQuery, JoinQuery, Row dataclass, Default{Domain}Repository 파일 목록
- specs/repository.md 업데이트
- 추가/변경된 스펙 정리본 출력
- JoinQuery 가 생겼으면 "테스트 필요 메서드 목록" 명시
- Provider 에 추가할 `provide(...)` 줄 (add-usecase 가 반영)
