---
name: be-add-usecase
description: service 패키지에 Reader/Writer 또는 Usecase를 생성/변경하고, app 패키지에 dishka Provider를 등록한다.
---

# add-usecase

service 패키지에 Reader/Writer 또는 Usecase 를 생성/변경한다. 생성한 클래스를 dishka Provider 에 등록한다.

## 입력

### 지시
- 기본적으로 도메인의 Reader/Writer 를 생성한다.
- 쓰기 API 가 있으면 create/update/delete Usecase 도 생성한다 (라우터가 Writer 를 직접 쓰지 않으므로).
- 단순 읽기/쓰기를 벗어나는 동작(여러 도메인 조합, 부수효과)은 Usecase 로 분리한다.

### Phase 0. 프로젝트 컨텍스트 해석
`{pkg}`, `{packages.service}`, `{packages.app}` 플레이스홀더 사용. 실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.

### 참고 문서 (읽어올 것)
- `${CLAUDE_PLUGIN_ROOT}/references/be-refs/service-references.md` — 5 패턴 (Foo/Bar/Baz/Qux/Quux) + 공통 요소 + DI 등록 + 선택 가이드
- specs/usecase.md — 기존 Reader/Writer 정보 (프로젝트 루트 기준, 없으면 소스에서 직접 추론)
- specs/infrastructure.md — 의존할 Repository ABC (프로젝트 루트 기준)
- specs/models.md — 도메인 모델 (반환 타입, Command 필드 참조) (프로젝트 루트 기준)

## 패턴 선택 (먼저 결정)

service-references.md §"패턴 선택 가이드" 를 적용해 도메인 특성에 맞는 패턴을 먼저 고른다:

| 도메인 특성 | Pattern |
|---|---|
| 조회만 (정적 / reference 데이터) | **Foo** — Reader 단독 |
| 기본 CRUD | **Bar** — Reader + Writer 쌍 |
| API 단위 여러 도메인 + 부수효과 | **Baz** — Usecase (Bar 위에) |
| 집계/대시보드, 중간 Reader 불필요 | **Qux** — Usecase 단독 |
| 고빈도 이벤트 집계 (특수) | **Quux** — ViewMemory |

대부분 **Bar + Baz** 조합. 쓰기 API 가 있으면 Baz 는 필수다.

## 생성 결과물 구조

### 기본 (Reader/Writer)

```
{packages.service}/src/{pkg}/service/{domain}/
├── __init__.py
├── {domain}_reader.py            ← ABC
├── {domain}_writer.py            ← ABC
├── impl/
│   ├── __init__.py
│   ├── default_{domain}_reader.py
│   └── default_{domain}_writer.py
└── dto/
    ├── __init__.py
    └── {domain}_{action}_command.py   ← 필요 시
```

### Usecase (단순 Reader/Writer 를 벗어나는 동작, 또는 쓰기 API 진입점)

```
{packages.service}/src/{pkg}/service/{domain}/usecase/
├── __init__.py
└── {domain}_{action}_usecase.py
```

### DI 등록 (app 패키지, be-init C-10)

```
{packages.app}/src/{pkg}/app/providers/{domain}_provider.py   ← 신규
{packages.app}/src/{pkg}/app/di.py                            ← DOMAIN_PROVIDERS 에 한 줄 추가
```

## 기본 원칙

1. **최소 스펙**으로 작업한다.
2. 도메인은 **Reader + Writer**를 기본으로 가진다.
3. 쓰기 API 가 있는 도메인은 **create/update/delete Usecase** 를 만든다. 라우터는 Writer 를 직접 주입하지 않는다.
4. 여러 도메인 조합·부수효과 같은 Usecase 동작은 **유저에게 계획을 보이고** 진행한다.
5. 기존 동작의 변형, Command 스펙 변형은 **유저에게 허락**을 받고 진행한다.

## 생성 규칙

### 1. Reader (조회)

**ABC:**

```python
class {Domain}Reader(ABC):
    @abstractmethod
    def get_by_id(self, identity: {Domain}Identity) -> {Domain}Read: ...

    @abstractmethod
    def get_by_user_id(self, user_id: int, page: int, size: int) -> list[{Domain}Read]: ...
```

**구현체:**

```python
logger = logging.getLogger(__name__)


class Default{Domain}Reader({Domain}Reader):
    def __init__(self, repository: {Domain}Repository) -> None:
        self._repository = repository

    def get_by_id(self, identity: {Domain}Identity) -> {Domain}Read:
        read = self._repository.find_by_id(identity)
        if read is None:
            raise {Domain}NotFoundException(f"{domain} not found: {identity.{domain}_id}")
        return read
```

- 데코레이터 없음. 생성자 주입만
- infrastructure Repository ABC 만 의존
- 조회 실패 시 도메인 예외 raise
- **`tx` 없음, `@transactional` 없음**

### 2. Writer (명령)

**ABC:**

```python
class {Domain}Writer(ABC):
    @abstractmethod
    def create(self, command: {Domain}CreateCommand) -> {Domain}Identity: ...

    @abstractmethod
    def update(self, command: {Domain}UpdateCommand) -> {Domain}: ...

    @abstractmethod
    def delete(self, identity: {Domain}Identity) -> None: ...
```

**구현체:**

```python
from {pkg}.transaction import TransactionManager, transactional


class Default{Domain}Writer({Domain}Writer):
    def __init__(self, tx: TransactionManager, repository: {Domain}Repository) -> None:
        self.tx = tx
        self._repository = repository

    @transactional
    def create(self, command: {Domain}CreateCommand) -> {Domain}Identity:
        domain = {Domain}.create(command.request_user_id, command.title, ...)
        saved = self._repository.save(domain)
        assert saved.{domain}_id is not None
        return {Domain}Identity(saved.{domain}_id)
```

- 생성자 첫 파라미터 `tx: TransactionManager`, `self.tx` 에 설정 (be-init C-9)
- **모든 public 메서드에 `@transactional`**
- update/delete 는 `find_write_by_id` 선검증으로 404 확보
- `create` → Identity, `update` → Write 모델, `delete` → `None`. Read 반환 금지
- 크로스 도메인 Repository 주입 가능 (카운터 증감 등)

### 3. Command

```python
@dataclass(frozen=True, slots=True)
class {Domain}{Action}Command:
    request_user_id: int
    title: str
    categories: tuple[{Domain}Category, ...]
```

- **frozen dataclass** (불변, 간결)
- `request_user_id` 를 첫 필드로 명시
- 컬렉션은 `tuple`. 라우터가 Request 의 list 를 `tuple(...)` 로 변환해 넣는다
- `dto/` 하위에 위치

### 4. Usecase (쓰기 API 진입점 / Reader/Writer 를 벗어나는 동작)

여러 도메인을 조율하거나, 단순 CRUD 가 아닌 비즈니스 흐름이거나, 라우터가 호출할 쓰기 진입점인 경우.

**부수효과·크로스 도메인이 있으면 유저에게 계획을 보일 것:**
- 어떤 동작인지
- 어떤 부수효과가 트랜잭션 안에 들어가는지
- as-is/to-be (기존 변형인 경우)

```python
class {Domain}{Action}Usecase:
    def __init__(
        self,
        tx: TransactionManager,
        {domain}_writer: {Domain}Writer,
        # 필요한 Repository / Reader
    ) -> None:
        self.tx = tx
        self._{domain}_writer = {domain}_writer

    @transactional
    def execute(self, user_id: int, command: {Domain}{Action}Command) -> {Domain}Identity:
        # 선검증 → Writer → 부수효과
        ...
```

- ABC 없음, 클래스 직접 노출
- `execute` 1개, `@transactional`. 안에서 부르는 Writer/Repository 는 합류
- 소유권 체크는 Usecase 에서 (`existing.user_id != user_id → ForbiddenException`)
- 읽기 전용 Usecase (Qux) 는 `tx` 와 `@transactional` 생략 가능

### 5. Provider 등록

`{packages.app}/src/{pkg}/app/providers/{domain}_provider.py`:

```python
from dishka import Provider, Scope, provide


class {Domain}Provider(Provider):
    scope = Scope.REQUEST

    entity_query = provide({Domain}EntityQuery)
    join_query = provide({Domain}JoinQuery)                # JoinQuery 파일이 있을 때만
    repository = provide(Default{Domain}Repository, provides={Domain}Repository)
    # Stats 가 있으면:
    # stats_entity_query = provide({Domain}StatsEntityQuery)
    # stats_repository = provide(Default{Domain}StatsRepository, provides={Domain}StatsRepository)
    reader = provide(Default{Domain}Reader, provides={Domain}Reader)
    writer = provide(Default{Domain}Writer, provides={Domain}Writer)
    create_usecase = provide({Domain}CreateUsecase)
    update_usecase = provide({Domain}UpdateUsecase)
    delete_usecase = provide({Domain}DeleteUsecase)
```

`{packages.app}/src/{pkg}/app/di.py` 의 `DOMAIN_PROVIDERS` 리스트에 `{Domain}Provider()` 추가 + import.

- persistence 의 구현체·EntityQuery·JoinQuery 바인딩도 여기서 한다 (add-query 는 클래스만 만든다). 구현체 생성자가 받는 쿼리 클래스는 전부 `provide(...)` 로 등록해야 dishka 가 주입한다
- 생성자에 `Session`, `TransactionManager` 가 있으면 be-init `RequestProvider` 가 자동 주입
- Usecase 가 다른 도메인의 Repository 를 받으면 그 도메인 Provider 가 이미 등록되어 있어야 한다. 없으면 이 Provider 에 함께 바인딩

## 규칙

- Writer 는 같은 도메인 Reader 를 주입할 수 있다
- 다른 도메인 Reader/Writer 주입은 유저에게 문의 (Repository 주입은 허용)
- 비즈니스 로직은 Writer 또는 Usecase 에만 둔다
- Reader 는 변환과 위임만
- `{pkg}.persistence` import 금지 (import-linter). Provider 파일은 app 패키지라 허용

## 기존 변경 시

- as-is/to-be 를 유저에게 제시하고 확인받는다
- ABC 에 메서드 추가 시 Default 구현도 함께 (누락 시 인스턴스 생성 실패)
- 시그니처 변경, Command 스펙 변경은 유저 확인 필수
- 기존 메서드와 유사한 요청이 오면 병합 여부를 유저에게 문의

## 검증

```bash
uv run ruff check {packages.service} {packages.app} && uv run lint-imports && uv run mypy && \
uv run python -c "import {pkg}.app.main as m; m.create_app()"
```

마지막 명령이 dishka 컨테이너 조립까지 검증한다. Provider 의 `provides` 타입과 생성자 타입 힌트가 안 맞으면 여기서 `NoFactoryError` 가 난다 (지연 생성이라 라우터 첫 호출 시 나는 경우도 있으므로, 통합 테스트에서 재확인).

## 산출물

- 생성/변경된 Reader, Writer, Usecase, Command, Provider 파일 목록
- `di.py` 에 추가된 줄
- `specs/usecase.md` 업데이트 (없으면 `specs/` 와 함께 생성) — ABC 시그니처와 Command 스펙, 각 Usecase 의 `execute` 시그니처와 부수효과를 반드시 기록. api 레이어가 이 문서만 읽고 의존한다
- 추가/변경된 스펙 정리본 출력
