---
title: 'Story 5.2 — Маппинг со справочником и нормализация ЕИ с confidence'
type: 'feature'
created: '2026-10-20'
status: 'done'
route: 'dispatch'
baseline_commit: '8d611b0f63d37967fb7bc5ecb3bf55499738aeff'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-5-context.md'
  - '_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 5.1 даёт список позиций, но без сопоставления со справочником и уверенности: ручная проверка (5.3) и расчёт (Epic 6) не знают, какую расценку и нормализованную ЕИ использовать.

**Approach:** Добавить read-endpoint активных расценок в rate-catalog и stateless `mapping-service` с `POST /v1/mapping/proposals`: сервис принимает позиции 5.1, читает активную версию справочника по API, нормализует ЕИ, детерминированно подбирает лучшую расценку и возвращает `confidence` + `needs_review` (`< 0.70`). Сохранение и подтверждение — Story 5.3.

## Boundaries & Constraints

**Always:** company-scope из `X-Company-Id` (AD-013); расценки читаются только через API rate-catalog, не из его БД (AD-014); маппинг детерминированный, без LLM/облака; `confidence` в `[0,1]`, `needs_review = confidence < 0.70`; вход — позиции 5.1 (`name`, `kind`, `unit`, `quantity`, `section`), не PDF/текст; лимит 1000 позиций; ошибки — `application/problem+json`; слои mapping-service по AD-011; нормализация ЕИ возвращается в каждом предложении.

**Never:** сохранение предложений/подтверждений (5.3), изменение справочника, импорт Excel, расчёт стоимости, UI, LLM-переранжирование, очередь, чтение чужой БД, новый сервис вместо mapping-service.

**Решения (2026-10-20):** `GET /v1/rate-catalog/versions/current/rates` → `200 [{id, code, name, unit, rate_amount}]`, нет активной версии → `404`; `POST /v1/mapping/proposals` с `{positions:[...]}` → `200 {items:[{index, position, match, normalized_unit, confidence, needs_review}]}`, `match = {rate_id, code, name, unit, rate_amount}` или `null`; mapping-service stateless, без БД; gateway-префикс `/v1/mapping`; compose-сервис `mapping-service` с `RATE_CATALOG_URL`. Подбор: нормализация ЕИ по словарю алиасов, кандидаты с совпавшей ЕИ, иначе все с штрафом; нормализация названия (регистр, ё→е, пунктуация, пробелы); score = `0.85 * name_similarity + 0.15 * unit_match`, где name_similarity — максимум из SequenceMatcher и token Jaccard, unit_match = 1 при совпадении и 0.35 иначе; точное совпадение названия и ЕИ → 1.0; `round(score, 3)`; пустой справочник → `404`, нет кандидата → `match=null, confidence=0.0, needs_review=true`. Вход > 1000 позиций → `422`; недоступен rate-catalog → `502`. Качество 90–95% / доля `<0.70` ≤15% на реальном наборе отложено: нет каталога и корпуса; тесты — синтетические. Реальные метрики 90–95% и доля `<0.70` ≤15% отложены до появления справочника и эталонного набора позиций.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Точное совпадение | позиция совпадает с расценкой по названию и ЕИ | `200`, `confidence=1.0`, `needs_review=false` | — |
| Вариант ЕИ | `м³`/`куб.м` против `м3` | нормализованная ЕИ `м3`, кандидат найден | — |
| Fuzzy-название | близкое название | лучшая расценка и `confidence ≥ 0.70` либо ниже | — |
| Нет совпадения по ЕИ | ЕИ не совпадает ни с одной | лучший кандидат со штрафом, низкий confidence | — |
| Пустой список | `positions=[]` | `200`, `items=[]` | — |
| Нет активной версии | справочник без current | — | `404` |
| rate-catalog недоступен | таймаут/5xx | — | `502` |
| Нет identity | без `X-Company-Id` | — | `401` |
| Слишком много позиций | > 1000 | — | `422` |

</frozen-after-approval>

## Code Map

- `rate-catalog/app/domain/rate.py` -- добавить `id: UUID | None = None` в `Rate`; новый `RateRow` не нужен.
- `rate-catalog/app/domain/ports.py` -- `RateCatalogRepository.list_rates(version_id)`.
- `rate-catalog/app/infrastructure/repository.py` -- `list_rates` по `rate_catalog_version_id`, порядок стабильный (`code`, `id`).
- `rate-catalog/app/application/current_rates.py` -- новый `ListCurrentRates`: компания → current version → строки; нет версии → `CatalogNotFound`.
- `rate-catalog/app/transport/{schemas,routes}.py` -- `RateResponse` (`rate_amount` строкой), `GET /v1/rate-catalog/versions/current/rates`.
- `rate-catalog/app/wiring.py` -- `CatalogVersions.list_current_rates`.
- `rate-catalog/tests/test_current_rates.py` -- новый: 200, 404, tenancy, пустая версия, Decimal-сериализация.
- `mapping-service/app/domain/{position,rate,mapping}.py` -- новые `PositionInput`, `RateCandidate`, `MappingSuggestion` (`index`, `position`, `match`, `normalized_unit`, `confidence`, `needs_review`).
- `mapping-service/app/application/units.py` -- словарь алиасов и `normalize_unit`: `м³/куб.м` → `м3`, `м²/кв.м` → `м2`, `пог.м/м.п.` → `м`, `шт./штук` → `шт`, `т./тонн` → `т`, `комплект` → `компл`, `упак/уп.` → `упак`; fallback — lowercase, ё→е, без хвостовых точек, пробелы схлопнуты.
- `mapping-service/app/application/matcher.py` -- нормализация названий, candidate blocking по токенам, SequenceMatcher + Jaccard, формула confidence, порог 0.70.
- `mapping-service/app/application/propose_mapping.py` -- `ProposeMapping`: валидация лимита, загрузка активных расценок, построение предложений в исходном порядке.
- `mapping-service/app/infrastructure/rate_catalog_client.py` -- httpx-клиент `GET /v1/rate-catalog/versions/current/rates` с `X-Company-Id`; 404 → `CatalogNotConfigured`, сеть/5xx → `RateCatalogUnavailable`.
- `mapping-service/app/transport/{schemas,routes,problem}.py` -- `POST /v1/mapping/proposals`, `X-Company-Id`, problem+json 401/404/422/502.
- `mapping-service/app/wiring.py`, `main.py` -- сборка use case, env `RATE_CATALOG_URL` (default `http://rate-catalog:8000`).
- `mapping-service/pyproject.toml` -- runtime `httpx`, `pydantic` (FastAPI уже есть).
- `mapping-service/tests/{test_units,test_matcher,test_proposals,test_rate_catalog_client}.py` -- нормализация, confidence, матрица I/O, клиент на MockTransport.
- `api-gateway/app/application/routing.py`, `wiring.py` -- `DEFAULT_MAPPING_SERVICE_URL`, префикс `/v1/mapping`, env `MAPPING_SERVICE_URL`.
- `api-gateway/tests/test_gateway.py` -- 401 и проксирование `/v1/mapping/proposals` с identity.
- `deploy/docker-compose.yml`, `.env.example`, `README.md` -- сервис `mapping-service` (без БД), env/route/depends; обновить список.
- `llm-gateway/*`, `pdf-extract/*`, `estimate/*`, `calc-service/*` -- не трогать.

## Tasks & Acceptance

**Execution:**
- [x] `rate-catalog/app/domain/{rate,ports}.py`, `infrastructure/repository.py` -- id и чтение строк версии -- контракт для mapping.
- [x] `rate-catalog/app/application/current_rates.py`, `transport/{schemas,routes}.py`, `wiring.py` -- `GET .../current/rates` -- активная версия и tenancy.
- [x] `rate-catalog/tests/test_current_rates.py` -- новый -- 200/404/изоляция/Decimal.
- [x] `mapping-service/app/domain/*`, `application/units.py` -- типы и нормализация ЕИ -- единый канон.
- [x] `mapping-service/app/application/{matcher,propose_mapping}.py` -- score/confidence/порог -- детерминированный подбор.
- [x] `mapping-service/app/infrastructure/rate_catalog_client.py` -- HTTP-клиент -- 404/502.
- [x] `mapping-service/app/transport/{schemas,routes,problem}.py`, `wiring.py`, `main.py` -- endpoint -- 200/401/404/422/502.
- [x] `mapping-service/pyproject.toml` -- httpx -- зависимость.
- [x] `mapping-service/tests/*` -- матрица, units, matcher, клиент -- регрессия.
- [x] `api-gateway/app/{application/routing,wiring}.py`, `tests/test_gateway.py` -- `/v1/mapping` -- edge.
- [x] `deploy/*` -- compose/env/README -- сервис поднимается.
- [x] `rate-catalog` + `mapping-service` + `api-gateway` -- ruff/pytest; deploy -- compose config -- верификация.

**Acceptance Criteria:**
- Given позиции 5.1 и активная версия справочника, when `POST /v1/mapping/proposals`, then `200` и для каждой позиции предложение с `rate_id`, нормализованной ЕИ, `confidence` и `needs_review`.
- Given ЕИ `м³`/`куб.м`, when маппинг, then нормализованная ЕИ `м3`, кандидат ищется среди `м3`.
- Given точное совпадение названия и ЕИ, when маппинг, then `confidence=1.0`, `needs_review=false`.
- Given `confidence < 0.70`, when ответ, then `needs_review=true`; при отсутствии кандидата `match=null`, `confidence=0.0`.
- Given нет активной версии, when маппинг, then `404`; rate-catalog недоступен → `502`; >1000 позиций → `422`; нет `X-Company-Id` → `401`.
- Given реальный набор, when измеряется качество, then цель 90–95% и доля `<0.70` ≤15% (отложено до появления данных — см. Open Questions).

## Implementation Notes

- `match=null` при непустом справочнике недостижим: `best_match` всегда возвращает лучшего кандидата из пула, а пустой каталог отдаёт `404 CatalogNotConfigured` ещё до подбора. Поле `match` остаётся nullable в контракте на будущее (например, если появится порог отбрасывания кандидатов). Поведение подтверждено walkthrough 2026-10-21; I/O-матрица с «нет кандидата → match=null» читается как недостижимая в текущей реализации.

## Spec Change Log

## Review Triage Log

- **2026-10-20 · blind-hunter + edge-case-hunter · пустой активный справочник даёт 200/null, а frozen-решение требует 404** — verdict: medium. evidence: `ListCurrentRates` возвращает `[]`, `ProposeMapping` строит null-предложения; README обещает 404 для пустого справочника. route: patch → 404 при пустом списке rates, обновить тест/README.
- **2026-10-20 · blind-hunter + edge-case-hunter · `match=null` при непустом справочнике недостижим** — verdict: medium. evidence: `best_match` всегда выбирает кандидата из непустого списка; null остаётся только для пустого справочника, который теперь 404. route: patch (следствие предыдущего) — контракт null остаётся зарезервированным, README уточняется.
- **2026-10-20 · blind-hunter + edge-case-hunter · hard-filter по ЕИ отбрасывает exact-name кандидата с другой ЕИ** — verdict: medium. evidence: для `Бетон/м3` при `Кирпич/м3` и `Бетон/шт` выбирается кирпич; фильтр зафиксирован frozen-решением «совпавшая ЕИ, иначе все». route: defer → калибровка на реальном справочнике.
- **2026-10-20 · blind-hunter + edge-case-hunter · exact name + неверная ЕИ даёт confidence 0.9025 и needs_review=false** — verdict: medium. evidence: формула frozen-решения `0.85*name + 0.15*unit`; 1.0 имени и штраф 0.35 дают 0.9025. route: defer → калибровка порога/штрафа на реальных данных.
- **2026-10-20 · blind-hunter + edge-case-hunter · нет пагинации/кэша активных расценок** — verdict: low. evidence: mapping загружает весь справочник на каждый запрос; объём каталога в MVP неизвестен. route: defer → кэш по версии/пагинация при появлении реального объёма.
- **2026-10-20 · blind-hunter + edge-case-hunter · экстремальный exponent quantity принимается и раздувается `format(..., "f")`** — verdict: medium. evidence: `Decimal("1e999999999")` finite, сериализация даёт огромную строку/OOM. route: patch → ограничение `adjusted()` и тест.
- **2026-10-20 · blind-hunter · нет `max_length` на `positions`, лимит проверяется после парсинга** — verdict: medium. evidence: `ProposalsRequest.positions` без ограничения; 1001 позиция парсится полностью. route: patch → `Field(max_length=MAX_POSITIONS)`.
- **2026-10-20 · blind-hunter · глобальный `httpx.Client` не закрывается** — verdict: low. evidence: `wiring._client` живёт без shutdown; утечка соединений при reload. route: patch → lifespan close.
- **2026-10-20 · blind-hunter + edge-case-hunter · `Rate.id` optional, а `RateResponse.id` обязателен → возможен 500** — verdict: medium. evidence: `Rate` можно создать без id, `_rate_response` передаст None в обязательное поле. route: patch → id с default uuid4/явная проверка.
- **2026-10-20 · blind-hunter + verification-gap · порядок `list_rates` по `id` не проверен для NULL/duplicate code** — verdict: medium. evidence: тест использует только разные непустые коды; удаление `id.asc()` не ломает тесты. route: patch → тест с NULL/одинаковым кодом и обратным порядком вставки.
- **2026-10-20 · blind-hunter · при равном confidence выбирается первый кандидат, tie-break не зафиксирован** — verdict: medium. evidence: строгое `>`; порядок зависит от ответа rate-catalog. route: patch → tie-break `(score, code, rate_id)` + тест.
- **2026-10-20 · blind-hunter + edge-case-hunter · словарь ЕИ не покрывает кг/л/ч/км/га/% и `M³`/`M²`, «куб. метр»** — verdict: medium. evidence: fallback оставляет `m³`/«куб. метр» как есть, ЕИ считается несовпавшей. route: patch → алиасы и тесты.
- **2026-10-20 · blind-hunter + verification-gap · нет кросс-сервисного контрактного теста rate-catalog → mapping** — verdict: medium. evidence: обе стороны используют свои схемы/моки; дрейф `id`/`rate_amount` пройдёт незаметно. route: patch → golden payload + проверка ключей/типов с обеих сторон.
- **2026-10-20 · blind-hunter · README rate-catalog не описывает новый endpoint, 401 и порядок** — verdict: low. evidence: контракт есть в коде/OpenAPI, но не в README. route: patch → документация.
- **2026-10-20 · blind-hunter · Dockerfile/pyproject/README сервисов всё ещё «каркас Story 1.1»** — verdict: low. evidence: mapping Dockerfile говорит «без внешних зависимостей», api-gateway README не знает `/v1/mapping`. route: patch → метаданные/документация.
- **2026-10-20 · edge-case-hunter · тысяча позиций без общих токенов × большой справочник = миллионы SequenceMatcher** — verdict: medium. evidence: после fallback весь справочник скорится для каждой позиции; нужен cap кандидатов. route: patch → top-N по token overlap.
- **2026-10-20 · edge-case-hunter · пустой batch возвращает 200 без проверки активного справочника** — verdict: false. evidence: матрица спеки явно задаёт `positions=[] → 200 items=[]`, каталог не нужен. route: reject.
- **2026-10-20 · verification-gap · production-композиция mapping-service не проверяется** — verdict: medium. evidence: тесты всегда инжектят use case; удаление `include_router` или игнор `RATE_CATALOG_URL` не ломает 60 тестов. route: patch → тест через `main.app` и MockTransport.
- **2026-10-20 · verification-gap · token blocking не покрыт тестом** — verdict: medium. evidence: удаление `_block_by_tokens` не ломает набор. route: patch → кейс с кандидатом без общего токена.
- **2026-10-20 · blind-hunter · malformed `X-Company-Id` даёт 422, а не 401** — verdict: low. evidence: невалидный UUID — ошибка валидации; отсутствие заголовка по-прежнему 401; gateway присылает UUID. route: reject.


## Verification

**Commands (выполнено 2026-10-20, после review-фиксов):**
- `cd rate-catalog && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `58 passed`
- `cd mapping-service && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `83 passed`
- `cd api-gateway && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `44 passed`
- `cd deploy && docker compose config -q` → exit 0
- `cd mapping-service && docker build -q -t mapping-service:ci .` → образ собран (`sha256:59051cd…`)

**Не покрыто автоматически:** реальный справочник и корпус позиций; метрики 90–95% / ≤15% не измеряются без данных.

**Walkthrough (2026-10-21):** approved. Замечание: при непустом справочнике `match` не бывает `null` — сервис всегда возвращает лучшего кандидата с низким `confidence`; `null` достижим только для пустого справочника, который отдаёт `404`. Поведение принято как есть, спека не менялась.
