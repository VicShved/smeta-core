---
title: 'Story 8.3 — «Верно» по умолчанию и удобная проверка таблицы'
type: 'feature'
created: '2026-10-26'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/spec-8-2-обратная-связь-и-привязка-материала.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** После `match-rates` все найденные позиции остаются `proposed`, хотя сметчик в большинстве случаев согласен с сопоставлением. Проверка требует ручного подтверждения каждой строки; кнопки действий уезжают при горизонтальной прокрутке, а статус занимает отдельную колонку.

**Approach:** Найденную расценку сохранять как `confirmed` с `reviewed_at=null` («верно» по умолчанию). Ручное действие (`confirm/correct/reject`) заполняет `reviewed_at`; повторное сопоставление перезаписывает авто-подтверждённые позиции и пропускает ручные. В UI — чекбокс «Верно» вместо кнопок «Верно/Неверно», закреплённые первая и последняя колонки.

## Boundaries & Constraints

**Always:** `confirmed` после match; `reviewed_at` только у ручных решений; ручные решения приоритетнее matching; `rejected` не входит в расчёт; повторный идентичный `PATCH` идемпотентен; UI не пересчитывает стоимость.

**Never:** не менять алгоритм/порог/rerank matching; не запускать обучение; не менять контракт `match-rates`; не добавлять отдельный экран проверки.

**Решения (2026-10-26):** Авто-подтверждение распространяется только на найденные позиции. Ненайденные остаются `proposed` без расценки. Чекбокс доступен только при наличии `rate_catalog_id`. Первая колонка — наименование, последняя — действия.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Match found | `match-rates` нашёл расценку | `confirmed`, `reviewed_at=null` | — |
| Match not found | `match-rates` не нашёл | `proposed`, без расценки | — |
| Manual reject | чекбокс снят | `rejected`, `reviewed_at` заполнен | ошибка — текст, чекбокс возвращается |
| Manual confirm | чекбокс возвращён | `confirmed`, `reviewed_at` заполнен | — |
| Re-match | авто-подтверждённая | пересопоставляется | — |
| Re-match | ручное решение | не перезаписывается, `skipped` | — |
| No rate | `rate_catalog_id=null` | чекбокс disabled | — |
| Calculated version | `calculated_at != null` | действия недоступны | `409` API |

</frozen-after-approval>

## Code Map

- `estimate/app/domain/item.py` -- `reviewed_at`.
- `estimate/app/infrastructure/models.py`, `repository.py` -- колонка и маппинг.
- `estimate/migrations/versions/0009_reviewed_at.py`, `0010_auto_confirm_existing.py` -- колонка и backfill старых matched-позиций.
- `estimate/app/application/match_rates.py` -- авто-`confirmed`, `_needs_matching`.
- `estimate/app/application/review_item.py` -- `reviewed_at` при ручном решении.
- `estimate/app/transport/{schemas,routes}.py` -- `reviewed_at` в ответе.
- `ui/src/components/EstimateTable.vue` -- чекбокс, sticky-колонки.
- `ui/src/views/EstimateView.ts` -- `confirm/reject`, счётчик пропущенных.

## Tasks & Acceptance

**Execution:**
- [x] `estimate` -- миграции/домен/репозиторий -- `reviewed_at` и backfill `proposed` с расценкой в `confirmed`.
- [x] `estimate` -- matching/review -- авто-подтверждение и защита ручных решений.
- [x] `ui` -- чекбокс и sticky-колонки -- проверка без прокрутки.
- [x] тесты estimate/ui -- сценарии match/rematch/review.

**Acceptance Criteria:**
- Given `match-rates` нашёл расценку, when ответ, then статус `confirmed` и `reviewed_at=null`.
- Given ручной `reject`, when повторный `match-rates`, then позиция остаётся `rejected`, попадает в `skipped`.
- Given авто-подтверждённая позиция, when повторный `match-rates`, then она пересопоставляется.
- Given `rate_catalog_id=null`, when таблица, then чекбокс disabled.
- Given длинная строка, when прокрутка, then наименование и действия остаются видимыми.

## Implementation Notes

- Идентичный повторный `PATCH` не меняет `updated_at` и не пишет feedback.
- `reviewed_at` выставляется только при фактическом изменении решения.
- `rejected` сохраняет исходную расценку для аудита.
- Миграция `0010` одноразово переводит старые `proposed` с `rate_catalog_id` в `confirmed`, чтобы после деплоя чекбоксы отражали уже выполненные сопоставления.

## Spec Change Log

## Review Triage Log

## Design Notes

- Авто-подтверждение — состояние без ручного решения, поэтому `reviewed_at` (а не отдельный флаг) отделяет его от действий сметчика.
- Sticky-колонки реализованы CSS `position: sticky`; таблица сохраняет горизонтальную прокрутку для остальных полей.

## Verification

**Commands:**
- `cd estimate && .venv/bin/ruff check . && .venv/bin/pytest -q` -- **355 passed, 2 skipped**.
- `cd ui && npm run typecheck && npm test` -- **158 passed**, typecheck green.
- Миграции `0009`/`0010` применены на живом Postgres: **791** старых matched-позиций стали `confirmed`, `rejected` не затронута; `GET items` отдаёт `reviewed_at=null`.
