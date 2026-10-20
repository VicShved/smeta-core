---
title: 'Story 3.1 — Импорт справочника расценок из Excel с маппером колонок'
type: 'feature'
created: '2026-10-20'
status: 'done'
route: 'dispatch'
baseline_commit: '95850f9724838abb56db856ecbe9f18c0d487071'
review_loop_iteration: 0
context:
  - '_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md'
  - '_bmad-output/implementation-artifacts/epic-3-context.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Справочник расценок отсутствует — сметы не могут опираться на утверждённые расценки и ЕИ; сервис rate-catalog существует только как скелет с `/health`.

**Approach:** Вертикаль импорта в rate-catalog по паттерну эпика 2: Postgres-БД `rate_catalog`, слоистая структура domain/application/infrastructure/transport, один эндпоинт импорта (загрузка `.xlsx` + маппинг колонок в одном запросе), сохранение исходника в MinIO, подключение сервиса в api-gateway, docker-compose и CI.

## Boundaries & Constraints

**Always:** Только `.xlsx` (openpyxl, без pandas); identity только из заголовков `X-User-Id`/`X-Company-Id` (AD-010), 401 без company; справочник скоупится по компании (AD-013, `company_id` на `rate_catalog`); ошибки — `application/problem+json` (паттерн company-org); первый лист книги, первая строка — заголовки, данные со 2-й строки; парсинг и импорт в одной транзакции, частичный успех (валидные строки импортируются, инвалидные возвращаются с причинами); десятичный разделитель — и точка, и запятая, неразрывные пробелы отбрасываются; пределы: ≤20 МБ файл, ≤50 000 строк данных (сверху — 413/422).

**Never:** UI-маппера (только API); глобальная (не companyId) схема; импорт без сохранения исходного `.xlsx` в MinIO; изменение скелетонного `/health`-контракта и тестов вне rate-catalog, кроме точечной регистрации роутов шлюза/компоуза.

**Решения (2026-10-20):** Поля строки справочника — `code` (Код материала/изделия/конструкции, опционален), `name` (Наименование по КСР), `unit` (ЕИ по КСР), `rate_amount` (сметная цена, ₽/ед., **без НДС** — флаг НДС не хранится). Обязательные при маппинге: `name`, `unit`, `rate_amount`. Маппер предсопоставляет колонки по четырём заголовкам КСР. Импорт — единый эндпоинт: загрузка `.xlsx` + маппинг в одном запросе, без отдельного превью и `upload_id`. Версия становится активной при импорте, если в справочнике ещё нет активной версии. Исходный `.xlsx` сохраняется в MinIO.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Импорт | multipart `.xlsx` + маппинг колонок | 200: `version_id`, `imported`, `rejected` [{row, reason}], исходник сохранён в MinIO | 400 не-xlsx; 413 >20 МБ; 422 >50k строк / нет строк данных / маппинг не покрывает обязательные поля |
| Не-xlsx файл | `.csv`/`.xls`/без расширения | — | 400 `problem+json` с типом `unsupported-format` |
| Инвалидная строка | пустое наименование / нечисловая цена / дубль `code` | строка в `rejected` с номером и причиной, остальные импортированы | не прерывает импорт |
| Нет identity | без `X-Company-Id` | — | 401 |
| Повтор импорта того же файла | дубликат загрузки | создаётся новая версия справочника | — |

</frozen-after-approval>

## Code Map

- `/home/shved/projects/smeta/rate-catalog/app/main.py` -- точка входа, добавить router и problem-хендлеры
- `/home/shved/projects/smeta/rate-catalog/pyproject.toml` -- зависимости: `openpyxl`, `sqlalchemy`, `psycopg`, `alembic`, dev: `pytest`, `ruff` (уже есть)
- `/home/shved/projects/smeta/rate-catalog/.github/workflows/ci.yml` -- ruff+pytest уже есть; шаг `docker build` отсутствует (как в company-org добавить)
- `/home/shved/projects/smeta/company-org/app/wiring.py` -- референс composition root: `make_*_runner()` «одна сессия на запрос», rollback/commit
- `/home/shved/projects/smeta/company-org/app/infrastructure/db.py` -- референс `make_session_factory(dsn)`: `POSTGRES_DSN`, sqlite `connect_args`, `pool_pre_ping`
- `/home/shved/projects/smeta/company-org/app/transport/routes.py` -- референс `create_router`: префикс `/v1`, identity-зависимости, `Idempotency-Key`
- `/home/shved/projects/smeta/company-org/app/transport/problem.py` -- референс RFC7807 `problem_response`/`install_problem_handlers` (копируется, shared-lib не вводится)
- `/home/shved/projects/smeta/company-org/app/domain/ports.py` -- референс Protocol-портов репозиториев
- `/home/shved/projects/smeta/company-org/migrations/env.py`, `tests/test_migrations.py` -- референс alembic + тест `alembic upgrade head` на sqlite
- `/home/shved/projects/smeta/api-gateway/app/application/routing.py` -- добавить префикс `("/v1/rate-catalog", rate_catalog_url)`
- `/home/shved/projects/smeta/api-gateway/app/wiring.py` -- env `RATE_CATALOG_URL`, дефолт `http://rate-catalog:8000`
- `/home/shved/projects/smeta/deploy/docker-compose.yml` -- сервис rate-catalog (build, expose 8000, `POSTGRES_DSN_RATE_CATALOG`, depends_on postgres)
- `/home/shved/projects/smeta/deploy/docker-compose.yml` -- сервис minio для импортов rate-catalog (если отсутствует)
- `/home/shved/projects/smeta/deploy/initdb/01-create-databases.sql` -- `CREATE DATABASE rate_catalog;`
- `/home/shved/projects/smeta/deploy/.env.example` -- `POSTGRES_DSN_RATE_CATALOG`, `RATE_CATALOG_URL`, `MINIO_ENDPOINT`, `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD`, `RATE_CATALOG_IMPORT_BUCKET`
- `/home/shved/projects/smeta/rate-catalog/app/infrastructure/storage.py` -- MinIO adapter: ensure bucket, put-object исходного `.xlsx` при импорте

## Tasks & Acceptance

**Execution:**
- [x] `rate-catalog/pyproject.toml` -- `openpyxl`, `sqlalchemy`, `psycopg`, `alembic`, `minio` в dependencies; dev: `pytest`, `ruff` -- сервис парсит xlsx, хранит в Postgres и MinIO
- [x] `rate-catalog/alembic.ini` + `rate-catalog/migrations/` -- 0001: `rate_catalog` (id, company_id, current_version_id), `rate_catalog_version` (id, rate_catalog_id, version, imported_at, source_file_hash, created_by), `rate_catalog_rate` (id, rate_catalog_version_id, code, name, unit, rate_amount) -- схема импорта и база под версионирование 3.2
- [x] `rate-catalog/app/domain/` -- entity `Rate`/`RateVersion`, порты `RateCatalogRepository`, `ImportFileStorage`, errors -- слой домена без SQLAlchemy
- [x] `rate-catalog/app/application/` -- use case `import_rates` (парс `.xlsx` через openpyxl, авто-предложение маппинга по заголовкам КСР, валидация строк, импорт валидных в одну транзакцию, отчёт rejected, активация версии если первая) -- вся бизнес-логика импорта
- [x] `rate-catalog/app/infrastructure/` -- `db.py` (по образцу company-org), `models.py` (SQLAlchemy), `repository.py`, `excel_parser.py` (openpyxl: чтение первого листа, нормализация чисел), `storage.py` (MinIO: ensure bucket, put-object) -- инфраструктура
- [x] `rate-catalog/app/transport/` -- `routes.py` (`POST /v1/rate-catalog/imports` — multipart `.xlsx` + маппинг колонок, `GET /health` без изменений), `schemas.py`, `problem.py` (копия company-org) -- HTTP-контракт
- [x] `rate-catalog/app/wiring.py` + правка `app/main.py` -- докомпоновка, problem-хендлеры -- работоспособное приложение
- [x] `rate-catalog/tests/test_import.py`, `tests/test_migrations.py` -- по образцу company-org (sqlite, `_migrate`, reload main) -- покрытие I/O-матрицы
- [x] `api-gateway/app/application/routing.py` + `app/wiring.py` -- префикс `/v1/rate-catalog`, env `RATE_CATALOG_URL`, проброс `X-User-Id`/`X-Company-Id`/`X-Role` downstream -- доступ через edge
- [x] `api-gateway/tests/test_gateway.py` -- стаб upstream rate-catalog -- регрессия шлюза
- [x] `deploy/docker-compose.yml` -- сервис rate-catalog + сервис minio (build, expose, `POSTGRES_DSN_RATE_CATALOG`, `MINIO_*`, depends_on postgres)
- [x] `deploy/initdb/01-create-databases.sql` -- `CREATE DATABASE rate_catalog;`
- [x] `deploy/.env.example` -- `POSTGRES_DSN_RATE_CATALOG`, `RATE_CATALOG_URL`, `MINIO_ENDPOINT`, `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD`, `RATE_CATALOG_IMPORT_BUCKET`
- [x] `rate-catalog/.github/workflows/ci.yml` -- шаг `docker build` -- паритет с company-org

**Acceptance Criteria:**
- Given валидный `.xlsx` с четырьмя колонками КСР (код, наименование, ЕИ, цена) и маппингом, when POST `/v1/rate-catalog/imports`, then 200: `version_id`, `imported`, `rejected` [{row, reason}], валидные строки сохранены в новую версию справочника, версия активна если первая, исходник в MinIO
- Given файл `.csv` или не-xlsx, when POST, then 400 `problem+json` `unsupported-format`
- Given строка с пустым наименованием / нечисловой ценой / дублем `code`, when import, then строка в `rejected` с причиной, остальные импортированы, HTTP 200
- Given запрос без `X-Company-Id`, when любой вызов, then 401
- Given миграции применены на чистой Postgres/sqlite, when `alembic upgrade head`, then схема создана без ошибок

## Implementation Notes

- **Маппинг колонок:** multipart-поле `mapping` — JSON-объект `{field: header | 0-based index}`, `field ∈ {code, name, unit, rate_amount}`. Явные значения главнее автопредложения; недостающие поля предсопоставляются по заголовкам КСР (`app/application/mapping.py`). `name`, `unit`, `rate_amount` обязательны, колонки не должны пересекаться; иначе 422.
- **Слои:** openpyxl изолирован в `app/infrastructure/excel_parser.py` за портом `WorkbookParser`; application-слой (`import_rates.py`) не импортирует openpyxl/SQLAlchemy/FastAPI (AD-011). Парсинг вызывается use case через порт.
- **Транзакция:** репозиторий не коммитит; `app/wiring.py` открывает одну сессию на запрос, `ImportRates` пишет каталог, версию, строки и активную версию, затем commit. Ошибка → rollback. Исходник кладётся в MinIO перед commit (key `{company_id}/{version_id}/{filename}`).
- **Активная версия:** `rate_catalog.current_version_id`; при импорте выставляется только если был `NULL`. FK на `current_version_id` не создан из-за циклической ссылки `rate_catalog ↔ rate_catalog_version` (совместимость миграции с sqlite); инвариант держит use case.
- **Валидация строк:** пустое/длинное (>1000) наименование, пустая/длинная (>100) единица, длинный (>255) `code`, нечисловая/нефинитная/отрицательная/вне `Numeric(18,4)` цена, дубль `code` внутри импорта → `rejected` (`row` — номер строки Excel, данные с 2). Пустые строки пропускаются. Если валидных строк не осталось — 422 (`workbook has no valid rows`), каталог/версия/файл не создаются.
- **Форматы/лимиты:** расширение `.xlsx` (регистронезависимо), ≤20 МиБ, ≤50 000 непустых строк данных; иначе 400 `unsupported-format`, 413, 422 соответственно. Повреждённый xlsx (включая битый XML внутри валидного zip) → 422 `invalid-workbook`.
- **Ошибки:** копия `problem.py` company-org; доменные ошибки маппятся в `application/problem+json` в `app/transport/problem.py`.
- **Deploy:** в compose добавлены `minio` (volume `minio-data`, healthcheck) и `rate-catalog`; api-gateway получил `RATE_CATALOG_URL` и depends_on. `initdb` создаёт БД `rate_catalog` (только на свежем volume).
- **MinIO-образ (исправлено на верификации):** `minio/minio` удалён с Docker Hub (minio/minio#21647, проверено: `denied`), поэтому по решению Story 1.1 используется зеркало `alpine/minio:latest-release` (проверено: манифест есть, `docker pull` прошёл), `user: root` и healthcheck через `wget` к `/minio/health/live` — в зеркале нет `mc`.

## Spec Change Log

- 2026-10-20: реализация без изменения frozen-блока. Контракт поля `mapping` (JSON в multipart) уточнён на уровне реализации: спека не фиксировала форму, выбран JSON-объект как достаточный для API-only маппера.

## Review Triage Log

- **2026-10-20 · blind-hunter · нет проверок длины `name`/`unit`/`code`/`created_by` и диапазона `Numeric(18,4)`** — verdict: medium. evidence: `validate_rows` проверяет только пустое имя, цену и дубль; `models.py` объявляет `String(1000)`/`String(100)`/`String(255)`/`Numeric(18,4)`, поэтому длинная строка падает на flush → 500 и откат всего импорта. route: patch → отклонять строку с причиной до записи.
- **2026-10-20 · blind-hunter · пустой `unit` принимается** — verdict: medium. evidence: `unit` не валидируется, `_cell` возвращает `""`, строка попадает в каталог без единицы измерения. route: patch → `rejected` с причиной.
- **2026-10-20 · blind-hunter · отрицательная/нулевая цена и экспонента принимаются** — verdict: low. evidence: `parse_rate_amount` принимает `-5`, `0`, `1e5`; отрицательная цена — недостоверные данные, нулевая допустима (бесплатный материал), `1e5` — корректное число 100000. route: patch → отклонять только отрицательные.
- **2026-10-20 · blind-hunter · дубли `code` без case/whitespace-нормализации** — verdict: false. evidence: спека требует отклонять «повторяющийся code»; реализация сравнивает точные строки после `strip()`, регистронезависимое равенство не заявлено и именованного вреда нет. 
- **2026-10-20 · blind-hunter · нет DB unique на `(version, code)`** — verdict: low. evidence: дубли ловятся в памяти по спеке; DB-констрейнт потребовал бы менять миграцию и защищает состояние, которое use case уже не пропускает. route: reject (fix > direct correction).
- **2026-10-20 · blind-hunter · гонка `next_version_number` (max+1 без lock)** — verdict: low. evidence: дефект реален только при одновременных импортах одной компании; в текущем single-instance MVP не воспроизводится, фикс требует блокировок/ретраев. route: reject.
- **2026-10-20 · blind-hunter · `current_version_id` без FK** — verdict: false. evidence: отсутствие FK намеренно задокументировано (циклическая ссылка, sqlite); записать чужую версию нечем — единственный writer всегда берёт версию того же каталога. 
- **2026-10-20 · blind-hunter · необработанные исключения (MinIO/IntegrityError) отдают не problem+json** — verdict: low. evidence: спека требует копию `problem.py` company-org, где тоже нет catch-all; добавление глобального хендлера — больше, чем прямая правка. route: reject.
- **2026-10-20 · blind-hunter · файл и строки читаются в память до проверки лимитов** — verdict: low. evidence: лимит 20 МиБ всё равно применяется, сервис внутренний (за gateway); потоковая обработка — нетривиальная переделка. route: reject.
- **2026-10-20 · blind-hunter · orphan-объект в MinIO при ошибке commit** — verdict: low. evidence: исходник кладётся до commit, при сбое остаётся объект без версии; пользовательского вреда нет, компенсация усложняет код. route: reject.
- **2026-10-20 · blind-hunter · имя файла без санитизации в ключе MinIO** — verdict: low. evidence: ключ плоский и уникален по `version_id`, `..`/`/` не выходят за bucket; вреда не показано. route: reject.
- **2026-10-20 · edge-case-hunter · импорт с нулём валидных строк создаёт и активирует пустую версию** — verdict: medium. evidence: `ImportRates` не проверяет `rates`; первый полностью невалидный импорт выставляет `current_version_id`, после чего следующий валидный импорт уже не активируется — расценки навсегда остаются вне активной версии. route: patch → 422 «no valid rows» (переиспользуется существующий путь для «нет строк данных»), без записи каталога/версии/файла.
- **2026-10-20 · edge-case-hunter · битый XML внутри валидного zip → ParseError → 500** — verdict: medium. evidence: `load_workbook` ловит `ValueError`, но `xml.etree.ElementTree.ParseError` (подкласс `SyntaxError`) всплывает из `iter_rows` вне `try`; спека обещает 422 для повреждённого xlsx. route: patch → ловить `ParseError` и отдавать `InvalidWorkbook`.
- **2026-10-20 · edge-case-hunter · частичный явный маппинг конфликтует с автоподстановкой** — verdict: low. evidence: `resolve_columns` добавляет автопредложение поверх явных колонок и затем падает на «distinct columns», хотя явное значение должно главенствовать; воспроизведено на `{"name": 0}`. route: patch → исключать явно занятые индексы из автоподстановки.
- **2026-10-20 · edge-case-hunter · `RecordingUpstream` ослаблен (`.get(...) or ...`)** — verdict: low. evidence: правка маскирует отсутствующий стаб для метода; тесты проходят и с исходным `[request.method]` (проверено). route: patch → вернуть строгий доступ.
- **2026-10-20 · edge-case-hunter · побочные эффекты в composition root** — verdict: false. evidence: `create_engine` и `Minio(...)` не открывают соединений при импорте; сетевых вызовов нет.
- **2026-10-20 · edge-case-hunter · дефолтные креды `minioadmin`** — verdict: low. evidence: это локальный dev-compose, порты MinIO наружу не публикуются; менять дефолты — отдельное инфраструктурное решение. route: reject.
- **2026-10-20 · edge-case-hunter · `initdb` создаёт БД только на свежем volume** — verdict: low. evidence: существующий dev-volume не получит `rate_catalog`, контейнер упадёт; ограничение уже отмечено в Implementation Notes, паттерн общий для всех сервисов. route: defer (pre-existing infra).
- **2026-10-20 · edge-case-hunter · автоподстановка может привязать колонку «цена с НДС»** — verdict: maybe-false. evidence: при наличии в файле и «цена», и «цена с НДС» возможна неверная привязка; нужен реальный образец файла/решение о колонках НДС, чтобы подтвердить. route: defer (если подтвердится — medium).
- **2026-10-20 · edge-case-hunter · строки-заголовки/объединённые ячейки ломают разбор** — verdict: false. evidence: спека фиксирует «первый лист, первая строка — заголовки»; такие файлы вне интента.
- **2026-10-20 · edge-case-hunter · нет просмотра/активации версий** — verdict: false. evidence: версионирование — Story 3.2, интент 3.1 его исключает.
- **2026-10-20 · edge-case-hunter · 401 без `WWW-Authenticate`, 415 вместо 400** — verdict: false. evidence: спека явно задаёт 400 `unsupported-format` и 401 без требования заголовка.
- **2026-10-20 · edge-case-hunter · нет авторизации по роли** — verdict: false. evidence: интент требует только identity через gateway, ролевых ограничений в 3.1 нет.
- **2026-10-20 · edge-case-hunter · нет идемпотентности импорта** — verdict: false. evidence: спека прямо задаёт «повторный импорт создаёт новую версию».
- **2026-10-20 · edge-case-hunter · 50k строк синхронно без прогресса/таймаута** — verdict: false. evidence: спека фиксирует синхронный одиночный эндпоинт и лимиты.
- **2026-10-20 · blind-hunter · в диффе нет спеки/BMAD-артефактов** — verdict: false. evidence: спека лежит в `core/_bmad-output` (untracked), в дифф сервисов не входит; sprint-status обновлён.
- **2026-10-20 · edge-case-hunter · контракт ответа/OpenAPI недоописан** — verdict: false. evidence: `response_model` соответствует AC (`version_id`, `imported`, `rejected`), причины — свободный текст по AC.
- **2026-10-20 · edge-case-hunter · префикс `/v1/rate-catalog` может матчить чужие пути** — verdict: false. evidence: `_matches_prefix` сравнивает путь целиком или с `/` на границе, `-other` не матчится.
- **2026-10-20 · blind-hunter · нет структурного логирования** — verdict: low. evidence: архитектурное требование, отсутствует и в остальных сервисах; вне рамок истории. route: defer.
- **2026-10-20 · verification-gap · тест `test_corrupt_xlsx_returns_422` не покрывает битый XML в валидном zip** — verdict: medium. evidence: тест шлёт `b"not-a-zip"` (BadZipFile), ветка `ParseError` не исполняется. route: patch (та же группа, что и ParseError).
- **2026-10-20 · verification-gap · дрейф ORM и миграции: `uq_rate_catalog_version_catalog_version` есть в 0001, но не в модели** — verdict: medium. evidence: `alembic check` показывает «removed unique constraint»; следующая автогенерация удалит защиту от дублей версий. route: patch → добавить `UniqueConstraint` в модель.
- **2026-10-20 · verification-gap · тавтологичные тесты** — verdict: low. evidence: `test_empty_rows_are_skipped` сравнивает `resp.json()` сам с собой, `test_engine_uses_env_dsn` проверяет, что sqlite — это sqlite; реальные утверждения не проверяются. route: patch → заменить на проверяемые.
- **2026-10-20 · verification-gap · нет теста отката при ошибке storage** — verdict: low. evidence: заявленный rollback (Implementation Notes) не проверяется; нужен тест «ошибка put → в БД нет каталога/версии/строк». route: patch.
- **2026-10-20 · verification-gap · нет тестов passthrough 413/422 через gateway** — verdict: low. evidence: проксирование статусов проверено только на 200; негативные коды не покрыты. route: patch → добавить тест 422 problem+json.
- **2026-10-20 · verification-gap · приоритет явного маппинга не покрыт тестом** — verdict: low. evidence: тесты покрывают только явный полный и индексный маппинг, не частичный. route: patch (та же группа, что и конфликт маппинга).
- **2026-10-20 · verification-gap · Postgres-нативные поведения не автоматизированы (sqlite в тестах)** — verdict: medium. evidence: тесты идут на sqlite; Alembic на Postgres 18.6 проверен вручную (Verification); в CI нет postgres service. route: defer (открытый пункт ретро Epic 2, общий для сервисов).
- **2026-10-20 · verification-gap · README deploy устарел** — verdict: low. evidence: список БД не содержит `rate_catalog`, а MinIO назван «ещё не подключён», хотя сервис уже в compose. route: patch → обновить README.
- **2026-10-20 · verification-gap · `MINIO_SECURE` отсутствует в `.env.example`** — verdict: low. evidence: `storage.py` читает переменную, но пример её не документирует (дефолт `false`). route: patch → задокументировать.

## Verification

**Commands (выполнено 2026-10-20, после патчей ревью):**
- `cd rate-catalog && .venv/bin/ruff check .` → `All checks passed!`
- `cd rate-catalog && .venv/bin/pytest -q` → `36 passed` (импорт, I/O-матрица, ревью-регрессии, миграции, production composition)
- `cd api-gateway && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `32 passed` (вкл. новые тесты префикса rate-catalog и passthrough ошибки)
- `cd deploy && docker compose config -q && docker compose config --services` → OK; в списке `minio`, `rate-catalog`
- `cd rate-catalog && docker build -q -t rate-catalog:ci .` → образ собран
- Alembic на реальном Postgres 18.6 (`smeta1-postgres`, временная БД `rate_catalog_check`): `POSTGRES_DSN=... alembic upgrade head` → версия `0001`; `alembic check` → `No new upgrade operations detected` (модели и миграция совпадают); временная БД удалена

**Не покрыто автоматически:** импорт против реального Postgres/MinIO (тесты используют sqlite и fake-storage); smoke через `docker compose up` не запускался.
