# 03 — Доменная модель и бизнес-правила

## Деньги

```go
// internal/platform/money
type Money struct {
    Amount   int64  // минорные единицы: 12345 = 123.45
    Currency string // ISO 4217, например "RUB"
}
```

- Сумма транзакции всегда положительна (`amount > 0`), знак определяется типом категории.
- Арифметика выполняется только между значениями одной валюты, иначе возвращается ошибка `ErrCurrencyMismatch`.
- Проценты и доли считаются в `float64` или `decimal` **только для отображения**. Обратно в `Money` значения не превращаются.
- В API сумма передаётся строкой в минорных единицах (`"12345"`) или целым числом. Фронт сам форматирует её по локали.

## Пользователь (auth)

| Поле | Правила |
|---|---|
| `email` | Нормализуется (trim, lower), уникален |
| `username` | `^[a-z0-9_]{3,30}$`, уникален без учёта регистра, менять можно раз в 30 дней. Список зарезервированных имён (`admin`, `support`, …) хранится в конфиге |
| `password` | Минимум 8 символов, хеш argon2id (параметры в конфиге) |
| `timezone` | IANA (`Europe/Moscow`), проверяется через `time.LoadLocation` |
| `base_currency` | ISO 4217, в MVP после создания не меняется |

### Токены
- **Access** — JWT (EdDSA или HS256), TTL 15 мин. Claims: `sub` (user_id), `exp`, `iat`, `jti`.
- **Refresh** — случайные 32 байта (base64url), TTL 30 дней. В Redis хранится только SHA-256 токена.
- **Ротация.** При `Refresh` старый токен удаляется и выдаётся новый. Повторное использование уже использованного токена означает кражу: отзываются все сессии пользователя (семейство токенов).

## Счета и доступ

```
Account {
  id, type (personal|shared), name, currency, timezone,
  owner_id, split_mode (none|equal|custom) — только для shared,
  archived_at?, created_at
}
AccountMember { account_id, user_id, role (owner|editor|viewer), joined_at }
```

- **Каждая** финансовая сущность (транзакция, цель, взнос, категория совместного счёта) принадлежит счёту. `account_id` — главный ключ изоляции данных вместо `user_id`.
- Личный счёт: ровно один участник, его роль `owner`. Пригласить в личный счёт нельзя. Чтобы делить деньги, нужно создать совместный счёт.
- У совместного счёта ровно один `owner`. Правила ролей, приглашений и выхода описаны в [09-social.md](09-social.md#роли-и-права).
- Часовой пояс совместного счёта задаётся при создании (берётся пояс владельца) и меняется владельцем. Отчёты по счёту считаются в его поясе. Отчёт по личным счетам считается в поясе пользователя: при `user.timezone_changed` finance обновляет `timezone` всех его личных счетов и публикует `account.updated`.
- Валюта счёта задаётся при создании и не меняется. Личные счета в MVP создаются только в базовой валюте пользователя.

### Проверка доступа
Каждый use-case в finance начинается с одного и того же шага:

```go
role, err := s.access.Role(ctx, accountID, userID) // ErrNotFound, если не участник
if err != nil {
    return err
}
if !role.Can(domain.ActionCreateTransaction) {
    return domain.ErrInsufficientRole
}
```

- Не участник получает `NotFound`, а не `Forbidden`: так нельзя узнать, существует ли счёт.
- Матрица `role → actions` объявлена в одном месте (`domain/permissions.go`) и покрыта табличным тестом.
- Роль кешируется в Redis `acl:{account_id}:{user_id}` на 60 с. Кеш сбрасывается при `member.*` событиях внутри finance.
- В списочных запросах (`ListTransactions` по нескольким счетам) фильтр строится через `JOIN account_members ON user_id = $1`, а не через список ID от клиента.

### Переводы
- Перевод создаёт две связанные транзакции: `kind = transfer_out` в счёте-источнике и `kind = transfer_in` в счёте-получателе, с общим `transfer_group_id`. Категории у них нет.
- Пользователь должен иметь роль `editor` или выше в обоих счетах. Валюты счетов должны совпадать (ограничение MVP).
- Переводы не входят в доходы и расходы отчёта. Удаление одной половины удаляет и вторую.

## Категории (finance)

```
Category {
  id, user_id? | account_id?    // личный набор пользователя ИЛИ набор совместного счёта
  parent_id?, name, type (income|expense),
  budget_class? (needs|wants|savings) — только у корня expense,
  depth (1..3), sort_order, archived_at?
}
```

### Инварианты
0. Заполнено ровно одно из полей `user_id` и `account_id`. Транзакция личного счёта использует категорию из набора пользователя-владельца, транзакция совместного счёта — из набора этого счёта. Родитель всегда из того же набора.
1. `depth = parent.depth + 1`, корень имеет `depth = 1`, всегда `depth ≤ 3`.
2. `type` дочерней категории совпадает с `type` родителя.
3. `budget_class` задаётся только у корневых категорий расходов. Дочерние наследуют его при чтении, у них в БД `NULL`.
4. Имя уникально среди соседей (`user_id, parent_id, lower(name)`), без учёта архивных.
5. Перемещение (смена `parent_id`) разрешено, если после него глубина поддерева не превысит 3 и не образуется цикл. Проверка: `parent.depth + высота поддерева ≤ 3`.
6. Удаление: если в поддереве нет транзакций, категория удаляется физически, иначе только `archived_at = now()` для всего поддерева.
7. В архивную категорию нельзя добавить новую транзакцию. Старые транзакции остаются в отчётах.

Инварианты 1–2 дублируются триггером в БД (см. [05-database.md](05-database.md)). Сервис проверяет их сам, чтобы вернуть понятную ошибку, а триггер страхует от гонок и багов.

### Категории по умолчанию
Создаются по событию `user.registered` вместе с личным счётом «Основной» (идемпотентно: если у пользователя уже есть категории, ничего не делается). Тот же шаблон копируется в набор нового совместного счёта.

```
Расходы
├── Жильё (needs)            ├── Коммуналка, Аренда/ипотека
├── Продукты (needs)
├── Транспорт (needs)        ├── Общественный, Такси, Авто
├── Здоровье (needs)
├── Еда вне дома (wants)     ├── Рестораны, Кофе, Доставка
├── Развлечения (wants)      ├── Подписки, Хобби
├── Одежда (wants)
└── Сбережения (savings)     ├── Подушка, Инвестиции, Цели
Доходы
├── Зарплата
├── Подработка
└── Прочее
```

## Транзакции (finance)

```
Transaction {
  id, account_id, created_by, paid_by?,
  kind (regular|transfer_in|transfer_out|settlement),
  category_id?   // обязательна для regular, пуста для остальных
  amount (int64), currency,
  occurred_at (timestamptz), note?, goal_id?, transfer_group_id?,
  created_at, updated_at, deleted_at?
}
```

- `currency` равна валюте счёта.
- `created_by` — кто создал запись, `paid_by` — кто фактически заплатил (имеет смысл в совместном счёте, по умолчанию равен `created_by`). Сплиты и расчёты описаны в [09-social.md](09-social.md#разделение-расходов).
- Права: создавать может `editor`+, менять и удалять свою транзакцию — автор, чужую — только `owner`.
- `occurred_at` не может быть больше `now() + 1 день`, чтобы опечатки в годе ловились сразу.
- `goal_id` разрешён только для категорий класса `savings`. Такая транзакция одновременно является взносом в цель.
- Удаление мягкое (`deleted_at`), в событие уходит `transaction.deleted`.
- Каждое изменение публикует событие с полным снимком **до** и **после**. analytics вычитает старое значение из агрегатов и добавляет новое.

### Фильтр «категория с поддеревом»
Запрос `category_id=X&include_children=true` раскрывается в `category_id IN (X, дети X, внуки X)`. Глубина ≤ 3, поэтому достаточно двух уровней self-join или рекурсивного CTE.

## Отчёты (analytics)

### Входные параметры
| Параметр | Правила |
|---|---|
| `from`, `to` | Даты (`YYYY-MM-DD`) в часовом поясе пользователя, `to` включительно. Максимум 5 лет |
| `granularity` | `day` / `week` / `month` / `auto`. Для `auto`: ≤ 31 день → day, ≤ 180 дней → week, иначе month |
| `category_depth` | Глубина разбивки, 1–3, по умолчанию 2 |
| `scope` | `personal` (по умолчанию: все личные счета пользователя) или `account` (один счёт по `account_id`, личный или совместный) |
| `include_shared` | Только для `personal`: добавить **долю пользователя** в тратах совместных счетов той же валюты. Доходы совместных счетов не добавляются |

Отчёт по одному совместному счёту объединяет категории его набора. Отчёт `personal` с `include_shared` показывает доли совместных счетов отдельной веткой «Совместные: <название счёта>», потому что категории у них другие.

### Содержимое отчёта
```
SummaryReport {
  period:           {from, to}
  previous_period:  {from, to}         // такая же длина, сразу перед текущим
  totals:           {income, expense, net, savings_rate}
  previous_totals:  {income, expense, net, savings_rate}
  deltas:           {income_pct, expense_pct, net_pct}
  by_category:      [CategoryNode{ id, name, type, amount, share, children[] }]
  by_budget_class:  {needs, wants, savings}  // суммы и доли от дохода
  series:           [{bucket_start, income, expense, net}]
  top_expenses:     [CategoryNode] (5)
  recommendations:  [Recommendation]
}
```

- `savings_rate = net / income`. Если `income = 0`, поле равно `null`, и показатели в процентах тоже `null`.
- `share` — доля от суммы своего типа за период.
- Источник данных — таблицы `daily_aggregates` (по счёту) и `daily_share_aggregates` (доли участников). Переводы и расчёты в агрегаты не попадают.
- Перед сохранением агрегата `occurred_at` переводится в локальную дату **счёта**. После смены часового пояса счёта запускается пересборка его агрегатов.

## Цели (finance)

```
Goal {
  id, account_id, created_by, name, target_amount, currency,
  deadline? (date), status (active|achieved|archived),
  created_at, achieved_at?
}
GoalContribution {
  id, goal_id, user_id (кто внёс), amount (может быть < 0 — изъятие),
  transaction_id?, occurred_at, note?
}
```

- В совместном счёте прогресс показывается и общим, и по участникам: `SUM(amount) GROUP BY user_id`.
- Изъятие из общей цели делает `owner` или сам автор взноса в пределах своего вклада.

- `current_amount = SUM(contributions.amount)`. Хранится денормализованно в `goals.current_amount` и обновляется в той же транзакции, что и взнос.
- Если `current_amount ≥ target_amount`, статус становится `achieved`, а цель получает `achieved_at` и событие `goal.achieved`. При изъятии ниже цели статус возвращается в `active`.
- Взнос через транзакцию: создаётся транзакция в категории класса `savings` с `goal_id`, и в той же БД-транзакции добавляется `goal_contribution` с `transaction_id`. Удаление транзакции удаляет и взнос.

### Расчёт для цели с дедлайном
```
remaining        = max(target - current, 0)
months_left      = max(месяцев между today и deadline, округлить вверх, 1)
required_monthly = ceil(remaining / months_left)
avg_free_cash    = средний net за последние 3 полных месяца
on_track         = required_monthly ≤ avg_free_cash
```

## Рекомендации

Движок — набор независимых правил. Каждое правило получает `RuleInput` (итоги текущего и предыдущих периодов, агрегаты по категориям, цели, профиль) и возвращает 0..N рекомендаций.

```go
type Rule interface {
    ID() string
    Evaluate(ctx context.Context, in RuleInput) []Recommendation
}

type Recommendation struct {
    RuleID   string
    Severity Severity          // info | warning | critical
    Title    string
    Message  string            // готовый текст для UI
    Facts    map[string]any    // цифры, на которых основан вывод
}
```

Пороги правил хранятся в конфиге analytics, а не в коде.

| ID | Правило | Условие срабатывания | Severity |
|---|---|---|---|
| `savings_rate_low` | Норма сбережений | `savings_rate < 10%` (critical, если `< 0`) | warning / critical |
| `savings_rate_good` | Позитивная обратная связь | `savings_rate ≥ 20%` | info |
| `budget_50_30_20` | Правило 50/30/20 | `needs > 50%` или `wants > 30%` дохода, либо `savings < 20%` дохода | warning |
| `category_spike` | Рост категории | Расход категории за месяц > среднего за 3 предыдущих месяца на ≥ 30% и на ≥ 3 000 ₽ (абсолютный порог отсекает мелочь) | warning |
| `emergency_fund` | Подушка безопасности | Накопления в категории «Подушка» < 3 × средних месячных расходов | info / warning (< 1 мес.) |
| `goal_off_track` | Цель не успевает | `on_track = false` | warning |
| `goal_required_monthly` | Сколько откладывать | Есть активная цель с дедлайном | info |
| `subscriptions_share` | Подписки | Подписки > 5% расходов | info |
| `expense_over_income` | Тратите больше, чем зарабатываете | `expense > income` за период | critical |
| `shared_debt_open` | Непогашенный долг в совместном счёте | `balance < 0` дольше 30 дней | info |
| `shared_share_high` | Большая доля совместных трат | Доля пользователя в тратах совместных счетов > 40% его личных расходов | info |

Для отчёта по совместному счёту правила считаются по данным счёта; правила, которым нужен личный доход (`budget_50_30_20`, `emergency_fund`), пропускаются.

Правила тестируются табличными юнит-тестами: вход → ожидаемый набор `RuleID` и `Facts`.

## Доменные ошибки

Объявляются в `domain` каждого сервиса и маппятся в gRPC-коды, а gateway переводит их в HTTP:

| Ошибка | gRPC | HTTP |
|---|---|---|
| `ErrNotFound` | `NotFound` | 404 |
| `ErrAlreadyExists` (email, имя категории) | `AlreadyExists` | 409 |
| `ErrValidation` (+ поля) | `InvalidArgument` | 422 |
| `ErrCategoryDepthExceeded`, `ErrCategoryTypeMismatch`, `ErrCategoryArchived` | `FailedPrecondition` | 422 |
| `ErrInvalidCredentials`, `ErrTokenExpired`, `ErrTokenReused` | `Unauthenticated` | 401 |
| `ErrForbidden`, `ErrInsufficientRole` | `PermissionDenied` | 403 |
| `ErrNotMember` (не участник счёта) | `NotFound` | 404 |
| `ErrMemberLimit`, `ErrOwnerCannotLeave`, `ErrSplitSumMismatch` | `FailedPrecondition` | 422 |
| `ErrRateLimited` | `ResourceExhausted` | 429 |
