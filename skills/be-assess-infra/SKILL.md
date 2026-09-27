---
name: be-assess-infra
description: infrastructure 레이어 평가. 상위(service)에서 요청된 기능이 기존 Repository ABC로 처리 가능한지 평가하고, model에 필요한 기능을 도출한다.
---

# assess-infra

infrastructure 레이어에서 요청된 기능을 처리할 수 있는지 평가한다.

## 입력

- assess-usecase 의 산출물 (infrastructure 에 대한 요청)
- specs/infrastructure.md (기존 Repository ABC) — 프로젝트 루트 기준
- specs/models.md (기존 도메인 모델)

specs 문서가 없으면 `packages/infrastructure`, `packages/model` 소스에서 직접 추론한다.

## 평가 절차

### 1. 기존 infrastructure 확인

specs/infrastructure.md를 읽고, 요청된 기능과 매칭되는 Repository 메서드가 있는지 확인한다.

- 완전 일치 → "충족". 작업 불필요.
- 기존 Repository 에 메서드 추가로 처리 가능 → "부분 충족". 변경 계획 수립.
- 해당 도메인의 Repository 자체가 없음 → "미충족". 신규 생성 계획 수립.

### 2. model 가용성 판단

specs/models.md를 읽고, Repository 메서드의 입출력에 필요한 도메인 타입이 있는지 확인한다.

**기존 model로 처리 가능한 경우:**
- 반환할 `{Domain}Read` / `{Domain}` 의 타입이 존재
- 파라미터에 필요한 Identity VO 가 존재
- → model 변경 불필요

**model 변경/추가가 필요한 경우:**
- 새 도메인의 Repository → 해당 도메인 모델 자체가 필요
- 기존 도메인에 새 필드 → Read/Write 모델 모두 확장 필요
- 새 Identity VO 필요
- → model 에 기능 요청

### 3. infrastructure 계획 수립

다음을 결정한다:

- 기존 Repository 에 추가 or 신규 Repository 생성
- Stats 분리 여부 (카운터 증감이 있으면 `{Domain}StatsRepository` 별도 파일)
- 메서드 시그니처 (파라미터 + 반환 타입, Python 타입 힌트)
- 읽기 메서드 → `{Domain}Read | None` / `list[{Domain}Read]`
- 선검증 메서드 → `find_write_by_id` → `{Domain} | None`
- 쓰기 메서드 → `{Domain}`
- 원자적 연산 여부 (`increase_xxx_count` 등) — 어댑터가 `update()` 문으로 구현할 것을 명시
- 어댑터 영향: 기존 ABC 에 메서드를 추가하면 기존 어댑터가 그 메서드를 구현할 때까지 인스턴스 생성이 실패한다. add-query 가 반드시 뒤따라야 함을 계획에 명시

## 규칙

- ABC 만 정의한다. 구현은 persistence 의 영역.
- 읽기 → `{Domain}Read` 반환, 쓰기 → `{Domain}` 반환 원칙을 따른다.
- model 과 exception 만 의존한다. SQLAlchemy 타입을 시그니처에 노출하지 않는다.
- 트랜잭션은 port 의 관심사가 아니다. 어댑터가 `@transactional` 로 처리한다 (계획에 트랜잭션 언급 불필요).
- 기존 ABC 변경 시 as-is/to-be 를 제시하고 유저 확인을 받는다.

## 하위 요청 시 컨벤션 지정

model 에 신규 타입을 요청할 때 다음을 함께 명시한다:

- Write 모델 / Read 모델 / Identity / Stats / enum / VO 중 무엇인지
- Read 모델에 JOIN 으로 들어올 필드가 있으면 그 필드명과 출처 도메인 (to_read 파라미터가 된다)
- 자식 컬렉션이 있으면 그 원소 타입 (enum 인지 VO 인지)

## 산출물

### 1. 충족 여부

```
충족 / 부분 충족 / 미충족
```

### 2. 하위 레이어에 필요한 기능

**A. model:**
```
기존 사용 (변경 불필요):
  - {Domain}, {Domain}Read, {Domain}Identity 기존 타입으로 충분

신규 요청:
  - 대상: {도메인명} (신규/신규 아님)
  - 종류: Write / Read / Identity / Stats / enum / VO
  - {필요한 데이터 구조 설명}
  - 예상: {VO/필드 + 관계 힌트, JOIN 필드와 출처}
  - {왜 필요한지 구현 맥락}
```

### 3. 현재 레이어 작업 계획

```
- [신규/변경] {Repository}.{메서드명}({파라미터}) -> {반환}
- 종류: 읽기 / 선검증 / 쓰기 / 삭제 / 원자 연산
- 파라미터: {타입 목록}
- 반환: {Domain}Read | None / list[{Domain}Read] / {Domain} / None
- 원자적 연산: 여부 (어댑터는 update() 문)
- Stats 분리: 여부
- 어댑터 갱신 필요: 여부 (기존 ABC 변경 시)
```
