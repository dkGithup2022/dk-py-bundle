# Service References

`be-add-usecase` 스킬이 참고하는 Reader/Writer/Usecase 작성 레퍼런스.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 패키지)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

---

## 공통 요소

### 클래스 구성과 네이밍

| 역할 | ABC (service/{domain}/) | 구현 (impl/) | 배치 |
|---|---|---|---|
| 조회 | `{Domain}Reader` | `Default{Domain}Reader` | ABC / 구체 클래스 |
| 쓰기 | `{Domain}Writer` | `Default{Domain}Writer` | ABC / 구체 클래스 |
| 조합 (선택) | — (ABC 없음) | `{Domain}{Action}Usecase` 또는 `{Action}Usecase` | 클래스 직접 노출 |

- 구현체에 데코레이터 없음 (`@Service` 대응 없음). 조립은 dishka Provider 가 한다
- 모듈 상단 `logger = logging.getLogger(__name__)`. 형식(JSON 한 줄)과 `request_id` / `user_id` 같은 요청 키는 `my_logging` 이 붙이므로 메시지에 다시 넣지 않는다 (be-init C-13)
- 요청 범위 키를 더할 일이 있으면 `from {pkg}.my_logging import set_context` 로 `set_context("article_id", 3)` 처럼. 그 요청의 이후 로그 줄 전부에 붙는다
- Usecase 는 ABC 없이 클래스 직접 노출 (재사용 Reader/Writer 와 달리 API 단위 전용)

### 의존 방향

- Reader/Writer/Usecase 는 **infra port** (`{Domain}Repository` ABC) 와 `TransactionManager` 만 의존. `Default{Domain}Repository`, `{Domain}Entity`, `Session` 직접 참조 금지 (import-linter 가 잡는다)
- Usecase 는 Reader, Writer, 다른 도메인의 Repository 를 자유롭게 주입
- 모든 의존은 **생성자 파라미터**로 받는다. dishka 가 타입 힌트를 보고 주입한다

### 트랜잭션 주입

`@transactional` 을 쓰는 클래스(Writer, Usecase)는 생성자에서 `self.tx: TransactionManager` 를 설정한다 (be-init C-9).

```python
from {pkg}.transaction import TransactionManager, transactional


class DefaultBarWriter(BarWriter):
    def __init__(self, tx: TransactionManager, repository: BarRepository) -> None:
        self.tx = tx
        self._repository = repository

    @transactional
    def create(self, command: BarCreateCommand) -> BarIdentity: ...
```

- Reader: `tx` 받지 않음, `@transactional` 없음. Reader 가 부르는 Repository 메서드가 각자 짧은 트랜잭션을 연다
- Writer: 생성자에 `tx`, 쓰기 메서드마다 `@transactional`
- Usecase: 생성자에 `tx`, `execute` 에 `@transactional`. 선검증 + Writer 호출 + 부수효과를 한 트랜잭션으로. 안에서 호출되는 Writer/Repository 의 `@transactional` 은 합류한다

### 메서드 네이밍

- **Reader**: `get_by_id(...)`, `get_all(...)`, `get_by_user_id(...)` — `get` prefix. 없으면 `raise {Domain}NotFoundException`. `find_xxx` 는 port(Repository) 에서만 사용
- **Writer**: 동사로 — `create`, `update`, `delete`, `increase_view_count`, `hide`, `show` 등
- **Usecase**: public `execute(...)` 1개. 입력은 인증된 `user_id: int` + Command

### Command

- `@dataclass(frozen=True, slots=True) class {Action}Command`
- 패키지: `service/{domain}/dto/`
- 첫 필드 관례: `request_user_id: int` (인증된 사용자 ID)
- 컬렉션 필드는 도메인과 같이 `tuple`. 라우터가 Request 의 `list` 를 `tuple(...)` 로 바꿔서 넣는다

```python
@dataclass(frozen=True, slots=True)
class SetupCreateCommand:
    request_user_id: int
    title: str
    body: str
    categories: tuple[SetupCategory, ...]
```

### 예외 처리

- `{Domain}NotFoundException` (add-model 이 생성) — 조회 실패 시
- `ForbiddenException` — 소유권/권한 체크 실패
- `ConflictException` — 중복 / 상태 충돌
- `InvalidInputException` — 비즈니스 규칙 위반 입력
- be-init §4.5 의 `DomainException` + `http_status_code` / `http_error_message` 구조 준수 (C-7)

### DI 등록 (app 패키지)

service 클래스는 데코레이터로 등록되지 않는다. add-usecase 스킬이 `app/providers/{domain}_provider.py` 를 만들고 `app/di.py` 의 `DOMAIN_PROVIDERS` 에 추가한다 (C-10).

```python
# packages/app/src/{pkg}/app/providers/bar_provider.py
from dishka import Provider, Scope, provide

from {pkg}.infrastructure.bar.bar_repository import BarRepository
from {pkg}.persistence.bar.bar_entity_query import BarEntityQuery
from {pkg}.persistence.bar.bar_repository import DefaultBarRepository
from {pkg}.service.bar.bar_reader import BarReader
from {pkg}.service.bar.bar_writer import BarWriter
from {pkg}.service.bar.impl.default_bar_reader import DefaultBarReader
from {pkg}.service.bar.impl.default_bar_writer import DefaultBarWriter
from {pkg}.service.bar.usecase.bar_create_usecase import BarCreateUsecase


class BarProvider(Provider):
    scope = Scope.REQUEST

    entity_query = provide(BarEntityQuery)
    repository = provide(DefaultBarRepository, provides=BarRepository)
    reader = provide(DefaultBarReader, provides=BarReader)
    writer = provide(DefaultBarWriter, provides=BarWriter)
    create_usecase = provide(BarCreateUsecase)
```

```python
# packages/app/src/{pkg}/app/di.py  (변경 부분)
from .providers.bar_provider import BarProvider

DOMAIN_PROVIDERS: list[Provider] = [
    BarProvider(),
]
```

`Session` 과 `TransactionManager` 는 be-init 의 `RequestProvider` 가 이미 제공한다. EntityQuery 생성자의 `session: Session`, 구현체·Writer 생성자의 `tx: TransactionManager` 가 자동으로 채워진다.

---

## Pattern Foo — Reader 단독 (조회 전용 도메인)

**언제 쓰나**: 사용자 action 으로 변경되지 않는 정적 / reference 성격 도메인.

```python
from abc import ABC, abstractmethod


class FooReader(ABC):
    @abstractmethod
    def get_all(self) -> list[FooRead]: ...

    @abstractmethod
    def get_by_id(self, identity: FooIdentity) -> FooRead: ...
```

```python
import logging

from {pkg}.exception.foo.foo_not_found_exception import FooNotFoundException
from {pkg}.infrastructure.foo.foo_repository import FooRepository

logger = logging.getLogger(__name__)


class DefaultFooReader(FooReader):
    def __init__(self, repository: FooRepository) -> None:
        self._repository = repository

    def get_all(self) -> list[FooRead]:
        return self._repository.find_all()

    def get_by_id(self, identity: FooIdentity) -> FooRead:
        read = self._repository.find_by_id(identity)
        if read is None:
            raise FooNotFoundException(f"foo not found: {identity.foo_id}")
        return read
```

**특징**:
- Writer 파일 생성 금지 (변경 로직 없음)
- Usecase 불필요 (조회만이라 조합 없음)
- 구성 파일: `foo_reader.py`, `impl/default_foo_reader.py` 2개

---

## Pattern Bar — Reader + Writer 쌍 (기본 CRUD)

**언제 쓰나**: 일반 CRUD 도메인. 대부분 도메인이 여기에 해당.

### Reader

```python
class BarReader(ABC):
    @abstractmethod
    def get_by_id(self, identity: BarIdentity) -> BarRead: ...

    @abstractmethod
    def get_by_user_id(self, user_id: int, page: int, size: int) -> list[BarRead]: ...

    @abstractmethod
    def count_by_user_id(self, user_id: int) -> int: ...
```

```python
class DefaultBarReader(BarReader):
    def __init__(self, repository: BarRepository) -> None:
        self._repository = repository

    def get_by_id(self, identity: BarIdentity) -> BarRead:
        read = self._repository.find_by_id(identity)
        if read is None:
            raise BarNotFoundException(f"bar not found: {identity.bar_id}")
        return read

    def get_by_user_id(self, user_id: int, page: int, size: int) -> list[BarRead]:
        return self._repository.find_by_user_id(user_id, page, size)

    def count_by_user_id(self, user_id: int) -> int:
        return self._repository.count_by_user_id(user_id)
```

### Writer

```python
class BarWriter(ABC):
    @abstractmethod
    def create(self, command: BarCreateCommand) -> BarIdentity: ...

    @abstractmethod
    def update(self, command: BarUpdateCommand) -> Bar: ...

    @abstractmethod
    def delete(self, identity: BarIdentity) -> None: ...
```

```python
import logging

from {pkg}.transaction import TransactionManager, transactional

logger = logging.getLogger(__name__)


class DefaultBarWriter(BarWriter):
    def __init__(self, tx: TransactionManager, repository: BarRepository) -> None:
        self.tx = tx
        self._repository = repository

    @transactional
    def create(self, command: BarCreateCommand) -> BarIdentity:
        bar = Bar.create(command.request_user_id, command.title, command.body)
        saved = self._repository.save(bar)
        logger.info("Bar created: id=%s", saved.bar_id)
        assert saved.bar_id is not None
        return BarIdentity(saved.bar_id)

    @transactional
    def update(self, command: BarUpdateCommand) -> Bar:
        identity = BarIdentity(command.bar_id)
        existing = self._repository.find_write_by_id(identity)
        if existing is None:
            raise BarNotFoundException(f"bar not found: {command.bar_id}")
        updated = existing.with_update(command.title, command.body)
        return self._repository.save(updated)

    @transactional
    def delete(self, identity: BarIdentity) -> None:
        existing = self._repository.find_write_by_id(identity)
        if existing is None:
            raise BarNotFoundException(f"bar not found: {identity.bar_id}")
        self._repository.delete_by_id(identity)
        logger.info("Bar deleted: id=%s", identity.bar_id)
```

**특징**:
- Writer 의 update/delete 는 **선검증** (`find_write_by_id` → `None` 이면 raise) 으로 404 의미 확보 — 어댑터 `delete_by_id` 가 조용한 no-op 이므로 여기서 막는다
- `create` 는 Identity, `update` 는 Write 모델 반환. Read 모델 반환은 Reader 책임
- Reader 에는 `tx` 도 `@transactional` 도 없음
- 구성 파일: 4개 (`bar_reader.py`, `bar_writer.py`, `impl/default_bar_reader.py`, `impl/default_bar_writer.py`) + Command dataclass(들)

---

## Pattern Baz — Usecase (API 단위 조합 + 부수효과)

**언제 쓰나**: API 1개가 여러 도메인을 건드리거나, 부수효과(Stats/Outbox/Notification) 를 동반.

```python
import logging

from {pkg}.transaction import TransactionManager, transactional

logger = logging.getLogger(__name__)


class BazCreateUsecase:
    def __init__(
        self,
        tx: TransactionManager,
        user_repository: UserRepository,
        user_stats_repository: UserStatsRepository,
        baz_writer: BazWriter,
        outbox_event_recorder: OutboxEventRecorder,
    ) -> None:
        self.tx = tx
        self._user_repository = user_repository
        self._user_stats_repository = user_stats_repository
        self._baz_writer = baz_writer
        self._outbox_event_recorder = outbox_event_recorder

    @transactional
    def execute(self, user_id: int, command: BazCreateCommand) -> BazIdentity:
        # 1. 선검증
        if self._user_repository.find_by_id(UserIdentity(user_id)) is None:
            raise UserNotFoundException(f"user not found: {user_id}")

        # 2. Writer 호출 (같은 트랜잭션에 합류)
        created = self._baz_writer.create(command)

        # 3. 부수효과
        self._user_stats_repository.increase_post_count(UserIdentity(user_id))
        self._outbox_event_recorder.record(
            RecordOutboxEventCommand.created(TargetType.BAZ, created.baz_id)
        )
        return created
```

권한 체크가 필요한 Usecase:

```python
class BazDeleteUsecase:
    def __init__(
        self,
        tx: TransactionManager,
        user_repository: UserRepository,
        baz_repository: BazRepository,
        baz_writer: BazWriter,
    ) -> None:
        self.tx = tx
        self._user_repository = user_repository
        self._baz_repository = baz_repository
        self._baz_writer = baz_writer

    @transactional
    def execute(self, user_id: int, identity: BazIdentity) -> None:
        if self._user_repository.find_by_id(UserIdentity(user_id)) is None:
            raise UserNotFoundException(f"user not found: {user_id}")

        existing = self._baz_repository.find_write_by_id(identity)
        if existing is None:
            raise BazNotFoundException(f"baz not found: {identity.baz_id}")

        if existing.user_id != user_id:
            raise ForbiddenException("not owner")

        self._baz_writer.delete(identity)
```

**특징**:
- public 메서드는 `execute(...)` 1개
- `@transactional` 은 `execute` 에. 안에서 Writer 의 `@transactional` 은 합류하므로 전체가 한 트랜잭션. 부수효과 도중 예외가 나면 Writer 가 한 것까지 롤백된다
- 선검증 → Writer/Reader → 부수효과 순서
- ABC 없이 **클래스 직접 노출** — API 단위라 재사용성 없음
- Provider 에 `provide(BazCreateUsecase)` 로 등록 (provides 생략 = 자기 타입)

**파일 배치**: `service/{domain}/usecase/{domain}_{action}_usecase.py`
- 예: `SetupCreateUsecase`, `SetupDeleteUsecase`, `RatingUsecase`, `BookmarkUsecase`

---

## Pattern Qux — Usecase 단독 (Reader/Writer 없이 Repository 직접 조립)

**언제 쓰나**: 대시보드/마이페이지처럼 여러 도메인의 집계를 조합하지만 **API 1개 전용** — 중간 Reader 로 추상화할 가치 낮음.

```python
class MypageUsecase:
    def __init__(
        self,
        user_repository: UserRepository,
        setup_repository: SetupRepository,
        comment_repository: CommentRepository,
        user_stats_repository: UserStatsRepository,
    ) -> None:
        self._user_repository = user_repository
        self._setup_repository = setup_repository
        self._comment_repository = comment_repository
        self._user_stats_repository = user_stats_repository

    def execute(self, user_id: int) -> MypageRead:
        identity = UserIdentity(user_id)
        user = self._user_repository.find_by_id(identity)
        if user is None:
            raise UserNotFoundException(f"user not found: {user_id}")

        my_setups = self._setup_repository.find_by_user_id(user_id, 0, 5)
        my_comments = self._comment_repository.find_by_user_id(identity, 0, 5)
        stats = self._user_stats_repository.find_by_user_id(identity) or UserStats.init(user_id)

        return MypageRead(user, tuple(my_setups), tuple(my_comments), stats)
```

**특징**:
- Reader/Writer ABC 없음
- Repository 를 직접 다수 주입
- 읽기만이라 `tx` 도 `@transactional` 도 없음. 각 Repository 호출이 짧은 트랜잭션을 연다. 한 스냅샷이 꼭 필요하면 `tx` 를 받고 `@transactional` 을 붙여 한 트랜잭션으로 묶는다
- `MypageRead` 는 여러 도메인을 합친 Read 이므로 `service/{usecase}/dto/` 에 dataclass 로 둔다 (model 패키지가 아님)
- 구성 파일: Usecase 1개 + dto

---

## Pattern Quux — 인메모리 버퍼링 + 주기 플러시 (특수)

**언제 쓰나**: 조회수·좋아요 같은 **고빈도 이벤트** 가 그대로 DB 에 가면 락 경합이 심할 때. 메모리에 누적 → 주기적으로 배치 flush.

```python
import logging
import threading
from collections import defaultdict

logger = logging.getLogger(__name__)


class FooViewMemory:
    """APP 스코프 싱글턴. Provider 에서 scope=Scope.APP 으로 등록."""

    def __init__(self, session_factory: sessionmaker[Session]) -> None:
        self._session_factory = session_factory
        self._lock = threading.Lock()
        self._view_counts: dict[int, int] = defaultdict(int)

    def count_up(self, foo_id: int) -> None:
        with self._lock:
            self._view_counts[foo_id] += 1

    def flush(self) -> None:
        with self._lock:
            if not self._view_counts:
                return
            snapshot = dict(self._view_counts)
            self._view_counts.clear()

        count = 0
        for foo_id, increment in snapshot.items():
            try:
                # APP 스코프라 요청 세션이 없다. 스스로 세션을 열고 어댑터를 조립한다
                with self._session_factory() as session:
                    stats_repo = DefaultFooStatsRepository(session)
                    stats_repo.increase_view_count(FooIdentity(foo_id), increment)
                count += 1
            except Exception as e:  # noqa: BLE001  개별 row 격리
                logger.error("[FooViewMemory] flush failed: foo_id=%s, error=%s", foo_id, e)

        if count > 0:
            logger.info("[FooViewMemory] flushed: %s rows", count)
```

플러시 스케줄은 `app/lifespan.py` 에서 APScheduler `BackgroundScheduler` 로 `flush` 를 10초마다 호출한다 (`apscheduler` 를 app 패키지 dependencies 에 추가, 유저 확인 필요).

**특징**:
- APP 스코프 싱글턴 — 요청 세션을 쓸 수 없으므로 `session_factory` 를 받아 스스로 세션을 연다. **service 가 어댑터를 직접 import 하는 유일한 예외**이며 import-linter 계약을 어기므로 실제로는 `app` 패키지에 배치한다
- `threading.Lock` + `dict` — `def + Session` 스레드 모델에서 스레드 안전
- flush 중 새 데이터 유실 방지: lock 안에서 snapshot → clear
- 예외는 **개별 row 단위로 격리**
- 일반 도메인엔 쓰지 않음. 고부하 지표 계측이 필요할 때만

---

## 패턴 선택 가이드

| 도메인 특성 | Pattern |
|---|---|
| 조회만 (정적 / reference 데이터) | **Foo** — Reader 단독 |
| 기본 CRUD | **Bar** — Reader + Writer 쌍 |
| API 1개가 여러 도메인 + 부수효과 | **Baz** — Usecase (Bar 위에 얹음) |
| 집계/대시보드, 중간 Reader 불필요 | **Qux** — Usecase 단독 |
| 고빈도 이벤트 집계 (특수) | **Quux** — ViewMemory 류 |

조합 예:
- 일반 도메인: **Bar + Baz** (Reader/Writer 쌍 + Create/Update/Delete 각 Usecase)
- 단순 정적 데이터: **Foo** 만
- 대시보드: **Qux** 만
- 조회수 높은 도메인: **Bar + Baz + Quux**

라우터는 Writer 를 직접 주입하지 않고 Usecase 를 거친다 (api-references). 그래서 쓰기 API 가 있는 도메인은 항상 Baz Usecase 가 최소 하나 있다.

---

## 💡 팁 — 위 패턴들로 풀리지 않을 때

1. **중간 레이어 추가**를 우선 고민하지 않기 — Reader/Writer 로 충분하면 거기서 끝낸다. 과한 추상화는 유지 비용만 증가.
2. **Usecase 수가 급증**하면 도메인 경계 재검토 — 한 도메인의 Usecase 가 10개 이상이면 aggregate 를 쪼개야 할 신호.
3. **여러 서비스 호출 조합이 복잡**하면 Usecase 내부 private 메서드로 추출.
4. 그래도 표현이 어색하면 **유저에게 질의** — 이벤트 드리븐 / 비동기 파이프라인 / 스케줄러 도입 등 구조적 선택지.

---

## 파일 구성 요약

```
packages/service/src/{pkg}/service/{domain}/
├── __init__.py
├── {domain}_reader.py                  ← ABC (Pattern Foo/Bar)
├── {domain}_writer.py                  ← ABC (Pattern Bar)
├── dto/
│   ├── __init__.py
│   ├── {domain}_create_command.py      ← frozen dataclass
│   └── {domain}_update_command.py
├── impl/
│   ├── __init__.py
│   ├── default_{domain}_reader.py
│   └── default_{domain}_writer.py
└── usecase/                            ← Pattern Baz / Qux (선택)
    ├── __init__.py
    ├── {domain}_create_usecase.py
    ├── {domain}_update_usecase.py
    └── {domain}_delete_usecase.py

packages/app/src/{pkg}/app/providers/{domain}_provider.py   ← DI 등록 (add-usecase 가 생성)
```

---

## 공통 금지 사항

- Default 구현에서 `{pkg}.persistence` / `sqlalchemy` 직접 참조 금지 — 항상 port ABC (import-linter 가 잡는다)
- Reader 에서 mutation (save/delete 등) 호출 금지
- Reader 에 `@transactional` / `tx` 금지
- Writer/Usecase 에 `@transactional` 누락 금지 — 어댑터가 각자 커밋해 원자성이 깨진다
- Writer 의 조회 반환은 Write 모델 (`{Domain}`) — Read 모델(`{Domain}Read`) 반환은 Reader 의 책임
- Usecase 간 상호 호출은 지양. 공통 로직은 Writer 나 Repository 로 내리기 (Usecase 체이닝은 트랜잭션·의존 그래프 혼란)
- Command 에 `list` 필드 금지 (`tuple`)
- Provider 등록 누락 금지 — dishka `NoFactoryError` 로 라우터 첫 호출 시 실패한다
