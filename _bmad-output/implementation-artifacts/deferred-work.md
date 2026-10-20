- source_spec: `_bmad-output/implementation-artifacts/spec-2-1-авторизация-и-привязка-к-компании.md`
  summary: Edge-слой аутентификации — api-gateway проверяет JWT, затирает/устанавливает identity-заголовки и проксирует запросы, наружу публикуется только он
  evidence: Вынесено из Story 2.1 при сужении объёма (спека ~3560 токенов > 1600). Ядро истории (регистрация, компания, вход/токен) закрывается без gateway; edge-контракт AD-010 — отдельный shippable deliverable, зависит только от формата claims из этой истории.
  status: resolved
  resolved_by: `_bmad-output/implementation-artifacts/spec-2-gateway-jwt-i-identity-заголовки.md` (2026-10-19, внеплановая build-спека)
- source_spec: `_bmad-output/implementation-artifacts/spec-2-2-договоры-произвольной-глубины-adjacency-list.md`
  summary: `epic-2-context.md` потерял упоминание «один контейнер PostgreSQL в compose» в AD-014 и сохраняет противоречие: цель эпика говорит, что edge-слой уже реализован Story 2.0, а Cross-Story Dependencies — что «Epic 2 вводит фактический edge-слой»
  evidence: Правки `epic-2-context.md` сделаны до baseline Story 2.2 (docs-работа прошлой сессии), не этой историей; при ревью диффа видно, что AD-014 сокращён до «Отдельная база на сервис», а строка про edge-слой осталась от прежней редакции.

## Deferred from: code review of spec-2-1-авторизация-и-привязка-к-компании.md (2026-10-20)

- Миграции и запросы проверяются только на SQLite, а деплой идёт на PostgreSQL: нет ни одного CI-прогона `alembic upgrade head`/запросов на `postgresql+psycopg`. Требуется postgres service-контейнер в CI или compose-smoke (закрывается patch-находкой о проверке контура контейнера).

## Deferred from: code review of spec-2-2-договоры-произвольной-глубины-adjacency-list.md (2026-10-20)

- `deploy/README.md` рассинхронизирован с compose (заявляет «Postgres 16» при `postgres:18.6`; «database-per-service — open question» при реализованном AD-014). Правка относится к репозиторию `deploy` и к Story 2.1 — сделана в рамках её Review Findings.

## Deferred from: code review of spec-2-3-этапы-договора.md (2026-10-20)

- `make_session_factory` не кэширует engine: каждый `make_*_runner` строит свою фабрику, на один импорт `main.py` создаётся 5 engine с отдельными пулами. Pre-existing паттерн Story 2.1/2.2; фикс — `@lru_cache`/единый sessionmaker в wiring.


## Deferred from: code review of spec-3-1-импорт-справочника-из-excel-с-маппером-колонок.md (2026-10-20)

- source_spec: `_bmad-output/implementation-artifacts/spec-3-1-импорт-справочника-из-excel-с-маппером-колонок.md`
  summary: `initdb` создаёт базу `rate_catalog` только на свежем volume; существующий dev-volume не получит её, и контейнер rate-catalog упадёт.
  evidence: `deploy/initdb/01-create-databases.sql` выполняется Postgres только при инициализации пустого `postgres-data`; ограничение отмечено в Implementation Notes спеки и общее для всех сервисов (pre-existing infra-паттерн).
- source_spec: `_bmad-output/implementation-artifacts/spec-3-1-импорт-справочника-из-excel-с-маппером-колонок.md`
  summary: Автоподстановка маппера может привязать колонку «цена с НДС» к `rate_amount`, хотя справочник должен хранить цену без НДС.
  evidence: `FIELD_ALIASES["rate_amount"]` содержит подстроки «цена»/«стоимость»; при наличии в файле и «цена», и «цена с НДС» выбор зависит от порядка колонок. Требуется образец реального файла или правило приоритета колонок НДС; если подтвердится — severity medium.
- source_spec: `_bmad-output/implementation-artifacts/spec-3-1-импорт-справочника-из-excel-с-маппером-колонок.md`
  summary: Тесты rate-catalog выполняются на sqlite, CI не поднимает Postgres, поэтому `Numeric(18,4)`, `DataError` и уникальные констрейнты не проверяются автоматически.
  evidence: `tests/test_import.py` мигрирует sqlite-БД; прогон Alembic на Postgres 18.6 и `alembic check` сделаны вручную (Verification). Закрывается добавлением postgres service-контейнера в CI — уже числится открытым пунктом ретро Epic 2.
- source_spec: `_bmad-output/implementation-artifacts/spec-3-1-импорт-справочника-из-excel-с-маппером-колонок.md`
  summary: Нет структурного JSON-логирования с `request_id` и именем сервиса, хотя этого требует архитектура.
  evidence: В rate-catalog, как и в остальных сервисах, отсутствует logging-конфигурация; требование общее для системы, а не следствие Story 3.1.

## Deferred from: Story 3.3 scope check (2026-10-20)

- source_spec: none
  summary: Story 3.3 «Snapshot расценок в позициях смет» отложена до появления фундамента Epic 6 (estimate-модели/версии/позиции и детерминированный расчёт).
  evidence: `estimate` и `calc-service` — каркасы только с `/health`: нет моделей `estimate`/`estimate_version`/`estimate_item`, миграций, БД `estimate` в compose и API; у rate-catalog нет чтения строк версии. AC «позиции сохраняют snapshot при расчёте» и «повторный расчёт» проверить нечем. Решение владельца (2026-10-20): отложить до Epic 6; при возврате учесть, что позиции принадлежат estimate, а rate-catalog должен отдавать неизменяемые расценки версии по API (AD-014 запрещает прямое чтение чужой БД).

## Deferred from: code review of spec-4-1-загрузка-цифрового-pdf-в-minio-с-валидацией.md (2026-10-20)

- source_spec: `_bmad-output/implementation-artifacts/spec-4-1-загрузка-цифрового-pdf-в-minio-с-валидацией.md`
  summary: Нет лимита числа страниц/времени разбора PDF: крафтовый 5-МиБ документ может надолго занять CPU.
  evidence: Теоретический CPU-DoS не воспроизведён; нужен бенчмарк PDF с большим числом страниц, чтобы выбрать безопасный лимит, не отсекая легитимные сметы.
- source_spec: `_bmad-output/implementation-artifacts/spec-4-1-загрузка-цифрового-pdf-в-minio-с-валидацией.md`
  summary: `initdb` создаёт `pdf_extract` только на свежем volume; существующий dev-volume требует ручного `CREATE DATABASE`.
  evidence: `deploy/initdb/01-create-databases.sql` исполняется только при инициализации пустого `postgres-data`; ограничение общее для всех сервисов и уже зафиксировано для `rate_catalog`, в README добавлена команда создания базы.
- source_spec: `_bmad-output/implementation-artifacts/spec-4-1-загрузка-цифрового-pdf-в-minio-с-валидацией.md`
  summary: Dockerfile ставит зависимости через `pip install .`, игнорируя закоммиченный `uv.lock`.
  evidence: Тот же паттерн у rate-catalog; воспроизводимость сборки по lock-файлу — общая infra-задача, не следствие Story 4.1.
- source_spec: `_bmad-output/implementation-artifacts/spec-4-1-загрузка-цифрового-pdf-в-minio-с-валидацией.md`
  summary: api-gateway буферизует тело запроса целиком и не имеет edge-лимита на размер загрузки.
  evidence: `content=await request.body()` — pre-existing дизайн прокси для всех маршрутов (rate-catalog тоже принимает файлы); ограничение на edge меняет общий контракт gateway.
