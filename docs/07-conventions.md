# 07 — Соглашения по разработке

## Go

- Форматирование `gofmt` / `goimports`, линтер `golangci-lint` (минимум: `govet`, `staticcheck`, `errcheck`, `errorlint`, `revive`, `gosec`, `sqlclosecheck`).
- Роутер — `github.com/go-chi/chi/v5`, middleware — `github.com/go-chi/chi/v5/middleware`. Пакеты chi v1 (`github.com/go-chi/chi`) не используются.
- `context.Context` — первый аргумент любой функции, которая ходит в сеть или БД.
- Интерфейсы объявляются на стороне потребителя (в `service`), а не рядом с реализацией. Они маленькие.
- Конструкторы `NewX(deps...) *X`, без глобального состояния и `init()` с побочными эффектами.
- Время берётся через интерфейс `Clock`, ID — через генератор. Так тесты детерминированы.

### Слои на примере создания транзакции

```go
// transport/grpc: парсинг, валидация формата, маппинг ошибок
func (s *Server) CreateTransaction(ctx context.Context, req *financev1.CreateTransactionRequest) (*financev1.Transaction, error) {
    userID := grpcx.UserID(ctx)
    tx, err := s.svc.CreateTransaction(ctx, userID, toCreateInput(req)) // account_id внутри input
    if err != nil {
        return nil, grpcx.Error(err)
    }
    return toProto(tx), nil
}

// service: бизнес-правила, транзакция БД, outbox
func (s *TransactionService) CreateTransaction(ctx context.Context, userID uuid.UUID, in CreateInput) (domain.Transaction, error) {
    var out domain.Transaction
    role, err := s.access.Role(ctx, in.AccountID, userID)
    if err != nil {
        return domain.Transaction{}, err
    }
    if !role.Can(domain.ActionCreateTransaction) {
        return domain.Transaction{}, domain.ErrInsufficientRole
    }
    err = s.tx.WithinTx(ctx, func(ctx context.Context) error {
        cat, err := s.categories.GetForAccount(ctx, in.AccountID, in.CategoryID)
        if err != nil {
            return fmt.Errorf("get category: %w", err)
        }
        if cat.IsArchived() {
            return domain.ErrCategoryArchived
        }
        out = domain.NewTransaction(s.ids.New(), in.AccountID, userID, cat, in.Amount, in.OccurredAt, in.Note)
        if err := s.transactions.Insert(ctx, out); err != nil {
            return fmt.Errorf("insert transaction: %w", err)
        }
        return s.outbox.Add(ctx, events.TransactionCreated(out))
    })
    return out, err
}
```

## Ошибки

- Ошибка оборачивается с контекстом: `fmt.Errorf("insert transaction: %w", err)`. Слова «failed to» не пишутся.
- Доменные ошибки — sentinel (`var ErrNotFound = errors.New(...)`) или типы (`*ValidationError` с полями). Проверяются через `errors.Is` / `errors.As`.
- Репозиторий переводит `pgx.ErrNoRows` → `domain.ErrNotFound`, unique violation (`23505`) → `domain.ErrAlreadyExists`.
- Маппинг в gRPC-статус выполняется в одном месте (`grpcx.Error`), в HTTP — в одном месте в gateway (по gRPC-коду). Таблица — в [03-domain.md](03-domain.md#доменные-ошибки).
- Наружу не уходят тексты внутренних ошибок: для `Internal` клиент получает только `request_id`.

## Тестирование

| Уровень | Что | Инструменты |
|---|---|---|
| Unit | `domain`, `service` с фейками, правила рекомендаций, расчёты целей и периодов | `testing`, табличные тесты, `testify/require` по желанию |
| Repository | SQL-запросы против настоящего Postgres | `testcontainers-go`, build tag `integration` |
| Contract | gRPC-сервер с in-memory зависимостями | `bufconn` |
| E2E | Регистрация → транзакция → отчёт через gateway | docker-compose + Go-тест или Playwright с фронта |

Что обязательно покрыть:
- инварианты категорий (глубина, тип, набор, перемещение, архивация);
- **изоляцию счетов**: попытка прочитать или изменить сущность чужого счёта возвращает `NotFound`;
- **матрицу прав** совместного счёта: каждое действие × каждая роль × «не участник» (один табличный тест на сервис);
- балансы сплитов: `Σ balance = 0`, распределение копеек (см. [09-social.md](09-social.md#что-обязательно-протестировать));
- идемпотентность консьюмеров (одно событие дважды даёт тот же агрегат);
- границы периодов в разных часовых поясах (транзакция в 23:30 по Москве попадает в правильный день);
- update/delete транзакции корректно меняют агрегаты.

## Git

- Ветки: `feat/<что>`, `fix/<что>`, `chore/<что>` от `master`.
- Коммиты в стиле Conventional Commits: `feat(finance): add category archiving`.
- Один PR — одна логическая задача. Миграции, proto и OpenAPI меняются в том же PR, что и код, который их использует.
- `master` всегда собирается и проходит тесты.

## Документация

- Изменение поведения обновляет соответствующий файл в `docs/` в том же PR.
- Архитектурные решения, которые неочевидно откатить (выбор сериализации событий, переход на монолит и т. п.), фиксируются короткими ADR в `docs/adr/NNNN-title.md`: контекст → решение → последствия.
