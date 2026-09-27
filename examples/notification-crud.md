# Plan: Notification CRUD (6 APIs)

## 요청 스펙

알림 도메인 신규 생성 + CRUD API 6개.

| # | Method | URL | 설명 |
|---|--------|-----|------|
| 1 | GET | /api/notifications?page&size | 알림 목록 조회 |
| 2 | GET | /api/notifications/unread-count | 안 읽은 개수 |
| 3 | PUT | /api/notifications/{id}/read | 개별 읽음 처리 |
| 4 | PUT | /api/notifications/read-all | 모두 읽기 |
| 5 | DELETE | /api/notifications/{id} | 개별 삭제 |
| 6 | DELETE | /api/notifications | 모두 삭제 |

인증: 전부 필요 (`FromDishka[CurrentUser]`)

---

## Phase 1: 평가

### assess-api

**충족 여부: 미충족** — Notification 관련 엔드포인트 없음. 전부 신규.

**service 요청:**

```
신규 요청:
  - [Reader] NotificationReader
    - get_by_receiver(user_id: int, page: int, size: int) -> list[NotificationRead]
    - count_by_receiver(user_id: int) -> int          # total 용
    - count_unread(user_id: int) -> int
  - [Usecase] 쓰기 진입점 4개 (라우터는 Writer 를 직접 쓰지 않음)
    - NotificationMarkReadUsecase.execute(user_id: int, identity: NotificationIdentity) -> None
    - NotificationMarkAllReadUsecase.execute(user_id: int) -> None
    - NotificationDeleteUsecase.execute(user_id: int, identity: NotificationIdentity) -> None
    - NotificationDeleteAllUsecase.execute(user_id: int) -> None
  - (create 는 별도 계획에서 추가 예정)
```

**model 요청:**

```
신규 요청:
  - 대상: Notification (신규)
  - NotificationType StrEnum (REACTION_1, REACTION_10, REACTION_100, COMMENT, REPLY)
  - NotificationTargetType StrEnum (SETUP, AIDOC)
  - 필드: type, target_type, target_id, receiver_user_id, actor_user_id, actor_name, preview, is_read
  - actor_name/preview 는 생성 시점 스냅샷 (이후 불변)
```

**API 계획:**

```
- [신규] GET /api/notifications
  - Params: page (default 0), size (default 10)
  - Response: NotificationListResponse { data: list[NotificationResponse], total: int, has_more: bool }
  - 상태 코드: 200
  - 인증: 필요
  - 사용 service: NotificationReader.get_by_receiver, NotificationReader.count_by_receiver

- [신규] GET /api/notifications/unread-count
  - Response: UnreadCountResponse { count: int }
  - 상태 코드: 200
  - 인증: 필요
  - 사용 service: NotificationReader.count_unread

- [신규] PUT /api/notifications/{id}/read
  - Response: 200, 본문 없음 (response_class=Response)
  - 인증: 필요
  - 사용 service: NotificationMarkReadUsecase

- [신규] PUT /api/notifications/read-all
  - Response: 200, 본문 없음
  - 인증: 필요
  - 사용 service: NotificationMarkAllReadUsecase

- [신규] DELETE /api/notifications/{id}
  - Response: 204
  - 인증: 필요
  - 사용 service: NotificationDeleteUsecase

- [신규] DELETE /api/notifications
  - Response: 204
  - 인증: 필요
  - 사용 service: NotificationDeleteAllUsecase
```

---

### assess-usecase

**충족 여부: 미충족** — Notification 관련 service 없음. 전부 신규.

**infrastructure 요청:**

```
신규 요청:
  - NotificationRepository
    - find_write_by_id(identity: NotificationIdentity) -> Notification | None     # 선검증
    - find_by_receiver_user_id(user_id: int, page: int, size: int) -> list[NotificationRead]   # 읽기
    - count_by_receiver_user_id(user_id: int) -> int                                # 읽기
    - count_unread_by_receiver_user_id(user_id: int) -> int                         # 읽기
    - save(notification: Notification) -> Notification                              # 쓰기
    - mark_as_read(identity: NotificationIdentity) -> None                          # 원자 연산 (update 문)
    - mark_all_as_read(receiver_user_id: int) -> None                               # 원자 연산 (bulk update 문)
    - delete_by_id(identity: NotificationIdentity) -> None                          # soft delete
    - delete_all_by_receiver_user_id(receiver_user_id: int) -> None                 # soft delete (bulk update 문)
```

**model 요청:**

```
상위(assess-api)에서 도출한 model 요청과 동일.
(NotificationCreateCommand 는 별도 계획에서 추가 예정)
```

**service 계획:**

```
- [신규] NotificationReader (Foo 패턴 아님 — Writer 가 있으므로 Bar)
  - get_by_receiver(user_id, page, size) -> list[NotificationRead]
  - count_by_receiver(user_id) -> int
  - count_unread(user_id) -> int
  - 의존: NotificationRepository
  - 트랜잭션: 없음

- [신규] NotificationWriter
  - mark_as_read(identity: NotificationIdentity) -> None
  - mark_all_as_read(receiver_user_id: int) -> None
  - delete(identity: NotificationIdentity) -> None
  - delete_all(receiver_user_id: int) -> None
  - 의존: TransactionManager, NotificationRepository
  - 트랜잭션: 모든 public 메서드 @transactional
  - (create 는 별도 계획에서 추가 예정)

- [신규] Usecase 4개 (Baz 패턴 — 수신자 확인 후 Writer 호출)
  - NotificationMarkReadUsecase.execute(user_id, identity)
    - find_write_by_id → 없으면 NotificationNotFoundException → receiver_user_id != user_id 면 ForbiddenException → writer.mark_as_read
  - NotificationMarkAllReadUsecase.execute(user_id) → writer.mark_all_as_read(user_id)
  - NotificationDeleteUsecase.execute(user_id, identity) → 선검증 동일 → writer.delete
  - NotificationDeleteAllUsecase.execute(user_id) → writer.delete_all(user_id)
  - 의존: TransactionManager, NotificationRepository, NotificationWriter
  - 트랜잭션: execute @transactional
  - 소유권 체크: 단건 2개는 있음, bulk 2개는 user_id 가 곧 필터라 없음

- DI: NotificationProvider (REQUEST)
  - repository = provide(DefaultNotificationRepository, provides=NotificationRepository)
  - reader = provide(DefaultNotificationReader, provides=NotificationReader)
  - writer = provide(DefaultNotificationWriter, provides=NotificationWriter)
  - Usecase 4개 provide(...)
```

---

### assess-infra

**충족 여부: 미충족** — NotificationRepository 없음. 신규 생성.

**model 요청:**

```
상위에서 도출한 model 요청과 동일.
infra 추가 관점:
  - NotificationIdentity (notification_id: int)
  - Notification Write 모델: save / find_write_by_id 용
  - NotificationRead: find_by_receiver_user_id 반환용. JOIN 없음 (actor_name/preview 가 스냅샷)
```

**infrastructure 계획:**

```
- [신규] NotificationRepository (Bar 패턴, Stats 없음)
  - find_write_by_id(NotificationIdentity) -> Notification | None        종류: 선검증
  - find_by_receiver_user_id(int, int, int) -> list[NotificationRead]    종류: 읽기
  - count_by_receiver_user_id(int) -> int                                종류: 읽기
  - count_unread_by_receiver_user_id(int) -> int                         종류: 읽기
  - save(Notification) -> Notification                                   종류: 쓰기
  - mark_as_read(NotificationIdentity) -> None                           종류: 원자 연산 (update 문, updated_at 포함)
  - mark_all_as_read(int) -> None                                        종류: 원자 연산 (bulk update 문)
  - delete_by_id(NotificationIdentity) -> None                           종류: soft delete
  - delete_all_by_receiver_user_id(int) -> None                          종류: soft delete (bulk update 문: is_deleted=true, deleted_at=now)
- 어댑터 갱신 필요: 신규라 해당 없음
```

---

### assess-model

**충족 여부: 미충족** — Notification 도메인 없음. 전부 신규.

**model 계획:**

```
- 패턴: Bar (Write + Read, JOIN 없음. is_read 전이가 있어 Qux 요소 일부 합성)

- [신규] NotificationType (notification_type.py)
  - 값: REACTION_1, REACTION_10, REACTION_100, COMMENT, REPLY

- [신규] NotificationTargetType (notification_target_type.py)
  - 값: SETUP, AIDOC

- [신규] Notification (notification.py)
  - 필드: notification_id: int | None, type: NotificationType, target_type: NotificationTargetType,
          target_id: int, receiver_user_id: int, actor_user_id: int | None, actor_name: str | None,
          preview: str, is_read: bool, created_at: datetime, updated_at: datetime
  - 팩토리: create(type, target_type, target_id, receiver_user_id, actor_user_id, actor_name, preview)
    → is_read=False, created_at=updated_at=utcnow()
  - 상태 변경: mark_read() -> Self  (동사 메서드, replace(is_read=True, updated_at=now))
  - to_read: () -> NotificationRead  (인자 없음, Bar)

- [신규] NotificationRead (notification_read.py)
  - 필드: Notification 필드 전부, notification_id: int (None 아님)

- [신규] NotificationIdentity (notification_identity.py)
  - 필드: notification_id: int

- [신규] NotificationNotFoundException (exception/notification/)
  - 기본 메시지 사용
```

---

## Phase 2: 구현 계획

### 평가 결과 요약

| 레이어 | 충족 여부 | 비고 |
|--------|----------|------|
| api | 미충족 | 6개 엔드포인트 신규 |
| service | 미충족 | Reader + Writer + Usecase 4개 신규 |
| infra | 미충족 | NotificationRepository 신규 |
| model | 미충족 | Notification 도메인 전체 신규 |

**케이스 A: 새 도메인** — 전 레이어 구현.

### 설계 결정 사항 (확정)

**1. soft delete 사용**
- `notifications` 테이블은 4필드 보유 (C-4). delete = soft delete (`delete_by_id` 는 `soft_delete()`, bulk 는 `update()` 문).

**2. is_read 상태 변경은 `update()` 문**
- `mark_as_read`: `update(NotificationEntity).where(id == :id).values(is_read=True, updated_at=utcnow())`
- `mark_all_as_read`: `.where(receiver_user_id == :uid, is_read.is_(False), is_deleted.is_(False))`
- 도메인 모델 로드 없이 직접 UPDATE. 원자적이라 동시성 TC 불필요.

**3. 이 계획의 범위: 조회 + 삭제 + 읽음 처리만**
- 알림 생성(create)은 별도 계획으로 분리.
- NotificationWriter 에 create 메서드 없음. NotificationCreateCommand 없음.

**4. 트랜잭션**
- Writer 4개 메서드와 Usecase 4개 `execute` 에 `@transactional`. 단건 Usecase 는 선검증(`find_write_by_id`) + Writer 호출이 한 트랜잭션.

**5. 응답 형식 — 프로젝트 컨벤션**
- PUT read → 200 본문 없음 (`response_class=Response`), DELETE → 204.

**6. DI / 라우터 등록**
- `app/providers/notification_provider.py` 생성 + `di.py` DOMAIN_PROVIDERS 에 추가 (add-usecase)
- `app/routers.py` ROUTERS 에 `notification_router` 추가 (add-api)

### 구현 순서

```
1. add-model
   - NotificationType, NotificationTargetType StrEnum
   - Notification (Write), NotificationRead (Read), NotificationIdentity
   - NotificationNotFoundException

2. add-infrastructure
   - NotificationRepository ABC

3. add-entity-specs
   - NotificationEntity (4필드 직접 선언, Foo 패턴)
   - entities.py 등록
   - migration: alembic revision --autogenerate -m "notification"

4. add-query
   - DefaultNotificationRepository (select / update 문, @transactional)

5. add-usecase
   - NotificationReader + DefaultNotificationReader
   - NotificationWriter + DefaultNotificationWriter
   - Usecase 4개
   - NotificationProvider + di.py 등록

6. add-api
   - notification_router (DishkaSyncRoute, 6 핸들러)
   - NotificationResponse, NotificationListResponse, UnreadCountResponse
   - routers.py 등록
```

---

## Phase 3: 구현 진행

### 1. add-model (완료)

생성된 파일:
- `packages/model/src/{pkg}/model/notification/notification_type.py` — StrEnum (REACTION_1, REACTION_10, REACTION_100, COMMENT, REPLY)
- `packages/model/src/{pkg}/model/notification/notification_target_type.py` — StrEnum (SETUP, AIDOC)
- `packages/model/src/{pkg}/model/notification/notification_identity.py` — frozen dataclass, `notification_id: int`
- `packages/model/src/{pkg}/model/notification/notification.py` — Write 모델 (frozen dataclass)
  - `create()` 팩토리, `mark_read()` 상태변경, `to_read()` 변환
- `packages/model/src/{pkg}/model/notification/notification_read.py` — Read 모델 (Notification 과 동일 필드, JOIN 없음)
- `packages/exception/src/{pkg}/exception/notification/notification_not_found_exception.py`

검증: `uv run ruff check packages/model packages/exception && uv run mypy` 통과.
specs/models.md 업데이트 완료.
