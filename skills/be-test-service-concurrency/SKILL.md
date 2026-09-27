---
name: be-test-service-concurrency
description: service 패키지의 동시성/순서 보장 통합 테스트(실제 Postgres, 스레드)를 평가하고 작성한다.
---

# be-test-service-concurrency

읽고→계산→쓰기 패턴이 있는 Writer 메서드의 동시성 통합 테스트를 3단계로 수행한다.

---

## 입력

```
대상 Writer 클래스 이름 (예: DefaultCommentWriter)
```

입력이 없으면 "어떤 Writer를 테스트할까요?" 로 확인한다.

---

## Phase 0. 프로젝트 컨텍스트 해석

`{pkg}`, `{packages.service}` 플레이스홀더 사용.
실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.

---

## Phase 1. assess-infra

`${CLAUDE_PLUGIN_ROOT}/references/be-test-refs/infra-checklist.md` §service(통합 — 동시성) 참조.

| 항목 | 경로 | 확인 |
|------|------|------|
| 루트 conftest | `{projectRoot}/conftest.py` | `session_factory`, `truncate_all` 픽스처 |
| Docker | `docker info` | 데몬 실행 중 |
| Entity 레지스트리 / migration | | 대상 Writer 가 만지는 테이블 전부 |

**세이브포인트 `session` 픽스처는 쓰지 않는다.** 스레드 간 공유가 안 되고, 커밋이 실제로 일어나야 경쟁이 재현된다. 대신 스레드마다 `session_factory()` 로 세션을 열고, `truncate_all` 로 정리한다. 별도 `ConcurrencyTestBase` 는 없다 — 픽스처 두 개가 대체한다.

---

## Phase 2. assess-scope

### 2-1. 대상 판별

대상 Writer 의 각 public 메서드에 대해 specs 를 읽고 동시성 위험을 판단한다.

**읽을 specs:**
- `specs/repository.md` — 의존 Repository 메서드의 구현 방식 (select / update 문)
- `specs/usecase.md` — Writer ABC 동작 설명

### 2-2. 동시성 위험 판단 규칙

| Repository 메서드 구현 방식 (specs/repository.md 에서 확인) | 원자성 | 위험 |
|--------------------------------------------------------|--------|------|
| `update(...).values(col=col + n)` — 단일 UPDATE 문 | 원자적 | 안전 |
| `find_xxx → 계산 → save` — 조회 후 저장 | 비원자 | **위험** |
| `get_max_xxx → +1 → save` — 최댓값 기반 생성 | 비원자 | **위험** |
| `exists_xxx → save/update` — 존재 확인 후 변경 | 비원자 | **위험** |

**판단 흐름:**

```
Writer 메서드 내부에서 Repository 를 어떻게 호출하는가?

Q1: 조회 결과를 기반으로 계산 후 쓰기를 하는가?
├─ NO  → 동시성 TC 불필요 (이 메서드 건너뜀)
└─ YES
    Q2: 그 쓰기가 update() 문 (단일 SQL) 인가?
    ├─ YES → 쓰기 자체는 안전하지만 조회-판단 갭 존재 → check-then-act TC
    └─ NO  → 전체 비원자 → 순서/합산 TC
```

`@transactional` 이 있어도 Postgres 기본 격리 수준(READ COMMITTED)에서는 두 트랜잭션이 같은 값을 읽고 각자 +1 할 수 있다. 트랜잭션은 원자성을 주지 격리를 보장하지 않는다.

### 2-3. TC 유형 결정

| 위험 패턴 | TC 유형 | 검증 방법 |
|----------|---------|----------|
| 최댓값 기반 순번 생성 (max+1) | **순서 유일성** | N개 동시 생성 → 번호 중복 없는지 |
| 카운터 갱신 (조회→계산→save) | **합산 정확성** | N개 동시 갱신 → 최종값 == 기대값 |
| 존재 확인 후 생성/수정 | **중복 방지** | N개 동시 시도 → 정확히 1개만 생성 or 합산 정확 |

### 2-4. 산출물

```
## 분석 결과: {ClassName}
### 동시성 대상 메서드
| 메서드 | 위험 패턴 | TC 유형 | 검증 |
```

사용자 확인 후 Phase 3 진행.

---

## Phase 3. add-test

### 파일 위치

```
{packages.service}/tests/{domain}/
└── impl/test_default_{domain}_writer_concurrency.py
```

### 모듈 템플릿

```python
import threading
from concurrent.futures import ThreadPoolExecutor

import pytest
from sqlalchemy import select, text
from sqlalchemy.orm import Session, sessionmaker

from {pkg}.persistence.transaction import DefaultTransactionManager
from {pkg}.persistence.{domain}.{domain}_entity_query import XxxEntityQuery
from {pkg}.persistence.{domain}.{domain}_repository import DefaultXxxRepository
from {pkg}.service.{domain}.impl.default_{domain}_writer import DefaultXxxWriter

pytestmark = pytest.mark.usefixtures("truncate_all")


def _writer(session: Session) -> DefaultXxxWriter:
    """스레드마다 세션 → 어댑터 → Writer 를 새로 조립한다. Provider 와 같은 구성."""
    tx = DefaultTransactionManager(session)
    return DefaultXxxWriter(
        tx=tx,
        repository=DefaultXxxRepository(tx=tx, entities=XxxEntityQuery(session)),
    )


@pytest.fixture
def seed(session_factory: sessionmaker[Session]) -> dict[str, int]:
    """선행 데이터. 실제 커밋되어야 다른 스레드가 본다."""
    with session_factory() as session:
        tx = DefaultTransactionManager(session)
        user_repo = DefaultUserRepository(tx=tx, entities=UserEntityQuery(session))
        user = user_repo.save(User.create("tester", "test@test.com", "google123", ""))
        # ... 필요한 선행 데이터
        session.commit()   # 테스트 seed 에 한해 직접 commit 허용 (어댑터의 @transactional 은 이미 닫힘)
        return {"user_id": user.user_id}
```

### 동시 실행 패턴 (모든 TC 공통)

```python
def _run_concurrently(session_factory: sessionmaker[Session], n: int, action) -> list[BaseException | None]:
    barrier = threading.Barrier(n)
    errors: list[BaseException | None] = [None] * n

    def worker(i: int) -> None:
        with session_factory() as session:      # 스레드마다 자기 세션
            writer = _writer(session)
            barrier.wait()                       # 전부 준비되면 동시에 출발
            try:
                action(writer)
            except BaseException as e:           # noqa: BLE001
                errors[i] = e

    with ThreadPoolExecutor(max_workers=n) as ex:
        list(ex.map(worker, range(n)))
    return errors
```

- `Session` 은 스레드 안전이 아니다. **스레드마다 `session_factory()`**
- `threading.Barrier(n)` 이 Java 의 `CountDownLatch` 두 개 역할
- 예외는 모아서 테스트가 판단 (유니크 위반이 "정상" 인 TC 도 있다)

### TC 유형별 검증

**순서 유일성:**
```python
def test_should_assign_unique_orders_when_created_concurrently(session_factory, seed):
    n = 10
    errors = _run_concurrently(
        session_factory, n,
        lambda w: w.create(CommentCreateCommand(request_user_id=seed["user_id"], article_id=1, ...)),
    )
    assert all(e is None for e in errors), errors

    with session_factory() as s, s.begin():   # autobegin=False 라 begin 없이 조회하면 InvalidRequestError
        values = s.scalars(text("SELECT qux_order FROM quxs WHERE article_id = 1")).all()
    assert len(values) == n
    assert len(set(values)) == n
```

**합산 정확성:**
```python
def test_should_sum_exactly_when_incremented_concurrently(session_factory, seed):
    n = 10
    errors = _run_concurrently(session_factory, n, lambda w: w.increase_view_count(identity, 1))
    assert all(e is None for e in errors), errors

    with session_factory() as s, s.begin():   # autobegin=False 라 begin 없이 조회하면 InvalidRequestError
        total = s.scalar(text("SELECT view_count FROM baz_stats WHERE baz_id = :id"), {"id": seed["baz_id"]})
    assert total == n
```

**중복 방지:**
```python
def test_should_create_exactly_one_when_bookmarked_concurrently(session_factory, seed):
    n = 10
    errors = _run_concurrently(session_factory, n, lambda w: w.bookmark(seed["user_id"], seed["setup_id"]))
    # 유니크 제약이 있으면 n-1 개는 ConflictException 이 정상
    assert sum(e is None for e in errors) == 1

    with session_factory() as s, s.begin():   # autobegin=False 라 begin 없이 조회하면 InvalidRequestError
        count = s.scalar(text("SELECT COUNT(*) FROM saved_posts WHERE user_id = :u"), {"u": seed["user_id"]})
    assert count == 1
```

검증 조회는 `text()` 로 직접 해도 된다 (테스트의 관찰 목적, 프로덕션 코드 아님). 세션 팩토리가 `autobegin=False` 라서 검증 조회도 `s.begin()` 안에서 해야 한다 (2026-09-27 testbed 에서 확인).

### 검증

```bash
uv run pytest {packages.service}/tests/{domain}/impl/test_default_{domain}_writer_concurrency.py -v
```

실패하면 그것이 곧 **동시성 버그 발견**이다. 프로덕션 코드 수정(update 문으로 전환, `SELECT ... FOR UPDATE`, 유니크 제약 추가)은 유저에게 보고하고 별도 작업으로 넘긴다.

---

## 공통 규칙

- Phase 1은 최초 1회만 실행
- Phase 2 산출물을 사용자에게 보여주고 확인 후 Phase 3 진행
- 테스트 코드 내에서 프로덕션 코드를 수정하지 않는다
- 기존 테스트 파일이 있으면 덮어쓰지 않고 추가/수정한다
- 선행 데이터는 반드시 도메인 어댑터 API 를 통해 생성한다 (`text()` 직접 INSERT 금지 — 검증 SELECT 는 예외)
- `truncate_all` 이 테스트 후 전 테이블을 비우므로, 다른 테스트와 같은 DB 를 공유해도 격리된다. 단 병렬 실행(`pytest -n`)과는 함께 쓰지 않는다
