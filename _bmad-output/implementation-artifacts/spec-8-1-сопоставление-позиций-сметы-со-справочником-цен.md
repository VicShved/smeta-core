---
title: 'Story 8.1 — Сопоставление позиций сметы со справочником цен по эмбеддингам'
type: 'feature'
created: '2026-10-24'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
baseline_commit: '475c85f'
context:
  - 'docs/architecture-spine.md'
  - '_bmad-output/implementation-artifacts/spec-5-2-маппинг-со-справочником-и-нормализация-еи-с-confidence.md'
  - '_bmad-output/implementation-artifacts/spec-7-2-ui-просмотр-спецификации-маппинга-и-стоимости.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Позиции сметы хранят `rate_catalog_id` и `confidence`, но автоматического сопоставления со справочником цен нет. Существующий `POST /v1/rate-catalog/search` возвращает материал без строки цены и периода, поэтому записать в смету расценку для расчёта нельзя.

**Approach:** Добавить в `rate-catalog` endpoint `POST /v1/rate-catalog/matches`, который для каждого наименования считает E5 query-эмбеддинг и возвращает ближайшую по cosine цену за точный регион/год/квартал. В `estimate` добавить хранение периода справочника и endpoint `POST /v1/estimates/{id}/versions/{vid}/match-rates`, который сохраняет найденные `rate_catalog_id`/`rate_code`/`confidence`. В UI — поля периода и кнопка «Сопоставить со справочником».

## Boundaries & Constraints

**Always:** период справочника хранится в смете; задаётся только на экране сметы и сохраняется успешным сопоставлением; поиск идёт по наименованию через E5 query embedding; ближайшая цена выбирается только среди строк с ценой за точный регион/год/квартал; повторное сопоставление обновляет только `proposed`; `confirmed`/`corrected` не меняются; после поиска статус остаётся `proposed`, `confidence` = cosine score в `[0,1]` с 4 знаками, `needs_review` — по порогу 0.70; ответы в исходном порядке; ошибки — `application/problem+json`.

**Never:** менять контракт существующего `/search`; искать без периода; трогать рассчитанную/неактивную версию; автоматически подтверждать позиции; использовать LLM для сопоставления; читать чужую БД; добавлять новый сервис.

**Решения (2026-10-24):** период хранится в `estimate` (`catalog_region_code`, `catalog_year`, `catalog_quarter`, nullable для старых строк); поля периода задаются на экране сметы; первое успешное сопоставление сохраняет период; повторное нажатие обновляет только `proposed`, `confirmed`/`corrected` пропускаются; результат сохраняется сразу; статус остаётся `proposed`; порога отсечения нет.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Совпадение | имя позиции и цена за период | `200`, ближайшая цена и score, позиция обновлена | — |
| Цена другого периода | у ближайшего материала нет цены за период | выбирается следующий ближайший материал с ценой | — |
| Нет цен | пустой период | `200`, `rate=null`, позиция без расценки, `confidence=0` | — |
| Нет эмбеддингов | материалы без векторов | `200`, `rate=null` | — |
| Повторное нажатие | есть `confirmed`/`corrected` | обновляются только `proposed`; `skipped` в ответе | — |
| Рассчитанная версия | `calculated_at != null` | — | `409` |
| Неактивная версия | `is_active=false` | — | `409` |
| Чужая смета/версия | другой `company_id` | — | `404` |
| Неизвестный регион | нет в `region` | — | `404` |
| Модель недоступна | сбой эмбеддера | — | `503` |
| Справочник недоступен | сеть/5xx | — | `502`, без частичных записей |
| Слишком много имён | > 1000 | — | `422` |

</frozen-after-approval>

## Code Map

- `rate-catalog/app/domain/rate.py` -- `RateMatch`, `RateSearchResult`.
- `rate-catalog/app/domain/ports.py` -- `nearest_rate(embedding, region_code, year, quarter)`.
- `rate-catalog/app/infrastructure/repository.py` -- join эмбеддинга и цены периода, cosine distance, tie-break.
- `rate-catalog/app/application/match_rates.py` -- батчевый поиск по именам.
- `rate-catalog/app/transport/{schemas,routes}.py`, `app/wiring.py` -- `POST /v1/rate-catalog/matches`.
- `rate-catalog/tests/test_match_rates.py` -- новый набор.
- `estimate/app/domain/estimate.py` -- `CatalogPeriod` и поле в `Estimate`.
- `estimate/app/domain/rate.py`, `app/domain/ports.py` -- порт и результат сопоставления, запись периода/позиции.
- `estimate/app/infrastructure/models.py`, `app/infrastructure/repository.py` -- колонки периода и их запись.
- `estimate/app/infrastructure/http.py`, `rate_catalog_client.py` -- POST к справочнику, длинный таймаут.
- `estimate/app/application/match_rates.py` -- чтение контекста → сеть без сессии → транзакционная запись.
- `estimate/app/transport/{schemas,routes}.py`, `app/wiring.py` -- endpoint и `catalog_period` в `GET /estimates/{id}`.
- `estimate/migrations/versions/0006_add_catalog_period.py` -- три nullable-колонки и CHECK.
- `estimate/tests/` -- миграция, клиент, use case, API, tenancy, сессия.
- `api-gateway/app/transport/routes.py` -- длинный таймаут для новых маршрутов.
- `ui/src/api/client.ts`, `src/views/EstimateView.ts`, `tests/estimate_view.test.ts` -- период и кнопка.
- `ui/nginx.conf` -- длинный таймаут для match-routes.
- `rate-catalog/README.md`, `estimate/README.md`, `ui/README.md` -- контракт и UX.

## Tasks & Acceptance

**Execution:**
- [x] `rate-catalog` -- домен, репозиторий, use case, endpoint, wiring -- ближайшая цена по эмбеддингу.
- [x] `rate-catalog/tests/test_match_rates.py` -- период, порядок, null, лимиты, ошибки -- регрессия.
- [x] `estimate` -- миграция, домен, порт, HTTP-клиент, use case, endpoint -- сохранение совпадений.
- [x] `estimate/tests/` -- миграция, клиент, матрица I/O, сессия, tenancy -- регрессия.
- [x] `api-gateway` + `ui/nginx.conf` -- длинные таймауты -- запрос не обрывается.
- [x] `ui` -- период, кнопка, состояния и тесты -- рабочий сценарий.
- [x] README и verify -- ruff/pytest/typecheck/build/compose config.

**Acceptance Criteria:**
- Given позиции сметы и цены за период, when `POST .../match-rates`, then для каждой `proposed`-позиции сохранена ближайшая цена и `confidence`, ответ содержит счётчики.
- Given повторное нажатие, when есть `confirmed`/`corrected`, then они не меняются, а `proposed` пересопоставляются.
- Given рассчитанная/неактивная версия, when сопоставление, then `409`; чужая смета — `404`; справочник недоступен — `502`.
- Given нет цен/эмбеддингов, when сопоставление, then позиция без расценки с `confidence=0`, без ошибки.
- Given период впервые сохранён, when `GET /v1/estimates/{id}`, then `catalog_period` заполнен.
- Given UI, when нажата кнопка, then уходит период и после успеха таблица перезагружается.

## Spec Change Log

## Review Triage Log

## Verification

**Commands (выполнено 2026-10-24):**
- `cd rate-catalog && .venv/bin/ruff check . && .venv/bin/pytest -q` -> `All checks passed!`, `125 passed, 1 skipped`
- `cd estimate && .venv/bin/ruff check . && .venv/bin/pytest -q` -> `All checks passed!`, `347 passed, 2 skipped`
- `cd api-gateway && .venv/bin/ruff check . && .venv/bin/pytest -q` -> `All checks passed!`, `70 passed`
- `cd ui && npm test && npm run typecheck && npm run build` -> `149 passed`, typecheck/build без ошибок
- `cd deploy && docker compose config -q` -> OK

**Не покрыто автоматически:** качество cosine-порога на реальном справочнике.

## Implementation Notes

- **rate-catalog**: `POST /v1/rate-catalog/matches` ищет только среди материалов, у которых есть цена за точный регион/год/квартал/зону; сортировка cosine distance, затем `code`, затем `price.id`. Ответ в порядке входа, до 1000 имён (каждое ≤1000 символов); границы покрыты тестами. `/search` не менялся.
- **estimate**: период хранится в `estimate` четырьмя nullable-колонками (`region_code`, `year`, `quarter`, `zone`) с CHECK «все или ни одной»; миграции `0006_add_catalog_period.py` и `0007_add_catalog_zone.py`. Каждое успешное сопоставление обновляет период; `GET /v1/estimates/{id}` отдаёт `catalog_period` с `zone_code`.
- **estimate (сопоставление)**: чтение контекста → вызов справочника при закрытой сессии → транзакционная запись под блокировкой. Обновляются только `proposed`; `confirmed`/`corrected` попадают в `skipped`. `confidence` = cosine score, clamp 0..1, Numeric(5,4).
- **UI**: `EstimateMatchPanel.vue` с регионом/годом/кварталом/зоной; после успеха таблица перезагружается с первой страницы, период обновляется локально.
- **Таймауты**: gateway — 900 с (`PROXY_RATE_CATALOG_TIMEOUT_SECONDS`), nginx — 920 с, клиент UI — 900 с; HTTP-клиент estimate к rate-catalog — 600 с.

## As-built addendum (2026-10-25)

- Период справочника и `POST /v1/rate-catalog/matches` включают `zone_code`; миграция `estimate` — `0007_add_catalog_zone.py`. Это добавлено Story 3.5 после первой редакции 8.1.
- Период обновляется каждым успешным `match-rates`, а не только первым; UI позволяет менять период до расчёта версии.
- `matches` принимает 1–1000 имён; ровно 1000 — `200`, 1001 — `422`. Тесты: `rate-catalog/tests/test_match_rates.py::test_match_accepts_up_to_1000_names`, `::test_match_rejects_more_than_1000_names`.
- Браузерный e2e (`deploy/e2e`) дополнен шагом `match-rates` и проверкой сохранённого периода.
