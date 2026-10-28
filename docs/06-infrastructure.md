# 06 — Инфраструктура

## Конфигурация

- Источник: YAML-файл (`config.yaml`) + переопределение переменными окружения. Переменные окружения имеют приоритет. Подойдёт `cleanenv` или ручной `os.Getenv` поверх `yaml.v3`.
- Путь к файлу задаётся через `CONFIG_PATH` или флаг `--config`.
- **Секреты (пароли БД, JWT-ключи) в YAML не хранятся.** Они приходят только из env: локально через `.env` (в `.gitignore`), в k8s через `Secret`. В репозитории лежит `config.example.yaml` без секретов, а сам `config.yaml` добавлен в `.gitignore`.
- Конфиг валидируется при старте. При ошибке сервис завершается с понятным сообщением.

```yaml
# config.example.yaml (finance)
app:
  env: dev                  # dev | prod — влияет на формат логов
  name: finance
http:
  addr: ":8081"             # healthz / readyz / metrics
grpc:
  addr: ":9091"
postgres:
  host: localhost
  port: 5432
  name: graspy_finance
  user: graspy_finance
  # password: из POSTGRES_PASSWORD
  max_conns: 10
kafka:
  brokers: ["localhost:9092"]
  outbox_poll_interval: 500ms
shutdown_timeout: 15s
```

## Логирование (slog)

- `app.env=dev` → `slog.NewTextHandler`, уровень `debug`; `prod` → `slog.NewJSONHandler`, уровень `info`.
- Обязательные поля: `service`, `request_id`, `user_id` (если есть), `trace_id` (если есть трейсинг).
- Логгер с полями запроса кладётся в `context` в middleware/интерсепторе и берётся оттуда (`logger.FromContext(ctx)`).
- Не логируются: пароли, токены, полные тела запросов, суммы и комментарии транзакций на уровне `info`.
- Ошибка логируется **один раз** — там, где она обрабатывается (в транспортном слое), а не на каждом уровне стека.

## Локальная разработка

`deploy/docker-compose.yml`:

| Сервис | Образ | Порт |
|---|---|---|
| postgres | `postgres:17` + init-скрипт, создающий 4 базы и пользователей | 5432 |
| redis | `redis:7` | 6379 |
| kafka | `apache/kafka` (KRaft, без ZooKeeper) | 9092 |
| kafka-ui | `provectuslabs/kafka-ui` | 8090 |
| minio | `minio/minio` + init-контейнер, создающий bucket `avatars` | 9000 / 9001 (консоль) |

Go-сервисы локально запускаются вне docker (быстрее итерации), фронт — через `npm run dev` с прокси на gateway.

### Makefile (целевой набор)
```makefile
up:              ## поднять инфраструктуру
down:
migrate-up:      ## SERVICE=finance
migrate-create:  ## SERVICE=finance NAME=...
proto:           ## buf generate
lint:            ## golangci-lint run + buf lint
test:            ## go test ./... -race
test-integration:## go test -tags=integration ./... (testcontainers)
run-<service>:
build:           ## docker build всех сервисов
```

## Graceful shutdown

Каждый сервис:
1. Ловит `SIGINT` / `SIGTERM` через `signal.NotifyContext`.
2. Переводит `/readyz` в 503, чтобы k8s перестал слать трафик.
3. Останавливает HTTP (`Server.Shutdown`) и gRPC (`GracefulStop`) серверы.
4. Останавливает консьюмеры (коммит офсетов) и outbox-релей.
5. Закрывает пулы Postgres и Redis.
6. Укладывается в `shutdown_timeout`, который меньше `terminationGracePeriodSeconds`.

## Docker

- Multi-stage: `golang:1.x` → `gcr.io/distroless/static` (или `scratch`), `CGO_ENABLED=0`, запуск не от root.
- Один Dockerfile с аргументом `SERVICE`, тег образа — git SHA.
- Фронт: сборка Vite → `nginx:alpine`.

## Kubernetes

Для локального кластера подойдёт `kind` или `k3d`. Postgres, Redis и Kafka в кластере ставятся helm-чартами (Bitnami / Strimzi для Kafka) или остаются в docker-compose снаружи кластера — на старте так проще.

```
deploy/k8s/
├── namespace.yaml
├── gateway/      deployment, service, hpa, configmap
├── auth/         deployment, service, configmap, secret (шаблон)
├── finance/      deployment, service, configmap, migrate-job
├── analytics/    deployment, service, configmap, migrate-job
├── social/       deployment, service, configmap, migrate-job, cronjob (очистка ленты, дайджесты)
├── web/          deployment, service
└── ingress.yaml  /api → gateway, / → web
```

Для каждого Deployment:
- `readinessProbe` → `/readyz`, `livenessProbe` → `/healthz`.
- `resources.requests/limits` заданы.
- Конфиг через `ConfigMap`, секреты через `Secret` (env).
- Миграции выполняются в `Job` (или `initContainer`) с той же версией образа до раскатки.
- `replicas: 2` для gateway и finance. Количество реплик analytics не больше числа партиций топика.
- SSE через ingress: отключить буферизацию (`nginx.ingress.kubernetes.io/proxy-buffering: "off"`) и поднять `proxy-read-timeout` до 1 ч. Gateway при остановке закрывает SSE-соединения, фронт переподключается с `Last-Event-ID`.
- MinIO в кластере ставится helm-чартом. Для presigned URL наружу публикуется отдельный host, например `files.graspy.local`.
- Сервисы друг к другу обращаются по DNS (`finance.graspy.svc:9091`). Для балансировки gRPC нужен headless Service + `dns:///` резолвер с `round_robin` на клиенте, иначе все запросы уйдут на один под.

Позже: Helm chart или Kustomize-оверлеи `dev` / `prod`, NetworkPolicy (только gateway доступен из ingress), сертификаты через cert-manager.

## Наблюдаемость

| Что | Как |
|---|---|
| Метрики | `prometheus/client_golang`: RED-метрики HTTP/gRPC (интерсепторы), lag консьюмеров, размер неотправленного outbox, hit rate кеша отчётов, число открытых SSE-соединений, задержка «событие → уведомление» |
| Трейсинг | OpenTelemetry: HTTP → gRPC → Kafka headers, экспорт в Jaeger / Tempo |
| Логи | stdout JSON → Loki (или просто `kubectl logs` на старте) |
| Дашборды | Grafana: один дашборд на сервис + общий по Kafka |

Алерты для пет-проекта необязательны, но полезно хотя бы считать в метриках outbox, «застрявший» дольше 1 минуты, и сообщения в DLQ.

## CI (GitHub Actions)

1. `golangci-lint`, `buf lint`, `buf breaking` (против `master`).
2. `go test -race ./...` + интеграционные тесты с testcontainers.
3. Проверка, что `buf generate` и `sqlc generate` не меняют файлы (сгенерированный код закоммичен и актуален).
4. Фронт: `npm ci && npm run lint && npm run test && npm run build`.
5. На `master`: сборка и пуш образов с тегом SHA.
