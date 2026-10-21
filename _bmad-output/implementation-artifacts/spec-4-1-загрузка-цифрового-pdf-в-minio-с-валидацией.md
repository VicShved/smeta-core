---
title: 'Story 4.1 — Загрузка цифрового PDF в MinIO с валидацией'
type: 'feature'
created: '2026-10-20'
status: 'done'
route: 'dispatch'
baseline_commit: '8e7446490ab704a1e1a5857c9b7bbcd26603cf77'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-4-context.md'
  - '_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** `pdf-extract` — каркас только с `/health`: нет приёма PDF, проверки цифрового текстового слоя и сохранения файла, поэтому конвейер извлечения спецификации начать не с чего.

**Approach:** Вертикаль загрузки в pdf-extract по паттерну импорта rate-catalog: `POST /v1/files` (multipart) проверяет размер, формат и текстовый слой, кладёт файл в MinIO и создаёт запись `file` в собственной БД. Маршрут подключается через api-gateway, сервис — в compose.

## Boundaries & Constraints

**Always:** identity только из `X-Company-Id` (401 без него); `company_id` хранится в `file`, записи изолированы компанией (AD-013); лимит — 50 МиБ (`50 * 1024 * 1024`), ровно 50 МиБ допустимо; текстовый слой проверяется реальным парсером PDF, а не по расширению; ошибки — `application/problem+json`; слоистость AD-011; отдельная БД `pdf_extract` и свои миграции Alembic (AD-014); наружу экспонируется только gateway.

**Never:** OCR и конвертация PDF → текст (Story 4.2); привязка файла к смете/этапу; скачивание, список и удаление файлов; UI; правки контракта импорта rate-catalog; чтение чужих БД; хранение файла в БД.

**Решения (2026-10-20):** `POST /v1/files` → `201` с `FileResponse` (`id`, `mime_type`, `size`, `meta`, `created_at`), `storage_key` наружу не отдаётся; bucket `pdf-extract-files` (env `PDF_EXTRACT_BUCKET`), ключ `{company_id}/{file_id}.pdf`; `meta = {"filename", "pages"}`; `413` при размере > 50 МиБ; `415` если содержимое не PDF (объявленный Content-Type не доверяем); `422` для битого/зашифрованного PDF, PDF без текстового слоя и имени файла длиннее 255 символов. Текстовый слой — хотя бы один непробельный символ во всех страницах. Файл проверяется до записи; объект кладётся в MinIO до вставки строки, orphan-объект при сбое commit допустим (как в 3.1). Дедупликации нет: каждая загрузка — новый `file`. Bucket создаётся лениво при первой записи (как в rate-catalog).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Валидный цифровой PDF | PDF с текстом, ≤ 50 МиБ, `X-Company-Id` задан | `201`, запись `file` в БД, объект в MinIO | — |
| Файл больше лимита | > 50 МиБ | ничего не сохранено | `413` problem+json |
| Не PDF | содержимое без `%PDF-` (в т.ч. с именем `*.pdf`) | ничего не сохранено | `415` problem+json |
| PDF без текстового слоя | пустая или картинка без текста | ничего не сохранено | `422` `pdf-without-text-layer` |
| Битый/зашифрованный PDF | `%PDF-` + мусор либо пароль | ничего не сохранено | `422` problem+json |
| Нет identity | без `X-Company-Id` | — | `401` |
| Изоляция компаний | две компании загружают файлы | у каждой записи свой `company_id` | — |

</frozen-after-approval>

## Code Map

- `pdf-extract/app/domain/file.py` -- новый: `StoredFile` (`id`, `company_id`, `storage_key`, `mime_type`, `size`, `meta`, `created_by`, `created_at`); `StoredFile.create` генерирует `uuid4()` и `datetime.now(UTC)`.
- `pdf-extract/app/domain/errors.py` -- новый: база `PdfExtractError` + `UnsupportedFileFormat`, `FileTooLarge`, `PdfWithoutTextLayer`, `InvalidPdf`, `InvalidUpload`.
- `pdf-extract/app/domain/ports.py` -- новый: `FileRepository.add`, `FileStorage.put`, `PdfInspector.inspect -> PdfInspection(page_count, text_length)`.
- `pdf-extract/app/application/store_pdf.py` -- новый: `StorePdf` (`MAX_FILE_SIZE_BYTES`, `MAX_FILENAME_LENGTH=255`): длина имени → размер → magic → `inspect` → `storage.put` → `repository.add`.
- `pdf-extract/app/infrastructure/models.py` -- новый: `Base`, `FileRow` (`file`: id, company_id индекс, storage_key unique, mime_type, size, meta JSON, created_by, created_at).
- `pdf-extract/app/infrastructure/repository.py` -- новый: `SqlAlchemyFileRepository.add` (без commit).
- `pdf-extract/app/infrastructure/pdf.py` -- новый: `PdfiumInspector` на `pypdfium2`; `PdfiumError` → `InvalidPdf`; считает страницы и непробельные символы.
- `pdf-extract/app/infrastructure/storage.py` -- новый: `MinioFileStorage` (копия паттерна rate-catalog: env `MINIO_*`, `PDF_EXTRACT_BUCKET`, ленивое создание bucket).
- `pdf-extract/app/infrastructure/db.py` -- новый: `make_session_factory` (env `POSTGRES_DSN`, default `.../pdf_extract`).
- `pdf-extract/app/transport/schemas.py` -- новый: `FileResponse` (`from_attributes=True`).
- `pdf-extract/app/transport/routes.py` -- новый: `create_router(store_pdf=None)`, `_require_company_id`, `POST /v1/files` (multipart `file`, `X-Company-Id`, `X-User-Id`; `filename = file.filename or "upload.pdf"`).
- `pdf-extract/app/transport/problem.py` -- новый: обработчики `ProblemException`, `RequestValidationError`, 5 доменных ошибок (`InvalidUpload` → 422).
- `pdf-extract/app/wiring.py` -- новый: `make_store_pdf` (одна сессия на вызов, commit после use case).
- `pdf-extract/app/main.py` -- подключить проблемные обработчики и `create_router`.
- `pdf-extract/pyproject.toml` -- deps: `python-multipart`, `sqlalchemy`, `alembic`, `psycopg[binary]`, `minio`, `pypdfium2`; dev: `reportlab` для генерации PDF в тестах.
- `pdf-extract/alembic.ini`, `pdf-extract/migrations/env.py`, `pdf-extract/migrations/versions/0001_create_file.py` -- новые; миграция таблицы `file`.
- `pdf-extract/Dockerfile` -- `alembic upgrade head` перед uvicorn, копирование миграций.
- `pdf-extract/tests/test_upload.py` -- новый: I/O-матрица на sqlite+Alembic, fake-storage, настоящий `PdfiumInspector`; PDF генерируются reportlab (текст в фикстурах — ASCII, без зависимости от шрифтов).
- `pdf-extract/tests/test_migrations.py` -- новый: `upgrade head` создаёт `file`.
- `deploy/initdb/01-create-databases.sql` -- добавить `pdf_extract`.
- `deploy/docker-compose.yml` -- сервис `pdf-extract` (Postgres+MinIO, bucket env) и `PDF_EXTRACT_URL`/`depends_on` у gateway.
- `deploy/README.md` -- обновить состав сервисов, список БД и назначение MinIO.
- `api-gateway/app/application/routing.py` -- `DEFAULT_PDF_EXTRACT_URL` и префикс `/v1/files`.
- `api-gateway/app/wiring.py` -- env `PDF_EXTRACT_URL`.
- `api-gateway/tests/test_gateway.py` -- проксирование `/v1/files` (401 без токена, маршрут с токеном).
- `rate-catalog/*`, `estimate/*`, `calc-service/*` -- не трогать.

## Tasks & Acceptance

**Execution:**
- [x] `pdf-extract/app/domain/{file,errors,ports}.py` -- доменные типы, ошибки и порты -- application не знает про FastAPI/SQLAlchemy/MinIO.
- [x] `pdf-extract/app/application/store_pdf.py` -- `StorePdf` -- порядок проверок и создание записи.
- [x] `pdf-extract/app/infrastructure/{db,models,repository}.py` -- БД и таблица `file` -- компания на каждой записи.
- [x] `pdf-extract/app/infrastructure/{pdf,storage}.py` -- `PdfiumInspector` и MinIO-адаптер -- реальная проверка текстового слоя и хранение.
- [x] `pdf-extract/migrations/*`, `alembic.ini` -- миграция `0001` -- схема `file` совпадает с моделью.
- [x] `pdf-extract/app/transport/{schemas,routes,problem}.py`, `wiring.py`, `main.py` -- `POST /v1/files` -- контракт, 401 и problem+json.
- [x] `pdf-extract/pyproject.toml`, `Dockerfile` -- зависимости и запуск миграций -- сервис собирается и стартует.
- [x] `pdf-extract/tests/{test_upload,test_migrations}.py` -- I/O-матрица и миграции -- регрессия.
- [x] `api-gateway/app/{application/routing,wiring}.py`, `tests/test_gateway.py` -- префикс `/v1/files` -- загрузка доступна только через edge.
- [x] `deploy/initdb/01-create-databases.sql`, `deploy/docker-compose.yml`, `deploy/README.md` -- БД, сервис, env -- локальный стек поднимает pdf-extract.
- [x] `pdf-extract` + `api-gateway` -- ruff/pytest, `deploy` -- `docker compose config -q` -- верификация.

**Acceptance Criteria:**
- Given авторизованный пользователь с `X-Company-Id`, when загружает цифровой PDF ≤ 50 МиБ, then `201`, в БД есть `file` с `storage_key`/`mime_type`/`size`/`meta`, объект доступен в MinIO.
- Given PDF > 50 МиБ, when загрузка, then `413` problem+json и ни записи, ни объекта.
- Given PDF без текстового слоя, when загрузка, then `422` с типом `pdf-without-text-layer` и без сохранения.
- Given файл не PDF, when загрузка, then `415` problem+json.
- Given две компании, when каждая загружает файл, then записи содержат разные `company_id`, и выборка одной компании не видит файл другой.
- Given нет `X-Company-Id`, when загрузка, then `401` до домена.

## Implementation Notes

- **Слои (AD-011):** `domain` содержит `StoredFile`/`PdfInspection`, ошибки и Protocol-порты; `application/store_pdf.py` знает только порты; pypdfium2 изолирован в `infrastructure/pdf.py`, MinIO — в `infrastructure/storage.py`, SQLAlchemy — в `infrastructure/{db,models,repository}.py`; FastAPI — только в `transport`. Composition root — `app/wiring.py`.
- **Порядок валидации:** длина имени (≤255 символов) → размер (≤`50 * 1024 * 1024`, ровно 50 МиБ допустимо) → magic `%PDF-` (объявленный Content-Type и расширение не доверяются) → реальный разбор pypdfium2. Текстовый слой — хотя бы один непробельный символ на всех страницах; иначе `PdfWithoutTextLayer` (422, `pdf-without-text-layer`).
- **Запись:** `StoredFile.create` генерирует UUID и `datetime.now(UTC)`, ключ — `{company_id}/{file_id}.pdf`. Объект кладётся в MinIO (bucket `PDF_EXTRACT_BUCKET`, ленивое создание) до вставки строки `file`; репозиторий не коммитит, commit/rollback — в `make_store_pdf`. Orphan-объект при сбое commit допустим по решению истории. Дедупликации нет.
- **Контракт:** `POST /v1/files` (multipart `file`, `X-Company-Id`, опциональный `X-User-Id`) → `201` с `FileResponse` (`id`, `mime_type`, `size`, `meta`, `created_at`), `storage_key` наружу не отдаётся; `meta = {filename, pages}`. Нет `X-Company-Id` → 401 до use case. Ошибки — `application/problem+json`; битый/зашифрованный PDF → 422 `invalid-pdf`.
- **БД/деплой:** отдельная база `pdf_extract`, миграция `0001_create_file` (id, company_id с индексом, storage_key unique, mime_type, size, meta JSON, created_by, created_at); Dockerfile выполняет `alembic upgrade head` перед uvicorn. В compose добавлен внутренний сервис `pdf-extract` (Postgres + MinIO, только `expose`), gateway получил `PDF_EXTRACT_URL` и префикс `/v1/files`.
- **Тесты:** sqlite + реальные Alembic-миграции; fake-storage; настоящий `PdfiumInspector`; PDF генерируются reportlab (ASCII). Покрыты валидная загрузка, ровно 50 МиБ, >50 МиБ, не-PDF, PDF без текста, битый/зашифрованный PDF, длинное имя, 401, изоляция компаний, откат при сбое storage и production-композиция.

## Spec Change Log

- 2026-10-21: лимит файла изменён с 5 МиБ на 50 МиБ по решению владельца; обновлены `MAX_FILE_SIZE_BYTES`, текст `413`, тесты, nginx (`client_max_body_size 51m`) и документация.

## Review Triage Log

- **2026-10-20 · blind-hunter + edge-case-hunter · multipart читается целиком до проверки лимита** — verdict: medium. evidence: `app/transport/routes.py:43` делает `file.file.read()` без ограничения, а `FileTooLarge` бросается уже в use case (`store_pdf.py:22`); при 50 МиБ контракт соблюдён, но память растёт на любой размер. route: patch → читать не более `MAX_FILE_SIZE_BYTES + 1`.
- **2026-10-20 · blind-hunter + edge-case-hunter + verification-gap · `created_by` длиннее 255 вызывает ошибку БД после записи в MinIO** — verdict: medium. evidence: `X-User-Id` без валидации попадает в `String(255)` (`models.py:27`); в rate-catalog аналогичная проверка есть (`import_rates.py:167`), здесь её нет; через gateway `sub` — UUID, но прямой вызов с длинным заголовком даёт 500 и orphan-объект. route: patch → `MAX_CREATED_BY_LENGTH=255` и `InvalidUpload`.
- **2026-10-20 · blind-hunter + edge-case-hunter · гонка ленивого создания bucket** — verdict: medium. evidence: `storage.py:51-54` проверяет `bucket_exists` и затем `make_bucket`; два первых параллельных запроса дают `BucketAlreadyOwnedByYou` одному из них. route: patch → идемпотентно обрабатывать это S3-исключение.
- **2026-10-20 · blind-hunter + edge-case-hunter · `MINIO_SECURE` не передаётся в pdf-extract** — verdict: medium. evidence: `.env.example:39` документирует переменную, адаптер её читает (`storage.py:29`), но compose-сервис `pdf-extract` её не задаёт; при HTTPS-MinIO загрузки пойдут по HTTP. route: patch → добавить env в compose.
- **2026-10-20 · blind-hunter + edge-case-hunter · PDF, зашифрованный только owner-паролем, принимается как валидный** — verdict: medium. evidence: проверено на reportlab-фикстуре: `PdfDocument` открывается, `FPDF_GetSecurityHandlerRevision` = 2, текст извлекается, ответ 201; решение спеки требует 422 для зашифрованных. route: patch → проверять revision и бросать `InvalidPdf`.
- **2026-10-20 · blind-hunter + edge-case-hunter · 404/405 и неожиданные 500 отдаются не problem+json** — verdict: medium. evidence: `install_problem_handlers` покрывает только доменные ошибки и `RequestValidationError`; `BoomStorage` в тесте даёт обычный 500, `GET /v1/files` — обычный 405. route: patch → обработчики `StarletteHTTPException` и `Exception`.
- **2026-10-20 · blind-hunter · 401 без `WWW-Authenticate`** — verdict: low. evidence: статус и problem+json корректны, заголовок в RFC 7235 — SHOULD, gateway отдаёт 401 так же; пользователь не встречает функционального дефекта. route: reject.
- **2026-10-20 · blind-hunter · нет лимита страниц/времени разбора** — verdict: maybe-false. evidence: теоретический CPU-DoS на крафтовом PDF; вред не воспроизведён. route: defer → нужен бенчмарк 5-МиБ PDF с большим числом страниц.
- **2026-10-20 · blind-hunter · `/health` не проверяет БД/MinIO** — verdict: low. evidence: liveness-проба — общий паттерн всех сервисов; readiness-эндпоинт в интент не входит, фикс добавляет публичную поверхность. route: reject.
- **2026-10-20 · blind-hunter + verification-gap · тесты миграций не сверяют модель/nullable и не проверяют downgrade** — verdict: medium. evidence: проверено мутацией `created_by nullable=False` — набор зелёный; тест смотрит только имена колонок, модель не импортирует. route: patch → сравнение с `FileRow.__table__`, `downgrade`→`upgrade`, загрузка без `X-User-Id`.
- **2026-10-20 · blind-hunter + verification-gap · `MinioFileStorage` не исполняется ни одним тестом** — verdict: medium. evidence: все тесты внедряют `FakeStorage`, production-тест подменяет класс; удаление `_ensure_bucket`/`put_object` не ломает набор, AC «объект в MinIO» не проверен. route: patch → unit-тест со stub-клиентом (bucket, аргументы `put_object`).
- **2026-10-20 · blind-hunter · не покрыты edge-кейсы: нет part `file`, пустое имя, отсутствие `X-User-Id`, 0 байт, пробельный текст** — verdict: medium. evidence: матрица покрыта, но контрактные ветки не исполняются; отсутствие пользователя разрешено роутом и не проверено. route: patch → добавить тесты; кириллица отклонена как зависящая от шрифтов.
- **2026-10-20 · blind-hunter · gateway не проверяет проброс problem+json от pdf-extract** — verdict: low. evidence: прокси не переписывает статус/тело, но регрессию 413/415/422 ничто не ловит. route: patch → тест passthrough.
- **2026-10-20 · blind-hunter + verification-gap · `initdb` не создаёт `pdf_extract` на существующем volume** — verdict: medium. evidence: скрипт исполняется только на пустом `postgres-data`; это уже зафиксировано в `deferred-work.md` как общий infra-паттерн для `rate_catalog`. route: defer.
- **2026-10-20 · blind-hunter · README pdf-extract и комментарии deploy устарели** — verdict: low. evidence: `pdf-extract/README.md` говорит «каркас», нет контракта/переменных/миграций; комментарий MinIO в compose упоминает только rate-catalog. route: patch → обновить документацию.
- **2026-10-20 · blind-hunter · Dockerfile игнорирует `uv.lock`** — verdict: low. evidence: `pip install .` — тот же паттерн, что в rate-catalog; воспроизводимость сборки не входит в интент этой истории. route: defer.
- **2026-10-20 · edge-case-hunter · пустой `X-Company-Id` даёт 422 вместо 401** — verdict: false. evidence: пустая строка — невалидный UUID, а не отсутствие заголовка; отсутствие по-прежнему даёт 401 (`routes.py:31`), gateway всегда присылает UUID. route: reject.
- **2026-10-20 · blind-hunter · gateway буферизует тело и имеет таймаут 10 с** — verdict: false (таймаут). evidence: буферизация — существующий дизайн прокси для всех маршрутов; таймаут применяется к запросу gateway→pdf-extract внутри Docker-сети, где 50 МиБ передаются заведомо быстрее 10 с. Ограничение тела на edge — pre-existing, не этой истории. route: reject таймаут; edge-лимит → defer.
- **2026-10-20 · verification-gap · удаление problem-обработчиков не ломает тесты** — verdict: medium. evidence: проверено мутацией: production `main.app` без обработчиков даёт 500/plain на запрос без `X-Company-Id`, а набор остаётся зелёным. route: patch → ошибочный запрос через `main.app`.
- **2026-10-20 · verification-gap · ветка `POSTGRES_DSN` в `migrations/env.py` не тестируется** — verdict: medium. evidence: проверено мутацией: без env-ветки `alembic upgrade head` уходит в `alembic.ini` и падает; фактический путь сейчас рабочий (проверен вручную на sqlite). route: patch → тест только с env.
- **2026-10-20 · verification-gap · правило текстового слоя не закреплено для поздней страницы и пробелов** — verdict: medium. evidence: проверено мутацией: проверка только первой страницы и `len(text)` вместо непробельных символов оставляют набор зелёным. route: patch → тесты «текст на второй странице» и «только пробелы».
- **2026-10-20 · verification-gap · лимиты 50 МиБ и 255 символов не закреплены литералами** — verdict: low. evidence: проверено мутацией: замена констант на 4 МиБ/100 оставляет набор зелёным; тесты самосогласованы. route: patch → ассерты констант против литералов.
- **2026-10-20 · blind-hunter · нет тестов на Cyrillic и сбой БД** — verdict: low. evidence: кириллица зависит от шрифта фикстуры, а orphan-объект при сбое commit прямо разрешён решением спеки; отдельного контрактного дефекта нет. route: reject.
- **2026-10-20 · blind-hunter · схема не проверяется на реальном PostgreSQL** — verdict: low. evidence: типы `Uuid`/`DateTime(timezone=True)` совпадают в модели и миграции; поднятие Postgres в unit-CI — infra-задача вне истории. route: reject (отмечено в Verification как непокрытое).


## Verification

**Commands:**
- `cd pdf-extract && .venv/bin/ruff check .` -- expected: `All checks passed!`
- `cd pdf-extract && .venv/bin/pytest -q` -- expected: все тесты проходят (upload, миграции, health)
- `cd api-gateway && .venv/bin/ruff check . && .venv/bin/pytest -q` -- expected: без регрессий
- `cd deploy && docker compose config -q` -- expected: exit 0
- `cd pdf-extract && docker build -q -t pdf-extract:ci .` -- expected: образ собран

**Результаты (2026-10-20, после патчей ревью):**
- `pdf-extract`: `All checks passed!`; `36 passed` (health, upload, MinIO-адаптер, миграции).
- `api-gateway`: `All checks passed!`; `36 passed`.
- `deploy`: `docker compose config -q` — exit 0.
- `pdf-extract`: образ пересобран (`pdf-extract:ci`); в контейнере `import app.main` и `import pypdfium2` успешны.
- `uv lock --check` в `pdf-extract`: lock согласован с `pyproject.toml`.

**Не покрыто автоматически:** реальная связка Postgres + MinIO (тесты — sqlite и stub/fake-адаптеры); схема на PostgreSQL (типы/uuid проверены на sqlite); orphan-объект при сбое commit после успешного `put` (разрешён решением спеки); крайний случай гонки bucket проверен stub-клиентом, не реальным MinIO.
