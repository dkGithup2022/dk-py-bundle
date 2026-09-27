# dk-py-bundle Project Context Resolver

이 플러그인의 스킬은 실행 시 **프로젝트 컨텍스트**를 먼저 해석한다. 해석된 값은 각 스킬 본문의 `{플레이스홀더}`에 대응된다.

## 해석 우선순위 (hybrid)

1. **config 파일** — 프로젝트 루트의 `dk-py-bundle.config.json`이 있으면 그 값을 최우선으로 사용
2. **자동 탐지** — config 없으면 아래 규칙으로 추출
3. **사용자 질의** — 탐지 실패 또는 모호하면 유저에게 명시적으로 묻고, 결과를 config 파일로 저장 제안

## 변수 목록

| 변수 | 의미 | 기본값 / 탐지 방법 |
|---|---|---|
| `{projectName}` | 배포명 접두어, docker DB 이름 | 루트 `pyproject.toml`의 `[project] name` |
| `{pkg}` | Python import 루트 (Java의 `{packageRoot}`) | `packages/model/src/` 아래 유일한 디렉터리 이름 |
| `{packagesPath}` | workspace 멤버 디렉터리 루트 | `packages` (루트 `[tool.uv.workspace] members` 가 `packages/*` 이면) |
| `{packages.model}` | model 패키지 경로 | `{packagesPath}/model` |
| `{packages.exception}` | exception 패키지 경로 | `{packagesPath}/exception` |
| `{packages.infrastructure}` | infrastructure 패키지 경로 | `{packagesPath}/infrastructure` |
| `{packages.transaction}` | 트랜잭션 계약 패키지 경로 | `{packagesPath}/transaction` |
| `{packages.my_logging}` | 로깅 패키지 경로 | `{packagesPath}/my_logging` |
| `{packages.persistence}` | SQLAlchemy 어댑터 패키지 경로 | `{packagesPath}/persistence` |
| `{packages.service}` | service 패키지 경로 | `{packagesPath}/service` |
| `{packages.api}` | api 패키지 경로 | `{packagesPath}/api` |
| `{packages.schema}` | schema (Alembic) 패키지 경로 | `{packagesPath}/schema` |
| `{packages.app}` | app 패키지 경로 | `{packagesPath}/app` |
| `{esModule}` | ES 패키지 이름 | config 명시 또는 질의 (예: `ai_search`) |
| `{esIndexName}` | ES 인덱스명 | config 명시 또는 질의 |
| `{esHost}` | ES 호스트 | `localhost:9200` |

### 파생 변수 (자동 변환)

| 변수 | 변환 규칙 | 예시 |
|---|---|---|
| `{packages.model.src}` | `{packages.model}/src/{pkg}/model` | `packages/model/src/share_ai_setup/model` |
| `{packages.service.src}` | 동일 규칙 | `packages/service/src/share_ai_setup/service` |
| `{packages.model.tests}` | `{packages.model}/tests` | `packages/model/tests` |
| `{pkg}.{layer}` | FQCN 접두어 | `share_ai_setup.service.article.article_reader` |

사용 위치: 파일 시스템 경로에는 `{packages.xxx.src}` / `{packages.xxx.tests}`, import 문과 pytest `--pyargs`에는 `{pkg}.{layer}`.

Java 번들과의 차이: Gradle 태스크 경로(`modules:api`)가 없다. 테스트 실행은 `uv run pytest {packages.api.tests}/...` 처럼 파일 경로로 한다.

## 자동 탐지 절차

```bash
# 1. projectName
grep -E '^name\s*=' {project-root}/pyproject.toml | head -1
# → name = "share-ai-setup-be" → projectName = share-ai-setup-be

# 2. packagesPath 확인
grep -A2 'tool.uv.workspace' {project-root}/pyproject.toml   # members = ["packages/*"]

# 3. pkg 추출 (model 패키지의 src 아래 유일한 디렉터리)
ls {project-root}/packages/model/src/
# → share_ai_setup → pkg = share_ai_setup

# 4. 각 패키지 존재 확인
ls {project-root}/packages/{model,exception,transaction,my_logging,infrastructure,persistence,service,api,schema,app}
```

## config 파일 스키마

`{project-root}/dk-py-bundle.config.json`:

```json
{
  "projectName": "share-ai-setup-be",
  "pkg": "share_ai_setup",
  "packagesPath": "packages",
  "packages": {
    "model": "packages/model",
    "exception": "packages/exception",
    "infrastructure": "packages/infrastructure",
    "transaction": "packages/transaction",
    "my_logging": "packages/my_logging",
    "persistence": "packages/persistence",
    "service": "packages/service",
    "api": "packages/api",
    "schema": "packages/schema",
    "app": "packages/app"
  },
  "es": {
    "module": "ai_search",
    "indexName": "setup",
    "host": "localhost:9200"
  }
}
```

모든 필드 선택(optional). 빠진 값은 자동 탐지 또는 질의로 보완.

## 스킬 실행 시 체크리스트

스킬 본문에서 `{플레이스홀더}`를 만나면:

1. config 파일 읽었는가? (없으면 읽기 시도)
2. 해당 변수가 config에 있는가? → 사용
3. 없으면 자동 탐지 시도 — 추출 규칙 적용
4. 실패하면 유저에게 질의: "이 프로젝트의 `{변수명}`은 무엇인가요?" + 기본값 제시
5. 답 받으면 메모리에 캐시하고, `dk-py-bundle.config.json`에 저장할지 제안
