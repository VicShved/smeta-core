---
title: 'Story 8.2 — Обратная связь и ручная привязка материала'
type: 'feature'
created: '2026-10-26'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/spec-8-1-сопоставление-позиций-сметы-со-справочником-цен.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Сметчик видит неверное сопоставление, но не может пометить его и привязать корректный материал из справочника. Исправления и ошибки не сохраняются как данные, поэтому их нельзя разобрать и использовать для оценки/обучения.

**Approach:** Добавить в позицию статус `rejected`, поле `correction_link` и append-only журнал `match_feedback`. Расширить review API: `reject`, `correct` по ссылке/коду/id, `confirm`; каждая операция пишет событие. В UI — кнопки «Неверно»/«Исправить» и копирование ссылки из `/prices`. Экспорт журнала в JSONL — в этой же истории.

## Boundaries & Constraints

**Always:** append-only `match_feedback`; каждое действие — новое событие; текущий статус позиции — по последнему действию; `rejected` не входит в расчёт; `corrected` входит; ссылка проверяется на период/зону/существование; рассчитанная версия — `409`; ошибки — `application/problem+json`.

**Never:** не менять алгоритм сопоставления (8.2 attributes/rerank отложен); не запускать обучение модели; не добавлять внешние LLM; не удалять историю feedback; не менять контракт `match-rates`.

**Решения (2026-10-26):** 8.3 переименован в 8.2; 8.2 attributes/rerank убран из плана. Ссылка принимает URL `/prices`, material code и rate id. Комментарий опционален. Отмена — новое событие. Экспорт JSONL — в 8.2. Исправлять могут все авторизованные.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Confirm | proposed-позиция с расценкой | `confirmed`, событие `confirm` | — |
| Reject | позиция с расценкой | `rejected`, событие `reject`, не входит в итог | — |
| Correct by link | URL/code/id, материал есть за период | `corrected`, событие `correct`, ссылка сохранена | — |
| Bad link | материал не найден | позиция не меняется | `422` |
| Period mismatch | ссылка на другой период/зону | позиция не меняется | `422` |
| Calculated version | `calculated_at != null` | — | `409` |
| Repeat action | повторный PATCH | новое событие, статус по последнему | — |
| Export | `--format jsonl` | JSONL со всеми решениями | — |

</frozen-after-approval>

## Code Map

- `estimate/app/domain/item.py` -- `ItemStatus.rejected`, `correction_link`, `ReviewDecision.reject`.
- `estimate/app/domain/feedback.py` -- append-only `MatchFeedback`.
- `estimate/app/infrastructure/models.py` -- `correction_link`, `MatchFeedbackRow`.
- `estimate/migrations/versions/0008_match_feedback.py` -- поля и таблица.
- `estimate/app/application/review_item.py` -- confirm/correct/reject, link resolution, feedback.
- `estimate/app/application/calculate_cost.py` -- причина `rejected`.
- `estimate/app/infrastructure/repository.py` -- сохранение feedback, маппинг.
- `estimate/app/infrastructure/rate_catalog_client.py` -- lookup по code.
- `estimate/app/transport/{schemas,routes}.py` -- контракт и `X-User-Id`.
- `estimate/app/tools/export_match_feedback.py` -- JSONL.
- `rate-catalog/app/application/lookup_rate.py`, `wiring.py`, `transport/*`, `infrastructure/repository.py` -- lookup.
- `ui/src/components/EstimateTable.vue`, `views/EstimateView.ts`, `api/client.ts` -- действия.
- `ui/src/views/PriceCatalogView.vue` -- copy-link.

## Tasks & Acceptance

**Execution:**
- [ ] `estimate` -- модель, миграция, review use case, расчёт, API -- обратная связь.
- [ ] `rate-catalog` -- lookup по code/period -- разрешение ссылки.
- [ ] `ui` -- действия в таблице и copy-link -- ручная привязка.
- [ ] `estimate` -- export JSONL -- данные для анализа/обучения.
- [ ] тесты estimate/rate-catalog/ui -- верификация.

**Acceptance Criteria:**
- Given `reject`, when PATCH, then статус `rejected`, событие `reject`, позиция не в итоге.
- Given `correct` со ссылкой, when материал есть за период, then `corrected`, событие `correct`, ссылка сохранена.
- Given ссылка на другой период/несуществующий материал, when PATCH, then `422`.
- Given рассчитанная версия, when PATCH, then `409`.
- Given любое review-действие, when PATCH, then ровно одно append-only событие.
- Given export, when `--format jsonl`, then файл содержит confirm/correct/reject с метаданными.

## Implementation Notes

- `reject` сохраняет исходную расценку для аудита; расчёт её игнорирует.
- Feedback пишется в той же транзакции, что и изменение позиции.
- Ссылка без периода использует период сметы; ссылка с периодом обязана совпасть.
- `attributes` в feedback пока не заполняются: это задел под отложенный matching-эпизод.
- Реализовано: миграция `0008`, `match_feedback`, `correction_link`, `rejected`, JSONL-экспорт, lookup `GET /rates/lookup`, UI-кнопки и copy-link.

## Spec Change Log

## Review Triage Log

## Design Notes

- Append-only журнал отделён от текущего состояния позиции: история действий не теряется.
- Lookup в `rate-catalog` — один endpoint `GET /rates/lookup`; `estimate` не перебирает страницы `/prices`.

## Verification

**Commands:**
- `cd estimate && .venv/bin/ruff check . && .venv/bin/pytest -q` -- **351 passed, 2 skipped**.
- `cd rate-catalog && .venv/bin/ruff check . && POSTGRES_TEST_DSN=... .venv/bin/pytest -q` -- **141 passed, 1 skipped**.
- `cd ui && npm run typecheck && npm test` -- **154 passed**, typecheck green.
- `cd estimate && POSTGRES_DSN=.../estimate ESTIMATE_INTEGRATION=1 pytest -q tests/test_real_services.py` -- **2 passed**; миграция `0008` применена на реальном Postgres.
- Экспорт проверен тестом `test_export_match_feedback_includes_decisions`.
- `cd deploy && docker compose config -q` -- ожидается OK (не менялся).
