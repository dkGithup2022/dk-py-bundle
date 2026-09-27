---
name: be-assess-usecase
description: service 레이어 평가. 상위(api)에서 요청된 기능이 기존 Reader/Writer/Usecase로 처리 가능한지 평가하고, infrastructure와 model에 필요한 기능을 도출한다.
---

# assess-usecase

service 레이어에서 요청된 기능을 처리할 수 있는지 평가한다.

## 입력

- assess-api 의 산출물 (service 에 대한 요청)
- specs/usecase.md (기존 Reader/Writer/Usecase 시그니처) — 프로젝트 루트 기준
- specs/infrastructure.md (기존 Repository ABC)
- specs/models.md (기존 도메인 모델)

specs 문서가 없으면 `packages/service`, `packages/infrastructure`, `packages/model` 소스에서 직접 추론한다.

## 평가 절차

### 1. 기존 service 확인

specs/usecase.md를 읽고, 요청된 기능과 매칭되는 ABC 메서드 / Usecase 가 있는지 확인한다.

- 완전 일치 → "충족". 작업 불필요.
- 기존 Reader/Writer 에 메서드 추가, 또는 기존 Writer 위에 Usecase 추가로 처리 가능 → "부분 충족". 변경 계획 수립.
- 없음 → "미충족". 신규 생성 계획 수립.

### 2. infrastructure 가용성 판단

specs/infrastructure.md를 읽고, service 구현에 필요한 데이터 접근이 가능한지 확인한다.

**기존 infrastructure로 처리 가능한 경우:**
- 기존 Repository ABC 의 메서드로 데이터 조회/저장 가능
- → infrastructure 변경 불필요

**새로운 infrastructure 기능이 필요한 경우:**
- 기존 Repository 에 없는 조회 조건이 필요 (예: 정렬, 필터, 집계)
- 새 도메인의 Repository 가 필요
- 원자적 연산 메서드가 필요
- → infrastructure 에 기능 요청

### 3. model 가용성 판단

specs/models.md를 읽고, service 가 다루는 도메인 모델이 충분한지 확인한다.

**기존 model로 처리 가능한 경우:**
- Command 변환에 필요한 도메인 타입이 존재
- 반환할 `{Domain}Read` 의 필드가 충분
- → model 변경 불필요

**model 변경/추가가 필요한 경우:**
- Command 가 다루는 도메인 자체가 없음 → 새 도메인 필요
- 기존 도메인에 필드/VO 추가 필요
- → model 에 기능 요청

### 4. service 계획 수립

다음을 결정한다:

- Reader / Writer / Usecase 중 어디에 속하는지
  - 조회 → Reader
  - 단순 쓰기 → Writer 메서드 + 그것을 감싸는 Usecase (라우터 진입점)
  - 여러 도메인 조합·부수효과 → Usecase
- 메서드 시그니처 (Python 타입 힌트)
- Command 필요 여부 및 필드 구성 (frozen dataclass, 첫 필드 `request_user_id`, 컬렉션은 `tuple`)
- 반환 타입 (Reader → `{Domain}Read` / Writer `create` → `{Domain}Identity`, `update` → `{Domain}`, `delete` → `None` / Usecase → 상황별)
- 크로스 도메인 의존성 (다른 도메인의 Repository 참조 여부)
- 트랜잭션: Writer 모든 public 메서드와 Usecase `execute` 는 `@transactional` (be-init C-9). Reader 는 없음. 부수효과가 트랜잭션 안에 들어가는지 명시
- 소유권 체크 위치 (Usecase)

## 규칙

- service 에 DB 접근 코드를 넣지 않는다. Repository ABC 와 `TransactionManager` 만 의존.
- CQRS: 조회는 Reader, 명령은 Writer 에 배치한다.
- 쓰기 API 의 진입점은 Usecase 다. Writer 를 라우터에 직접 노출하는 계획을 세우지 않는다.
- 같은 도메인의 Reader/Writer 는 서로 호출 가능하다.
- 다른 도메인의 Reader/Writer 를 호출하는 경우 → 유저에게 문의. (다른 도메인의 Repository 참조는 허용)
- Command 는 frozen dataclass 로 정의한다.
- 기존 service 변경 시 as-is/to-be 를 제시하고 유저 확인을 받는다.

## 하위 요청 시 컨벤션 지정

infrastructure 에 신규 메서드를 요청할 때 다음을 함께 명시한다:

- 읽기(`find_*` → `{Domain}Read | None` / `list[{Domain}Read]`) 인지 쓰기(`save` → `{Domain}`) 인지 선검증용(`find_write_by_id` → `{Domain} | None`) 인지
- 원자적 연산이면 `update()` 문으로 구현할 것 (조회→계산→저장 금지)
- 파라미터는 Identity VO 우선

## 산출물

### 1. 충족 여부

```
충족 / 부분 충족 / 미충족
```

### 2. 하위 레이어에 필요한 기능

**A. infrastructure:**
```
기존 사용 (변경 불필요):
  - {Repository}.{메서드명} 사용

신규 요청:
  - {Repository}.{메서드명}({파라미터}) -> {반환}
  - 종류: 읽기 / 쓰기 / 선검증 / 원자 연산
  - {구현해야 하는 기능에 대한 설명}
  - {동작 요구사항}
```

**B. model:**
```
기존 사용 (변경 불필요):
  - {Domain} 의 기존 필드/타입으로 충분

신규 요청:
  - 대상: {도메인명} (신규/신규 아님)
  - {필요한 데이터 구조 설명}
  - 예상: {VO/필드 + 관계 힌트}
  - {왜 필요한지 기능 맥락}
```

### 3. 현재 레이어 작업 계획

```
- [신규/변경] {Reader / Writer / Usecase}.{메서드명}
- 입력: {Command 또는 파라미터}
- 반환: {Domain}Read / {Domain}Identity / {Domain} / None
- 의존: {사용하는 Repository ABC 목록, TransactionManager 여부}
- 트랜잭션: @transactional 여부, 안에 들어가는 부수효과
- 소유권 체크: 있음/없음
- DI: {Domain}Provider 에 등록할 클래스 목록
```
