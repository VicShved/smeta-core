---
title: 'Версионирование справочника: история версий и активная версия'
type: 'feature'
created: '2026-10-20'
status: 'done'
route: 'dispatch'
baseline_commit: '6d604f7be0977e3f8b4e8949529d28ede8aeec9a'
review_loop_iteration: 0
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** После импорта в справочнике уже есть версии, но API не отдаёт историю и не сообщает, какая версия активна; UI и потребители (маппинг, расчёт) не могут понять, на какой версии основана смета.

**Approach:** Read/activation API в rate-catalog поверх существующих `rate_catalog` и `rate_catalog_version` без изменения схемы: список версий, текущая активная версия и явное переключение активной версии. Активация переиспользует `current_version_id` и скоупится компанией.

## Boundaries & Constraints

**Always:** identity только из `X-Company-Id` (401 без него), все выборки и запись — в рамках компании (AD-013); список версий — новые первыми, `is_active` выводится из `current_version_id`; неизвестная и чужая версия дают одинаковый 404; изменяется только `current_version_id`, строки расценок и содержимое версий не мутируются; новых миграций нет; ошибки — `application/problem+json` (паттерн company-org); слоистость AD-011 (application не импортирует SQLAlchemy/FastAPI).

**Never:** UI; редактирование/удаление версий и строк; пагинация и фильтры (версий мало); snapshot-логика и расчёт смет (Story 3.3); изменение контракта импорта; правки gateway/compose (префикс `/v1/rate-catalog` уже маршрутизируется).

**Решения (2026-10-20):** `GET /v1/rate-catalog/versions` → `[RateVersionResponse]` (пустой справочник — `200 []`); `GET /v1/rate-catalog/versions/current` → объект, `404` если активной нет; `PUT /v1/rate-catalog/versions/current` с `{"version_id": ...}` → объект, идемпотентно. Переключать может любой аутентифицированный участник компании (ролевой модели в системе нет).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Список версий | две успешные версии, активна первая | `200`, порядок `2,1`, `is_active` только у первой | — |
| Пустой справочник | импортов не было | `200 []` | — |
| Текущая версия | активная задана | `200`, `is_active=true` | — |
| Активной нет | справочника нет или `current_version_id=null` | — | `404` problem+json |
| Активация | `version_id` своей версии | `200`, `current_version_id` изменён | — |
| Повторная активация | версия уже активна | `200`, состояние не меняется | — |
| Чужая/неизвестная версия | `version_id` другой компании | — | `404`, тело неотличимо от неизвестной |
| Нет identity | без `X-Company-Id` | — | `401` |

</frozen-after-approval>

## Code Map

- `rate-catalog/app/domain/rate.py` -- `RateCatalog.current_version_id`, `RateVersion`; добавить read-модель `RateVersionSummary` (`id`, `version`, `imported_at`, `source_file_hash`, `created_by`, `is_active`).
- `rate-catalog/app/domain/errors.py` -- базовый `RateCatalogError`; добавить `CatalogNotFound` и `RateVersionNotFound`.
- `rate-catalog/app/domain/ports.py` -- `RateCatalogRepository`; добавить `list_versions(rate_catalog_id)` и `get_version(rate_catalog_id, version_id)`.
- `rate-catalog/app/infrastructure/repository.py` -- `_version_to_row`, `get_catalog`, `set_current_version`; добавить обратное преобразование и выборки (сортировка `version.desc()`).
- `rate-catalog/app/application/catalog_versions.py` -- новый модуль: `ListVersions`, `GetCurrentVersion`, `ActivateVersion`.
- `rate-catalog/app/transport/schemas.py` -- `ImportResponse`; добавить `RateVersionResponse` и `ActivateVersionRequest`.
- `rate-catalog/app/transport/routes.py` -- `create_router`, `_require_company_id`; добавить три эндпоинта, `catalog_versions`-параметр с дефолтом из wiring.
- `rate-catalog/app/transport/problem.py` -- обработчики `InvalidWorkbook`/`InvalidMapping`; добавить 404 для новых ошибок.
- `rate-catalog/app/wiring.py` -- `make_import_rates_runner`; добавить `make_catalog_versions(session_factory=None)`, одна сессия на вызов, commit только в `activate`.
- `rate-catalog/tests/test_catalog_versions.py` -- новый файл; sqlite + Alembic, импорт через существующий use case.
- `rate-catalog/tests/test_import.py` -- не менять; `create_router()` остаётся совместимым (новые параметры опциональны).
- `api-gateway/app/application/routing.py`, `deploy/docker-compose.yml` -- не трогать.

## Tasks & Acceptance

**Execution:**
- [x] `rate-catalog/app/domain/rate.py` -- добавить `RateVersionSummary` -- единая read-модель для transport.
- [x] `rate-catalog/app/domain/errors.py` -- добавить `CatalogNotFound`, `RateVersionNotFound` -- 404 без утечки чужого скоупа.
- [x] `rate-catalog/app/domain/ports.py` -- расширить протокол репозитория -- application не знает SQLAlchemy.
- [x] `rate-catalog/app/application/catalog_versions.py` -- три use case -- бизнес-логика списка/текущей/активации.
- [x] `rate-catalog/app/infrastructure/repository.py` -- реализовать `list_versions`/`get_version` с фильтром по `rate_catalog_id` -- изоляция компаний.
- [x] `rate-catalog/app/transport/schemas.py` -- `RateVersionResponse`, `ActivateVersionRequest` -- контракт API.
- [x] `rate-catalog/app/transport/problem.py` -- 404-обработчики -- problem+json.
- [x] `rate-catalog/app/transport/routes.py` -- `GET /versions`, `GET /versions/current`, `PUT /versions/current` -- доступ к истории и переключение.
- [x] `rate-catalog/app/wiring.py` -- `make_catalog_versions` -- одна сессия на запрос, commit при активации.
- [x] `rate-catalog/tests/test_catalog_versions.py` -- покрыть I/O-матрицу и изоляцию компаний -- регрессия.
- [x] `rate-catalog` + `api-gateway` -- ruff/pytest, `deploy` -- `docker compose config -q` -- верификация.

**Acceptance Criteria:**
- Given две успешно импортированные версии, when `GET /v1/rate-catalog/versions`, then `200` со списком новых первыми и `is_active` у активной.
- Given активная версия существует, when `GET /v1/rate-catalog/versions/current`, then `200` с её полями.
- Given версия 2 существует, when `PUT /v1/rate-catalog/versions/current` с её `id`, then `200`, и последующий `GET current` возвращает версию 2.
- Given версия другой компании, when активация, then `404` с телом, неотличимым от неизвестной версии.
- Given выбранная версия 2 активна, when импортируется версия 3, then активной остаётся версия 2 (поведение импорта не меняется).

## Implementation Notes

## Spec Change Log

## Review Triage Log

- **2026-10-20 · blind-hunter · identity проверяется после валидации тела: `PUT` без `X-Company-Id` и с невалидным телом даёт 422, а не 401** — verdict: low. evidence: FastAPI валидирует body до вызова `_require_company_id` (`app/transport/routes.py`), но для запроса из матрицы (валидное тело) 401 соблюдён, утечки данных нет. route: reject (в обычном использовании не встречается; фикс — guard/dependency, больше прямой правки).
- **2026-10-20 · blind-hunter · существующий справочник без активной версии отдаёт detail «Rate catalog not found»; на одном endpoint два разных 404 (при `current_version_id=null` — `CatalogNotFound`, при висячем указателе — `RateVersionNotFound`)** — verdict: low. evidence: `GetCurrentVersion` бросает `CatalogNotFound` при `current_version_id=null`, статус 404 верный; detail контрактом не зафиксирован, висячий указатель возможен только прямой записью в БД. route: reject (косметика detail; различение состояний — новая ошибка/ветка, больше прямой правки).
- **2026-10-20 · blind-hunter · активация не фиксирует актора и не оставляет аудита** — verdict: low. evidence: спека задаёт только «любой аутентифицированный участник компании»; аудит-лог в интенте не заявлен. route: reject (фикс = добавлять аудит в контракт, т.е. править спеку).
- **2026-10-20 · blind-hunter · нет защиты от конкурентной активации (read-then-write, last-write-wins)** — verdict: low. evidence: два параллельных `PUT` с разными версиями оба вернут 200; одновременная активация одной компании в single-instance MVP не воспроизводится. route: reject (фикс — блокировка/`If-Match`/`409`, больше прямой правки).
- **2026-10-20 · blind-hunter + edge-case-hunter · `current`/`list` читают `current_version_id` и версии двумя SELECT-ами: при конкурентной активации возможен устаревший `is_active`** — verdict: low. evidence: `ListVersions`/`GetCurrentVersion` (`app/application/catalog_versions.py`) делают `get_catalog` и выборку версий раздельно; окно между запросами существует, но требует одновременной активации. route: reject (фикс — JOIN/снимок/уровень изоляции, больше прямой правки).
- **2026-10-20 · blind-hunter · нет теста изоляции чтения списка/текущей версии между компаниями (AD-013)** — verdict: low. evidence: `get_version`/`list_versions` скоупятся по `rate_catalog_id` и структурно изолированы, но двухкомпанейный сценарий чтения не проверяется. route: patch → добавить тест.
- **2026-10-20 · blind-hunter + verification-gap · ветка активации для компании без справочника не исполняется ни одним тестом; подмена `RateVersionNotFound`→`CatalogNotFound` не ломает набор (46 тестов)** — verdict: medium. evidence: проверено мутацией в одноразовой копии — регрессия раскрывала бы существование справочника различием тел 404, тесты остаются зелёными; ветка `catalog is None` (`app/application/catalog_versions.py`) не покрыта. route: patch → тест «компания без импортов, `PUT` случайного `version_id` → 404, тело совпадает с 404 неизвестной версии».
- **2026-10-20 · blind-hunter · `ActivateVersionRequest` принимает лишние поля; нет тестов на пустое тело, невалидный UUID, приоритет 401** — verdict: low. evidence: Pydantic по умолчанию игнорирует extras — стандартное поведение, контракт `extra="forbid"` не требует; невалидный UUID штатно даёт 422. route: reject (не охраняет состояние и не создаёт именованного вреда).
- **2026-10-20 · blind-hunter + verification-gap · в ответах истории не сверяются `source_file_hash` и `imported_at`; подмена на `""`/`datetime.now()` не ломает тесты** — verdict: low. evidence: проверено мутацией — набор остаётся зелёным, контракт полей не защищён. route: patch → точные ассерты (хэш sha256 исходника, дата из строки БД).
- **2026-10-20 · blind-hunter · разделы `Implementation Notes`/`Spec Change Log`/`Review Triage Log` пусты, статус не синхронизирован** — verdict: low. evidence: на момент снятия диффа статус был `in-progress`, triage-лог пуст. route: reject (фикс = правка этой же спеки; воркфлоу заполняет разделы в шагах review/present).
- **2026-10-20 · blind-hunter · README rate-catalog не описывает новые эндпоинты** — verdict: low. evidence: контракт доступен через OpenAPI/`response_model`; требование документации в интенте отсутствует. route: reject (фикс — новый раздел, больше прямой правки).
- **2026-10-20 · blind-hunter · устаревший docstring порта: «все операции — в транзакции импорта»** — verdict: low. evidence: `list_versions`/`get_version` используются read-only use case, формулировка `app/domain/ports.py:10` вводит в заблуждение. route: patch → прямая правка строки.
- **2026-10-20 · verification-gap · новые эндпоинты не проверены через production-композицию `create_router()` без аргументов; удаление default-ветки `make_catalog_versions()` не ломает набор (46 тестов)** — verdict: medium. evidence: проверено мутацией — все тесты версий инжектят `catalog_versions` явно, а `app/main.py` использует `create_router()`; при регрессии три эндпоинта в production отвечали бы 500, зелёный CI этого не покажет. route: patch → расширить `test_production_main_composition` вызовом `GET /v1/rate-catalog/versions` после импорта.

## Verification

**Commands (выполнено 2026-10-20, после патчей ревью):**
- `cd rate-catalog && .venv/bin/ruff check .` → `All checks passed!`
- `cd rate-catalog && .venv/bin/pytest -q` → `49 passed` (импорт, версии, I/O-матрица, изоляция компаний, production composition, миграции)
- `cd api-gateway && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `32 passed`
- `cd deploy && docker compose config -q` → OK

**Не покрыто автоматически:** поведение против реального PostgreSQL (тесты — sqlite); конкурентная активация и консистентность чтения при гонке не автоматизированы (отклонены как low в triage-логе); аудит активаций не входит в интент.
