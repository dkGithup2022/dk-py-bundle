# Infrastructure References

`be-add-infrastructure` 스킬이 참고하는 port (Repository ABC) 작성 레퍼런스.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 패키지)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

---

## 공통 요소

### 역할과 배치

- **port** (이 문서의 대상): service 레이어가 사용하는 **추상 계약**. 구현은 모른다.
- 파일 위치: `packages/infrastructure/src/{pkg}/infrastructure/{domain}/{domain}_repository.py`
- 구현체 위치: `packages/persistence/src/{pkg}/persistence/{domain}/{domain}_repository.py` 의 `Default{Domain}Repository` (be-add-query 가 생성)
- 트랜잭션 계약은 별도 패키지 `{pkg}.transaction` 에 있고 be-init 소유. 이 스킬은 건드리지 않는다.

### ABC 작성 규칙

- **`class {Domain}Repository(ABC)` + 모든 메서드 `@abstractmethod`** — 구현 본문 없음 (`...` 만), 일반 메서드·classmethod·staticmethod 없음
- Protocol 이 아니라 ABC 인 이유: 어댑터가 메서드를 빠뜨리면 인스턴스 생성 시점에 `TypeError` 로 즉시 실패한다. Java `implements` 처럼 구현 관계가 코드에 보인다
- docstring 한 줄로 각 메서드 의도 명시 (한국어) — 어댑터 구현자가 기대값을 알 수 있게
- 반환/파라미터 타입에 **`*Entity` 누설 금지** — 도메인 모델 (`{Domain}`, `{Domain}Read`, `{Domain}Identity`) 만 사용
- `sqlalchemy` import 금지 (import-linter 가 `infrastructure → persistence` 를 막고, 외부 라이브러리는 `pyproject.toml` dependencies 에 없어 import 자체가 실패한다)

### 메서드 네이밍

| 용도 | 네이밍 | 반환 |
|---|---|---|
| 단건 조회 (읽기 전용) | `find_by_id(...)` | `{Domain}Read \| None` (JOIN/집계 포함 가능) |
| 단건 조회 (쓰기 전 선검증) | `find_write_by_id(...)` | `{Domain} \| None` (순수 엔티티) |
| 리스트 조회 | `find_by_xxx(..., page: int, size: int)` | `list[{Domain}Read]` |
| 카운트 | `count_by_xxx(...)` | `int` |
| 존재 확인 | `exists_by_xxx(...)` | `bool` |
| 저장 (insert+update) | `save({domain})` | `{Domain}` (id 포함) |
| 삭제 | `delete_by_id({Domain}Identity)` | `None` — soft-delete, 이미 삭제면 no-op |
| 원자 증감 (Stats) | `increase_xxx` / `increment_xxx` / `add_xxx` | `None` 또는 갱신 결과 |

### 파라미터 관례

- **Identity VO 우선** — `int` 직접 받지 않는다 (`identity: SetupIdentity`, `user_identity: UserIdentity`)
- **원시 `int`** 는 page/size/user_id (인증 컨텍스트에서 받은 정수 키) 정도에서만 허용
- `ArticleType` 같은 StrEnum 은 그대로 (str 변환 금지)
- 반환 컬렉션은 `list` (호출자가 소비만 한다). 모델 **내부** 필드가 `tuple` 인 것과 구분

### Read 모델 vs Write 모델 분리 (Bar 패턴 핵심)

- `find_by_id` → **`{Domain}Read`** 반환 (JOIN·집계 포함, 화면용)
- `find_write_by_id` → **`{Domain}`** 반환 (순수 엔티티, update/delete 전 선검증 전용)
- 두 메서드를 같은 ABC 에 나란히 둔다

---

## Pattern Foo — 단순 CRUD port

**언제 쓰나**: JOIN/집계 불필요, 한 도메인 혼자 처리. 조회·저장·존재 확인만.

```python
from abc import ABC, abstractmethod

from {pkg}.model.foo.foo import Foo
from {pkg}.model.foo.foo_identity import FooIdentity


class FooRepository(ABC):
    @abstractmethod
    def find_by_id(self, identity: FooIdentity) -> Foo | None:
        """Foo 단건 조회."""

    @abstractmethod
    def find_by_email(self, email: str) -> Foo | None:
        """이메일로 Foo 조회."""

    @abstractmethod
    def exists_by_nickname(self, nickname: str) -> bool:
        """닉네임 중복 확인."""

    @abstractmethod
    def save(self, foo: Foo) -> Foo:
        """Foo 저장 (신규·수정)."""
```

**특징**:
- `find_by_id` 반환이 **Write 모델 `Foo`** — Read 모델 분리 가치 낮음 (JOIN 없음)
- `delete_by_id` 있으면 추가. Rating 처럼 upsert only 면 생략
- 메서드 3~5개가 보통

---

## Pattern Bar — Read/Write 분리 + 복수 조회 (주력 aggregate)

**언제 쓰나**: 조회 시 작성자명·통계·자식 컬렉션 JOIN 이 필요한 핵심 aggregate. 리스트/카운트/상세 조회·쓰기·삭제를 다 담당.

```python
class BarRepository(ABC):
    @abstractmethod
    def find_by_id(self, identity: BarIdentity) -> BarRead | None:
        """Bar 상세 조회. 작성자·통계·자식 컬렉션 포함."""

    @abstractmethod
    def find_write_by_id(self, identity: BarIdentity) -> Bar | None:
        """Bar Write 모델 조회. 수정/삭제 선검증 시 사용."""

    @abstractmethod
    def find_by_user_id(self, user_id: int, page: int, size: int) -> list[BarRead]:
        """Bar 목록 조회. 페이지 지원."""

    @abstractmethod
    def count_by_user_id(self, user_id: int) -> int:
        """Bar 목록 총 건수."""

    @abstractmethod
    def save(self, bar: Bar) -> Bar:
        """Bar 저장 (신규·수정)."""

    @abstractmethod
    def delete_by_id(self, identity: BarIdentity) -> None:
        """Bar 소프트 삭제. 이미 삭제된 건 no-op."""

    @abstractmethod
    def find_deleted_by_id(self, identity: BarIdentity) -> Bar | None:
        """소프트 삭제된 Bar 조회. ES 삭제 동기화 등 특수 용도."""

    @abstractmethod
    def find_existing_ids(self, ids: list[int]) -> set[int]:
        """주어진 ID 목록 중 존재(삭제되지 않음) 하는 ID 집합. 배치 존재 확인용."""
```

**특징**:
- **핵심 관례: `find_by_id` (Read) + `find_write_by_id` (Write) 병존** — 서비스가 용도로 선택
- `find_all` 은 보통 없음 (화면 요구사항이 필터·정렬을 동반하므로 `find_by_user_id` 같이 구체적 이름)
- `delete_by_id` 반환 `None` — soft-delete. 404 의미는 Usecase 에서 선검증으로 막음
- `find_deleted_by_id` 는 검색/알림/ES 동기화 같은 **특수 용도** 가 있을 때만 추가
- `find_existing_ids` 는 배치 작업·outbox 등에서 존재 확인 최적화용, 필요 없으면 생략

---

## Pattern Baz — Stats 집계 전용 port (본 aggregate 와 **별도**)

**언제 쓰나**: 통계 테이블의 카운터 원자적 증감. 쓰기 경쟁 격리 목적.

```python
class BarStatsRepository(ABC):
    @abstractmethod
    def find_by_bar_id(self, identity: BarIdentity) -> BarStats | None:
        """Bar 통계 조회."""

    @abstractmethod
    def save(self, stats: BarStats) -> BarStats:
        """통계 저장 (신규). Bar 생성 시 init() 으로."""

    @abstractmethod
    def increase_view_count(self, identity: BarIdentity, increment: int) -> None:
        """조회수 원자적 증가."""

    @abstractmethod
    def add_rating(self, identity: BarIdentity, score: int) -> BarStats:
        """평점 합계/건수 원자적 갱신 (평가 추가 시). 갱신된 통계 반환."""

    @abstractmethod
    def update_rating(self, identity: BarIdentity, old_score: int, new_score: int) -> BarStats:
        """평점 합계/건수 원자적 갱신 (평가 수정 시). 갱신된 통계 반환."""

    @abstractmethod
    def increment_comment_count(self, identity: BarIdentity) -> None:
        """댓글 수 원자적 증가."""
```

**특징**:
- **본 aggregate port 와 별도 파일** (`bar_repository.py` 와 `bar_stats_repository.py` 분리)
- 증감 메서드는 **도메인 의도로 네이밍** — `increase_view_count`, `add_rating` 같이
- 반환: 값 필요 없으면 `None`, 갱신된 상태가 필요하면 `BarStats`

---

## Pattern Qux — 트리·순서 조작 (Bar 에 병합)

**언제 쓰나**: 댓글·메뉴·카테고리 같은 트리 구조. 정렬·자식 카운트 조정이 필요.

**별도 ABC 로 분리하지 않고 Bar 에 병합**:

```python
class QuxRepository(ABC):
    # --- Bar 기본 (find_by_id / find_write_by_id / save / delete_by_id ...) ---
    @abstractmethod
    def find_by_id(self, identity: QuxIdentity) -> QuxRead | None: ...
    @abstractmethod
    def find_write_by_id(self, identity: QuxIdentity) -> Qux | None: ...
    @abstractmethod
    def save(self, qux: Qux) -> Qux: ...

    # --- 트리·순서 조작 ---
    @abstractmethod
    def get_max_comment_order(self, article_id: int, article_type: ArticleType) -> int:
        """아티클의 최대 comment_order 조회. 다음 댓글 순서 결정용. 없으면 0."""

    @abstractmethod
    def increment_sort_numbers_above(
        self, article_id: int, article_type: ArticleType, comment_order: int, sort_number: int
    ) -> None:
        """같은 comment_order 그룹에서 부모의 sort_number 이후 값들을 +1 밀어내기."""

    @abstractmethod
    def increment_child_count(self, parent_id: QuxIdentity) -> None:
        """부모 댓글의 child_count 를 1 증가."""
```

**특징**:
- **Bar 와 별도 분리하지 않는다** — 트리 연산이 같은 테이블에 작용하므로 한 port 에서 관리
- 트리 연산은 호출자(Usecase) 의 `@transactional` 경계에서 보호

---

## 패턴 선택 가이드

| 도메인 특성 | Pattern |
|---|---|
| 단순 CRUD, JOIN 불필요 | **Foo** |
| Read 모델에 JOIN/집계 필요, 리스트·카운트 조회 | **Bar** (주력) |
| 통계 카운터 증감 전용 | **Baz** (본 aggregate 의 **사이드** port, 파일 별도) |
| 트리·순서 연산 | **Qux** (Bar 에 병합) |

조합 예:
- Rating 도메인: **Foo**
- Setup 도메인: **Bar** (`SetupRepository`) + **Baz** (`SetupStatsRepository`) — 파일 2개
- Comment 도메인: **Bar + Qux** 병합 (`CommentRepository` 한 파일)
- User 도메인: **Foo** (`UserRepository`) + **Baz** (`UserStatsRepository`)

---

## 💡 팁 — 위 패턴들로 풀리지 않을 때

1. **별도 port 파일 분리**를 우선 고민하지 않기 — Bar 나 Qux 처럼 한 파일에 병합이 낫다. 파일이 커질 때 분리.
2. **새 도메인 등장**이면 보통 Foo 로 시작 — 필드가 단순하고 JOIN 필요 없으면.
3. **JOIN/집계가 생기면** Bar 로 승격 — `find_by_id` 반환을 `Read` 로 바꾸고 `find_write_by_id` 추가.
4. **쓰기 경쟁이 보이면** Baz 로 통계 분리 — 조회수·평점·댓글수가 별 테이블이 낫다.
5. 그래도 표현이 어색하면 **유저에게 질의** — 도메인 경계 재설계 / aggregate 분할 / 이벤트 기반 컴포즈 등 선택지.

---

## 파일 구성 요약

```
packages/infrastructure/src/{pkg}/infrastructure/{domain}/
├── __init__.py                       ← 빈 파일
├── {domain}_repository.py            ← Pattern Foo / Bar / Qux
└── {domain}_stats_repository.py      ← Pattern Baz (Bar 와 별도)
```

---

## 공통 금지 사항

- 데코레이터·메타클래스 추가 금지 (`ABC` + `@abstractmethod` 만)
- 구현 본문 금지 — `...` 또는 docstring 만. 기본 구현은 어댑터 전담
- `{pkg}.persistence` / `sqlalchemy` import 금지 — port 는 구현을 모른다
- `Select`, `Session`, `Row` 같은 SQLAlchemy 타입 노출 금지 — `page: int, size: int` 로 받아 내부 변환
- `raises` 문서화 외 예외 발생 금지 — port 는 던지지 않는다. 예외는 어댑터·service 에서
- `None` 대신 빈 컬렉션 — `list[...]` 반환 메서드는 결과 없을 때 `[]`
