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
  status: resolved
  resolved_by: `_bmad-output/implementation-artifacts/spec-6-3-детерминированный-расчёт-стоимости.md` (2026-10-21)

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

## Deferred from: code review of spec-4-2-конвертация-pdf-текст-без-ocr.md (2026-10-20)

- source_spec: `_bmad-output/implementation-artifacts/spec-4-2-конвертация-pdf-текст-без-ocr.md`
  summary: Обновление `extracted_at` при повторном extract не проверяется тестом: содержимое обновляется, но свежесть артефакта не закреплена.
  evidence: AC требует обновления содержимого и отсутствия дубля; точная проверка времени требует управления часами или отдельного источника времени, которого в сервисе нет.

## Deferred from: code review of spec-5-1-llm-извлечение-спецификации-local-first.md (2026-10-20)

- source_spec: `_bmad-output/implementation-artifacts/spec-5-1-llm-извлечение-спецификации-local-first.md`
  summary: Бюджет контекста 3000 символов чанка + 2048 токенов вывода и целая строка >3000 могут превысить n_ctx=4096; нужна проверка на реальном токенайзере/модели.
  evidence: Без GGUF-модели точный token-count не измерить; решение «не разрывать строку» зафиксировано в спеке.
- source_spec: `_bmad-output/implementation-artifacts/spec-5-1-llm-извлечение-спецификации-local-first.md`
  summary: `LlamaClient.chat` теряет `finish_reason`; обрезка по `max_tokens` неотличима от невалидного JSON и ведёт к 502 без диагностики.
  evidence: Для исправления нужен новый API клиента и прогон на реальной модели; сейчас fake-LLM не воспроизводит `length`.
- source_spec: `_bmad-output/implementation-artifacts/spec-5-1-llm-извлечение-спецификации-local-first.md`
  summary: Чанкер не переносит заголовок раздела и теряет пустые строки/маркеры страниц; `section` в поздних чанках может быть пустым или неверным.
  evidence: Качество извлечения раздела требует реальных ГРАНД/АВК PDF; без корпуса эффект не измерить.
- source_spec: `_bmad-output/implementation-artifacts/spec-5-1-llm-извлечение-спецификации-local-first.md`
  summary: У промпта нет версии и few-shot примеров для типовых случаев смет; результаты невоспроизводимы между изменениями промпта.
  evidence: Версионирование/примеры имеют смысл после появления модели и эталонного корпуса для сравнения качества.
- source_spec: `_bmad-output/implementation-artifacts/spec-5-1-llm-извлечение-спецификации-local-first.md`
  summary: Сериализация вызовов модели под общим lock не покрыта конкурентным тестом.
  evidence: Нужна event/timing-инфраструктура; lock реализован, но регрессию на перекрытие вызовов suite не проверяет.

## Deferred from: code review of spec-5-2-маппинг-со-справочником-и-нормализация-еи-с-confidence.md (2026-10-20)

- source_spec: `_bmad-output/implementation-artifacts/spec-5-2-маппинг-со-справочником-и-нормализация-еи-с-confidence.md`
  summary: Hard-filter по совпавшей ЕИ может выбрать нерелевантную расценку вместо exact-name кандидата с другой ЕИ; нужна калибровка на реальном справочнике.
  evidence: Решение «совпавшая ЕИ, иначе все» зафиксировано в frozen-спеке; без реальных данных нельзя оценить, какой вариант даёт лучшее покрытие.
- source_spec: `_bmad-output/implementation-artifacts/spec-5-2-маппинг-со-справочником-и-нормализация-еи-с-confidence.md`
  summary: Exact-name при несовпавшей ЕИ получает confidence 0.9025 и needs_review=false; формула/штраф требуют калибровки.
  evidence: Формула `0.85*name + 0.15*unit` зафиксирована в frozen-спеке; реальные метрики и порог отложены.
- source_spec: `_bmad-output/implementation-artifacts/spec-5-2-маппинг-со-справочником-и-нормализация-еи-с-confidence.md`
  summary: Нет пагинации/кэша активных расценок: mapping загружает весь справочник на каждый запрос.
  evidence: Объём реального справочника в MVP неизвестен; кэш по версии/пагинация нужны после появления данных.

## Deferred from: Story 5.3 scope check (2026-10-20)

- source_spec: none
  summary: Story 5.3 «API ручной проверки маппинга» отложена до появления фундамента Epic 6 (сметы, версии и позиции, Story 6.1), поверх которого строится review API.
  evidence: `estimate` — каркас только с `/health`, моделей `estimate`/`estimate_version`/`estimate_item`, миграций и API нет; Story 5.2 stateless и предложения не сохраняет. Решение владельца (2026-10-20): вариант C — не заводить review-хранилище в mapping-service и не создавать частичный estimate в 5.3. При возврате: позиции, статус и выбранная расценка/ЕИ принадлежат estimate (AD-013); review API подтверждения/исправления и выделение `confidence < 0.70` строятся поверх `estimate_item` после 6.1; снапшоты и расчёт — Epic 6 (6.3).
  status: resolved
  resolved_by: `_bmad-output/implementation-artifacts/spec-5-3-api-ручной-проверки-маппинга.md` (2026-10-20)

## Deferred from: code review of spec-6-1-создание-сметы-и-версий-ровно-одна-активная.md (2026-10-20)

- source_spec: `_bmad-output/implementation-artifacts/spec-6-1-создание-сметы-и-версий-ровно-одна-активная.md`
  summary: Postgres-ветка частичного уникального индекса `uq_estimate_version_one_active` (`postgresql_where`) не исполняется ни одним тестом; SQLite-only прогон не ловит её поломку.
  evidence: Все тесты миграций используют `sqlite:///`; в CI estimate/deploy нет Postgres-сервиса. Удаление `postgresql_where` оставляет suite зелёным, а на production-Postgres индекс становится обычным UNIQUE(estimate_id). Закрывается добавлением Postgres-service в CI — общая задача, уже отмеченная в ретро Epic 2/4.
- source_spec: `_bmad-output/implementation-artifacts/spec-6-1-создание-сметы-и-версий-ровно-одна-активная.md`
  summary: Entrypoint контейнера estimate (`alembic upgrade head` + uvicorn) и COPY миграций не проверяются автоматически; сломанный CMD проявится только при деплое.
  evidence: CI запускает ruff/pytest, deploy-CI — только `docker compose config -q`; `docker build` не исполняет CMD. Нужен container-smoke (build + run с DSN и проверка `/health`) — общая инфра-задача деплоя.
- source_spec: `_bmad-output/implementation-artifacts/spec-6-1-создание-сметы-и-версий-ровно-одна-активная.md`
  summary: Dockerfile estimate ставит зависимости через `pip install .`, игнорируя `uv.lock`; сборка не воспроизводима по lock-файлу.
  evidence: Тот же паттерн у rate-catalog/pdf-extract; воспроизводимость сборки — pre-existing инфра-задача, уже отмеченная в deferred-work по Story 4.1.

## Deferred from: code review of spec-6-3-детерминированный-расчёт-стоимости.md (2026-10-21)

- source_spec: `_bmad-output/implementation-artifacts/spec-6-3-детерминированный-расчёт-стоимости.md`
  summary: Расчёт версии делает до 1000 последовательных HTTP-вызовов к rate-catalog (по одному на уникальную расценку) внутри открытой транзакции.
  evidence: `estimate/app/application/calculate_cost.py` разрешает расценки поштучно через `GET /v1/rate-catalog/rates/{id}`; батч-эндпоинта и общего дедлайна нет, таймаут клиента 10 с на запрос. Нужен батч-endpoint в rate-catalog либо лимит/дедлайн на расчёт.
- source_spec: `_bmad-output/implementation-artifacts/spec-6-3-детерминированный-расчёт-стоимости.md`
  summary: Запись результата расчёта идёт построчно (SELECT с двумя JOIN'ами + UPDATE на каждую позицию) — до 1000 итераций с flush.
  evidence: `estimate/app/infrastructure/repository.py::update_item_snapshot_and_cost`; на объёмах MVP (100–500 позиций) приемлемо, но при росте нужны bulk-обновления.
- source_spec: `_bmad-output/implementation-artifacts/spec-6-3-детерминированный-расчёт-стоимости.md`
  summary: Расчёт не оставляет след об акторе: `X-User-Id` не читается, у версии нет `calculated_by`.
  evidence: `estimate/app/transport/routes.py` принимает только `X-Company-Id`; смета хранит `created_by`, версия — нет. Нужна колонка `calculated_by` и её заполнение.
- source_spec: `_bmad-output/implementation-artifacts/spec-6-3-детерминированный-расчёт-стоимости.md`
  summary: Нет переноса подтверждённых позиций в новую версию сметы: импорт 5.3 создаёт позиции в статусе `proposed`, поэтому подтверждения и скорректированная ЕИ теряются.
  evidence: `estimate/app/application/import_items.py` всегда создаёт `status=proposed`; новые цены по README требуют новой версии и повторного подтверждения всех позиций. Нужна операция копирования позиций с сохранением статусов.

## Deferred from: implementation of spec-7-1a-ui-вход-и-дерево-смет.md (2026-10-21)

- source_spec: `_bmad-output/implementation-artifacts/spec-7-1a-ui-вход-и-дерево-смет.md`
  summary: `GET /v1/estimates?stage_id=` делает лишний HTTP-вызов в company-org на каждый запрос (подтверждение скоупа этапа по дереву договоров) — кэша нет.
  evidence: `estimate/app/application/list_estimates.py` вызывает `CompanyOrgPort.list_company_stages` → `GET /v1/contracts/tree`; вызов нужен, чтобы отличать «пустой этап» (`200`, `items: []`) от «чужой/неизвестный этап» (`404`) — без него скоуп этапа неотличим от пустого. На объёмах MVP (десятки договоров) приемлемо; при росте иерархии нужен кэш этапов компании или отдельный endpoint этапов.
- source_spec: `_bmad-output/implementation-artifacts/spec-7-1a-ui-вход-и-дерево-смет.md`
  summary: UI покрыт только vitest/jsdom: браузерного e2e-прогона (Playwright/Cypress) в репозитории нет, реальный браузер и SPA-fallback проверены вручную (compose-стенд, curl).
  evidence: `ui/tests/*` — unit/component-тесты на jsdom; `ui/package.json` не содержит e2e-фреймворка. Для Epic 7.1b (загрузка PDF) и 7.2 (таблицы позиций) понадобится e2e-контур либо ручная проверка на стенде.
- source_spec: `_bmad-output/implementation-artifacts/spec-7-1a-ui-вход-и-дерево-смет.md`
  summary: Имя компании при регистрации UI выводится из локальной части email: `auth-users` требует `company_name` в теле, а frozen-спека оставляет в форме только email/пароль.
  evidence: `ui/src/views/RegisterView.ts` (`default_company_name`), контракт `POST /v1/auth/register` — `company_name` обязателен. Сейчас пользователь видит предполагаемое имя компании в форме и не может его изменить; отдельный выбор компании в скоупе 7.1a запрещён («новые роли/компании» в Never).

## Deferred from: code review of spec-7-1a-ui-вход-и-дерево-смет.md (2026-10-21)

- source_spec: `_bmad-output/implementation-artifacts/spec-7-1a-ui-вход-и-дерево-смет.md`
  summary: В UI не настроен CSP — при токене в `localStorage` это оставляет основной вектор кражи токена.
  evidence: `ui/nginx.conf` добавляет `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, но без `Content-Security-Policy`: у SPA есть встроенные стили и динамические импорты, слепой CSP сломал бы сборку. Нужен отдельный проход с проверкой сборки под строгим CSP.
- source_spec: `_bmad-output/implementation-artifacts/spec-7-1a-ui-вход-и-дерево-смет.md`
  summary: `GET /v1/estimates?stage_id=` считает `total` и `items` двумя запросами — при записи между ними счётчик и список могут разойтись.
  evidence: `estimate/app/infrastructure/repository.py` — `count_by_stage` и `list_by_stage`; на объёмах MVP (десятки смет на этап) расхождение маловероятно, но пагинация показывается пользователю. Исправляется одним запросом с `COUNT(*) OVER ()`.
- source_spec: `_bmad-output/implementation-artifacts/spec-7-1a-ui-вход-и-дерево-смет.md`
  summary: nginx-прокси жёстко привязан к docker-сети (`resolver 127.0.0.11`, `api-gateway:8000`) — переопределить upstream нельзя.
  evidence: `ui/nginx.conf`; в compose работает, но для внешнего стенда (Vite dev-server, иной хост) потребуется параметризация через envsubst.

## Deferred from: code review of spec-7-1b-ui-загрузка-pdf-и-статусы-обработки.md (2026-10-21)

- source_spec: `_bmad-output/implementation-artifacts/spec-7-1b-ui-загрузка-pdf-и-статусы-обработки.md`
  summary: Сбор сводного статуса удерживает сессию БД во время HTTP-вызова в pdf-extract — при медленном upstream пул соединений расходуется.
  evidence: `estimate/app/application/processing_status.py` вызывает `pdf_extract.get_file` внутри `with self._session_factory()`; на текущем объёме (десятки смет) это незаметно, но при росте нагрузки нужен вынос вызова за пределы транзакции либо отдельная сессия.
