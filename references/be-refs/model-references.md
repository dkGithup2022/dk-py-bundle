# Model References

`be-add-model` 스킬이 참고하는 도메인 모델 작성 레퍼런스.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 패키지)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

모든 모델은 **불변 값 객체(`frozen dataclass`) + 팩토리/전이 메서드** 형태다. 아래 4개 바리에이션과 공통 요소를 조합해 도메인 모델을 구성한다.

---

## 공통 요소

### AuditFields 구조 만족

모든 Write/Read 모델은 `created_at: datetime`, `updated_at: datetime` 필드를 보유한다 (be-init §4.4 / C-3). `{pkg}.model.audit.AuditFields`는 Protocol이라 명시 상속하지 않아도 두 필드가 있으면 만족한다. 시각은 `{pkg}.model.audit.utcnow()`로 만든다.

### id 필드 타입

- Write 모델: `{domain}_id: int | None` — 신규 생성 시 `None`, 저장 후 채워진다
- Read 모델: `{domain}_id: int` — 항상 영속화된 것만 Read 로 변환되므로 `None` 이 없다
- `to_read()` 첫 줄에서 `None` 을 걸러 `ValueError` 를 던진다. api Response 가 `id: int` 로 받을 수 있게 하는 장치

### Identity VO

단일 ID 래핑. 모든 도메인에 1개 존재.

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class FooIdentity:
    foo_id: int
```

**이유**: service/router 시그니처에서 `int` raw 타입보다 의미 명확. 여러 도메인의 ID가 메서드 파라미터에 섞일 때 타입 안전성.

### dataclass 규칙

- `@dataclass(frozen=True, slots=True)`: 불변 + 메모리 절약 + 오타 필드 대입 시 즉시 에러
- `__init__` 직접 선언 금지 — dataclass가 생성
- `__post_init__` 검증은 허용하되 최소로 (타입 힌트가 대신한다)
- 팩토리 외 생성자 직접 호출 금지 (내부 팩토리/전이 메서드 구현 제외). Python은 이를 언어로 막을 수 없으므로 **규칙으로** 지킨다
- 컬렉션 필드는 `tuple[T, ...]`. `list`는 가변이라 frozen 이 깨진다

### enum 필드

값 유한·고정이면 `StrEnum` (`company`, `category`, `type`, `role`, `source`).
자유 입력이면 `str` (`title`, `content`, `url`).

```python
from enum import StrEnum


class FooSource(StrEnum):
    COMPANY = "COMPANY"
    OFFICIAL = "OFFICIAL"
    SELF = "SELF"
```

멤버 값은 이름과 같게 둔다 (DB에 그대로 VARCHAR 저장, Pydantic 직렬화도 그대로). 표시 문자열 같은 추가 속성은 두지 않는다 — 프론트 책임.

### 팩토리 메서드 명명

- `create(...)`: 새 도메인 생성. `{domain}_id=None`, `created_at=now`, `updated_at=now`.
- `with_xxx(...)`: 필드 일부 변경. id/created_at 유지, `updated_at=now`. `dataclasses.replace` 사용.
- `to_read(...)`: Write → Read 변환. 필요한 JOIN 데이터를 파라미터로 받음.

반환 타입은 `typing.Self`.

### 하위 VO 분리 기준

3개 이상의 유사 의미 필드 묶음 → 별도 VO 로 추출. 같은 도메인 패키지 하위에 배치.

### 도메인 예외

도메인마다 `{Domain}NotFoundException`을 `exception` 패키지에 만든다 (be-init C-7). model 이 아니라 exception 패키지에 두지만, add-model 스킬이 도메인 모델과 짝으로 생성한다.

```python
# packages/exception/src/{pkg}/exception/foo/foo_not_found_exception.py
from {pkg}.exception import EntityNotFoundException


class FooNotFoundException(EntityNotFoundException):
    """기본 404 / "해당 리소스를 찾을 수 없습니다". 메시지를 바꾸려면 client_message 인자."""
```

---

## Pattern Foo — 단순 Write-only (Read 없음)

**언제 쓰나**: Upsert only 도메인, 혹은 조회 시 JOIN/집계가 필요 없는 단순 레코드 (평점, 반응 등).

```python
from dataclasses import dataclass, replace
from datetime import datetime
from typing import Self

from {pkg}.model.audit import utcnow

from .foo_source import FooSource


@dataclass(frozen=True, slots=True)
class Foo:
    foo_id: int | None
    user_id: int
    target_id: int
    source: FooSource
    score: int
    created_at: datetime
    updated_at: datetime

    @classmethod
    def create(cls, user_id: int, target_id: int, source: FooSource, score: int) -> Self:
        now = utcnow()
        return cls(None, user_id, target_id, source, score, now, now)

    def with_score(self, score: int) -> Self:
        return replace(self, score=score, updated_at=utcnow())
```

**특징**:
- Read 모델 없음 (`FooRead` 만들지 않음)
- 구성 파일: `foo.py`, `foo_identity.py`, `foo_source.py` 세 개

---

## Pattern Bar — Write + Read 분리 (단순 to_read)

**언제 쓰나**: 조회 결과에 민감 필드(내부 키, oauth 토큰 등) 제거나 구조 차이가 있는 기본 CRUD 도메인.

```python
@dataclass(frozen=True, slots=True)
class Bar:
    bar_id: int | None
    nickname: str
    email: str
    google_oauth_key: str  # 민감 — Read 에서 제외
    bio: str
    created_at: datetime
    updated_at: datetime

    @classmethod
    def create(cls, nickname: str, email: str, google_oauth_key: str, bio: str) -> Self:
        now = utcnow()
        return cls(None, nickname, email, google_oauth_key, bio, now, now)

    def with_profile(self, nickname: str, bio: str) -> Self:
        return replace(self, nickname=nickname, bio=bio, updated_at=utcnow())

    def to_read(self) -> BarRead:
        if self.bar_id is None:
            raise ValueError("bar must be persisted before to_read")
        return BarRead(self.bar_id, self.nickname, self.email, self.bio,
                       self.created_at, self.updated_at)
```

```python
@dataclass(frozen=True, slots=True)
class BarRead:
    bar_id: int
    nickname: str
    email: str
    bio: str            # google_oauth_key 없음
    created_at: datetime
    updated_at: datetime
```

**특징**:
- `to_read()` 가 **인자 없음** (self-contained)
- Read 가 Write 의 부분집합 (컬럼 추가 없이 프로젝션)
- `bar.py`가 `bar_read.py`를 import 한다 (단방향). `bar_read.py`는 `bar.py`를 모른다
- 구성 파일: `bar.py`, `bar_read.py`, `bar_identity.py`

---

## Pattern Baz — 하위 VO + Read 에 JOIN/집계 필드

**언제 쓰나**: 조회 시 다른 도메인 데이터(작성자명, 통계)나 자식 컬렉션을 함께 노출하는 핵심 aggregate.

### 하위 VO

```python
@dataclass(frozen=True, slots=True)
class BazLink:          # 여러 필드 묶음
    series_id: int
    name: str
```

### Write 모델

```python
@dataclass(frozen=True, slots=True)
class Baz:
    baz_id: int | None
    user_id: int
    type: BazType
    title: str
    body: str
    categories: tuple[BazCategory, ...]   # enum 튜플
    tags: tuple[BazTag, ...]
    source: BazSource
    created_at: datetime
    updated_at: datetime

    @classmethod
    def create(
        cls,
        user_id: int,
        type: BazType,
        title: str,
        body: str,
        categories: tuple[BazCategory, ...],
        tags: tuple[BazTag, ...],
        source: BazSource,
    ) -> Self:
        now = utcnow()
        return cls(None, user_id, type, title, body, categories, tags, source, now, now)

    def with_update(
        self,
        title: str,
        body: str,
        categories: tuple[BazCategory, ...],
        tags: tuple[BazTag, ...],
        source: BazSource,
    ) -> Self:
        return replace(self, title=title, body=body, categories=categories,
                       tags=tags, source=source, updated_at=utcnow())

    def to_read(
        self, author_nickname: str, stats: BazStats, links: tuple[BazLink, ...]
    ) -> BazRead:
        if self.baz_id is None:
            raise ValueError("baz must be persisted before to_read")
        rating = stats.rating_sum / stats.rating_count if stats.rating_count > 0 else 0.0
        return BazRead(
            self.baz_id, self.type, self.title, self.body, self.categories, self.tags,
            stats.view_count, author_nickname,
            rating, stats.rating_count, stats.comment_count,
            links, self.created_at, self.updated_at, self.source,
        )
```

### Read 모델 (JOIN/집계 필드 포함)

```python
@dataclass(frozen=True, slots=True)
class BazRead:
    baz_id: int
    type: BazType
    title: str
    body: str
    categories: tuple[BazCategory, ...]
    tags: tuple[BazTag, ...]
    views: int                    # ← 집계
    author: str                   # ← JOIN (nickname)
    rating: float                 # ← 집계 파생 (sum/count)
    reviews: int                  # ← 집계
    comments: int                 # ← 집계
    links: tuple[BazLink, ...]    # ← JOIN (자식 컬렉션)
    created_at: datetime
    updated_at: datetime
    source: BazSource
```

**특징**:
- `to_read()` 시그니처에 **의존 데이터를 파라미터로 명시** (어떤 JOIN 이 필요한지 모델이 선언)
- Read 에 user_id 같은 내부 키 대신 author 표시명
- 구성 파일: `baz.py`, `baz_read.py`, `baz_identity.py`, `baz_stats.py` (분리 집계), `baz_link.py` (하위 VO), `baz_type.py`/`baz_category.py`/`baz_tag.py`/`baz_source.py` (enum)

### 분리 Stats 모델

집계 컬럼은 빈번 업데이트로 쓰기 경쟁이 심하므로 본 테이블에서 **분리된 stats 테이블**로 관리. 모델도 별도.

```python
@dataclass(frozen=True, slots=True)
class BazStats:
    baz_id: int
    view_count: int
    rating_sum: int
    rating_count: int
    comment_count: int

    @classmethod
    def init(cls, baz_id: int) -> Self:
        return cls(baz_id, 0, 0, 0, 0)
```

**특징**:
- 감사 필드 없음 (audit 는 본 도메인에만)
- `init()` 팩토리로 0 초기화
- increment/decrement 같은 `with_xxx` 메서드는 필요 시 추가

---

## Pattern Qux — 상태 전이 + 트리 구조

**언제 쓰나**: 댓글·알림 같이 상태 플래그 전환(hide/show, read/unread)이나 부모-자식 관계가 있는 도메인.

```python
@dataclass(frozen=True, slots=True)
class Qux:
    qux_id: int | None
    article_id: int
    article_type: QuxArticleType
    user_id: int
    text: str
    parent_id: int | None     # None = 루트
    qux_order: int
    level: int                # 0 = 루트
    sort_number: int
    child_count: int
    is_hidden: bool           # 상태 플래그
    created_at: datetime
    updated_at: datetime

    @classmethod
    def create_root(
        cls, article_id: int, article_type: QuxArticleType, user_id: int, text: str, qux_order: int
    ) -> Self:
        now = utcnow()
        return cls(None, article_id, article_type, user_id, text, None,
                   qux_order, 0, 0, 0, False, now, now)

    @classmethod
    def create_reply(
        cls,
        article_id: int,
        article_type: QuxArticleType,
        user_id: int,
        text: str,
        parent_id: int,
        qux_order: int,
        level: int,
        sort_number: int,
    ) -> Self:
        now = utcnow()
        return cls(None, article_id, article_type, user_id, text, parent_id,
                   qux_order, level, sort_number, 0, False, now, now)

    def update_content(self, new_text: str) -> Self:
        return replace(self, text=new_text, updated_at=utcnow())

    def hide(self) -> Self:
        return replace(self, is_hidden=True, updated_at=utcnow())

    def show(self) -> Self:
        return replace(self, is_hidden=False, updated_at=utcnow())

    def to_read(self, author_nickname: str) -> QuxRead:
        if self.qux_id is None:
            raise ValueError("qux must be persisted before to_read")
        return QuxRead(
            self.qux_id, self.article_id, self.article_type, self.user_id, author_nickname,
            self.text, self.parent_id, self.qux_order, self.level, self.sort_number,
            self.child_count, self.is_hidden, self.created_at, self.updated_at,
        )
```

**특징**:
- `create_root()` vs `create_reply()` — **생성 시 분기** 를 팩토리 이름으로 드러냄
- `hide()` / `show()` — **상태 전이는 동사 메서드** (`with_is_hidden` 금지)
- soft-delete 미사용 (`is_hidden` 이 대체)
- 구성 파일: `qux.py`, `qux_read.py`, `qux_identity.py`, `qux_article_type.py` (enum)

---

## 패턴 선택 가이드

| 도메인 특성 | Pattern |
|---|---|
| Upsert 만 필요, 조회 시 추가 정보 없음 | Foo |
| 기본 CRUD, Read 는 Write 의 투영 | Bar |
| 조회 시 JOIN/집계 필요, 하위 VO 존재 | Baz |
| 상태 전이·트리·숨김 로직 있음 | Qux |

두 패턴이 섞이면 Baz 를 기본으로 하고 Qux 의 상태 전이 메서드를 추가하는 식으로 합성.

---

## 파일 구성 요약

```
packages/model/src/{pkg}/model/{domain}/
├── __init__.py                  ← 빈 파일
├── {domain}.py                  ← Write 모델
├── {domain}_read.py             ← Read 모델 (Bar/Baz/Qux 만)
├── {domain}_identity.py         ← Identity VO (필수)
├── {domain}_stats.py            ← 분리 통계 (Baz 만)
├── {name}_link.py / {name}.py   ← 하위 VO (필요 시)
└── {domain}_category.py
    {domain}_type.py
    ...                          ← StrEnum (필요 시)

packages/exception/src/{pkg}/exception/{domain}/
├── __init__.py
└── {domain}_not_found_exception.py
```

클래스당 파일 1개. `__init__.py`에서 re-export 하지 않는다 (import 경로가 파일을 그대로 가리키게 — 순환 import 예방).

---

## 공통 금지 사항

- `@dataclass` 에 `frozen=True` 누락 금지
- `list` 필드 금지 (항상 `tuple`)
- `__init__` 직접 선언 금지
- `NamedTuple` / `TypedDict` / Pydantic 모델을 도메인 모델로 사용 금지 (Pydantic 은 api DTO 전용)
- Read 모델에서 `with_xxx` / `to_write()` 금지 (단방향 투영)
- enum 에 `description` 같은 필드 과도 추가 금지 — 표시 문자열은 프론트 책임
- `datetime.now()` (naive) 금지 — 항상 `utcnow()`
- model 패키지에서 `sqlalchemy`, `pydantic`, `fastapi` import 금지 (import-linter 가 잡는다)
