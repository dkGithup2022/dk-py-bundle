---
name: be-assess-model
description: model 레이어 평가. 상위 레이어들에서 요청된 도메인 모델이 기존으로 충분한지 평가하고, 최종 model 작업 계획을 수립한다.
---

# assess-model

model 레이어에서 요청된 도메인 타입을 처리할 수 있는지 평가한다.

## 입력

- 상위 assess 스킬들의 산출물 (model 에 대한 요청을 통합)
- specs/models.md (기존 도메인 모델) — 프로젝트 루트 기준

specs 문서가 없으면 `packages/model` 소스에서 직접 추론한다.

## 평가 절차

### 1. 기존 model 확인

specs/models.md를 읽고, 요청된 도메인 타입이 있는지 확인한다.

- 완전 일치 → "충족". 작업 불필요.
- 기존 도메인에 필드/VO 추가로 처리 가능 → "부분 충족". 변경 계획 수립.
- 도메인 자체가 없음 → "미충족". 신규 생성 계획 수립.

### 2. 요청 항목별 분류

상위 레이어들의 요청을 항목별로 분류한다:

**A. 도메인 모델 (Write)**
- 새 도메인 필요? 기존 도메인 확장?
- 필드 구성, 팩토리 classmethod, 상태 변경 메서드

**B. Read 모델**
- 새 Read 모델 필요? 기존 Read 모델 확장?
- Write 모델 필드 + JOIN 정보 (nickname 등) → `to_read` 파라미터

**C. Identity VO**
- 새 도메인이면 Identity VO 필요

**D. StrEnum**
- 새 enum 필요? 기존 enum 에 값 추가?

**E. Value Object**
- 새 VO 필요? (Tag, LinkedContent 같은)
- 기존 공유 VO (Popularity 등) 변경 필요?

**F. Stats**
- 카운터 증감이 요청되었으면 분리 Stats 모델 (`{Domain}Stats`, `init()` 팩토리)

**G. 도메인 예외**
- 새 도메인이면 `{Domain}NotFoundException` (add-model 이 짝으로 생성)
- 상위 요청에 특화 예외(`WrongCommentException` 등)가 있으면 목록에

### 3. model 계획 수립

다음을 결정한다:

- 패턴 (Foo / Bar / Baz / Qux — model-references.md 선택 가이드)
- 생성/변경할 클래스 목록과 파일
- 각 클래스의 필드 구성 (타입 힌트, 컬렉션은 `tuple`)
- 팩토리 classmethod 설계 (`create`, `create_root` 등)
- 상태 변경 메서드 설계 (`with_xxx`, `hide` — `dataclasses.replace` 로 새 인스턴스)
- `to_read` 시그니처 (JOIN 파라미터)
- 기존 도메인 변경 시 영향 범위 — **필드 추가 위치**: 위치 인자 호출이 깨지지 않게 끝 또는 `updated_at` 앞

## 규칙

- `@dataclass(frozen=True, slots=True)` 불변 객체로 정의한다.
- 상태 변경은 새 인스턴스를 반환한다 (`replace`).
- 팩토리 classmethod 로 생성한다 (생성자 직접 호출 지양).
- 외부 의존성 없음. 순수 Python. `sqlalchemy`, `pydantic` import 금지.
- 컬렉션 필드는 `tuple`.
- soft delete 필드(`is_deleted`, `deleted_at`)는 모델에 포함하지 않는다. Entity 레이어에서만 관리.
- Read 모델의 id 는 `int` (None 아님). Write 모델은 `int | None`.
- 기존 도메인 변경 시 as-is/to-be 를 제시하고 유저 확인을 받는다.

## 산출물

### 1. 충족 여부

```
충족 / 부분 충족 / 미충족
```

### 2. 현재 레이어 작업 계획

**하위 레이어 없음. model 이 최하위.**

```
- 패턴: Foo / Bar / Baz / Qux

- [신규] {Domain}  ({domain}.py)
  - 필드: {필드: 타입 목록}
  - 팩토리: {classmethod 목록}
  - 상태 변경: {메서드 목록}
  - to_read: ({JOIN 파라미터}) -> {Domain}Read

- [신규] {Domain}Read  ({domain}_read.py)
  - 필드: {Write 모델 필드 + JOIN 필드}

- [신규] {Domain}Identity  ({domain}_identity.py)
  - 필드: {domain}_id: int

- [신규] {Domain}Stats  ({domain}_stats.py)   ← 있으면
  - 필드: {카운터 목록}

- [신규/변경] {EnumName}  ({enum_name}.py)
  - 값: {StrEnum 값 목록}

- [신규] {VOName}  ({vo_name}.py)
  - 필드: {필드 목록}
  - 팩토리: {메서드 목록}

- [신규] {Domain}NotFoundException  (exception/{domain}/)
  - 기본 메시지 사용 / 커스텀 메시지: "..."

- [변경] {기존 도메인명}
  - as-is: {기존 필드}
  - to-be: {변경 후 필드, 추가 위치}
  - 영향: {이 변경이 상위 레이어(to_read 시그니처, Entity, Response)에 미치는 영향}
```
