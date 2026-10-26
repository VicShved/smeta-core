---
title: 'Story 8.5 — Ссылка на материал справочника'
type: 'feature'
created: '2026-10-26'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/spec-8-4-выбор-материала-из-справочника.md'
---

## Intent

**Problem:** `estimate_item.rate_catalog_id` ссылался на `material_price.id` — цену конкретного периода, а не на карточку материала. Не было прямой ссылки на `material.id`, из-за чего интеграции и аналитика не могли однозначно определить материал без периода цены.

**Approach:** Заменить живую ссылку в позиции сметы на `material_id` → `rate_catalog.material.id` (логическая ссылка между БД). Цену искать по `material_id` + `estimate.catalog_period`; `rate_snapshot_id` оставить ссылкой на строку цены в рассчитанной версии. Выполнить backfill старых price-id → material-id.

## Boundaries & Constraints

**Always:** логическая ссылка UUID (межбазовый FK невозможен по AD-014); цена берётся за период сметы; `rate_snapshot_id` — id цены после расчёта; feedback получает material-id; backfill до удаления старой колонки.

**Never:** не хранить цену как живую ссылку; не менять алгоритм matching; не терять исторические снапшоты.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Match | найден материал | `material_id`, `rate_code`, `confirmed` | — |
| Correct | выбран `material_id` | цена за период, `corrected` | нет цены/периода — `422` |
| Calculate | material + period | `rate_snapshot_id`, цена, стоимость | нет цены — `rate_not_found` |
| List | material + period | код/название из справочника | справочник недоступен — деградация |
| Diff | material изменился | `changes: [rate]` | — |
| Backfill | старая цена | material-id заполнен | неразрешённые — отчёт, exit 1 |

## Code Map

- `rate-catalog/app/domain/rate.py`, `transport/schemas.py`, `routes.py`, `infrastructure/repository.py` -- `material_id`, `material_ids`.
- `estimate/app/domain/{item,rate,diff}.py`, `infrastructure/{models,repository,rate_catalog_client}.py` -- `material_id`, batch по материалам.
- `estimate/app/application/{match_rates,review_item,list_items,calculate_cost}.py` -- период сметы.
- `estimate/migrations/versions/0011_material_reference.py`, `0012_drop_rate_catalog_id.py`, `app/tools/backfill_material_ids.py`.
- `ui/src/api/client.ts`, `views/EstimateView.ts`, `components/EstimateTable.vue`.

## Tasks & Acceptance

- [x] `rate-catalog` — `material_id` в `RateResponse`, `material_ids` в `/rates`.
- [x] `estimate` — `material_id` в домене/БД/API, цена по периоду.
- [x] backfill — 792 позиции и 5 feedback.
- [x] удалить `rate_catalog_id` и старые feedback-колонки.
- [x] UI — picker отправляет `material_id`.

**Acceptance Criteria:**
- Given match/correct, when сохранение, then позиция содержит `material_id`.
- Given расчёт, when материал и период, then `rate_snapshot_id` = id цены.
- Given старые позиции, when backfill, then у всех 792 есть `material_id`.
- Given новая схема, when запрос items, then `rate_catalog_id` отсутствует, `material_id` присутствует.

## Implementation Notes

- `material_id` — логическая ссылка: `material` и `estimate_item` в разных БД.
- `rate_snapshot_id` остаётся id `material_price` и не участвует в matching.
- Diff сравнивает `material_id`, поэтому смена только периода при том же материале не показывается как смена расценки.
- Backfill использует старый `/rates?ids=` и удаляется вместе со старой колонкой.

## Verification

- `cd rate-catalog && .venv/bin/ruff check . && .venv/bin/pytest -q` — **148 passed, 2 skipped**.
- `cd estimate && .venv/bin/ruff check . && .venv/bin/pytest -q` — **360 passed, 2 skipped**.
- `cd ui && npm run typecheck && npm test` — **158 passed**.
- Живой backfill: `items 792/792, feedback 5/5, unresolved 0`.
- `alembic current` — `0012 (head)`; в `estimate_item` 792/792 `material_id`, старой колонки нет.
