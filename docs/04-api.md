# 04 — API

## REST (gateway)

База: `/api/v1`. Формат: JSON, `snake_case`. Авторизация: заголовок `Authorization: Bearer <access>`. Исключения — `register`, `login` и `refresh`: refresh-токен приходит в cookie.

### Общие соглашения
- **Ошибки:**
  ```json
  { "error": { "code": "validation_failed", "message": "invalid input",
               "details": [{ "field": "amount", "reason": "must be > 0" }] } }
  ```
- **Пагинация** курсорная: `?limit=50&cursor=<opaque>`, в ответе `next_cursor`. Курсор — base64 от `(occurred_at, id)`.
- **Идемпотентность создания:** необязательный заголовок `Idempotency-Key`. Ответ на ключ хранится в Redis 24 ч.
- **Деньги:** `{"amount": 12345, "currency": "RUB"}`, где `amount` — минорные единицы.
- **Даты:** `occurred_at` — RFC 3339 с зоной, параметры отчёта — `YYYY-MM-DD`.
- В каждом ответе есть заголовок `X-Request-Id`.

### Auth
| Метод | Путь | Описание |
|---|---|---|
| POST | `/auth/register` | `{email, username, password, timezone, base_currency}` → 201, `{user, access_token}` + refresh cookie |
| GET | `/auth/username-available?u=` | Проверка свободного username для формы регистрации |
| POST | `/auth/login` | `{email, password}` → `{user, access_token}` + refresh cookie |
| POST | `/auth/refresh` | refresh из cookie → новая пара |
| POST | `/auth/logout` | Отзыв текущего refresh. `?all=true` отзывает все сессии |
| GET | `/me` | Профиль |
| PATCH | `/me` | `{timezone, username}`. Смена зоны запускает пересборку агрегатов личных счетов |

### Счета
| Метод | Путь | Описание |
|---|---|---|
| GET | `/accounts` | Счета пользователя с его ролью и текущим остатком |
| POST | `/accounts` | `{type, name, currency?, timezone?, split_mode?}` |
| GET | `/accounts/{id}` | |
| PATCH | `/accounts/{id}` | `{name?, timezone?, split_mode?}` — только owner |
| POST | `/accounts/{id}/archive` | |
| POST | `/transfers` | `{from_account_id, to_account_id, amount, occurred_at, note?}` |

Участники, приглашения, балансы и расчёты описаны в [09-social.md](09-social.md#api).

### Категории
Набор категорий определяется счётом: личные счета используют личный набор, совместные — свой. Параметр `account_id` обязателен для набора совместного счёта, без него возвращается личный набор.

| Метод | Путь | Описание |
|---|---|---|
| GET | `/categories` | Дерево. `?account_id`, `?type=expense`, `?include_archived=true` |
| POST | `/categories` | `{name, type, parent_id?, budget_class?}` |
| PATCH | `/categories/{id}` | `{name?, parent_id?, budget_class?, sort_order?}` |
| DELETE | `/categories/{id}` | Удаление или архивация (см. правила). Ответ: `{"result": "deleted" \| "archived"}` |
| POST | `/categories/{id}/restore` | Разархивировать |

### Транзакции
| Метод | Путь | Описание |
|---|---|---|
| GET | `/transactions` | Фильтры: `account_id` (можно несколько; по умолчанию все счета пользователя), `from`, `to`, `type`, `kind`, `category_id`, `include_children`, `created_by`, `q`, `goal_id`; пагинация |
| POST | `/transactions` | `{account_id, category_id, amount, occurred_at, note?, goal_id?, paid_by?, splits?}` |
| GET | `/transactions/{id}` | |
| PATCH | `/transactions/{id}` | Частичное обновление |
| DELETE | `/transactions/{id}` | Мягкое удаление |

### Отчёты и рекомендации
| Метод | Путь | Описание |
|---|---|---|
| GET | `/reports/summary` | `?from&to&granularity=auto&category_depth=2&scope=personal&include_shared=false` или `?scope=account&account_id=…` → `SummaryReport` |
| GET | `/recommendations` | `?from&to&scope&account_id`, по умолчанию текущий месяц по личным счетам. Только список рекомендаций |

### Цели
| Метод | Путь | Описание |
|---|---|---|
| GET | `/goals` | `?account_id&status=active` (по умолчанию — цели всех счетов пользователя) |
| POST | `/goals` | `{account_id, name, target_amount, deadline?}` |
| GET | `/goals/{id}` | Цель + прогресс (общий и по участникам) + `required_monthly`, `on_track` |
| PATCH | `/goals/{id}` | `{name?, target_amount?, deadline?, status?}` |
| DELETE | `/goals/{id}` | Архивация |
| GET | `/goals/{id}/contributions` | Пагинация |
| POST | `/goals/{id}/contributions` | `{amount, occurred_at, note?}`. Ручной взнос (отрицательный — изъятие) |
| DELETE | `/goals/{id}/contributions/{cid}` | Удаление ручного взноса |

### Служебные
| Путь | Назначение |
|---|---|
| `/healthz` | liveness — процесс жив |
| `/readyz` | readiness — доступны зависимости (gRPC-сервисы, Redis) |
| `/metrics` | Prometheus (отдельный порт, наружу не публикуется) |

OpenAPI-спека хранится в `api/openapi/gateway.yaml` и является контрактом с фронтом. При изменении эндпоинта она обновляется в том же PR.

## gRPC

Контракты лежат в `api/proto/graspy/<service>/v1`, генерация через `buf generate`, линтер — `buf lint`, проверка обратной совместимости — `buf breaking`.

**Metadata:** gateway передаёт `x-user-id` и `x-request-id`. Серверный интерсептор кладёт их в `context`. Сервисы **не** принимают `user_id` текущего пользователя в теле запроса — только из metadata. `account_id` передаётся в теле, и право на него проверяет сервис.

Контракты `AccountService` (finance) и сервисов social — в [09-social.md](09-social.md#grpc).

```protobuf
// auth/v1/auth.proto
service AuthService {
  rpc Register(RegisterRequest) returns (AuthResponse);
  rpc Login(LoginRequest) returns (AuthResponse);
  rpc Refresh(RefreshRequest) returns (AuthResponse);
  rpc Logout(LogoutRequest) returns (google.protobuf.Empty);
  rpc GetProfile(google.protobuf.Empty) returns (Profile);
  rpc UpdateProfile(UpdateProfileRequest) returns (Profile);
}
```

```protobuf
// finance/v1/finance.proto
service CategoryService {
  rpc GetCategoryTree(GetCategoryTreeRequest) returns (CategoryTree);
  rpc CreateCategory(CreateCategoryRequest) returns (Category);
  rpc UpdateCategory(UpdateCategoryRequest) returns (Category);
  rpc DeleteCategory(DeleteCategoryRequest) returns (DeleteCategoryResponse);
}

service TransactionService {
  rpc CreateTransaction(CreateTransactionRequest) returns (Transaction);
  rpc GetTransaction(GetTransactionRequest) returns (Transaction);
  rpc ListTransactions(ListTransactionsRequest) returns (ListTransactionsResponse);
  rpc UpdateTransaction(UpdateTransactionRequest) returns (Transaction);
  rpc DeleteTransaction(DeleteTransactionRequest) returns (google.protobuf.Empty);
  rpc CreateTransfer(CreateTransferRequest) returns (Transfer);
  // для пересборки агрегатов в analytics (по account_id)
  rpc StreamTransactions(StreamTransactionsRequest) returns (stream Transaction);
}

service GoalService {
  rpc CreateGoal(CreateGoalRequest) returns (Goal);
  rpc GetGoal(GetGoalRequest) returns (Goal);
  rpc ListGoals(ListGoalsRequest) returns (ListGoalsResponse);
  rpc UpdateGoal(UpdateGoalRequest) returns (Goal);
  rpc AddContribution(AddContributionRequest) returns (GoalContribution);
  rpc ListContributions(ListContributionsRequest) returns (ListContributionsResponse);
  rpc DeleteContribution(DeleteContributionRequest) returns (google.protobuf.Empty);
}

message Money {
  int64 amount = 1;   // минорные единицы
  string currency = 2;
}
```

```protobuf
// analytics/v1/analytics.proto
service AnalyticsService {
  rpc GetSummaryReport(GetSummaryReportRequest) returns (SummaryReport);
  rpc GetRecommendations(GetRecommendationsRequest) returns (RecommendationList);
  rpc RebuildAggregates(RebuildAggregatesRequest) returns (google.protobuf.Empty);
}
```

**Таймауты по умолчанию:** CRUD — 2 с, отчёт — 5 с. Клиент всегда передаёт дедлайн через `context`.

## Kafka

### Топики
| Топик | Продюсер | Консьюмеры | Ключ | Партиции |
|---|---|---|---|---|
| `auth.users.v1` | auth | finance, social | `user_id` | 3 |
| `finance.accounts.v1` | finance | social, analytics | `account_id` | 3 |
| `finance.transactions.v1` | finance | analytics, social | `account_id` | 6 |
| `finance.goals.v1` | finance | analytics, social | `account_id` | 3 |
| `social.v1` | social | finance | `user_id` | 3 |
| `*.dlq` | консьюмеры | человек / скрипт переотправки | исходный | 1 |

Consumer group = имя сервиса (`analytics`, `finance`, `social`).

### Конверт события
Сериализация — Protobuf (схемы в `api/proto/graspy/events/v1`). Допустим JSON на старте, если так проще отлаживать. Выбрать один вариант и зафиксировать.

```json
{
  "event_id": "0b9f…",               // UUID, ключ идемпотентности
  "event_type": "transaction.updated",
  "occurred_at": "2026-10-28T10:00:00Z",
  "user_id": "…",                    // инициатор действия
  "account_id": "…",                 // для финансовых событий
  "version": 1,
  "payload": { "before": { … }, "after": { … } }
}
```

### События
| Тип | Payload |
|---|---|
| `user.registered` | `{user_id, username, email, timezone, base_currency}` |
| `user.username_changed` | `{user_id, old, new}` → social обновляет профиль |
| `user.timezone_changed` | `{user_id, old, new}` → analytics пересобирает агрегаты личных счетов |
| `user.deleted` | `{user_id}` → finance выходит из счетов / передаёт владение, social анонимизирует |
| `transaction.created` | `{after}` |
| `transaction.updated` | `{before, after}` |
| `transaction.deleted` | `{before}` |
| `goal.created` / `goal.updated` / `goal.achieved` | `{goal}` |
| `goal.contribution_added` / `goal.contribution_removed` | `{goal_id, contribution}` |
| `transfer.created` / `transfer.deleted` | `{transfer_group_id, from_account_id, to_account_id, amount}` |

Транзакции в событиях несут `created_by`, `paid_by`, `kind` и `splits`. События счетов, участников, приглашений и социальные события перечислены в [09-social.md](09-social.md#kafka).

### Правила эволюции
- Можно только добавлять поля. Переименование или удаление поля означает новую версию топика (`.v2`) и период двойной публикации.
- Консьюмер игнорирует неизвестные `event_type`.
