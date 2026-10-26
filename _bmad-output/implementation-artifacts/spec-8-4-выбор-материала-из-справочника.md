---
title: 'Story 8.4 — Выбор материала из справочника цен'
type: 'feature'
created: '2026-10-26'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/spec-8-3-верно-по-умолчанию-и-удобная-таблица.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** «Исправить» требует вставить ссылку/код материала, найденную на другом экране. Сметчик не видит варианты и цену, легко ошибается периодом; ручная привязка медленная.

**Approach:** Кнопка «Исправить» открывает модальный справочник цен за период сметы. Период read-only, поиск — по коду ИЛИ наименованию (`q`), строка показывает код, название, ЕИ и цену. Выбор отправляет `PATCH correct` с `rate_catalog_id`; отмена ничего не меняет. Backend проверяет период выбранной расценки.

## Boundaries & Constraints

**Always:** период picker = `estimate.catalog_period`; поиск в rate-catalog API с `q`; выбор только через `rate_catalog_id`; `correct` валидирует период; ошибки — текстом, модалка остаётся открытой; UI не пересчитывает цену.

**Never:** не менять алгоритм matching; не давать менять период в модалке; не загружать весь справочник в браузер; не добавлять свободный ввод ссылки как основной путь; не трогать `/prices` copy-link (остаётся для API/совместимости).

**Решения (2026-10-26):** Показывать первые 50 совпадений и общее число. Начальный запрос — наименование позиции. Escape/клик по фону/«×» закрывают без сохранения. Период без `zone_code` не поддерживается picker'ом (кнопка показывает ошибку периода).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Open picker | позиция и период сметы | модалка и первые 50 материалов периода; поиск пустой | — |
| Search by code | `q=C-1` | строки, где код содержит `C-1` | — |
| Search by name | `q=бетон` | строки, где наименование содержит `бетон` | — |
| Empty result | нет совпадений | «Ничего не найдено» | — |
| Select | клик по строке | `correct` с `rate_catalog_id`, модалка закрыта, таблица обновлена | `422` — ошибка в модалке |
| Cancel | Escape/фон/«×» | закрытие, позиция не меняется | — |
| No period | `catalog_period=null` | сообщение, поиск не выполняется | — |
| Period mismatch | выбранная расценка другого периода | позиция не меняется | `422` |

</frozen-after-approval>

## Code Map

- `rate-catalog/app/application/prices.py`, `domain/ports.py`, `infrastructure/repository.py` -- `q` по коду ИЛИ наименованию.
- `rate-catalog/app/transport/routes.py`, `wiring.py` -- параметр `q` в `/prices`.
- `estimate/app/domain/rate.py`, `infrastructure/rate_catalog_client.py` -- период в `RateRef`.
- `estimate/app/application/review_item.py` -- проверка периода `rate_catalog_id`.
- `ui/src/components/PricePickerDialog.vue` -- модалка, поиск, выбор.
- `ui/src/views/EstimateView.ts` -- открытие/закрытие, `correct`.
- `ui/src/api/client.ts` -- `q` и типы.

## Tasks & Acceptance

**Execution:**
- [x] `rate-catalog` -- `q`-поиск -- выбор материала.
- [x] `estimate` -- период `RateRef` и валидация `correct` -- защита от чужого периода.
- [x] `ui` -- `PricePickerDialog` и интеграция -- замена ссылки.
- [x] тесты rate-catalog/estimate/ui -- поиск, период, модалка.

**Acceptance Criteria:**
- Given «Исправить», when модалка открыта, then период сметы показан read-only.
- Given `q`, when поиск, then совпадения по коду или наименованию.
- Given выбор строки, when `PATCH correct`, then позиция `corrected` с выбранным `rate_catalog_id`.
- Given расценка другого периода, when `PATCH correct`, then `422`, позиция не меняется.
- Given отмена, when закрытие, then запросов на изменение нет.

## Implementation Notes

- Поиск дебаунсится на 300 мс; поздние ответы отбрасываются по поколению запроса. Начальный запрос идёт без `q`, чтобы справочник не выглядел пустым.
- Модалка не рендерится, пока `open=false`.
- `index.html` отдаётся с `expires -1`, чтобы после деплоя не открывалась старая сборка без модалки.
- Стили `PricePickerDialog` намеренно не `scoped`: компонент рисуется render-функцией, и scoped-атрибут не попадает на vnodes. Без этого overlay оставался `static` внизу страницы, и модалка была не видна.
- `RateRef` period-поля опциональны, чтобы старые fake-порты и тесты не ломались; при заполненных полях проверка обязательна.

## Spec Change Log

## Review Triage Log

## Design Notes

- Справочник остаётся серверным: браузер получает только страницу результатов.
- `rate_catalog_id` — устойчивый ключ выбора; код/ссылка остаются API-совместимым путём.

## Verification

**Commands:**
- `cd rate-catalog && .venv/bin/ruff check . && .venv/bin/pytest -q` -- **142 passed, 1 skipped**.
- `cd estimate && .venv/bin/ruff check . && .venv/bin/pytest -q` -- **355 passed, 2 skipped**.
- `cd ui && npm run typecheck && npm test` -- **158 passed**, typecheck green.
- Живой `GET /rate-catalog/prices?...&q=бетон` на реальном справочнике вернул **52156** совпадений.
- Проверено в Chromium на живой смете: клик «Исправить» открывает overlay `position:fixed` на весь viewport, 50 строк периода; UI/estimate/rate-catalog пересобраны и подняты.
