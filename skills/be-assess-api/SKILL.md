---
name: be-assess-api
description: API 레이어 평가. 요청된 API 스펙이 기존 엔드포인트로 처리 가능한지 평가하고, service(usecase)와 model에 필요한 기능을 도출한다.
---

# assess-api

API 레이어에서 요청된 기능을 처리할 수 있는지 평가한다.

## 입력

- 유저의 API 스펙 요청
- specs/api.md (기존 API 엔드포인트) — 프로젝트 루트 기준
- specs/usecase.md (기존 Reader/Writer/Usecase 시그니처)
- specs/models.md (기존 도메인 모델)

specs 문서가 없으면 `packages/api`, `packages/service`, `packages/model` 소스에서 직접 추론한다.

## 평가 절차

### 1. 기존 API 확인

specs/api.md를 읽고, 요청된 기능과 매칭되는 엔드포인트가 있는지 확인한다.

- 완전 일치 → "충족". 작업 불필요.
- 옵션/파라미터 추가로 처리 가능 → "부분 충족". 변경 계획 수립.
- 없음 → "미충족". 신규 생성 계획 수립.

### 2. service(usecase) 가용성 판단

specs/usecase.md를 읽고, 필요한 기능이 이미 있는지 확인한다.

**기존으로 처리 가능한 경우:**
- 조회: 기존 Reader 메서드로 충분
- 쓰기: 기존 Usecase (`{Domain}{Action}Usecase.execute`) 가 있음
- 라우터에서 조합만 하면 됨
- → service 변경 불필요

**새로운 service 기능이 필요한 경우:**
- 기존 Reader 에 없는 조회 메서드가 필요
- **쓰기 엔드포인트인데 해당 Usecase 가 없음** — 라우터는 Writer 를 직접 쓰지 않으므로 Usecase 가 반드시 있어야 한다
- 여러 도메인을 조합하는 사가성 로직이 필요
- 새로운 Command 가 필요
- → service 에 기능 요청

### 3. model 가용성 판단

specs/models.md를 읽고, API 응답에 필요한 데이터가 기존 도메인 모델에 있는지 확인한다.

**기존 model로 처리 가능한 경우:**
- `{Domain}Read` 의 기존 필드로 Response 구성 가능
- → model 변경 불필요

**model 변경/추가가 필요한 경우:**
- 요청된 API 가 다루는 도메인 자체가 없음 → 새 도메인 필요
- 기존 도메인에 필드 추가가 필요 → 기존 도메인 확장
- → model 에 기능 요청

### 4. API 계획 수립

다음을 결정한다:

- HTTP Method + URL 패턴 (`/api/{복수형}`)
- Request DTO 필드 구성 (Pydantic, 필수/optional, 검증 조건)
- Response DTO 필드 구성 (specs/models.md 의 `{Domain}Read` 기반)
- 인증 필요 여부 (`FromDishka[CurrentUser]` 선언 여부)
- 사용할 Reader 메서드 / Usecase (기존 or 신규 요청)
- 상태 코드 (200 / 201 / 204)

## 규칙

- 라우터에 비즈니스 로직을 넣지 않는다. 위임만.
- Request → Command 변환은 라우터에서 수행한다 (`list → tuple` 포함).
- Response 는 `from_read({Domain}Read)` classmethod 로 생성한다.
- 라우터는 Reader 와 Usecase 만 의존한다. Writer 직접 주입은 계획에 넣지 않는다.
- 입력 검증은 Pydantic 이 한다. 라우터 내부 파싱 분기(enum 변환 등)를 계획에 넣지 않는다.
- 기존 API 변경 시 as-is/to-be를 제시하고 유저 확인을 받는다.

## 하위 요청 시 컨벤션 지정

service 에 신규 기능을 요청할 때, 받는 쪽(assess-usecase / add-usecase)이 일관되게 작업할 수 있도록 다음을 함께 명시한다:

- Reader 인지 Usecase 인지 (조회 → Reader, 쓰기 → Usecase)
- Command 필요 여부와 필드 (첫 필드 `request_user_id: int`)
- 반환 타입 (`{Domain}Read` / `{Domain}Identity` / `None`)
- 인증 user_id 를 `execute(user_id, command)` 첫 인자로 받는지

## 산출물

### 1. 충족 여부

```
충족 / 부분 충족 / 미충족
```

### 2. 하위 레이어에 필요한 기능

**A. service:**
```
기존 사용 (변경 불필요):
  - {Reader}.{메서드명} 사용
  - {Usecase}.execute 사용

신규 요청:
  - [Reader] {Domain}Reader.{메서드명}({파라미터}) -> {반환}
  - [Usecase] {Domain}{Action}Usecase.execute(user_id: int, command: {Command}) -> {반환}
  - Command: {Command}({필드 목록})
  - {구현해야 하는 기능에 대한 설명}
  - {동작 요구사항, 부수효과}
```

**B. model:**
```
기존 사용 (변경 불필요):
  - {Domain}Read 의 기존 필드로 충분

신규 요청:
  - 대상: {도메인명} (신규/신규 아님)
  - {필요한 데이터 구조 설명}
  - 예상: {VO/필드 + 관계 힌트}
  - {왜 필요한지 기획 맥락}
```

### 3. 현재 레이어 작업 계획

```
- [신규/변경] {HTTP Method} {URL}
- Request: {필드 목록 + 검증 조건}
- Response: {필드 목록} (from_read {Domain}Read)
- 상태 코드: 200/201/204
- 인증: 필요/불필요
- 사용 service: {기존 Reader 메서드 / Usecase or 신규 요청}
```
