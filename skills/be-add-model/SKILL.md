---
name: be-add-model
description: model 패키지에 도메인 모델(frozen dataclass)과 exception 패키지에 도메인 예외를 생성/변경한다. assess-model의 산출물을 기반으로 작업한다.
---

# add-model

model 패키지에 도메인 모델을 생성하거나 변경한다. 도메인 예외(`{Domain}NotFoundException`)를 짝으로 생성한다.

## 입력

- assess-model 산출물 (작업 계획)
- specs/models.md (기존 도메인 모델) — 프로젝트 루트 기준 (없으면 소스에서 직접 추론)

## Phase 0. 프로젝트 컨텍스트 해석

`{pkg}`, `{packages.model}`, `{packages.exception}` 플레이스홀더 사용. 실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.

## 참고 문서 (읽어올 것)

- `${CLAUDE_PLUGIN_ROOT}/references/be-refs/model-references.md` — 4가지 패턴(Foo/Bar/Baz/Qux) 전체 예시 + 공통 요소 + 선택 가이드

## 패턴 선택 (먼저 결정)

model-references.md §"패턴 선택 가이드" 를 적용해 도메인 특성에 맞는 패턴을 먼저 고른다:

| 도메인 특성 | Pattern |
|---|---|
| Upsert 만 필요, 조회 시 추가 정보 없음 | **Foo** — Write-only, Read 생성 안 함 |
| 기본 CRUD, Read 는 Write 의 투영 | **Bar** — Write+Read, 단순 `to_read()` |
| 조회 시 JOIN/집계 필요, 하위 VO 존재 | **Baz** — Write+Read+Stats+하위 VO |
| 상태 전이·트리·숨김 로직 있음 | **Qux** — 상태 전이 메서드 |

## 생성 결과물 구조

선택한 패턴에 따라 아래 중 필요한 파일만 생성:

```
{packages.model}/src/{pkg}/model/{domain}/
├── __init__.py                 ← 빈 파일
├── {domain}.py                 ← Write 모델 (필수)
├── {domain}_read.py            ← Read 모델 (Bar/Baz/Qux 만)
├── {domain}_identity.py        ← Identity VO (필수)
├── {domain}_stats.py           ← 분리 통계 (Baz 만)
├── {name}.py                   ← 하위 VO (Baz 등 필요 시)
└── {domain}_category.py        ← StrEnum (필요 시)
    {domain}_type.py
    ...

{packages.exception}/src/{pkg}/exception/{domain}/
├── __init__.py
└── {domain}_not_found_exception.py   ← 필수 (Reader 가 던진다)
```

## 생성 규칙

### 1. 불변 객체
- `@dataclass(frozen=True, slots=True)`. `__init__` 직접 선언 없음.

### 2. Identity 분리
- 도메인마다 `{Domain}Identity` 별도 생성
- 필드명: `{domain}_id: int`

### 3. 상태 변경은 새 인스턴스 반환
- `dataclasses.replace(self, ..., updated_at=utcnow())`
- 반환 타입 `Self`

### 4. 하위 VO 분리
- 유사한 필드의 모임(3개 이상)은 별도 VO로 추출
- 도메인 패키지 하위에 위치

### 5. 감사 필드
- Write/Read 모델 모두 `created_at: datetime`, `updated_at: datetime` 필드를 반드시 포함 — `AuditFields` Protocol 구조 만족 (be-init §4.4 / C-3)
- 시각은 `{pkg}.model.audit.utcnow()` 로만 만든다. `datetime.now()` 금지
- 값은 Entity 레이어의 `default=utcnow` / `onupdate=utcnow` 가 채운 후 `to_domain` 으로 전달됨

### 6. str vs StrEnum 검증
- 값이 유한하고 고정적 → `StrEnum` (company, category, status, role)
- 자유 입력 → `str` (title, content, url)
- 애매하면 → 유저에게 문의

### 7. 컬렉션 필드는 tuple
- `tuple[T, ...]`. `list` 는 frozen 을 깨므로 금지
- Command 나 Request 에서 list 로 들어오면 팩토리 호출 전에 `tuple(...)` 로 변환하는 것은 호출자(service) 책임

### 8. Read 모델 — 패턴에 따라
- **Pattern Foo**: Read 생성 금지 (Write-only 도메인)
- **Pattern Bar/Baz/Qux**: Read 별도 생성
- JOIN/집계 필드 없는 Bar 도 Read 를 별도로 둬 투영(민감 필드 제거, 타입 안전성)의 기회를 유지

### 9. to_read 변환 메서드
- Bar/Baz/Qux 의 Write 모델에 `to_read(...)` 제공
- Bar: 인자 없음 (self-contained projection)
- Baz: 의존 데이터를 파라미터로 명시 (author_nickname, stats, links)
- 도메인 변경 시 `to_read` 시그니처도 함께 변경
- `{domain}.py` → `{domain}_read.py` 단방향 import. 역방향 금지

### 10. soft delete 필드 — 모델 레이어 제외
- 도메인이 soft-delete 를 **사용하더라도** `is_deleted`/`deleted_at` 은 모델에 포함하지 않는다.
- Entity/DB 레이어에서만 관리 (be-init C-4: 모든 테이블 보유, 사용만 도메인별).
- 모델에서 노출되는 "삭제" 개념은 조회 결과에 포함되지 않는 것으로 충분.

### 11. 도메인 예외
- `{Domain}NotFoundException(EntityNotFoundException)` 을 `{packages.exception}/src/{pkg}/exception/{domain}/` 에 생성 (be-init C-7)
- 기본 404 / "해당 리소스를 찾을 수 없습니다". 도메인 특화 메시지가 필요하면 `client_message` 기본값을 오버라이드
- 그 외 도메인 특화 예외(예: `WrongCommentException`)는 assess 산출물에 명시된 경우에만 추가

## 기존 도메인 변경 시

- as-is/to-be를 유저에게 제시하고 확인받는다
- 필드 추가 시 기존 팩토리 메서드 시그니처 변경 여부 확인
- Write 변경 시 Read + `to_read` 도 함께 변경
- 필드 순서 변경은 금지 (dataclass 위치 인자로 호출하는 코드가 깨진다). 새 필드는 끝에 추가하거나 `updated_at` 앞에 두고 호출부를 전수 확인

## 검증

```bash
uv run ruff check {packages.model} {packages.exception} && uv run mypy
```

실패 시 수정 후 재실행. `ruff --fix` 로 import 정렬은 자동 수정 가능.

## 산출물

- 생성/변경된 모델·예외 파일 목록
- `specs/models.md` 업데이트 (없으면 `specs/` 와 함께 생성). 각 도메인의 클래스·필드·팩토리·전이 메서드·enum 값을 기록
- 추가/변경된 스펙 정리본 출력
