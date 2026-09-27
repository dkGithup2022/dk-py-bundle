---
name: be-test-repository
description: persistence 패키지의 Default{Domain}Repository / EntityQuery / JoinQuery 통합 테스트를 평가하고 작성한다.
---

# be-test-repository

persistence 패키지의 통합 테스트를 3단계로 수행한다.

---

## 입력

```
대상 구현체 클래스 이름 (예: DefaultSetupRepository)
```

입력이 없으면 "어떤 Repository를 테스트할까요?" 로 확인한다. 같은 도메인의 EntityQuery / JoinQuery 를 함께 다룬다.

---

## Phase 0. 프로젝트 컨텍스트 해석

`{pkg}`, `{packages.persistence}`, `{packages.schema}` 플레이스홀더 사용.
실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.

## Phase 1. assess-infra — 테스트 인프라 점검

### 점검 항목

| 항목 | 경로 | 확인 내용 |
|------|------|----------|
| 루트 conftest | `{projectRoot}/conftest.py` | `database_url`, `engine`, `session_factory`, `session`, `truncate_all` 픽스처 |
| testcontainers | 루트 `pyproject.toml` dev 그룹 | `testcontainers[postgres]` |
| Docker | `docker info` | 데몬 실행 중 |
| Entity 레지스트리 | `{packages.persistence}/src/{pkg}/persistence/entities.py` | 대상 Entity 와 JOIN 대상 Entity 등록 여부 |
| migration | `{packages.schema}/alembic/versions/` | 대상 테이블과 JOIN 대상 테이블의 migration 존재 여부 |

### 점검 순서

1. 위 항목을 확인
2. 누락된 파일이 있으면 **즉시 생성** (be-init §4.10 conftest 템플릿 사용)
3. Docker 가 없으면 중단하고 유저에게 알린다
4. 모두 존재하면 "인프라 준비 완료" 출력 후 Phase 2로 진행

---

## Phase 2. assess-scope — 테스트 범위 분석

### 읽어야 할 파일

대상 Repository 에 대해 아래 파일을 반드시 읽는다:

1. `{domain}_repository.py` — 구현체 (public 메서드 = 테스트 대상)
2. `{domain}_entity_query.py` — `select`, `update` 문 확인
3. `{domain}_join_query.py` — SQL 의 LEFT JOIN 확인 (있으면). **메서드 전수 테스트 대상**
4. `{domain}_entity.py` — `relationship` 확인

추가로 자식 Entity(`{child}_entity.py`)와 Stats 파일이 있으면 함께 읽는다.

### 분석 절차

#### 2-1. 메서드 분류

구현체·EntityQuery·JoinQuery 의 모든 public 메서드를 아래 카테고리로 분류한다:

| 카테고리 | 판단 기준 | 예시 |
|---------|----------|------|
| Entity↔Domain 변환 | save + find_write_by_id 왕복 | save → find_write_by_id → 전 필드 일치 |
| 자식 컬렉션 변환 | `relationship` 존재 | categories, tags, items |
| JOIN 결과 | JoinQuery 메서드 | `{Domain}WithUserRow` — **메서드마다 필수** |
| `update()` 문 | EntityQuery 에 `update(...)` | increase_view_count |
| 동적 조건 | `if` 로 `.where()` 조건부 추가 | find_all with filters |
| 페이징 | `.limit().offset()` / `LIMIT :limit` | find_by_user_id(page, size) |
| 집계 | `func.count`, `COUNT(*)`, `MAX` | count_by_article, find_max_comment_order |
| 단건 조회 | `find_* → X \| None` | find_by_email, find_write_by_id |
| 삭제 | `delete_by_id` | soft delete 확인 |

#### 2-2. 조립 대상 결정

```
Q: 구현체가 다른 도메인의 Repository 를 사용하는가?
├─ NO → 자기 도메인만 조립
│   Default{Domain}Repository(tx, {Domain}EntityQuery(session), {Domain}JoinQuery(session))
│
└─ YES → 사용하는 도메인의 구현체도 같은 session 으로 조립
```

**주의**: JoinQuery 의 SQL JOIN 은 DB 레벨이므로 다른 도메인 구현체가 필요하지 않다.
다만 **선행 데이터를 생성**하려면 해당 도메인 구현체를 조립해야 한다.

**원칙**: 테스트 데이터 생성에 필요한 모든 구현체를 같은 `session` 으로 조립한다.

#### 2-3. 선행 데이터 판단

```
Q: JoinQuery 의 SQL 에 LEFT JOIN 이 있는가?
├─ JOIN users → User 선행 생성 필요 → DefaultUserRepository 조립
├─ JOIN {table}_stats → Stats 선행 생성 필요 → Default{Domain}StatsRepository 조립
└─ 없음 → 선행 데이터 불필요
```

JoinQuery 소스의 SQL 을 읽어 도출한다. 도메인별 고정 표는 없다.

#### 2-4. 산출물

분석 결과를 아래 형식으로 정리한다:

```
## 분석 결과: {ClassName}

### 테스트 설정
- 조립: [구현체 + 쿼리 클래스 목록]
- 선행 데이터: [목록]

### 테스트 케이스
| # | 클래스 | 메서드 | 카테고리 | 검증 포인트 |
|---|--------|--------|---------|------------|
| 1 | 구현체 | save → find_write_by_id | 변환 + 자식 컬렉션 | categories, tags 왕복 |
| 2 | JoinQuery | find_by_id_with_user | JOIN | nickname, 상대 행 없을 때 None |
| ... | ... | ... | ... | ... |
```

JoinQuery 메서드 중 케이스가 없는 것이 있으면 "미커버" 로 표시하고 사용자 확인 전에 채운다.

---

## Phase 3. add-test — 테스트 코드 작성

### 파일 위치

```
{packages.persistence}/tests/{domain}/
├── test_{domain}_repository.py       ← 구현체
├── test_{domain}_entity_query.py     ← EntityQuery
└── test_{domain}_join_query.py       ← JoinQuery (있을 때만)
```

### 클래스 템플릿

```python
import pytest
from sqlalchemy.orm import Session

from {pkg}.persistence.transaction import DefaultTransactionManager
from {pkg}.persistence.{domain}.{domain}_entity_query import {Domain}EntityQuery
from {pkg}.persistence.{domain}.{domain}_join_query import {Domain}JoinQuery      # 있을 때만
from {pkg}.persistence.{domain}.{domain}_repository import Default{Domain}Repository


@pytest.fixture
def repository(session: Session) -> Default{Domain}Repository:
    return Default{Domain}Repository(
        tx=DefaultTransactionManager(session),
        entities={Domain}EntityQuery(session),
        joins={Domain}JoinQuery(session),        # 있을 때만
    )


@pytest.fixture
def entities(session: Session) -> {Domain}EntityQuery:
    return {Domain}EntityQuery(session)


@pytest.fixture
def joins(session: Session) -> {Domain}JoinQuery:
    return {Domain}JoinQuery(session)


# Phase 2 에서 도출된 선행 데이터 픽스처 (같은 session 으로 조립)
# @pytest.fixture
# def test_user_id(session: Session) -> int: ...


# Phase 2 의 테스트 케이스 목록에 따라 test_ 함수 작성
```

쿼리 클래스에는 `@transactional` 이 없으므로 직접 호출하는 테스트는 `with session.begin():` 안에서 호출한다. `session` 픽스처는 세이브포인트 모드라 테스트 끝에 전부 롤백된다.

### 작성 규칙

1. **테스트 데이터는 반드시 구현체를 통해 생성한다.**
   - `session.execute(text("INSERT ..."))` raw SQL 로 INSERT 하지 않는다.
   - 다른 도메인 데이터가 필요하면 해당 도메인 구현체를 같은 `session` 으로 조립해 `save` 한다.
   - 이유: raw SQL 은 정당한 테스트 데이터인지 코드 리뷰 시 판별이 어렵다.

2. **JoinQuery 는 SQL 결과 자체를 검증한다.**
   - 구현체를 거치지 않고 JoinQuery 메서드를 직접 호출해 행 dataclass 필드를 확인한다.
   - 상대 테이블 행이 **없는 경우** 도 반드시 1개 (LEFT JOIN + COALESCE / None 확인).
   - 이유: SQL 문자열은 mypy 가 못 잡는 유일한 구간이다.

3. **메서드 네이밍**: `test_should_예상결과_when_조건`
   ```python
   test_should_return_setup_when_saved_and_found_by_id()
   test_should_return_none_when_non_existing_id()
   test_should_increment_view_count_when_increase_view_count_called()
   ```

4. **구조**: given / when / then 주석 블록

5. **카테고리별 패턴**: `${CLAUDE_PLUGIN_ROOT}/references/be-test-refs/repository-patterns.md` 참조

6. **동적 조건 테스트**:
   - 필터 조합을 여러 케이스로 분리
   - 빈 필터, 단일 필터, 복합 필터 각각 테스트
   - 정렬 옵션별 테스트

7. **soft delete 고려** (모든 Entity 가 is_deleted/deleted_at 필드 보유 전제):
   - 정석 soft-delete 를 쓰는 도메인은 `delete_by_id` 후 `find_by_id` 가 None 반환하는지 확인
   - is_deleted=true 인 데이터가 EntityQuery 조회와 JoinQuery SQL 양쪽에서 제외되는지 확인
   - hide/show 류나 upsert-only 도메인은 해당 항목 생략

### 작성 후 검증

```bash
uv run pytest {packages.persistence}/tests/{domain} -v
```

컴파일 실패 시 원인을 분석하고 수정한다. JoinQuery 테스트가 실패하면 SQL 의 컬럼명·테이블명·별칭을 Entity 및 migration 과 대조한다.

---

## 공통 규칙

- Phase 1은 최초 1회만 실행. 이후 동일 패키지의 다른 Repository 테스트 시 건너뛴다.
- Phase 2 → Phase 3는 항상 순차 실행. Phase 2 산출물을 사용자에게 보여주고 확인 후 Phase 3 진행.
- **JoinQuery 메서드에 테스트가 없으면 그 메서드는 미완성으로 보고한다.**
- 테스트 코드 내에서 프로덕션 코드를 수정하지 않는다.
- 기존 테스트 파일이 있으면 덮어쓰지 않고 추가/수정한다.
