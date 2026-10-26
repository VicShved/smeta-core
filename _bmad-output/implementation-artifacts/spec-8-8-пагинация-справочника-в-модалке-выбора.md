---
title: 'Story 8.8 — Пагинация справочника в модалке выбора'
type: 'feature'
created: '2026-10-26'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/spec-8-4-выбор-материала-из-справочника.md'
---

## Intent

**Problem:** Модалка выбора материала показывала только первые 50 строк справочника. Найти материал за пределами первой страницы без точного поиска было нельзя.

**Approach:** Добавить в модалку страницы по 50 строк: «Назад»/«Вперёд», диапазон «Показаны N–M из T» и номер страницы. Поиск сбрасывает пагинацию на первую страницу.

## Boundaries & Constraints

**Always:** страница запрашивается с `limit=50&offset=...`; поиск `q` сохраняется при листании; пустой выбор материала остаётся отдельной кнопкой.

**Never:** не загружать весь справочник в браузер; не менять backend-контракт `/prices`.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Open | период сметы | первая страница, 50 строк | — |
| Next | есть ещё строки | `offset += 50`, новые строки | — |
| Prev | offset > 0 | `offset -= 50` | — |
| Search | ввод `q` | offset сброшен, первая страница | — |
| Last page | строк меньше 50 | «Вперёд» disabled | — |

## Code Map

- `ui/src/components/PricePickerDialog.vue` — offset, кнопки страниц, счётчик.
- `ui/tests/estimate_view.test.ts` — проверка перехода и сброса.

## Tasks & Acceptance

- [x] пагинация `Назад`/`Вперёд` по 50 строк.
- [x] диапазон и номер страницы.
- [x] сброс offset при поиске.
- [x] тест перехода между страницами.

**Acceptance Criteria:**
- Given 120 строк, when открыть модалку, then «Показаны 1–50 из 120».
- Given первая страница, when «Вперёд», then запрос `offset=50` и «Показаны 51–100».
- Given вторая страница, when «Назад», then `offset=0`.

## Verification

- `cd ui && npm run typecheck && npm test` — **162 passed**.
