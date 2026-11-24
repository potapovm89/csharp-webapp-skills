# Agent Guidelines — <название-солюшена>

C# / .NET 10, монорепо-солюшен, PostgreSQL. Точные сигнатуры проектных хелперов смотрите в коде текущего солюшена (`Common`, `DataAccess`, `Domain`), не додумывайте API.

## Информация о проекте
<!-- Заполните для конкретного проекта: стек фронтенда, назначение приложения, входящие каналы (REST / Kafka / RabbitMQ / gRPC), внешние интеграции. Скиллы ниже от предметной области не зависят. -->
- Стек: C# .NET 10, <фронтенд>.
- Назначение: <кратко, 1–3 предложения>.
- Механизм Stories: <MediatR | IStoryExecutor> — ровно один на солюшен, выбирает пользователь (см. `cqrs-stories`).
- Топология хостов: <in-process | шлюз через RpcHost> (см. `solution-architecture`).
- WorkerHost на нескольких репликах: <одна реплика | Quartz cluster | advisory lock | SKIP LOCKED> (см. `worker-host-jobs`).
- Поставка: <CI-платформа, registry, ответственный>; контракт pipeline и продвижения — `docs/delivery/pipeline.md` (см. `delivery-and-supply-chain`).
- Критичные сценарии и цели SLI/SLO: `docs/reliability/`; допустимые потери/время восстановления RPO/RTO и ответственный — `docs/recovery/plan.md`. Значения выбираются по требованиям приложения, не копируются из примеров.

## Жёсткие правила (действуют всегда)
- **НЕ** устанавливать/удалять ПО, не удалять данные, таблицы и базы без разрешения.
- **НЕ** менять историю коммитов git (rebase опубликованного, force push).
- **НЕ** подключаться к продуктивной среде, не разворачивать и не тестировать на ней.
- **НЕ** хранить секреты в файлах, попадающих в git (кроме фиктивных значений локальных сред).
- Схема БД меняется **только миграциями**; уже влитые миграции не редактируются.
- Изменённое поведение проверено подходящими тестами; затронутые контракты, схема, настройки, права и архитектура отражены в `docs/`. Для косметических правок и документации — соответствующая валидация; формальные тесты и бессодержательные правки docs не нужны.
- Библиотеки — только с разрешёнными лицензиями (GPL/AGPL/SSPL/коммерческие запрещены); **MediatR ≤ 12.5.0**, **FluentAssertions ≤ 7.2.0**.
- Отдельные сервисы под AGPL/SSPL (Grafana, MongoDB Server, Garage) — допустимы по согласованию, если не модифицированы и доступны только по протоколу.
- **PDFsharp/MigraDoc ≤ 6.2.x** (с 6.5.0 — AGPL); HTML → PDF — только **PeachPDF**.
- **Безопасность**: всё закрыто по умолчанию (fallback-политика); владелец ресурса проверяется в Stories, его идентификатор берётся из claims, а не из заголовка; high/critical уязвимости NuGet ломают сборку.
- **Миграции без простоя**: expand → код и backfill → contract в следующем релизе; индексы на существующих таблицах — `CONCURRENTLY`; миграции запускаются один раз до деплоя.
- **WorkerHost на нескольких репликах**: Quartz в кластере (ADO.NET store), одна реплика, advisory lock или распределение строк через SKIP LOCKED — выбор явный и задокументирован; повторное выполнение не дублирует бизнес-эффект.
- **AutoMapper запрещён в любой версии.** Маппинг — только ручной, extension-методами `To<Цель>()` в `Common/Mappers`.
- **Сборка без предупреждений**: проверяется полной сборкой `dotnet build --no-incremental` (инкрементальная не перезапускает анализаторы); nullable-предупреждения исправляются, а не подавляются.
- **Пароли**: по возможности не храним (вход через Keycloak); если храним — PBKDF2 с параметрами в самом хеше, перехеширование при входе, сравнение за постоянное время.
- **Stories — один механизм**: MediatR **или** IStoryExecutor, как указано выше. Не указан и в коде неоднозначно — остановиться и спросить.
- **Шаблон не эталон**: код, пришедший из шаблона, может не соответствовать скиллам (DLQ, fallback-политика, NuGet-аудит и т.п.). Нарушение в коде, который вы трогаете или от которого зависит ваша фича, исправляется или явно сообщается пользователю — «так было в шаблоне» не оправдание.
- **Приведение проекта к скиллам** — по чеклистам и минимальным наборам правил самих скиллов (например, «Adopting in an Existing Solution» в `runtime-deployment`, минимальный набор архитектурных тестов в `testing`). Скопированный из другого репозитория код (включая шаблон) сверяется с текущим текстом скилла пункт за пунктом.
- **References обязательны**: если в SKILL.md есть строка «Before implementing … read …», соответствующий reference читается до написания кода.
- **Время** — только через `TimeProvider`; `DateTime.UtcNow`/`DateTime.Now` в Stories, DataAccess и хостах запрещены.
- **Поставка**: релизные проверки привязаны к точному digest артефакта; между средами продвигается проверенный артефакт без пересборки. Отсутствующая обязательная проверка или истёкшее исключение не считаются успехом (`delivery-and-supply-chain`).
- **Надёжность и восстановление** — только когда задача это затрагивает: новый или изменённый критичный пользовательский сценарий → SLI/SLO и условия деградации (`reliability-and-performance`); новое хранилище постоянных данных или смена способа хранения → RPO/RTO и план восстановления (`data-recovery`). Обычная фича в существующем сценарии и хранилище этого не требует. Пробы, реплики и успешный backup job не заменяют эти проверки; drill — только в разрешённой изолированной непродуктивной среде.

## Ключевые конвенции (кратко)
- **Слои**: Hosts → Stories (бизнес-логика, CQRS) → DataAccess (доменные сервисы) → Domain. Хосты тонкие, Contracts и Domain ни от кого не зависят.
- **Проекты**: `Itl.<Проект>.<Роль>`; таблицы/колонки БД, URL, топики/очереди — kebab/snake_case.
- **Форматы файлов**: решение — один `.slnx` (устаревший `.sln` переводится `dotnet sln migrate` и удаляется), проекты — SDK-style `.csproj` без `packages.config`. Формата `.csprojx` не существует.
- **Брокеры**: запись в БД + публикация сообщения — только через transactional outbox; консьюмеры идемпотентны (inbox или уникальный ключ), сбойные сообщения уходят в DLQ, а не теряются.
- **Настройки**: классы `XxxOptions` + `AddOptionsWithValidation`, секция = имя класса.
- **Каждый хост**: `AddEnvironmentConfigurations()`, `AddServiceDefaults()`, `MapDefaultEndpoints(port)`.
- **XML-документация**: на русском; интерфейсы, контроллеры, модели; описание обязательно заканчивается точкой. Логи — на английском.
- **Ветки**: `feature|bugfix/<задача>-<тезис>`, коммиты с номером задачи, MR обязателен.

---

## Каталог доступных скиллов (skills/)

Для решения задач активируйте соответствующие скиллы из папки `skills/`.

> Типовую фичу целиком (эндпоинт, консьюмер, джоба, сущность, интеграция) начинайте с `feature-recipes`: там упорядоченный чеклист по слоям со ссылками на профильные скиллы и сигнатуры проектных хелперов.
> Фронтенд: `react-frontend` — для React + TypeScript, `blazor-frontend` — только для Blazor-проектов.

| Задача / Область | Скилл | Путь к скиллу |
| ---------------- | ----- | ------------- |
| Порядок работы над задачей (план, проверка уверенности, ревью) | `task-lifecycle` | [skills/task-lifecycle/SKILL.md](skills/task-lifecycle/SKILL.md) |
| Сквозные рецепты фич (эндпоинт + outbox, консьюмер, джоба, сущность + миграция, внешний API, gRPC-метод), сигнатуры хелперов | `feature-recipes` | [skills/feature-recipes/SKILL.md](skills/feature-recipes/SKILL.md) |
| Архитектура монорепо, слои решения, composition root, мультитенантность | `solution-architecture` | [skills/solution-architecture/SKILL.md](skills/solution-architecture/SKILL.md) |
| C# код-стайл, именование, XML-документация, SOLID/KISS/DRY/YAGNI, лицензии | `csharp-code-style` | [skills/csharp-code-style/SKILL.md](skills/csharp-code-style/SKILL.md) |
| Бизнес-логика в Stories: MediatR **или** Executor-стори (один механизм на солюшен), валидаторы | `cqrs-stories` | [skills/cqrs-stories/SKILL.md](skills/cqrs-stories/SKILL.md) |
| Многошаговые процессы с компенсациями (саги): process manager vs хореография, таймауты, застрявшие процессы | `sagas-processes` | [skills/sagas-processes/SKILL.md](skills/sagas-processes/SKILL.md) |
| Доменные сущности, базовые классы ключей, миксины, DTO, EF конфигурации | `domain-modeling` | [skills/domain-modeling/SKILL.md](skills/domain-modeling/SKILL.md) |
| PostgreSQL, доменные сервисы (EF Core / Dapper), транзакции ITransactionScope, конкурентность, производительность запросов | `postgresql-data-access` | [skills/postgresql-data-access/SKILL.md](skills/postgresql-data-access/SKILL.md) |
| Миграции БД (EF Core, FluentMigrator), миграции без простоя, DBML-схемы | `database-migrations` | [skills/database-migrations/SKILL.md](skills/database-migrations/SKILL.md) |
| Документная БД MongoDB, распределённый кэш Redis (Valkey), стратегия кэширования (HybridCache) | `nosql-and-cache` | [skills/nosql-and-cache/SKILL.md](skills/nosql-and-cache/SKILL.md) |
| Брокеры Kafka и RabbitMQ (продюсеры, консьюмеры, IProcessor, outbox/inbox, retry и DLQ, версии сообщений) | `message-brokers` | [skills/message-brokers/SKILL.md](skills/message-brokers/SKILL.md) |
| S3-совместимое файловое хранилище (RustFS локально и в тестах) | `s3-storage` | [skills/s3-storage/SKILL.md](skills/s3-storage/SKILL.md) |
| PDF-отчёты: PeachPDF (HTML→PDF) и PDFsharp ≤ 6.2.x, шаблоны, статусы, выдача через S3 | `pdf-reports` | [skills/pdf-reports/SKILL.md](skills/pdf-reports/SKILL.md) |
| Вызовы внешних REST API (HttpClient) и внешних gRPC-сервисов | `api-clients` | [skills/api-clients/SKILL.md](skills/api-clients/SKILL.md) |
| WebHost REST API, контроллеры (User / Admin), middleware pipeline, Swagger | `web-host-api` | [skills/web-host-api/SKILL.md](skills/web-host-api/SKILL.md) |
| RpcHost gRPC-сервисы, структура и правила .proto файлов, интерцепторы | `grpc-services` | [skills/grpc-services/SKILL.md](skills/grpc-services/SKILL.md) |
| WorkerHost, фоновые задачи (Quartz, cron, работа на нескольких репликах) | `worker-host-jobs` | [skills/worker-host-jobs/SKILL.md](skills/worker-host-jobs/SKILL.md) |
| Real-time push клиентам: SignalR-хаб в RealtimeHost, авторизация подписок, outbox→push, backplane, BFF-прокси | `realtime-signalr` | [skills/realtime-signalr/SKILL.md](skills/realtime-signalr/SKILL.md) |
| Blazor WebAssembly frontend UI, компоненты, изоляция стилей, локализация | `blazor-frontend` | [skills/blazor-frontend/SKILL.md](skills/blazor-frontend/SKILL.md) |
| React + TypeScript фронтенд: сгенерированный API-клиент, TanStack Query, формы, OIDC PKCE, i18n, тесты, XSS/CSP | `react-frontend` | [skills/react-frontend/SKILL.md](skills/react-frontend/SKILL.md) |
| Запуск в контейнерах и Kubernetes: Dockerfile, пробы, graceful shutdown, миграции до деплоя, ресурсы, реплики | `runtime-deployment` | [skills/runtime-deployment/SKILL.md](skills/runtime-deployment/SKILL.md) |
| CI/CD, release gates, lock-файлы, SBOM, provenance, проверка подлинности и продвижение digest | `delivery-and-supply-chain` | [skills/delivery-and-supply-chain/SKILL.md](skills/delivery-and-supply-chain/SKILL.md) |
| Резервные копии, PostgreSQL PITR, RPO/RTO, согласованное восстановление данных и изолированные drills | `data-recovery` | [skills/data-recovery/SKILL.md](skills/data-recovery/SKILL.md) |
| Нефункциональные требования, SLI/SLO, error budget, burn-rate алерты, нагрузка и отказы | `reliability-and-performance` | [skills/reliability-and-performance/SKILL.md](skills/reliability-and-performance/SKILL.md) |
| Локальный полигон .NET Aspire, оркестрация контейнеров и ресурсов | `aspire-environment` | [skills/aspire-environment/SKILL.md](skills/aspire-environment/SKILL.md) |
| Настройки Options (AddOptionsWithValidation), среды, защита секретов | `configuration-secrets` | [skills/configuration-secrets/SKILL.md](skills/configuration-secrets/SKILL.md) |
| Безопасность: аутентификация и авторизация, владение ресурсами, CORS, rate limiting, инъекции, SSRF, персональные данные, уязвимые зависимости | `security` | [skills/security/SKILL.md](skills/security/SKILL.md) |
| Логирование Serilog, метрики и трейсы OpenTelemetry, эндпоинты health | `observability` | [skills/observability/SKILL.md](skills/observability/SKILL.md) |
| Юнит-тесты (xUnit, Moq, BaseTestCasesClass), интеграционные (Testcontainers), архитектурные и контрактные, детерминизм тестов | `testing` | [skills/testing/SKILL.md](skills/testing/SKILL.md) |
| Git flow, именование веток, формат коммитов, шаблон Merge Request | `git-workflow` | [skills/git-workflow/SKILL.md](skills/git-workflow/SKILL.md) |
| Документирование в docs/ (C4 DSL, DBML, контракты API/брокеров, роли, ADR, runbook) | `project-documentation` | [skills/project-documentation/SKILL.md](skills/project-documentation/SKILL.md) |
