---
title: 'Story 6.1 — Создание сметы и версий (ровно одна активная)'
type: 'feature'
created: '2026-10-20'
status: 'done'
route: 'dispatch'
baseline_commit: 'f1875e7b7cd1409fc0fb47ef2e61b0332aa44939'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-6-context.md'
  - '_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** `estimate` — пустой каркас: нет смет, версий и позиций. Без них нельзя закрыть проверку маппинга (5.3), diff (6.2) и детерминированный расчёт (6.3).

**Approach:** Создать в `estimate` персистентную модель сметы, версий и позиций и API: создать смету на существующем этапе (сразу первая активная версия), создать следующую пустую версию и активировать её атомарно. Версия хранит позиции; snapshot-поля расценок присутствуют, но заполняются при расчёте (6.3).

## Boundaries & Constraints

**Always:** `estimate` владеет сметами/версиями/позициями (AD-013); `company_id` из `X-Company-Id`, все выборки фильтруются в репозитории; `contract_id`+`stage_id` проверяются через `GET /v1/contracts/{contract_id}/stages` (AD-014), без FK между БД; у сметы ровно одна `is_active=true` (гарантия БД + транзакция активации); `estimate_item` хранит тип, наименование, исходную/нормализованную ЕИ, количество, раздел, `rate_catalog_id`, `confidence`, статус и nullable snapshot-поля; ошибки — `application/problem+json`; UUID, ISO-8601 UTC, `snake_case`, `/v1/...`; слои AD-011; миграции Alembic и отдельная БД `estimate`.

**Решения (2026-10-20):** этап проверяется через список этапов договора company-org (без нового endpoint и без изменений company-org); новая версия создаётся пустой — позиции появятся из нового PDF/импорта Story 5.3.

**Never:** UI и diff-API (6.2, 1.1); расчёт и заполнение снапшотов (6.3); API создания/редактирования позиций и ручная проверка (5.3); LLM; импорт Excel; изменение rate-catalog; прямое чтение чужих БД; очередь; несколько активных версий; удаление истории версий.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Создание сметы | существующие `contract_id` и `stage_id` компании | `201`, смета + версия 1 `is_active=true` | — |
| Чужой/несуществующий этап | `stage_id` не в списке этапов договора | — | `404` |
| Чужой/несуществующий договор | `contract_id` вне компании | — | `404` |
| company-org недоступен | сеть/5xx | — | `502` |
| Новая версия | смета с активной версией | `201`, `version_no+1`, `is_active=false`, позиций нет | — |
| Активация | неактивная версия | `200`, она `is_active=true`, прежняя `false` | — |
| Повторная активация | уже активная версия | `200`, состояние не меняется | — |
| Чужая смета/версия | другой `company_id` | — | `404` |
| Неизвестная версия | `version_id` не принадлежит смете | — | `404` |
| Нет identity | без `X-Company-Id` | — | `401` |
| Невалидное тело | пустое имя, не-UUID | — | `422` |

</frozen-after-approval>

## Code Map

- `estimate/pyproject.toml` -- добавить `sqlalchemy`, `alembic`, `psycopg[binary]`, `httpx`.
- `estimate/alembic.ini`, `estimate/migrations/{env.py,script.py.mako,versions/0001_create_estimate.py}` -- схема по образцу `rate-catalog`; `POSTGRES_DSN`, default `.../estimate`.
- `estimate/app/domain/{estimate,item,errors,ports}.py` -- `Estimate` (`company_id`, `contract_id`, `stage_id`), `EstimateVersion`, `EstimateItem`, статусы, ошибки, порты репозитория и company-org.
- `estimate/app/application/{create_estimate,create_version,activate_version,get_estimate}.py` -- use cases; проверка этапа по списку этапов договора, нумерация версий, атомарная активация.
- `estimate/app/infrastructure/{db,models,repository,company_org_client}.py` -- сессии, ORM, tenancy-JOIN, `GET /v1/contracts/{contract_id}/stages` с `X-Company-Id`.
- `estimate/app/transport/{schemas,routes,problem}.py` -- `POST /v1/estimates`, `POST /v1/estimates/{id}/versions`, `POST .../versions/{vid}/activate`, `GET /v1/estimates/{id}`; problem+json.
- `estimate/app/{wiring,main}.py` -- composition root, роутер, lifespan httpx-клиента.
- `estimate/tests/{test_create_estimate,test_versions,test_tenancy,test_migrations,test_company_org_client,test_main_composition}.py` -- матрица I/O, одна активная версия, миграции, композиция.
- `estimate/{Dockerfile,README.md}` -- `alembic upgrade head`, контракт, env.
- `api-gateway/app/application/routing.py`, `wiring.py`, `tests/test_gateway.py` -- префикс `/v1/estimates` → estimate, `ESTIMATE_URL`.
- `deploy/docker-compose.yml`, `initdb/01-create-databases.sql`, `.env.example`, `README.md` -- сервис `estimate`, база `estimate`, DSN, depends_on postgres/company-org.
- Не трогать: `company-org/*`, `mapping-service/*`, `rate-catalog/*`, `llm-gateway/*`, `pdf-extract/*`, `calc-service/*`, `ui/*`.

## Tasks & Acceptance

**Execution:**
- [x] `estimate/app/domain/*`, `application/*` -- модель сметы/версии/позиции и use cases -- контракт версионирования.
- [x] `estimate/app/infrastructure/*` -- ORM, репозиторий, клиент company-org -- tenancy и проверка этапа по договору.
- [x] `estimate/migrations/*`, `pyproject.toml` -- Alembic-схема с частичным unique-индексом активной версии -- инвариант одной active.
- [x] `estimate/app/transport/*`, `wiring.py`, `main.py` -- HTTP-контракт -- 201/200/401/404/422/502.
- [x] `estimate/tests/*` -- матрица I/O, активация, миграции, композиция -- регрессия.
- [x] `estimate/{Dockerfile,README.md}` -- запуск и документация -- сервис поднимается.
- [x] `api-gateway/*` -- `/v1/estimates` -- edge.
- [x] `deploy/*` -- compose/initdb/env/README -- сметы доступны в стеке.
- [x] `estimate` + `api-gateway` -- ruff/pytest; `deploy` -- compose config -- верификация.

**Acceptance Criteria:**
- Given существующий этап договора компании, when `POST /v1/estimates` с `X-Company-Id`, then `201`, созданы смета и версия 1 с `is_active=true`.
- Given неизвестный/чужой договор или этап, when создание, then `404`; company-org недоступен → `502`; без `X-Company-Id` → `401`.
- Given смета, when `POST /v1/estimates/{id}/versions`, then `201`, новая пустая версия `is_active=false`.
- Given две версии, when активация одной, then `200`, ровно одна `is_active=true`, прежняя деактивирована в той же транзакции; повторная активация идемпотентна.
- Given чужая смета/версия, when запрос, then `404`; `GET /v1/estimates/{id}` отдаёт смету и все версии по возрастанию `version_no`.

## Implementation Notes

## Spec Change Log

## Review Triage Log

- **2026-10-20 · blind-hunter · tenancy держится на соглашении: repo-методы версий/позиций не принимают `company_id`** — verdict: medium. evidence: `get_version`/`list_versions`/`next_version_number`/`activate_version` фильтруют только по `estimate_id`, `add_item` не проверяет компанию; сейчас use cases проверяют смету, но 5.3 получит небезопасный `add_item`. route: patch → company-scope в порт/репозиторий, обновить callers и тесты.
- **2026-10-20 · blind-hunter · гонка `max(version_no)+1` и отсутствие идемпотентности** — verdict: low. evidence: два параллельных POST могут получить один номер и `IntegrityError`; сценарий требует одновременных запросов к одной смете, а продукт — 20–50 смет/мес с одним сметчиком. route: reject (fix требует блокировок/ретраев, не покрыт AC).
- **2026-10-20 · blind-hunter · `X-User-Id` не ограничен по длине → 500 на Postgres** — verdict: false. evidence: заголовок ставит только api-gateway из `sub` JWT, а auth-users выпускает `sub` как UUID; внешний клиент не может передать произвольный `X-User-Id` (AD-010, внутренняя сеть — граница доверия).
- **2026-10-20 · blind-hunter · `created_by` не возвращается в API** — verdict: low. evidence: README обещает сохранение, а не выдачу; поле сохраняется и доступно в БД, AC этого не требует. route: reject.
- **2026-10-20 · blind-hunter · RFC 7807 не покрывает 404/405/500 и `HTTPException`** — verdict: low. evidence: доменные ошибки и 422 отдают problem+json; неизвестный маршрут/405/500 уходят в дефолт Starlette, но через edge неизвестные пути отсекает api-gateway, а fix требует catch-all-обработчиков. route: reject.
- **2026-10-20 · blind-hunter · миграции в `CMD` контейнера при нескольких репликах** — verdict: low. evidence: локальный MVP — одна реплика, k8s отложен; тот же паттерн у rate-catalog/pdf-extract. route: reject.
- **2026-10-20 · blind-hunter · дефолтный dev-DSN вместо fail-fast** — verdict: false. evidence: README явно документирует default, а не «обязателен»; паттерн совпадает с остальными сервисами.
- **2026-10-20 · blind-hunter · образ не использует `uv.lock`** — verdict: low. evidence: `pip install .` по диапазонам — pre-existing инфра-паттерн, уже отмечен в deferred-work по Story 4.1. route: defer.
- **2026-10-20 · blind-hunter · нет CHECK-ограничений в БД** — verdict: low. evidence: значения `material_work`/`status` пишутся только через доменные enum'ы; raw SQL вне контракта, требование CHECK в спеке отсутствует. route: reject.
- **2026-10-20 · blind-hunter + verification-gap · Postgres-ветка миграций, `downgrade` и FK не тестируются** — verdict: medium. evidence: тесты идут на SQLite; `postgresql_where`, `Uuid`, `timestamptz` и CASCADE не исполняются, downgrade не вызывается. route: defer → Postgres/CI — pre-existing инфра-задача (см. ретро Epic 2/4).
- **2026-10-20 · blind-hunter · транзакционность не проверяется fault-injection** — verdict: low. evidence: счастливый путь и инвариант одной active покрыты; тест на сбой между UPDATE потребовал бы инъекции в репозиторий. route: reject.
- **2026-10-20 · blind-hunter · тест пустой версии не наполняет исходную версию** — verdict: medium. evidence: `test_new_version_is_empty_inactive_and_numbered` проверяет глобальный `count(*)`, который пуст и при копировании. route: patch → наполнить версию 1, проверить пустоту версии 2 и сохранность позиции в версии 1.
- **2026-10-20 · blind-hunter · порядок позиций не задан (`created_at, id`)** — verdict: low. evidence: API позиций в 6.1 нет; порядок импорта определит Story 5.3. route: reject.
- **2026-10-20 · blind-hunter · snapshot-поля нельзя заполнить через `EstimateItem.create`, `updated_at` не меняется** — verdict: low. evidence: snapshot-поля заполняет 6.3, review-обновления — 5.3; доменная модель допускает прямое создание с этими полями. route: reject.
- **2026-10-20 · blind-hunter · нет логирования и correlation id** — verdict: low. evidence: не входит в AC; логирование отсутствует во всех сервисах, это общая задача. route: reject.
- **2026-10-20 · blind-hunter · company-org client маппит 4xx в 502 и не делает retry** — verdict: low. evidence: клиент всегда отправляет `X-Company-Id`; неожиданный 401/403 — конфигурационная ошибка upstream, 502 допустим; retry/backoff не требуется MVP. route: reject.
- **2026-10-20 · blind-hunter · README: «чужая/неизвестная смета или версия» для GET без `version_id`** — verdict: low. evidence: фраза неверно описывает маршрут. route: patch → исправить формулировку.
- **2026-10-20 · blind-hunter · README не даёт локального запуска и противоречит себе по DSN** — verdict: low. evidence: `cd estimate` излишен, но команды запуска и default DSN описаны; поведение не затронуто. route: reject.
- **2026-10-20 · blind-hunter · «позиции» в названии, но API позиций нет** — verdict: low. evidence: 6.1 создаёт схему хранения; API импорта/проверки — 5.3, это зафиксировано в спеке. route: reject.
- **2026-10-20 · blind-hunter · diff смешивает три репозитория с одинаковыми путями** — verdict: false. evidence: это артефакт генерации review-diff, а не изменение продукта; в каждом репозитории пути корректны.
- **2026-10-20 · blind-hunter · `/health` статический, без readiness** — verdict: low. evidence: readiness/БД-пинг не входит в AC; `/health` — общий контракт всех сервисов. route: reject.
- **2026-10-20 · edge-case-hunter · отсутствие identity при невалидном теле даёт 422, а не 401** — verdict: low. evidence: FastAPI валидирует тело до endpoint; внешний путь всегда проходит JWT-gateway, который ставит заголовок; fix требует middleware. route: reject (тот же паттерн отклонён в 5.1/5.2).
- **2026-10-20 · edge-case-hunter · неизвестный маршрут/405/500 не в problem+json** — verdict: low. evidence: см. строку blind-hunter про RFC 7807; через edge неизвестные префиксы отсекает api-gateway. route: reject.
- **2026-10-20 · edge-case-hunter + verification-gap · ambient `POSTGRES_DSN` переопределяет tmp-DSN в тестах миграций** — verdict: medium. evidence: `migrations/env.py` читает env поверх `sqlalchemy.url`, а `test_migrations._config` и `test_main_composition` не изолируют env до `upgrade`; при заданном DSN тесты могут мигрировать чужую БД. route: patch → `monkeypatch.setenv` до `command.upgrade`/`migrate`.
- **2026-10-20 · edge-case-hunter · гонка выдачи номера версии** — verdict: low. evidence: см. строку blind-hunter про гонку; не покрыто AC и маловероятно при одном сметчике. route: reject.
- **2026-10-20 · edge-case-hunter · длинный `X-User-Id` → 500** — verdict: false. evidence: см. строку blind-hunter про `X-User-Id`; gateway ставит UUID `sub`.
- **2026-10-20 · edge-case-hunter · repo-методы версий/позиций без `company_id`** — verdict: medium. evidence: то же, что у blind-hunter; текущие use cases проверяют смету, но порт провоцирует будущий cross-tenant вызов из 5.3. route: patch (общий с blind-hunter).
- **2026-10-20 · verification-gap · тест новой версии не проверяет сохранение прежней active** — verdict: medium. evidence: `_active_count` проверяется только после явной активации; регрессия «создание версии снимает active» прошла бы suite. route: patch → после POST versions утверждать `_active_count == 1` и active = версия 1.
- **2026-10-20 · verification-gap · тест пустой версии не наполняет исходную** — verdict: medium. evidence: то же, что у blind-hunter. route: patch (общий).
- **2026-10-20 · verification-gap · nullable-поля позиции не проверяются round-trip** — verdict: medium. evidence: `test_items_are_scoped...` задаёт только обязательные поля; потеря `section`/snapshot/`confidence` в маппинге пройдёт незамеченной. route: patch → заполнить все nullable-поля и проверить после `list_items`.
- **2026-10-20 · verification-gap · Postgres-ветка частичного индекса не исполняется** — verdict: medium. evidence: SQLite-only CI; удаление `postgresql_where` не ломает тесты. route: defer → Postgres в CI (pre-existing инфра-задача).
- **2026-10-20 · verification-gap · entrypoint контейнера и COPY миграций не исполняются** — verdict: medium. evidence: CI собирает/тестирует Python, но не запускает образ; сломанный `CMD alembic` не будет замечен. route: defer → container-smoke в CI (общая инфра-задача деплоя).
- **2026-10-20 · verification-gap · ambient `POSTGRES_DSN` (other finding)** — verdict: medium. evidence: тот же дефект изоляции тестов, что у edge-case-hunter. route: patch (общий).

## Design Notes

- `POST /v1/estimates`: `{contract_id, stage_id, name}` → `201 {id, contract_id, stage_id, name, created_at, active_version}`; `created_by` — из `X-User-Id`. Версия 1 создаётся в той же транзакции. До записи вызывается `GET /v1/contracts/{contract_id}/stages`; `stage_id` должен быть в списке, иначе `404`; сеть/5xx — `502`. Cross-service FK нет; `contract_id` хранится для провенанса.
- `POST /v1/estimates/{estimate_id}/versions`: без тела → `201` сводка пустой версии; `version_no = max(version_no)+1` в транзакции (первая версия — 1), уникальность `(estimate_id, version_no)`. Позиции не копируются (решение 2026-10-20): версия наполняется импортом Story 5.3.
- `POST /v1/estimates/{estimate_id}/versions/{version_id}/activate`: в одной транзакции снять `is_active` у текущей и выставить у выбранной; повторная активация — no-op `200`. Частичный уникальный индекс `(estimate_id) WHERE is_active` страхует инвариант; в Alembic задать `sqlite_where` и `postgresql_where`, иначе тест миграций на SQLite не поймает регрессию.
- `GET /v1/estimates/{estimate_id}`: `200 {id, contract_id, stage_id, name, created_at, versions:[{id, version_no, is_active, calculated_at, created_at}]}` по возрастанию `version_no`; чужая/неизвестная — `404`, чтобы не раскрывать существование.
- `estimate_item`: `row_ref`, `material_work`, `name`, `unit_orig`, `unit_normalized`, `qty Numeric(18,4)`, `section`, `rate_catalog_id`, `rate_snapshot_id`, `unit_snapshot`, `rate_amount_snapshot`, `confidence Numeric(5,4)`, `status` (`proposed|confirmed|corrected`), timestamps. Snapshot-поля nullable и не заполняются в 6.1 (6.3); review-поля готовы для 5.3.
- Деньги/количества — Decimal в домене и строки в API; никаких float.

## Verification

**Commands (выполнено 2026-10-20):**
- `cd estimate && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `42 passed`
- `cd api-gateway && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `48 passed`
- `cd deploy && docker compose config -q` → exit 0
- `cd estimate && docker build -q -t estimate:ci .` → образ собран (`sha256:2093e76…`)

**Не покрыто автоматически:** живой PostgreSQL и полный compose-стек не поднимались; миграции проверены на SQLite, образ собран.

**Walkthrough (2026-10-21):** approved — модель сметы/версий, инвариант одной активной и проверка этапа через company-org приняты; статус истории `done`.
