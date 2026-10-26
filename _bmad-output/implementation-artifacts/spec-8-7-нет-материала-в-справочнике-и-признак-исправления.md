---
title: 'Story 8.7 — Нет материала в справочнике и признак ручного исправления'
type: 'feature'
created: '2026-10-26'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/spec-8-5-ссылка-на-материал-справочника.md'
---

## Intent

**Problem:** В форме выбора материала нельзя явно сказать «подходящего материала нет»: можно только закрыть модалку. В таблице не видно, что связанный материал исправлен сметчиком, а не подтверждён автоматически.

**Approach:** Добавить действие `unmatch`: из модалки возвращается пустой выбор, backend очищает `material_id`/`rate_code`, ставит `no_match`, пишет feedback и исключает позицию из расчёта. В таблице показать метку «исправлено вручную» для `corrected` и «нет в справочнике» для `no_match`.

## Boundaries & Constraints

**Always:** `unmatch` сохраняет исходный материал в `match_feedback`; `no_match` не пересопоставляется и не входит в итог; пустой выбор не подменяется другим материалом.

**Never:** не удалять историю feedback; не менять сохранённые снапшоты рассчитанной версии; не считать `no_match` как `rejected`.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Empty picker | кнопка «Нет подходящего материала» | `unmatch`, материал очищен, `no_match` | `422` при material/rate/unit |
| Repeat unmatch | повторный `unmatch` | статус сохраняется, feedback не дублируется | — |
| Calculate | `no_match` | вне итога, причина `no_match` | — |
| Re-match | `no_match` | не пересопоставляется | — |
| Manual correction | `corrected` + `reviewed_at` | метка «исправлено вручную» | — |
| No match UI | `no_match` | «нет в справочнике», чекбокс disabled | — |

## Code Map

- `estimate/app/domain/item.py` -- `ItemStatus.no_match`, `ReviewDecision.unmatch`.
- `estimate/app/application/review_item.py` -- очистка связи и feedback.
- `estimate/app/application/calculate_cost.py` -- причина `no_match`.
- `estimate/migrations/versions/0013_unmatch_decision.py` -- check constraint feedback.
- `estimate/migrations/versions/0014_reviewed_by.py` -- автор ручного решения.
- `estimate/migrations/versions/0015_clear_manual_confidence.py` -- очистка confidence у `corrected`/`no_match`.
- `auth-users`/`api-gateway` -- claim `email` и `X-User-Email`.
- `ui/src/components/PricePickerDialog.vue` -- кнопка пустого выбора.
- `ui/src/components/EstimateTable.vue` -- метка `исправлено: <логин>`, подписи под чекбоксом, `no_match` без чекбокса.
- `ui/src/views/EstimateView.ts` -- PATCH `unmatch`.

## Tasks & Acceptance

- [x] backend `unmatch`/`no_match` и миграция 0013.
- [x] пустой выбор в модалке.
- [x] метка «исправлено вручную» и статус `no_match` в таблице.
- [x] тесты backend/UI, README, PRD.

**Acceptance Criteria:**
- Given модалка выбора, when «Нет подходящего материала», then PATCH `unmatch`, модалка закрыта, материал очищен.
- Given `no_match`, when расчёт, then позиция вне итога с причиной `no_match`.
- Given `corrected`, when таблица, then видна метка `исправлено: <логин>`, confidence пустой.
- Given `no_match`, when таблица, then нет чекбокса, только «Нет в справочнике».
- Given повторный `unmatch`, when PATCH, then feedback не дублируется.

## Implementation Notes

- Для `unmatch` очищаются `material_id`, `rate_code`, `correction_link`, `confidence=0`; `unit_normalized` остаётся исходной.
- `update_item` теперь сохраняет `confidence`, чтобы идемпотентность `unmatch` работала.
- `match_feedback.decision` расширен значением `unmatch`.

## Verification

- `cd estimate && .venv/bin/ruff check . && .venv/bin/pytest -q` — **364 passed, 2 skipped**.
- `cd auth-users && .venv/bin/pytest -q` — **19 passed**.
- `cd api-gateway && .venv/bin/pytest -q` — **70 passed**.
- `cd ui && npm run typecheck && npm test` — **162 passed**.
- Миграции `0013`–`0015` применяются после `0012`; `correct`/`unmatch` сохраняют `confidence=NULL`.
