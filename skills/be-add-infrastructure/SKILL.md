---
name: be-add-infrastructure
description: infrastructure 패키지에 Repository port(ABC)를 생성/변경한다. assess-infra의 산출물을 기반으로 작업한다.
---

# add-infrastructure

infrastructure 패키지에 Repository ABC 를 생성하거나 변경한다.

## 입력

- assess-infra 산출물 (작업 계획)
- specs/infrastructure.md (기존 Repository ABC) — 프로젝트 루트 기준 (없으면 소스에서 추론)
- specs/models.md (도메인 모델 — 반환 타입 참조) — 프로젝트 루트 기준

## Phase 0. 프로젝트 컨텍스트 해석

`{pkg}`, `{packages.infrastructure}` 플레이스홀더 사용. 실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.

## 참고 문서 (읽어올 것)

- `${CLAUDE_PLUGIN_ROOT}/references/be-refs/infrastructure-references.md` — 4 패턴 (Foo/Bar/Baz/Qux) + 공통 요소 + 선택 가이드

## 패턴 선택 (먼저 결정)

infrastructure-references.md §"패턴 선택 가이드" 로 도메인 특성에 맞는 패턴을 먼저 고른다:

| 도메인 특성 | Pattern |
|---|---|
| 단순 CRUD, JOIN 불필요 | **Foo** |
| Read 모델에 JOIN/집계 필요, 리스트·카운트 | **Bar** (주력) |
| 통계 카운터 증감 | **Baz** (별도 파일: `{domain}_stats_repository.py`) |
| 트리·순서 연산 | **Qux** (Bar 에 병합) |

조합: Setup = Bar + Baz(파일 2개), Comment = Bar + Qux(한 파일), Rating = Foo.

## 생성 결과물 구조

```
{packages.infrastructure}/src/{pkg}/infrastructure/{domain}/
├── __init__.py                       ← 빈 파일
├── {domain}_repository.py            ← Pattern Foo / Bar / Qux
└── {domain}_stats_repository.py      ← Pattern Baz (필요 시)
```

## 생성 규칙

### 1. ABC 만 정의
- `class {Domain}Repository(ABC)`, 모든 메서드 `@abstractmethod`, 본문은 docstring 만.
- 구현 없음. 구현은 persistence 의 영역.

### 2. 반환 타입 원칙
- 읽기 메서드 → `{Domain}Read | None` / `list[{Domain}Read]`
- 쓰기 메서드 → `{Domain}`
- 삭제/원자적 연산 → `None`
- 결과 없음은 `None`(단건) 또는 `[]`(목록). `Optional` 표기 대신 `| None`

### 3. 파라미터 원칙
- ID 조회 → `{Domain}Identity` 사용 (`int` 직접 사용 지양)
- 벌크 조회 → `list[{Domain}Identity]` 또는 `list[int]`
- page/size, 인증 user_id 는 `int` 허용

### 4. 원자적 연산은 별도 메서드
- `increase_view_count`, `increase_like_count` 등
- 도메인 객체 전체를 save 하지 않고 특정 컬럼만 업데이트하는 경우

### 5. 의존
- `{pkg}.model` 과 `{pkg}.exception` 만 import
- `sqlalchemy`, `{pkg}.persistence` import 금지 (import-linter 검증)
- 트랜잭션 계약(`{pkg}.transaction`)은 be-init 소유. 수정 금지

### 6. 한국어 docstring
- ABC 는 다른 레이어의 설명서 역할을 한다
- 각 메서드에 한국어 docstring 한 줄
- 목적: 이 메서드가 무엇을 하는지
- 특이사항: 시간복잡도가 높은 경우, 부수효과가 있는 경우, "이미 삭제면 no-op" 같은 계약 명시

```python
class {Domain}Repository(ABC):
    @abstractmethod
    def find_by_id(self, identity: {Domain}Identity) -> {Domain}Read | None:
        """게시글 단건 조회. 유저 닉네임 포함."""

    @abstractmethod
    def search_tags(self, keyword: str) -> list[str]:
        """키워드로 태그 검색. prefix 매칭. LIKE 검색으로 데이터 많을 시 느려질 수 있음."""
```

## 기존 ABC 변경 시

- as-is/to-be를 유저에게 제시하고 확인받는다
- 메서드 추가는 하위 호환이지만, **기존 어댑터가 새 abstractmethod 를 구현하지 않으면 인스턴스 생성이 실패한다**. 추가 후 add-query 가 어댑터를 반드시 갱신해야 함을 산출물에 명시
- 시그니처 변경은 유저 확인 필수
- 기존 메서드와 기능적으로 유사한 요청이 있는 경우, 병합 여부를 유저에게 문의

## 검증

```bash
uv run ruff check {packages.infrastructure} && uv run lint-imports && uv run mypy
```

`lint-imports` 가 `infrastructure → persistence` 나 외부 라이브러리 import 를 잡는다.

## 산출물

- 생성/변경된 ABC 파일 목록
- `specs/infrastructure.md` 업데이트 (없으면 `specs/` 와 함께 생성). 메서드 시그니처와 docstring 을 그대로 기록 — usecase 레이어가 이 문서만 읽고 의존한다
- 추가/변경된 스펙 정리본 출력
- 기존 ABC 에 메서드를 추가한 경우: "어댑터 갱신 필요" 항목을 명시
