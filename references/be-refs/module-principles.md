# BE 패키지 원칙과 목적 (RDBMS 기준)

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 패키지)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

## 본질

**패키지를 왜 나누는가?**
→ 제작/수정에서 케어의 범위를 좁히기 위해서.

---

## 목적

### 사람적 이유: 인지의 범위를 제한
- 이 패키지에서는 이것만 신경 쓰면 된다
- 도메인마다 동일한 패턴이므로, 한 도메인만 이해하면 나머지는 공짜

### 기계적 이유

#### 1. 레퍼런스의 범위를 좁힌다
기능을 제작하기 위해 탐색해야 하는 코드베이스의 범위를 패키지로 제한.
"A 패키지는 B, C만 참조" → 스캔해야 하는 코드를 줄인다.

#### 2. 컨텍스트에 레퍼런스되는 양을 줄인다
구현체 대신 ABC(port)만 읽으면 된다.
패키지 경계 = LLM이 읽어야 할 컨텍스트의 경계.

#### 3. 각 패키지의 동작과 검증의 범위를 의도적으로 narrow하게 만든다
패키지가 수행하는 것이 좁고 명확하면, 스킬로 자동화할 수 있다.
패턴이 고정되어 있으므로, 도메인명과 필드 스펙만 주면 전 계층을 생성 가능.

---

## 패키지 분리의 근거

### 기본 Layered 분리: api / service / infra

각 계층이 케어하는 것이 다르고, 변경 이유가 다르다.

```
api:      "HTTP 요청을 어떻게 받고 응답하나" 만 케어
service:  "비즈니스 규칙이 뭔가" 만 케어
infra:    "데이터를 어떻게 저장/조회하나" 만 케어
```

### infra의 port 분리 (infrastructure 패키지)

infra는 레퍼런스하는 쪽이 여러 개이므로 port(ABC)를 별도 패키지로 분리한다.

```
service (API용 비즈니스 로직) → infrastructure port
batch task (크롤링, 동기화)   → infrastructure port
outbox (이벤트 처리)          → infrastructure port
```

소비자가 많을수록 port 분리의 이득이 커진다.
구현체(persistence)의 내부 변경이 소비자에게 전파되지 않는다.

### Python에서 물리적 강제가 안 되는 점

uv workspace는 venv 하나를 공유하므로 어느 패키지든 다른 패키지를 import할 수 있다.
그래서 **import-linter 계약**(루트 `pyproject.toml`)이 의존 방향을 검사한다. `uv run lint-imports`가 통과해야 헥사고날이 지켜진 것이다.

---

## 패키지 구조 (RDBMS 기본 흐름)

```
   model, exception, transaction, my_logging (공유 기반)
              ↑
        infrastructure (port ABC)
         ↑          ↑
   service     persistence
      ↑               ↑
     api            schema (Alembic)
      ↑
     app (DI 바인딩, 조립, 로깅 설정, 요청 로깅 미들웨어)
```

`transaction` 과 `my_logging` 은 도메인과 무관한 프레임워크 성격의 코드라 `infrastructure` 와 섞지 않고 맨 아래 층에 둔다. 둘 다 다른 패키지를 import 하지 않는다.

---

## 각 패키지의 원칙

### model
- **역할**: 도메인의 본질을 표현
- **규칙**:
  - `@dataclass(frozen=True, slots=True)` 불변 값 객체
  - 상태 변경은 새 인스턴스 반환 (`dataclasses.replace`)
  - 팩토리 classmethod로 생성 (생성자 직접 호출 지양)
  - 외부 의존성 없음 (순수 Python)
  - 컬렉션 필드는 `tuple`
- **포함하는 것**: 도메인 모델, Identity VO, StrEnum, 공유 VO, `AuditFields` Protocol
- **포함하지 않는 것**: DB 매핑, 프레임워크 import

### exception
- **역할**: 도메인별 예외 정의
- **규칙**:
  - `DomainException` 상속 + `http_status_code` / `http_error_message` 속성
  - 404는 `EntityNotFoundException` 의 도메인별 하위 클래스
  - 도메인별 서브패키지 분리
- **포함하는 것**: 공통 5종 + 도메인 특화 예외
- **포함하지 않는 것**: 비즈니스 로직, 에러 핸들링 (핸들러는 app)

### infrastructure
- **역할**: 데이터 접근 계층의 port
- **규칙**:
  - ABC만 존재, 구현 없음
  - 읽기 메서드 → Read 모델 반환
  - 쓰기 메서드 → Write 모델 반환
  - 원자적 연산은 별도 메서드 (increase_view_count 등)
  - model과 exception만 의존
- **포함하는 것**: Repository ABC
- **포함하지 않는 것**: 구현체, SQLAlchemy import, 쿼리, 트랜잭션 계약 (→ transaction)

### transaction
- **역할**: 트랜잭션 계약 (Java 의 spring-tx 자리)
- **규칙**:
  - `TransactionManager` ABC + `NoopTransactionManager` + `@transactional` 데코레이터. be-init 소유, 도메인 코드가 손대지 않는다
  - 외부 의존 없음, 다른 패키지 import 없음
  - 사용하는 쪽은 `from {pkg}.transaction import TransactionManager, transactional`
- **포함하는 것**: `manager.py`, `decorator.py`
- **포함하지 않는 것**: 구현체 (`DefaultTransactionManager` 는 persistence), 세션

### my_logging
- **역할**: 로그 형식(JSON 한 줄)과 로그 컨텍스트 키를 한 곳에서 정한다
- **규칙**:
  - 엔진은 structlog. `structlog` 직접 import 는 이 패키지 안에서만
  - 도메인 코드는 `logging.getLogger(__name__)` 를 그대로 쓴다. 형식은 이 패키지가 정한다
  - 요청 범위 키(`set_context`)는 HTTP 요청 1회가 범위. 전역 키(`set_global_context`)는 프로세스 수명
  - 요청 진입 로그는 찍지 않는다. app 의 `RequestLoggingMiddleware` 가 `http.request` 한 줄로 대신한다
- **포함하는 것**: `configure_logging`, `log` / `error` (+ `info` / `warning`), `set_context` / `remove` / `clear` / `keys`, `set_global_context` / `remove_global`, `get_logger`
- **포함하지 않는 것**: 미들웨어 (app), 도메인 로직

### persistence
- **역할**: infrastructure port의 SQLAlchemy 구현 (Adapter)
- **규칙**:
  - port ABC를 상속
  - Entity ↔ Domain 변환은 이 패키지 안에서
  - VO 분해/재구성
  - aggregate 자식만 `relationship(lazy="selectin")`, cross-aggregate relationship 금지
  - 원자적 UPDATE는 `update()` 문으로 직접 작성
  - **`commit()` 호출 금지**. public 메서드에 `@transactional`
- **포함하는 것**: Entity, 어댑터(쿼리 포함), `DefaultTransactionManager` (transaction 계약의 SQLAlchemy 구현), 세션 팩토리
- **포함하지 않는 것**: 비즈니스 로직

### service
- **역할**: 비즈니스 로직 (애플리케이션 유스케이스)
- **규칙**:
  - CQRS: Reader(조회) / Writer(명령) 분리
  - infrastructure의 port ABC와 transaction 의 `TransactionManager`만 의존 (구현체 모름)
  - `@transactional`은 Writer와 Usecase에만. Reader는 없음
  - 크로스 도메인 참조는 다른 도메인의 port를 통해
- **포함하는 것**: Reader/Writer ABC + Default 구현, Usecase, Command dataclass
- **포함하지 않는 것**: 라우터 로직, DB 접근 코드

### api
- **역할**: REST API 엔드포인트
- **규칙**:
  - service의 Reader ABC와 Usecase만 의존 (Writer 직접 주입 금지)
  - Request → Command 변환은 라우터 함수에서
  - Response는 `from_read()` classmethod
  - `FromDishka[CurrentUser]`로 인증 정보 수신
  - 비즈니스 로직 없음, 위임만
- **포함하는 것**: APIRouter, Pydantic Request/Response, `auth.py`
- **포함하지 않는 것**: 비즈니스 로직, DB 접근, `app` import

### schema
- **역할**: DB 테이블 정의 (Alembic)
- **규칙**:
  - migration 파일만 포함
  - enum은 VARCHAR로 저장
  - VO는 flatten
  - soft delete (is_deleted)
  - audit 필드 (created_at, updated_at)
  - DB FK 제약 없음 (`env.py` 필터)
- **포함하는 것**: `alembic.ini`, `env.py`, `versions/`
- **포함하지 않는 것**: DML, 시드 데이터

### app
- **역할**: 조립 지점
- **규칙**:
  - dishka Provider 바인딩, 라우터 등록, 예외 핸들러, settings, lifespan, 로깅 설정(`configure_logging` 을 `create_app()` 첫 줄에서), 요청 로깅 미들웨어
  - 도메인 코드 없음
- **포함하는 것**: `main.py`, `di.py`, `routers.py`, `error_handlers.py`, `settings.py`, `middleware/request_logging.py`
- **포함하지 않는 것**: 비즈니스 로직

---

## 의존성 규칙

```
허용되는 의존 방향:
  app → api, service, persistence, infrastructure, schema, transaction, my_logging, model, exception
  schema → persistence
  api → service, my_logging, model, exception
  service → infrastructure, transaction, my_logging, model, exception
  persistence → infrastructure, transaction, my_logging, model, exception
  infrastructure → model, exception
  transaction → (없음)
  my_logging → structlog 만

금지:
  service → persistence (구현체 직접 의존)
  api → infrastructure (계층 건너뛰기)
  api → app (역방향. dishka 가 이걸 막기 위해 있다)
  model → 다른 패키지 (순수 도메인)
  persistence / schema / app 밖에서 sqlalchemy, psycopg, alembic import (import-linter forbidden 계약)
```

---

## 스펙 문서 (계층별)

플래너가 견적을 내고, 각 스킬이 작업할 때 참조하는 문서.
계층별로 분리하여 각 스킬이 자기 계층 문서만 읽으면 된다.

```
specs/
├── models.md           ← add-model 스킬이 참조
├── infrastructure.md   ← add-infrastructure 스킬이 참조
├── schema.md           ← add-entity-specs 스킬이 참조 (migration 요약)
├── repository.md       ← add-entity-specs / add-query 스킬이 참조
├── usecase.md          ← add-usecase 스킬이 참조
└── api.md              ← add-api 스킬이 참조
```

- 플래너: 전체 specs/를 훑어서 기존 상태 파악 → 견적
- 스킬: 자기 계층 문서만 읽고 작업 → 케어 범위 제한
- 작업 완료 후: 해당 계층 문서 업데이트
- `specs/` 디렉터리와 빈 문서 6개는 첫 add-* 스킬이 없으면 생성한다

---

## 스킬 구조

### 스킬 종류: assess (평가) + add (구현)

각 레이어에 두 종류의 스킬이 있다.

```
평가 스킬 (위에서 아래로):
  assess-api            → API 레이어 평가
  assess-usecase        → service 레이어 평가
  assess-infra          → infrastructure 레이어 평가
  assess-model          → model 레이어 평가

구현 스킬 (아래에서 위로):
  add-model             → model 패키지의 작업 규칙 (도메인 예외 포함)
  add-infrastructure    → infrastructure 패키지의 작업 규칙
  add-entity-specs      → Entity + Alembic migration 생성 규칙
  add-query  → 어댑터(쿼리 포함) 구현 규칙
  add-usecase           → service 패키지의 작업 규칙 + DI provider 등록
  add-api               → api 패키지의 작업 규칙 + 라우터 등록
```

### assess 스킬 공통 원칙

**행동 규칙:**
1. 가능하면 현재 레이어에서 처리
2. 데이터 스펙(domain, entity) 변경이 필요하면 → 유저에게 문의 + 제안 동봉
3. 명시된 패키지에 대해서만 쿼리
4. 패키지 간 통신은 정해진 스펙으로만
   - infra → return: domain model
   - service → return: {Domain}Read, input: Command

**공통 산출물:**
1. 현재 레이어에서 작업 충족 여부
2. 필요한 하위 기능 (패키지별)
3. 현재 레이어의 작업 계획과 스펙

산출물이 다음 레이어의 입력이 된다.

### 진행 방향

```
평가: 위에서 아래로 (top-down)
  api → service → infra → model
  "이 요청을 처리할 수 있는가? 아래에 뭐가 필요한가?"

제작: 아래에서 위로 (bottom-up)
  model → infra → entity/migration → adapter → service → api
  "확정된 설계를 기반으로 구현"
```

### 플래너 = 경량 오케스트레이터

플래너는 판단을 직접 하지 않는다. assess 스킬의 결과를 모아서 조율만 한다.

1. 유저 요청 + API 스펙 수신
2. assess 스킬을 위에서 아래로 체이닝
3. 결과를 모아 유저에게 전체 계획 제시
4. 유저 확정 후 add 스킬을 아래에서 위로 실행

### 케이스별 실행 범위

assess 결과에 따라 필요한 스킬만 실행한다.

```
케이스 A: 새 도메인
  평가: assess-api → assess-usecase → assess-infra → assess-model (전부 "없음")
  제작: add-model → add-infrastructure → add-entity-specs → add-query → add-usecase → add-api

케이스 B: 기존 도메인에 API 추가
  평가: assess-api("없음") → assess-usecase("충분") → 여기서 끝
  제작: add-usecase 변경 → add-api 추가

케이스 C: 테이블 구조 변경
  평가: assess-api → assess-usecase → assess-infra("변경필요") → assess-model("변경필요")
  제작: add-model → add-infrastructure → add-entity-specs → add-query (port 안 바뀌면 나머지 그대로)
```

---

## 도메인 추가 시 생성 순서 (전체 실행의 경우)

```
1. model/                  → 도메인 모델, Identity, enum + 도메인 예외 (exception/)
2. infrastructure/         → Repository ABC
3. persistence/  → Entity + entities.py 등록 → schema/ migration
4. persistence/  → 어댑터 (쿼리 포함)
5. service/                → Reader/Writer + Command + Usecase → app/di.py provider 등록
6. api/                    → Router + Request/Response → app/routers.py 등록
```

---

## 미결 사항

- auth, elasticsearch 패키지의 헥사고날 적용 여부 → 스킬 제작 중 판단
- service 패키지의 ABC 분리 여부 → batch에서 service를 쓰게 될 경우 필요할 수 있음. 선택의 문제로 보류.
