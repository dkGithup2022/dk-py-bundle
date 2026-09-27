---
name: be-test-service
description: service 패키지의 단위 테스트(unittest.mock + NoopTransactionManager)를 평가하고 작성한다.
---

# be-test-service

service 패키지의 단위 테스트를 3단계로 수행한다.

---

## 입력

```
대상 서비스 클래스 이름 (예: SetupCreateUsecase, DefaultSetupReader)
```

입력이 없으면 "어떤 클래스를 테스트할까요?" 로 확인한다.

---

## Phase 0. 프로젝트 컨텍스트 해석

`{pkg}`, `{packages.service}` 플레이스홀더 사용. 실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.

---

## Phase 1. assess-infra

`${CLAUDE_PLUGIN_ROOT}/references/be-test-refs/infra-checklist.md` §공통 + §service(단위) 참조.

| 항목 | 경로 | 확인 |
|------|------|------|
| pytest | 루트 `pyproject.toml` dev 그룹 | 있는지 |
| tests 디렉터리 | `{packages.service}/tests/` | 있는지 |
| NoopTransactionManager | `{pkg}.transaction` | 있는지 (be-init 이 만든다) |

있으면 PASS, 없으면 추가.

**불필요**: conftest, DB, Docker, dishka — 컨텍스트 없음.

---

## Phase 2. assess-scope

### 2-1. 유형 판별

```
클래스 이름?
├─ *Usecase        → USECASE  (tx 있음, execute 1개)
├─ Default*Reader  → READER   (tx 없음)
└─ Default*Writer  → WRITER   (tx 있음)
```

### 2-2. 의존성 추출 + specs lookup

1. 대상 클래스 생성자 파라미터 전부 → `Mock(spec=...)` 후보. `tx: TransactionManager` 는 `NoopTransactionManager()`
2. 각 의존성의 **API 시그니처**를 specs에서 읽는다:
   - `specs/usecase.md` — Reader/Writer ABC 메서드 시그니처 + 동작 설명
   - `specs/infrastructure.md` — Repository ABC 메서드 시그니처
   - `specs/models.md` — 도메인 팩토리, 상태 변경 메서드, 반환 타입
3. **실제 ABC 소스**도 Read로 확인한다 (specs와 실제 반환 타입이 다를 수 있음)

### 2-3. 테스트 케이스 도출

대상 클래스의 각 public 메서드에 대해, **의존 API의 반환 타입**으로 분기를 결정한다.

#### 분기 판단 규칙

| 의존 API 반환 타입 | 분기 | TC |
|-------------------|------|-----|
| `X \| None` | 값 / None | 정상 처리 + 예외(NotFoundException 등) |
| `bool` | True / False | 각 분기의 후속 동작 |
| `int \| None` | 값 / None | 각 분기의 후속 동작 |
| `list[X]` | 비어있지 않음 / 빈 리스트 | (Reader에서만 — 양쪽 TC) |
| `int` / `None` | 단일 | 위임 결과 반환 or assert_called |

#### 추가 TC 규칙

| 코드 패턴 | TC 추가 |
|----------|---------|
| `existing.user_id != user_id` | 소유권 불일치 → ForbiddenException |
| `if/else` 분기 | 분기별 TC 각각 |
| 부수효과 (stats 갱신 등) | happy path에서 `assert_called_once_with` 포함 |
| 예외 경로 후 위임 호출 | `assert_not_called()` 포함 |
| Writer / Usecase | `@transactional` 부착 확인 TC 1개 (service-patterns.md §R) |

#### 예시: SetupCreateUsecase

```
specs lookup:
  UserRepository.find_by_id(UserIdentity) -> User | None     ... 값/None 분기
  SetupWriter.create(SetupCreateCommand) -> SetupIdentity      ... 단일 반환
  UserStatsRepository.increase_post_count(UserIdentity) -> None ... assert_called 대상

TC 도출:
  1. test_should_create_setup_when_valid_request     — 값 → create → increase_post_count 호출 확인
  2. test_should_raise_user_not_found_when_no_user   — None → pytest.raises, create assert_not_called
  3. test_execute_is_transactional                   — __wrapped__ 존재
```

### 2-4. 산출물

```
## 분석 결과: {ClassName}
### 유형: USECASE / READER / WRITER
### Mock 의존성 (specs 출처)
| 의존성 | 메서드 | 반환 타입 | specs 위치 |
### 테스트 케이스
| # | 테스트명 | Mock 설정 요약 | 검증 | 패턴 |
```

사용자 확인 후 Phase 3 진행.

---

## Phase 3. add-test

### 파일 위치

```
{packages.service}/tests/{domain}/
├── impl/test_default_{domain}_reader.py
├── impl/test_default_{domain}_writer.py
└── usecase/test_{domain}_{action}_usecase.py
```

### 모듈 템플릿

```python
from dataclasses import replace
from unittest.mock import ANY, Mock

import pytest

from {pkg}.transaction import NoopTransactionManager
from {pkg}.infrastructure.{domain}.{domain}_repository import XxxRepository
from {pkg}.service.{domain}.impl.default_{domain}_writer import DefaultXxxWriter


@pytest.fixture
def repository() -> Mock:
    return Mock(spec=XxxRepository)


@pytest.fixture
def sut(repository: Mock) -> DefaultXxxWriter:
    return DefaultXxxWriter(tx=NoopTransactionManager(), repository=repository)
```

### 작성 규칙

1. **네이밍**: `test_should_예상결과_when_조건`
2. **구조**: `# given` → `# when` → `# then`
3. **예외**: `with pytest.raises(XxxException):`
4. **부수효과**: `repo.xxx.assert_called_once_with(...)` / `assert_not_called()`
5. **저장 반환**: `repo.save.side_effect = lambda d: replace(d, xxx_id=100)` (id 부여) 또는 `lambda d: d`
6. **도메인 객체**: specs/models.md 의 팩토리로 생성 우선. id 필요 시 `replace(..., xxx_id=1)`. `Mock` 으로 만들지 않는다
7. **Mock 은 항상 `spec=ABC`**: 없는 메서드 호출을 잡는다
8. **컨텍스트 금지**: dishka 컨테이너, DB 없음. `NoopTransactionManager` 사용

코드 예제는 `${CLAUDE_PLUGIN_ROOT}/references/be-test-refs/service-patterns.md`의 패턴 레터 참조.

### 검증

```bash
uv run pytest {packages.service}/tests/{domain}/{subpackage}/test_{module}.py -v
```

---

## 공통 규칙

- Phase 1은 최초 1회만 실행
- Phase 2 산출물을 사용자에게 보여주고 확인 후 Phase 3 진행
- 테스트 코드 내에서 프로덕션 코드를 수정하지 않는다
- 기존 테스트 파일이 있으면 덮어쓰지 않고 추가/수정한다
- 트랜잭션 동작 자체(커밋·롤백)는 이 스킬 범위 밖. be-test-service-concurrency 또는 통합 테스트 영역
