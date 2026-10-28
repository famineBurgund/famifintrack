# 05 — Данные

## PostgreSQL

Один кластер, у каждого сервиса **своя база** (`graspy_auth`, `graspy_finance`, `graspy_analytics`, `graspy_social`) и свой пользователь БД с правами только на неё. Драйвер — `pgx/v5` (`pgxpool`). SQL пишется руками или генерируется через `sqlc`; ORM не используется.

Общие правила:
- PK — `uuid` (UUIDv7, генерируется в Go: сортируется по времени, удобно для курсоров и индексов).
- Время — `timestamptz`, значения в UTC.
- Деньги — `bigint` (минорные единицы) + `char(3)` валюты.
- Финансовые таблицы содержат `account_id`, и **все** запросы к ним проходят через проверку членства в счёте (см. [03-domain.md](03-domain.md#проверка-доступа)). Таблицы без счёта (профиль, личные категории, уведомления) фильтруются по `user_id`.

Схема social описана в [09-social.md](09-social.md#данные-social).

### auth

```sql
CREATE TABLE users (
    id             uuid PRIMARY KEY,
    email          text NOT NULL,
    username       text NOT NULL,
    username_changed_at timestamptz,
    password_hash  text NOT NULL,
    timezone       text NOT NULL DEFAULT 'UTC',
    base_currency  char(3) NOT NULL DEFAULT 'RUB',
    created_at     timestamptz NOT NULL DEFAULT now(),
    updated_at     timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX users_email_uq ON users (lower(email));
CREATE UNIQUE INDEX users_username_uq ON users (lower(username));

-- общая для всех сервисов форма outbox
CREATE TABLE outbox (
    id            bigserial PRIMARY KEY,
    event_id      uuid NOT NULL UNIQUE,
    topic         text NOT NULL,
    key           text NOT NULL,
    payload       bytea NOT NULL,
    created_at    timestamptz NOT NULL DEFAULT now(),
    published_at  timestamptz
);
CREATE INDEX outbox_unpublished_idx ON outbox (id) WHERE published_at IS NULL;
```

Refresh-токены хранятся в Redis (см. ниже). Если понадобится список активных сессий в UI, их можно перенести в таблицу `sessions`.

### finance

```sql
CREATE TYPE category_type AS ENUM ('income', 'expense');
CREATE TYPE budget_class  AS ENUM ('needs', 'wants', 'savings');
CREATE TYPE account_type  AS ENUM ('personal', 'shared');
CREATE TYPE member_role   AS ENUM ('owner', 'editor', 'viewer');
CREATE TYPE tx_kind       AS ENUM ('regular', 'transfer_in', 'transfer_out', 'settlement');

CREATE TABLE accounts (
    id           uuid PRIMARY KEY,
    type         account_type NOT NULL,
    name         text NOT NULL,
    currency     char(3) NOT NULL,
    timezone     text NOT NULL,
    owner_id     uuid NOT NULL,
    split_mode   text NOT NULL DEFAULT 'none' CHECK (split_mode IN ('none','equal','custom')),
    archived_at  timestamptz,
    created_at   timestamptz NOT NULL DEFAULT now(),
    CHECK (type = 'shared' OR split_mode = 'none')
);

CREATE TABLE account_members (
    account_id  uuid NOT NULL REFERENCES accounts (id),
    user_id     uuid NOT NULL,
    role        member_role NOT NULL,
    joined_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (account_id, user_id)
);
CREATE INDEX account_members_user_idx ON account_members (user_id);
CREATE UNIQUE INDEX account_members_one_owner_uq ON account_members (account_id) WHERE role = 'owner';
-- лимит 10 участников и «у личного счёта один участник» проверяются в сервисе
-- под SELECT ... FROM accounts WHERE id = $1 FOR UPDATE

CREATE TABLE account_invitations (
    id          uuid PRIMARY KEY,
    account_id  uuid NOT NULL REFERENCES accounts (id),
    inviter_id  uuid NOT NULL,
    invitee_id  uuid,
    token_hash  bytea UNIQUE,
    role        member_role NOT NULL CHECK (role <> 'owner'),
    status      text NOT NULL DEFAULT 'pending',
    expires_at  timestamptz NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now(),
    CHECK ((invitee_id IS NULL) <> (token_hash IS NULL))
);
CREATE UNIQUE INDEX account_invitations_pending_uq
    ON account_invitations (account_id, invitee_id) WHERE status = 'pending';
CREATE INDEX account_invitations_invitee_idx ON account_invitations (invitee_id, status);

CREATE TABLE user_blocks_replica (          -- из social.v1, для проверки приглашений
    blocker_id  uuid NOT NULL,
    blocked_id  uuid NOT NULL,
    PRIMARY KEY (blocker_id, blocked_id)
);

CREATE TABLE categories (
    id            uuid PRIMARY KEY,
    user_id       uuid,                       -- личный набор
    account_id    uuid REFERENCES accounts (id), -- набор совместного счёта
    parent_id     uuid REFERENCES categories (id),
    name          text NOT NULL,
    type          category_type NOT NULL,
    budget_class  budget_class,
    depth         smallint NOT NULL CHECK (depth BETWEEN 1 AND 3),
    sort_order    int NOT NULL DEFAULT 0,
    archived_at   timestamptz,
    created_at    timestamptz NOT NULL DEFAULT now(),
    CHECK ((parent_id IS NULL) = (depth = 1)),
    CHECK (budget_class IS NULL OR (depth = 1 AND type = 'expense')),
    CHECK ((user_id IS NULL) <> (account_id IS NULL))
);
CREATE UNIQUE INDEX categories_name_uq
    ON categories (COALESCE(user_id, account_id),
                   COALESCE(parent_id, '00000000-0000-0000-0000-000000000000'), lower(name))
    WHERE archived_at IS NULL;
CREATE INDEX categories_user_idx ON categories (user_id) WHERE user_id IS NOT NULL;
CREATE INDEX categories_account_idx ON categories (account_id) WHERE account_id IS NOT NULL;

-- страховка инвариантов: глубина и тип по родителю
CREATE FUNCTION categories_check_parent() RETURNS trigger AS $$
DECLARE p categories%ROWTYPE;
BEGIN
    IF NEW.parent_id IS NULL THEN RETURN NEW; END IF;
    SELECT * INTO p FROM categories WHERE id = NEW.parent_id;
    IF p.user_id IS DISTINCT FROM NEW.user_id OR p.account_id IS DISTINCT FROM NEW.account_id THEN
        RAISE EXCEPTION 'category parent belongs to another category set';
    END IF;
    IF p.type <> NEW.type THEN RAISE EXCEPTION 'category type mismatch'; END IF;
    IF NEW.depth <> p.depth + 1 THEN RAISE EXCEPTION 'invalid category depth'; END IF;
    RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER categories_check_parent_trg
    BEFORE INSERT OR UPDATE OF parent_id, depth, type ON categories
    FOR EACH ROW EXECUTE FUNCTION categories_check_parent();

CREATE TABLE goals (
    id              uuid PRIMARY KEY,
    account_id      uuid NOT NULL REFERENCES accounts (id),
    created_by      uuid NOT NULL,
    name            text NOT NULL,
    target_amount   bigint NOT NULL CHECK (target_amount > 0),
    current_amount  bigint NOT NULL DEFAULT 0,
    currency        char(3) NOT NULL,
    deadline        date,
    status          text NOT NULL DEFAULT 'active' CHECK (status IN ('active','achieved','archived')),
    created_at      timestamptz NOT NULL DEFAULT now(),
    achieved_at     timestamptz
);
CREATE INDEX goals_account_idx ON goals (account_id, status);

CREATE TABLE transactions (
    id                 uuid PRIMARY KEY,
    account_id         uuid NOT NULL REFERENCES accounts (id),
    created_by         uuid NOT NULL,
    paid_by            uuid NOT NULL,          -- = created_by, если не указан
    kind               tx_kind NOT NULL DEFAULT 'regular',
    category_id        uuid REFERENCES categories (id),
    goal_id            uuid REFERENCES goals (id),
    transfer_group_id  uuid,
    amount             bigint NOT NULL CHECK (amount > 0),
    currency           char(3) NOT NULL,
    occurred_at        timestamptz NOT NULL,
    note               text,
    created_at         timestamptz NOT NULL DEFAULT now(),
    updated_at         timestamptz NOT NULL DEFAULT now(),
    deleted_at         timestamptz,
    CHECK ((kind = 'regular') = (category_id IS NOT NULL)),
    CHECK ((kind IN ('transfer_in','transfer_out')) = (transfer_group_id IS NOT NULL))
);
CREATE INDEX transactions_account_time_idx
    ON transactions (account_id, occurred_at DESC, id DESC) WHERE deleted_at IS NULL;
CREATE INDEX transactions_account_cat_idx
    ON transactions (account_id, category_id) WHERE deleted_at IS NULL;
CREATE INDEX transactions_transfer_idx
    ON transactions (transfer_group_id) WHERE transfer_group_id IS NOT NULL;

CREATE TABLE transaction_splits (
    transaction_id  uuid NOT NULL REFERENCES transactions (id) ON DELETE CASCADE,
    user_id         uuid NOT NULL,
    amount          bigint NOT NULL CHECK (amount >= 0),
    PRIMARY KEY (transaction_id, user_id)
);
CREATE INDEX transaction_splits_user_idx ON transaction_splits (user_id);
-- Σ splits.amount = transactions.amount проверяется CONSTRAINT TRIGGER ... DEFERRABLE INITIALLY DEFERRED

CREATE TABLE settlements (                  -- статус расчёта; сама сумма — в transactions
    transaction_id  uuid PRIMARY KEY REFERENCES transactions (id),
    to_user_id      uuid NOT NULL,
    status          text NOT NULL DEFAULT 'pending' CHECK (status IN ('pending','confirmed','rejected')),
    confirmed_at    timestamptz
);

CREATE TABLE goal_contributions (
    id              uuid PRIMARY KEY,
    goal_id         uuid NOT NULL REFERENCES goals (id),
    user_id         uuid NOT NULL,              -- кто внёс
    amount          bigint NOT NULL CHECK (amount <> 0),
    transaction_id  uuid UNIQUE REFERENCES transactions (id),
    occurred_at     timestamptz NOT NULL,
    note            text,
    created_at      timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX goal_contributions_goal_idx ON goal_contributions (goal_id, occurred_at DESC);

CREATE TABLE processed_events (
    event_id      uuid PRIMARY KEY,
    processed_at  timestamptz NOT NULL DEFAULT now()
);

-- + outbox (как в auth)
```

### analytics

```sql
CREATE TABLE accounts_replica (       -- из finance.accounts.v1
    account_id  uuid PRIMARY KEY,
    type        text NOT NULL,
    currency    char(3) NOT NULL,
    timezone    text NOT NULL
);

CREATE TABLE account_members_replica (
    account_id  uuid NOT NULL,
    user_id     uuid NOT NULL,
    role        text NOT NULL,
    PRIMARY KEY (account_id, user_id)
);
CREATE INDEX account_members_replica_user_idx ON account_members_replica (user_id);

CREATE TABLE daily_aggregates (
    account_id   uuid NOT NULL,
    day          date NOT NULL,          -- локальная дата счёта
    category_id  uuid NOT NULL,          -- категория транзакции (как есть, без раскрытия)
    type         text NOT NULL,          -- income | expense
    amount       bigint NOT NULL,
    tx_count     int NOT NULL,
    PRIMARY KEY (account_id, day, category_id)
);

-- доли участников в тратах совместных счетов (для include_shared и балансов)
CREATE TABLE daily_share_aggregates (
    user_id      uuid NOT NULL,
    account_id   uuid NOT NULL,
    day          date NOT NULL,
    category_id  uuid NOT NULL,
    amount       bigint NOT NULL,
    PRIMARY KEY (user_id, account_id, day, category_id)
);

CREATE TABLE processed_events (
    event_id      uuid PRIMARY KEY,
    processed_at  timestamptz NOT NULL DEFAULT now()
);
```

Применение события `transaction.updated`:

```sql
-- в одной транзакции с INSERT INTO processed_events
INSERT INTO daily_aggregates AS a (account_id, day, category_id, type, amount, tx_count)
VALUES ($1, $2, $3, $4, -$5, -1)                       -- before
ON CONFLICT (account_id, day, category_id)
DO UPDATE SET amount = a.amount + EXCLUDED.amount, tx_count = a.tx_count + EXCLUDED.tx_count;
-- затем то же самое с +after
-- аналогично для daily_share_aggregates по каждому split (before с минусом, after с плюсом)
```

Транзакции `kind ≠ regular` в агрегаты не попадают.

Свёртка до родительских категорий выполняется в Go по дереву из `finance.GetCategoryTree`. Глубина 3, данных мало, так что это проще, чем хранить денормализованные пути.

> Если агрегатов станет много (годы данных × сотни категорий), можно добавить `monthly_aggregates` и выбирать источник по длине периода.

## Миграции (golang-migrate)

- Каталоги: `migrations/<service>/`, файлы `NNNNNN_description.up.sql` / `.down.sql` (`migrate create -ext sql -dir migrations/finance -seq add_goals`).
- Одна миграция — одно логическое изменение. У каждой `up` есть рабочий `down`.
- Миграции **не** запускаются при старте сервиса. Отдельная команда `cmd/<service> migrate up` или init-контейнер / k8s Job перед деплоем (см. [06-infrastructure.md](06-infrastructure.md)).
- Опасные изменения (переименование или удаление колонки, `NOT NULL` на большой таблице) проводятся в два релиза: expand → migrate data → contract.
- Индексы на заполненных таблицах создаются `CREATE INDEX CONCURRENTLY`, в отдельной миграции без транзакции.

```bash
make migrate-up SERVICE=finance
make migrate-down SERVICE=finance N=1
make migrate-create SERVICE=finance NAME=add_goals
```

## Redis

| Ключ | Тип | TTL | Владелец | Назначение |
|---|---|---|---|---|
| `auth:refresh:{sha256}` | hash `{user_id, family_id, expires_at}` | 30 д | auth | Refresh-токен |
| `auth:family:{family_id}` | set хешей | 30 д | auth | Семейство токенов (отзыв при reuse) |
| `auth:user_sessions:{user_id}` | set family_id | 30 д | auth | «Выйти везде» |
| `ratelimit:login:ip:{ip}` | counter | 15 мин | auth | ≤ 20 попыток |
| `ratelimit:login:email:{email}` | counter | 15 мин | auth | ≤ 5 попыток |
| `ratelimit:api:{user_id}` | counter (fixed window) | 1 мин | gateway | ≤ 300 запросов |
| `idem:{user_id}:{key}` | string (ответ) | 24 ч | gateway | Idempotency-Key |
| `report:ver:{account_id}` | counter | — | analytics | Версия данных счёта, инкрементируется на каждое событие |
| `report:{user_id}:{versions_hash}:{params_hash}` | string (protobuf) | 10 мин | analytics | Кеш отчёта; `versions_hash` — хеш версий всех счетов отчёта |
| `cattree:{user_id\|account_id}` | string | 5 мин | analytics | Кеш дерева категорий из finance |
| `acl:{account_id}:{user_id}` | string (роль) | 60 с | finance | Кеш роли участника |

Схема инвалидации через версию не требует удаления ключей: после `INCR report:ver` старые ключи просто перестают читаться и истекают по TTL. Кеш дерева категорий сбрасывается тем же способом. Изменение одного совместного счёта меняет `versions_hash` у всех его участников, поэтому их отчёты тоже инвалидируются.

Ключи social-service (Pub/Sub уведомлений, кеш профилей, лимиты) перечислены в [09-social.md](09-social.md#данные-social).

Redis используется как кеш и хранилище эфемерного состояния. Потеря Redis означает разлогин пользователей и холодный кеш, но не потерю данных.
