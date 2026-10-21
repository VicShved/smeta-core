---
title: 'Story 3.3 — Snapshot расценок в позициях смет (пост-фактум)'
type: 'feature'
created: '2026-10-21'
status: 'done'
route: 'documentation'
baseline_commit: '8173ca7'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-3-context.md'
  - '_bmad-output/implementation-artifacts/epic-3-retro-10-21-2026.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 3.3 была закрыта без собственной спеки: снапшоты расценок реализованы в историях 6.1 (колонки) и 6.3 (заполнение), но контракт нигде не зафиксирован, а AC эпика проверялся косвенно.

**Approach:** Зафиксировать пост-фактум as-built контракт снапшотов в `estimate_item` и неизменяемость рассчитанной версии. Код не меняется; спека описывает фактическое поведение и его проверку.

## Boundaries & Constraints

**Always:** снапшоты заполняются при расчёте; рассчитанная версия неизменяема; изменения справочника не влияют на итог и снапшоты.

**Never:** новые миграции, поля или API; пересчёт рассчитанной версии; изменение rate-catalog.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Расчёт | подтверждённые позиции с `rate_catalog_id` | `rate_snapshot_id`, `unit_snapshot`, `rate_amount_snapshot`, `cost`, `total_cost`, `calculated_at` заполнены | — |
| Новая версия справочника | рассчитанная смета + изменённая расценка | итог и снапшоты прежние | — |
| Повторный расчёт | версия уже рассчитана | — | `409`, версия не меняется |
| Позиция без расценки | `rate_catalog_id=null` | не входит в итог, попадает в `uncovered` | — |

</frozen-after-approval>

## Code Map

- `../estimate/app/infrastructure/models.py` -- колонки снапшотов в `estimate_item`.
- `../estimate/app/application/calculate_cost.py` -- заполнение снапшотов и итога.
- `../estimate/app/application/calculate_cost.py` -- запрет повторного расчёта.
- `../estimate/tests/test_calculate_cost.py` -- тесты снапшотов и неизменяемости.

## Tasks & Acceptance

**Execution:**
- [x] `estimate` -- колонки снапшотов -- схема 6.1.
- [x] `estimate` -- заполнение при расчёте -- 6.3.
- [x] `estimate/tests/test_calculate_cost.py` -- тест неизменяемости при изменении справочника -- регрессия.

**Acceptance Criteria:**
- Given рассчитанная версия сметы, when импортируется новая версия справочника, then итог, `unit_snapshot`, `rate_amount_snapshot` и `rate_snapshot_id` не меняются.
- Given рассчитанная версия, when повторный расчёт, then `409` и состояние версии не меняется.
- Given позиция без подтверждённой расценки, when расчёт, then она в `uncovered` и не влияет на итог.

## Implementation Notes

- Колонки: `rate_snapshot_id`, `unit_snapshot` (`String(100)`), `rate_amount_snapshot` (`Numeric(18,4)`), `cost` (`Numeric(18,2)`) в `estimate_item` (миграция 0001, Story 6.1).
- Расчёт читает расценку по неизменяемому id (`find_rate_by_id`), пишет снапшоты и `cost` по формуле `qty × rate_amount` c `ROUND_HALF_UP`, итог — сумма округлённых стоимостей (`calculate_cost.py:79-81`).
- Повторный расчёт запрещён проверкой `calculated_at` → `VersionAlreadyCalculated` (409).
- Фактическая модель: версия замораживается, новая редакция — новая версия сметы. AC 3.3 в `epics.md` переформулирован 2026-10-21 по решению владельца.

## Verification

- `cd estimate && .venv/bin/pytest -q` → `287 passed, 1 skipped`.
- `test_calculate_persists_snapshot_cost_and_total` — снапшоты и итог.
- `test_calculated_version_is_immutable_when_rate_catalog_changes` — новая цена в справочнике не меняет итог/снапшоты, повторный расчёт `409`.
