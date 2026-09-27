---
name: be-add-entity-specs
description: Entity 클래스(SQLAlchemy 2.0)와 Alembic migration을 생성/변경한다. 도메인 모델 스펙 또는 별도 지시를 기반으로 작업한다.
---

# add-entity-specs

Entity 클래스와 Alembic migration 을 생성/변경한다.

## 입력

### 지시
- 지시받은 내용에 따라 Entity 와 migration 을 생성/변경한다.
- model 스펙을 맞추는 요청인 경우, 네이밍을 통해 model 패키지에서 해당 도메인 스펙을 참조한다.

### Phase 0. 프로젝트 컨텍스트 해석
`{pkg}`, `{packages.persistence}`, `{packages.schema}` 플레이스홀더 사용. 실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.

### 참고 문서 (읽어올 것)
- `${CLAUDE_PLUGIN_ROOT}/references/be-refs/entity-references.md` — 4필드 직접 선언, relationship 규칙, 4 패턴 예시
- `${CLAUDE_PLUGIN_ROOT}/references/be-refs/schema-references.md` — migration 생성 절차, autogenerate 검토 체크리스트, 타입·네이밍 규칙
- specs/schema.md — 기존 테이블 요약 (프로젝트 루트 기준)
- specs/repository.md — 기존 Entity 정보 (프로젝트 루트 기준)

## 생성 결과물 구조

단순 도메인:
```
{packages.persistence}/src/{pkg}/persistence/{domain}/
├── __init__.py
└── {domain}_entity.py            ← 4필드 직접 선언 + to_domain / from_domain / soft_delete
```

복합 도메인 (자식 컬렉션):
```
{packages.persistence}/src/{pkg}/persistence/{domain}/
├── __init__.py
├── {domain}_entity.py            ← relationship(lazy="selectin") 보유
├── {child}_entity.py             ← 자식 테이블 (평면 배치, 서브디렉터리 금지)
└── {domain}_stats_entity.py      ← Stats (필요 시)
```

레지스트리와 migration (be-init C-6):
```
{packages.persistence}/src/{pkg}/persistence/entities.py
    ← from .{domain}.{domain}_entity import {Domain}Entity  # noqa: F401  한 줄 append

{packages.schema}/alembic/versions/{date}_{rev}_{domain}.py
    ← alembic revision --autogenerate 결과를 검토·정리한 파일
```

## 생성 규칙

### 1. Auditing + soft-delete 필드 직접 선언

dataclass 도메인과 SQLAlchemy Entity 는 공통 부모를 가질 수 없으므로 `Base` 에 컬럼을 두지 않고 각 Entity 가 직접 선언한다 (be-init §4.4 / C-3, C-4).

**모든 Entity 가 가져야 할 필드 4개 (전수 적용)**:
- `is_deleted: Mapped[bool]` (default False, server_default false)
- `deleted_at: Mapped[datetime | None]`
- `created_at: Mapped[datetime]` (default utcnow)
- `updated_at: Mapped[datetime]` (default utcnow, onupdate utcnow)

추가 규칙:
- `__tablename__ = "{snake_case_복수형}"` 필수
- `id: Mapped[int] = mapped_column(BigInteger, Identity(), primary_key=True)` — `int | None` 금지 (alembic check 가 nullable 차이를 계속 보고), `Identity()` 필수 (SERIAL 방지)
- `utcnow` 는 `{pkg}.persistence.base` 에서 import
- entity-references.md 참고

### 1-1. soft-delete 사용 방식 (도메인별 선택)

필드 자체는 모든 Entity 가 보유하지만, **사용 방식**은 도메인별로 결정:

- **정석 soft-delete** (Setup/Series 류):
  - `soft_delete()` 메서드 추가 → 어댑터 `delete_by_id` 에서 호출
  - 조회 쿼리에 `is_deleted.is_(False)` 필터 적용
- **hide/show** (Comment 류):
  - soft-delete 필드는 두되 `soft_delete` 메서드 미제공
  - 별도 플래그(`is_hidden`) 로 상태 관리, `hide()` / `show()` 제자리 전이
- **upsert only / 삭제 불가** (Rating 류):
  - 필드만 두고, `delete_by_id` 자체를 port 에 정의하지 않음

### 1-2. 4필드 예외
- Stats 분리 Entity (`{domain}_stats_entity.py`) 와 자식 컬렉션 Entity 는 4필드 **없음**

### 2. 도메인 → Entity 매핑
1:1 매칭이 되는 도메인이 존재한다면 해당 스펙을 따른다.

- Identity VO → `int` (unwrap). Entity 필드명은 `id`, 도메인은 `{domain}_id`
- StrEnum → `Enum(X, native_enum=False, length=255)`
- 단순 VO (필드 2~3개, 변환 로직 없음) → 개별 컬럼으로 분해
- 자식 컬렉션 (tuple of enum / tuple of VO) → 자식 Entity + `relationship(lazy="selectin", cascade="all, delete-orphan")`
- `composite()` 사용 금지

### 3. 변환 메서드
- `to_domain()`: Entity → Domain. 자식 컬렉션은 `tuple(...)` 로 변환.
- `from_domain()`: classmethod. Domain → Entity. kwargs 로 호출. 자식 컬렉션은 자식 Entity 리스트로.
- `soft_delete()`: 제자리 변경. 새 인스턴스 반환 금지.

### 4. 자식 컬렉션 (1:N)
- 부모 Entity 에 `relationship(lazy="selectin", cascade="all, delete-orphan")` 필수
- 자식 Entity 는 같은 도메인 디렉터리에 평면 배치 (`{child}_entity.py`)
- 자식 FK 컬럼은 `ForeignKey("{parent_table}.id")` — ORM 조인 조건용. DB FK 제약은 `env.py` 가 제외 (C-5)
- `back_populates` / `backref` 금지. 자식은 부모를 모른다
- 순서가 의미 있는 자식만 `order_by`

### 5. 레지스트리 등록
- `entities.py` 에 `from .{domain}.{domain}_entity import {Domain}Entity  # noqa: F401` append
- 자식 Entity 와 Stats Entity 도 각각 한 줄
- 등록 누락 시 Alembic autogenerate 가 테이블을 보지 못한다. **migration 생성 전에 반드시 확인**

### 6. migration
- schema-references.md 의 절차를 따른다: Docker Postgres 기동 → `upgrade head` → `revision --autogenerate -m "{domain}"` → 검토 체크리스트 → `upgrade head` → `alembic check`
- Docker 가 없으면 유저에게 알리고, migration 파일은 schema-references.md 의 예시 형식으로 손으로 작성한 뒤 "적용 미검증" 으로 산출물에 표시
- 자식 컬렉션 테이블의 `{parent}_id` 인덱스는 autogenerate 가 만들지 않으므로 `op.create_index` 를 직접 추가
- migration 이 시드 행을 넣는 참조 테이블(예: `categories`)이면 루트 `conftest.py` 의 `SEED_TABLES` 에 테이블 이름을 추가한다. 빠지면 동시성 테스트의 `truncate_all` 이 시드를 지워 뒤 테스트가 깨진다
- 도메인당 migration 1개. 다른 도메인의 변경이 섞이면 분리

### 7. 도메인 모델 없이 Entity 생성
- 모니터링, 테스트, 중간 테이블 등 별도 지시로도 생성 가능
- 이 경우 도메인 스펙 대신 지시 내용을 따른다
- 4필드 직접 선언 규칙 (§1), migration 규칙 (§6) 은 동일하게 적용

## 기존 변경 시

- as-is/to-be를 유저에게 제시하고 확인받는다
- Entity 필드 추가 시 migration, `to_domain`/`from_domain` 도 함께 변경
- NOT NULL 컬럼 추가는 `server_default` 동반 또는 3단계 (nullable 추가 → 백필 → NOT NULL)
- 컬럼 삭제는 유저 확인 필수 (데이터 손실)

## 검증

```bash
uv run ruff check {packages.persistence} {packages.schema} && uv run mypy
# Docker 가 있으면:
uv run alembic -c {packages.schema}/alembic.ini upgrade head && \
uv run alembic -c {packages.schema}/alembic.ini check
```

`alembic check` 가 "No new upgrade operations detected" 를 출력해야 Entity 와 migration 이 일치한 것.

## 산출물

- 생성/변경된 Entity, 자식 Entity, Stats Entity, migration 파일 목록
- `entities.py` 에 추가된 줄
- `specs/schema.md` (테이블·컬럼 요약), `specs/repository.md` (Entity 정보) 업데이트 (없으면 `specs/` 와 함께 생성)
- 추가/변경된 스펙 정리본 출력
- Docker 없이 진행한 경우 "migration 적용 미검증" 명시
