# Review: web/reality-check архитектурного spine

- **Verdict: FAIL — нужна правка до принятия.** 2 high, 3 medium, 3 low.
- Reviewer: web/reality-check lens, run inline (subagents unavailable).
- Date: 2026-10-26.
- Scope: `ARCHITECTURE-SPINE.md`, код `smeta/estimate`, `smeta/rate-catalog`, `smeta/ui`, `smeta/deploy`, lock-файлы. Web-проверка — только для спорных версий (FastAPI, PostgreSQL, MinIO).
- Spine не редактировался.

## Метод

- Версии сверены с фактическими lock-файлами/манифестами репозитория, а не с памятью: `estimate/uv.lock`, `spec-service/uv.lock`, `llm-proxy/uv.lock`, `ui/package-lock.json`, `*/pyproject.toml`, `*/Dockerfile`, `deploy/docker-compose.yml`.
- Web: PyPI/release notes для FastAPI, postgresql.org для PostgreSQL 18.6, Docker Hub/GitHub для MinIO.
- Наличие AD-017/AD-018 в коде проверено grep-ом по `estimate`, `rate-catalog`, `ui` (включая миграции, transport, domain, UI-роуты).

## Verified (чем подтверждено)

| Tech | Pin в spine | Факт / источник | Итог |
| --- | --- | --- | --- |
| Python | >=3.12 | `estimate/pyproject.toml:5`, `Dockerfile:2` (`python:3.12-slim`) | OK |
| FastAPI | 0.146.0 | Версия существует (PyPI, release 2026-10-18), но ни один lock её не использует — см. M-1 | **не as-built** |
| Uvicorn | 0.54.0 | `estimate/uv.lock:833-834`, `auth-users:846-847`, `company-org:833-834`, `pdf-extract:1828-1829`, `spec-service:1041-1042`, `llm-proxy:538-539`, `llm-gateway:1089-1090` | OK |
| Pydantic | 2.14.1 | `estimate/uv.lock:492-493` и все остальные lock-файлы | OK |
| SQLAlchemy | 2.1.5 | `estimate/uv.lock:718-719`; в `spec-service` 2.1.6 (`:917-918`) — см. L-3 | OK с дрейфом |
| Alembic | 1.20.0 | `estimate/uv.lock:6-7`; в `spec-service` 1.20.1 (`:10-11`) — см. L-3 | OK с дрейфом |
| psycopg | 3.3.6 | `estimate/uv.lock:423-424` | OK |
| PyJWT | 2.15.1 | `api-gateway/uv.lock:410-411`, `auth-users/uv.lock:626-627` | OK |
| PostgreSQL | 18.6 | Web: postgresql.org, релиз 18.6 от 2026-08-13. Compose: `pgvector/pgvector:0.8.7-pg18` (`deploy/docker-compose.yml:4`) — минор не запинен, но 18.6 существует | OK |
| pgvector | 0.8.7 | `deploy/docker-compose.yml:4` | OK |
| MinIO | RELEASE.2025-10-15T17-29-55Z | Тег существует на `alpine/minio` (Docker Hub), но compose использует `latest-release` — см. L-1 | **не enforced** |
| llama-cpp-python | >=0.3.14 | `llm-gateway/pyproject.toml:9`; lock 0.3.36 (`llm-gateway/uv.lock:488-489`) | OK |
| sentence-transformers | >=3.0 | `rate-catalog/pyproject.toml` (extra `embeddings`) | OK |
| Embedding model | intfloat/multilingual-e5-small (384, CPU) | `rate-catalog/Dockerfile:3-4` (model + revision), `rate-catalog/app/infrastructure/models.py:27` (`EMBEDDING_DIMENSION = 384`), HNSW cosine (`models.py:147-163`) | OK |
| Vue | 3.5 | `ui/package-lock.json` → 3.5.44; `package.json:9` `^3.5.13` | OK |
| Vite | 5.4 | `ui/package-lock.json` → 5.4.21; `package.json:19` `^5.4.11` | OK (as-built, не latest) |
| Node.js | 22 (>=22.20.0; `node:22-alpine`) | `ui/package.json:24`, `ui/Dockerfile:2` | OK |
| pytest / ruff | >=8 / >=0.6 | `estimate/pyproject.toml`, `rate-catalog/pyproject.toml` | OK |
| Docker Compose | v2 | инструмент, registry-пин не применим | OK |

## Findings

### H-1 (high) — «As-built sync» ложно: AD-017/AD-018 и пересмотренная модель снапшотов отсутствуют в коде

**Где:**
- Spine: `ARCHITECTURE-SPINE.md:16` (заявление «As-built sync… синхронизирован с текущим кодом; добавлены AD-017/AD-018»), `:39` (AD-001), `:44` (AD-002), `:116` (AD-016), `:121` (AD-017), `:126` (AD-018), `:214` (estimate_item), `:227-229` (match_feedback, unit_conversion_rule, unit_conversion_audit), `:252-255` (шаги 6–7).
- Код: `estimate/app/infrastructure/models.py:103-141` (нет `qty_catalog`, `unit_catalog`, `conversion_factor`, `conversion_*_snapshot`), `estimate/app/domain/item.py:18-24` (`ItemStatus`), `estimate/app/application/calculate_cost.py:145-149` (`calculate_line_cost(item.qty, rate.rate_amount)`), `estimate/migrations/versions/` (последняя — `0015_clear_manual_confidence.py`), `ui/src/router.ts:35-64` (нет маршрута словаря ЕИ), `ui/src/api/client.ts` (нет conversion-эндпоинтов).
- Самопротиворечие в companion-документе: `docs/architecture-overview.md:3` («As-built sync… добавлен пересчёт ЕИ») против `docs/architecture-overview.md:121` («**Далее:** пересчёт ЕИ и словарь `unit_conversion_rule` … — новый скоуп»).

**Evidence:**
- Grep по `estimate`, `rate-catalog`, `ui` не находит `unit_conversion`, `qty_catalog`, `conversion_factor`, `conversion_required`, `unit_catalog` ни в одном `.py`/`.ts`/`.vue`/миграции. Совпадения есть только в `core/docs/` и `_bmad-output/`.
- `estimate_item` хранит только `unit_orig`, `unit_normalized`, `qty` (`models.py:116-118`); полей пересчёта и снапшотов пересчёта нет.
- `match_feedback` (`models.py:144-185`) не содержит `conversion_factor`/`conversion_source`, которые обещает Data Model `ARCHITECTURE-SPINE.md:227`.
- Расчёт до сих пор использует `item.qty`, а не `qty_catalog_snapshot`, хотя AD-001 (`:39`) и AD-017 (`:121`) фиксируют формулу через `qty_catalog_snapshot`.
- UI не имеет ни редактора словаря ЕИ, ни полей `qty_catalog`/`conversion_factor`, хотя AD-018 (`:126`) и Services (`:139`) их обещают.

**Impact:** читатель и реализатор принимают нереализованное поведение за гарантированное. Сегодня расчёт может молча посчитать стоимость при несовпадающих ЕИ (пересчёта и `conversion_required` нет) — ровно тот отказ, который AD-017 заявлен предотвращать. Это не конфликт с Decimal/Postgres, а ложное заявление о текущем состоянии.

**Fix:** либо пометить AD-017/AD-018 как target/proposed и убрать «as-built sync», либо реализовать до `status: final`: миграция полей/таблиц, `conversion_required` в модели/расчёте, API и UI-редактор, затем синхронизировать `docs/architecture-overview.md:121`.

### H-2 (high) — Глобальная `unit_conversion_rule` в `estimate` противоречит закрытому списку исключений AD-013

**Где:** `ARCHITECTURE-SPINE.md:97-101` (AD-013, особенно `:100-101`), `:126` (AD-018), `:228` (Data Model).

**Evidence:** AD-013 требует `company_id` на доменных данных, фильтрацию всех выборок по `company_id` в репозитории и говорит, что «запрос без `company_id` невозможен». Исключения перечислены явно: `rate-catalog` (регионы, материалы, цены, эмбеддинги) и глобальные каталоги `llm_provider`/`llm_model` в `spec-service`. AD-018 помещает глобальную таблицу без `company_id` в `estimate` и разрешает её менять любому авторизованному пользователю. AD-013 при этом не расширен.

**Impact:** две реализации, формально следующие spine, разойдутся: одна добавит спец-исключение в estimate, другая будет требовать `company_id`. Security-ревью не может отличить разрешённое исключение от нарушения tenancy. Запись в глобальные правила, влияющие на стоимость всех компаний, любым тенантом — кросс-тенантный риск, не покрытый AD.

**Fix:** явно добавить `unit_conversion_rule`/`unit_conversion_audit` в исключения AD-013 и описать authz на запись (кто может менять глобальные правила, аудит/лимиты). Альтернатива — перенести словарь в `rate-catalog` как владельца глобального справочника, но тогда меняется владелец в AD-017.

### M-1 (medium) — FastAPI-пин не as-built и не current

**Где:** `ARCHITECTURE-SPINE.md:154`.

**Evidence:** Web: FastAPI 0.146.0 существует (PyPI, release 2026-10-18), текущая — 0.162.0 (2026-10-23). Фактические lock-файлы: `estimate/uv.lock:110-111` = **0.143.3**; `api-gateway/uv.lock:92-93` = 0.143.3; `auth-users/uv.lock:114-115` = 0.143.3; `company-org/uv.lock:110-111` = 0.143.3; `pdf-extract/uv.lock:416-417` = 0.143.3; `llm-gateway/uv.lock:217-218` = 0.143.3; `spec-service/uv.lock:222-223` = **0.159.1**; `llm-proxy/uv.lock:64-65` = **0.160.0**. Ни один сервис не использует 0.146.0.

**Impact:** таблица Stack утверждает версию, которой нет в системе, и скрывает уже существующий разброс трёх линий FastAPI. «Current»-утверждение устарело на 16 минорных релизов.

**Fix:** указать per-service фактические версии или поддерживаемый диапазон; если нужен 0.146.0 — обновить lock-файлы; если as-built — перечислить 0.143.3 / 0.159.1 / 0.160.0.

### M-2 (medium) — У `conversion_required` нет зафиксированного представления

**Где:** `ARCHITECTURE-SPINE.md:116` (AD-016: `needs_review` — производный флаг, не статус), `:121` (AD-017: «позиция получает `conversion_required`»), `:214` (Data Model: только `status`); код: `estimate/app/domain/item.py:18-24` (`ItemStatus`), `estimate/app/application/calculate_cost.py:34-41` (`UncoveredReason`), `estimate/app/transport/routes.py:267` и `estimate/app/application/list_items.py:68` (`needs_review` только из `confidence`).

**Evidence:** AD-016 говорит, что `needs_review` = порог 0.70 **или** `conversion_required`, но ни поле, ни enum не несут `conversion_required`; AD-017 говорит, что позиция его «получает»; в Data Model колонки нет. Текущий код считает `needs_review` только по `confidence` (`item.py:97`). В `ItemStatus` нет значения `conversion_required`; в `UncoveredReason` — тоже.

**Impact:** реализаторы могут сделать это статусом, булевым полем или вычисляемым предикатом; API-фильтры/счётчики и исключение из расчёта разойдутся. Существующий `ItemStatus` его выразить не может.

**Fix:** определить `conversion_required` как persisted reason/enum **или** как производный предикат с точными входами; описать формулу `needs_review` после match (`confidence` + `conversion_required`); добавить причину в `UncoveredReason` и схему ответа.

### M-3 (medium) — Модель правила AD-018 не выражает собственную валидацию (алиасы/циклы)

**Где:** `ARCHITECTURE-SPINE.md:126` (AD-018: «алиасы ЕИ», «отсутствие противоречивых алиасов и циклов»), `:228` (`unit_conversion_rule` с одной колонкой `alias`), `:229`.

**Evidence:** в Data Model у правила одна колонка `alias`; для нескольких алиасов нужен child-table или массив, которого нет. «Циклы» подразумевают граф единиц (alias → canonical → base), но схема плоская: `alias + factor_to_base + multiple + priority`. `dimension` включает `package`, хотя упаковка — не физическое измерение с базовой единицей; смешаны `dimension` и `kind=package`.

**Impact:** требование «нет противоречивых алиасов и циклов» не проверяемо и не реализуемо в заявленной схеме; две реализации нормализуют по-разному. Это внутренний контрактный пробел, не web-факт.

**Fix:** добавить `unit_alias(alias PK, rule_id)` (или массив), определить рёбра графа и проверку циклов; разделить `dimension` и `kind=package`; зафиксировать каноническую базовую единицу на измерение.

### L-1 (low) — MinIO-пин реален, но deploy его не соблюдает

**Где:** `ARCHITECTURE-SPINE.md:162`; `deploy/docker-compose.yml:135-138`.

**Evidence:** Web: `alpine/minio:RELEASE.2025-10-15T17-29-55Z` существует (digest `f93b...`), upstream `minio/minio` перешёл на source-only (issue #21647). Compose использует `alpine/minio:latest-release` (`:138`), а не заявленный тег; сейчас digest совпадает, но тег mutable. Зеркало — стороннее.

**Impact:** воспроизводимость, обещанная spine, не enforced; обновление зеркала молча меняет runtime.

**Fix:** запинить тег/digest в compose либо зафиксировать в spine `latest-release` как осознанное решение и принять дрейф.

### L-2 (low) — Утверждение AD-011 «417 строк, flat-модуль» устарело

**Где:** `ARCHITECTURE-SPINE.md:89`; `llm-gateway/app/*.py`.

**Evidence:** `wc -l llm-gateway/app/*.py` = **1426** строк в 10 модулях (`main.py` 107, `openai_api.py` 335, `model_artifacts.py` 362 и т.д.), а не 417 строк одного flat-модуля.

**Impact:** обоснование исключения опирается на неверный факт. Само исключение остаётся валидным, но «as-built» точность страдает.

**Fix:** обновить размер/структуру или переформулировать: «legacy package без слоёв Clean Architecture».

### L-3 (low) — Мелкий version drift скрыт single-version таблицей

**Где:** `ARCHITECTURE-SPINE.md:157-158`; `estimate/uv.lock:718-719` (SQLAlchemy 2.1.5), `:6-7` (Alembic 1.20.0); `spec-service/uv.lock:917-918` (SQLAlchemy 2.1.6), `:10-11` (Alembic 1.20.1).

**Impact:** читатель считает версию единой; `spec-service` на patch впереди. Не поломка, но таблица не Literally true для poly-repo.

**Fix:** писать «SQLAlchemy 2.1.x (estimate 2.1.5, spec-service 2.1.6)», «Alembic 1.20.x», либо выровнять lock-файлы.

## AD-017/AD-018: совместимость со стеком и кодом (явно)

- **Decimal — совместимо.** `estimate/app/domain/cost.py:1-15` уже использует `Decimal`, `ROUND_HALF_UP`; модели — `Numeric(18,2)`/`Numeric(18,4)` (`estimate/app/infrastructure/models.py:128-132`). `qty_catalog Numeric(18,4)` совпадает с текущей точностью `qty` (`models.py:118`). Противоречия с политикой «деньги — Decimal, не float» нет.
- **`Numeric(24,12)` — совместимо с Postgres 18.6/SQLAlchemy 2.1**, но миграции нет (H-1). Ограничений БД или кода, запрещающих такую точность, не найдено. Риск переполнения `cost Numeric(18,2)` уже обрабатывается `ensure_money_range` (`cost.py:39-52`).
- **`material.unit` как источник — совместимо.** `rate-catalog` хранит `unit: String(100)` (`rate-catalog/app/infrastructure/models.py:57`); реальные значения — `м3`, `шт.`, `шт` (`rate-catalog/tests/test_rates_for_period.py:38-47`, `test_get_rate.py:53`). Детерминированный словарь/размерный разбор реализуем, но нормализация/backfill существующих строк не описаны — это задача реализации, не конфликт.
- **Владение — совместимо с AD-014**, но глобальный scope конфликтует с AD-013 (H-2).
- **Расчёт — требует согласованного изменения.** `calculate_cost.py:145-149` использует `item.qty`, а AD-001/AD-017 — `qty_catalog_snapshot`. Детерминизм/Decimal не нарушаются, но текущее поведение не равно зафиксированной формуле (H-1).
- **UI — target gap.** AD-017/AD-018 bind `ui`, но в `ui/src/router.ts:35-64` нет маршрута словаря, в `ui/src/api/client.ts` — conversion-методов (H-1).
- **Snapshot — target gap.** AD-002 добавляет `conversion_factor_snapshot`, в модели `estimate_item` его нет (H-1).

## Considered and not flagged

- **PostgreSQL 18.6** — реальный релиз (2026-08-13); образ `pgvector/pgvector:0.8.7-pg18` подходит, минор не запинен, для локального compose приемлемо.
- **pgvector 0.8.7 + HNSW cosine + 384** — совпадает с `rate-catalog/app/infrastructure/models.py:147-163`, миграцией `0002_global_materials_and_embeddings.py` и Dockerfile-пином модели; AD-016 подтверждён.
- **JWT HS256/168ч/заголовки** — совпадает с `api-gateway/app/infrastructure/jwt.py:13`, `auth-users/app/infrastructure/tokens.py:10-11`, `api-gateway/app/infrastructure/proxy.py:54-59`.
- **Vite 5.4 / Node 22 / Vue 3.5** — соответствуют `ui/package-lock.json` и Dockerfile; старее 2026-latest, но это осознанный as-built.
- **llama-cpp-python / sentence-transformers / E5-small** — соответствуют pyproject/Dockerfile; модель реальна и запинена по revision.
- **MinIO removal from Docker Hub** — комментарий `deploy/docker-compose.yml:135-136` соответствует upstream issue #21647.

## Counts

- High: **2**
- Medium: **3**
- Low: **3**
- Total: **8**
