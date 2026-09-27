# 테스트 인프라 체크리스트

패키지별 테스트 인프라 요구사항. `be-test-*` 스킬의 Phase 1에서 참조한다.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 패키지)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

---

## 공통 (모든 패키지)

### 필수

| 항목 | 경로 | 확인 |
|------|------|------|
| pytest | 루트 `pyproject.toml` `[dependency-groups] dev` | `pytest` 있는지 |
| testpaths | 루트 `pyproject.toml` `[tool.pytest.ini_options]` | `testpaths = ["packages"]` |
| 테스트 디렉터리 | `{packages.xxx}/tests/` | 존재 (be-init 이 `.gitkeep` 으로 생성). `__init__.py` 는 두지 않는다 |

be-init 이 만든 프로젝트면 전부 이미 있다. 없으면 be-init 미실행이거나 수동 프로젝트 — 위 세 가지를 추가.

### 실행

```bash
uv run pytest {packages.xxx}/tests/{domain}/test_xxx.py
uv run pytest {packages.xxx}/tests/{domain}/test_xxx.py -k "test_should_x"
```

---

## persistence 패키지 (통합 테스트)

### 필수

| 항목 | 경로 | 확인 |
|------|------|------|
| 루트 conftest | `{projectRoot}/conftest.py` | `database_url`, `engine`, `session_factory`, `session`, `truncate_all` 픽스처 (be-init §4.10) |
| testcontainers | 루트 dev 그룹 | `testcontainers[postgres]` |
| Docker | `docker info` | 데몬 실행 중 |
| Entity 레지스트리 | `{packages.persistence}/src/{pkg}/persistence/entities.py` | 테스트 대상 Entity 가 등록되어 있는지 |
| migration | `{packages.schema}/alembic/versions/` | 테스트 대상 테이블의 migration 이 있는지 (`engine` 픽스처가 `upgrade head` 를 돈다) |

### 픽스처 사용

- 일반 통합 테스트: `session` (외부 트랜잭션 + 세이브포인트, 테스트 끝에 전부 롤백)
- 구현체는 `Default{Domain}Repository(tx=DefaultTransactionManager(session), entities={Domain}EntityQuery(session), joins=...)` 로 직접 조립
- 선행 데이터는 다른 도메인 구현체를 같은 `session` 으로 조립해 넣는다

### 불필요

- 패키지별 conftest, TestApplication, yml — 루트 conftest 가 전부 대체

---

## service 패키지 (단위 테스트)

### 필수

공통 항목만.

### 불필요

- conftest, DB, Docker — Mockito 대응인 `unittest.mock` 만 사용
- `NoopTransactionManager` 는 `{pkg}.transaction` 에 이미 있다 (be-init)

---

## service 패키지 (통합 테스트 — 동시성)

### 필수

| 항목 | 경로 | 확인 |
|------|------|------|
| 루트 conftest | `{projectRoot}/conftest.py` | `session_factory`, `truncate_all` 픽스처 |
| Docker | `docker info` | 데몬 실행 중 |
| Provider | `{packages.app}/src/{pkg}/app/providers/{domain}_provider.py` | 대상 Writer 의 의존이 조립 가능한지 (테스트는 직접 조립하지만 참고) |

### 픽스처 사용

- `session` (세이브포인트) 은 **쓰지 않는다** — 스레드 간 공유 불가, 커밋이 실제로 일어나야 경쟁이 재현된다
- 스레드마다 `session_factory()` 로 세션을 새로 열고 어댑터 + Writer 를 직접 조립
- `truncate_all` 로 테스트 후 전 테이블 정리 (`Base.metadata.sorted_tables` 역순)

---

## api 패키지 (단위 테스트)

### 필수

공통 항목만.

### 불필요

- TestClient, dishka 컨테이너, DB — 라우터 함수를 직접 호출하고 `FromDishka` 파라미터에 Mock 을 넘긴다
- 직접 호출이 안 되는 경우(Phase 1 에서 확인)에만 `httpx` + `TestClient` + 테스트용 Provider 로 전환

---

## app 패키지 (E2E, 선택)

### 필수

| 항목 | 경로 | 확인 |
|------|------|------|
| httpx | 루트 dev 그룹 | `httpx` (be-init 포함) |
| Docker | | 실제 DB 를 타는 E2E 라면 |

`TestClient(create_app())` 로 HTTP 레벨 검증. be-test-* 스킬 범위 밖. 필요하면 별도 스킬.

---

## exception / model / infrastructure 패키지

### 필수

공통 항목만. 순수 pytest.

### 불필요

나머지 전부.
