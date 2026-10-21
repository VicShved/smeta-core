---
title: 'Story 7.4 — хвосты Epic 7: восстановление UI, таймауты загрузки и утечка страницы PDF'
type: 'bugfix'
created: '2026-10-21'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-7-context.md'
  - '_bmad-output/implementation-artifacts/epic-7-retro-10-21-2026.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Ретро Epic 7 оставило девять дефектов: загрузка PDF переезжает между этапами, повтор позиций сбрасывает список, цена снапшота округляется в UI вопреки «как пришло от API», gateway обрывает загрузку на 10 с при клиентских 120 с, `pdf-extract` теряет нативную страницу PDF при исключении `get_textpage()`, а часть путей восстановления UI не покрыта тестами. Цель эпика обещает больше, чем UI делает.

**Approach:** Закрыть эти хвосты точечными правками: ключ по этапу у загрузчика, повтор текущей страницы, точная цена снапшота, таймауты gateway из env с длинным значением только для файловых маршрутов, закрытие страницы в `finally`, недостающие тесты. Цель Epic 7 привести к фактическому поведению: UI показывает стадии и запускает загрузку с извлечением.

## Boundaries & Constraints

**Always:** UI — проекция: не пересчитывает и не округляет значения из API; этап и смета — единственный контекст загрузки; длинный таймаут — только для `/v1/files*`, остальные маршруты остаются на 10 с; нативная страница PDF закрывается при любом исходе `get_textpage()`; ошибки — `application/problem+json`; access-токен — единственное, что хранится в браузере.

**Never:** новые эндпоинты и поля сверх необходимых; изменения в `mapping-service`, `rate-catalog`, `company-org`, `auth-users`; UI-валидация и UI-diff (1.1); пересчёт стоимости; миграции; шаг «UI запускает маппинг и расчёт» (решение: цель эпика переписана, Story 7.1c не заводится); браузерный e2e (item-12 — отдельно).

**Решения (2026-10-20):** цена снапшота отображается как пришла от API (без округления); цель Epic 7 переписывается под фактическое поведение; WIP в `company-org`/`auth-users` закоммичен отдельно; e2e откладывается отдельной историей.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Смена этапа после сбоя загрузки | сбой привязки в этапе A, переход в B, повтор | смета создаётся в этапе B, файл привязан к ней | — |
| Ошибка второй страницы позиций | первая страница загружена, вторая упала | повтор догружает вторую, первая остаётся | — |
| Цена снапшота | `1234.5678` | отображается `1234.5678`, а не `1234.57` | — |
| Лимит опроса исчерпан | 30 попыток без изменений | кнопка «Обновить» возобновляет запрос, подсказка исчезает | — |
| Ошибка извлечения | `POST /v1/files/{id}/extract` → 502 | текст ошибки в панели, без внутренних деталей | — |
| Загрузка PDF через gateway | POST `/v1/files`, ответ дольше 10 с | запрос не обрывается до файлового таймаута | — |
| Остальные маршруты | запрос дольше 10 с | — | `502` по прежнему таймауту |
| Исключение `get_textpage()` | страница PDF не читается | страница закрывается, ошибка `422 invalid-pdf` | — |

</frozen-after-approval>

## Code Map

- `ui/src/views/HierarchyView.ts` -- `key` по `stage_id` у `EstimateUpload`; сброс состояния и защита от поздних ответов при смене сметы/этапа; таймаут запроса извлечения 120 с (`run_extract`).
- `ui/src/views/EstimateView.ts` -- `retry_items` повторяет текущую страницу (append при уже загруженных позициях).
- `ui/src/components/EstimateTable.vue` -- цена снапшота без округления; удалить `format_money`.
- `ui/tests/{upload_status,estimate_view}.test.ts` -- смена этапа, повтор страницы, «Обновить» после лимита, ошибка извлечения (в т.ч. поздняя), правило `can_extract`.
- `ui/nginx.conf`, `ui/README.md` -- `proxy_read_timeout`/`proxy_send_timeout` 130 с для `/v1/`; цепочка «клиент 120 с → nginx → gateway 120 с».
- `ui/package.json`, `ui/.github/workflows/ci.yml` -- описание на 7.1a/7.1b/7.2; `npm ci` без fallback.
- `api-gateway/app/{application/routing,infrastructure/proxy,transport/routes,wiring}.py` -- таймаут запроса как параметр; env `PROXY_TIMEOUT_SECONDS` (10) и `PROXY_FILES_TIMEOUT_SECONDS` (120) с валидацией; длинный только для `POST /v1/files` и `POST /v1/files/{id}/extract`.
- `api-gateway/tests/test_gateway.py` -- выбор таймаута по пути, значения из env (включая нечисловые/0/отрицательные), собранный `app.main`.
- `pdf-extract/app/infrastructure/pdf.py` -- `get_textpage()` внутри `try`, `page.close()` в `finally` независимо от `textpage.close()`.
- `pdf-extract/tests/test_upload.py` -- тесты на fake-page: закрытие при исключении `get_textpage()` и при исключении `textpage.close()`.
- `_bmad-output/implementation-artifacts/epic-7-context.md` -- цель эпика под фактическое поведение; убрать «цена с 2 знаками».
- `_bmad-output/implementation-artifacts/epic-7-retro-10-21-2026.md` -- поправить запись про item-17/13.
- Не трогать: `estimate/*`, `mapping-service/*`, `rate-catalog/*`, `company-org/*`, `auth-users/*`, `calc-service/*`.

## Tasks & Acceptance

**Execution:**
- [x] `ui/src/views/HierarchyView.ts` -- ключ этапа у загрузчика, защита от поздних ответов, таймаут извлечения -- контекст не переезжает.
- [x] `ui/src/views/EstimateView.ts` -- повтор текущей страницы -- восстановление не теряет данные.
- [x] `ui/src/components/EstimateTable.vue` -- цена снапшота как в API -- арифметика сходится.
- [x] `ui/tests/*` -- смена этапа, повтор страницы, «Обновить», ошибка извлечения, `can_extract` -- регрессия.
- [x] `ui/nginx.conf`, `ui/README.md` -- таймауты nginx в цепочке файловых запросов -- загрузка не обрывается на 60 с.
- [x] `api-gateway/*` -- таймауты из env и по маршруту -- загрузка не обрывается.
- [x] `pdf-extract/*` -- закрытие страницы в `finally` и тесты -- нет утечки.
- [x] `ui/package.json`, `ui/.github/workflows/ci.yml` -- манифест и CI -- нет дрейфа.
- [x] `core/.../epic-7-context.md`, ретро -- цель эпика и запись про item-17 -- документы соответствуют коду.
- [x] `ui` -- typecheck/build/test; `api-gateway` и `pdf-extract` -- ruff/pytest; `deploy` -- compose config -- верификация.

**Acceptance Criteria:**
- Given сбой привязки PDF в этапе A, when пользователь переходит в этап B и повторяет, then смета и файл создаются в этапе B.
- Given первая страница позиций загружена, when вторая упала и пользователь повторил, then первая остаётся и догружается вторая.
- Given цена снапшота `1234.5678`, when открыт экран сметы, then отображается `1234.5678`.
- Given лимит опроса исчерпан, when нажата «Обновить», then запрос статуса возобновляется и подсказка исчезает.
- Given загрузка PDF идёт дольше 10 с, when запрос идёт через gateway, then он не обрывается раньше файлового таймаута; остальные маршруты сохраняют 10 с.
- Given `get_textpage()` бросает исключение, when извлекается текст, then страница PDF закрыта, пользователь видит `422` без деталей.

## Implementation Notes

- UI: `HierarchyView.ts` пересоздаёт `EstimateUpload` по `stage_id`, повторяет загрузку позиций текущей страницей (`EstimateView.ts`), распознаёт поздние ответы статуса/извлечения по смете и этапу; `EstimateTable.vue` показывает `rate_amount_snapshot` как есть.
- Таймауты: клиент 120 с для загрузки/извлечения (`FILES_REQUEST_TIMEOUT_MS`), nginx 130 с для `/v1/`, gateway 120 с для `POST /v1/files` и `POST /v1/files/{id}/extract`, остальные — 10 с; env читаются с fallback на дефолт при нечисловых/неположительных значениях.
- `pdf-extract`: `get_textpage()` внутри `try`, `textpage.close()` и `page.close()` в раздельных `try`/`finally`.
- Тесты закрывают смену этапа на том же `File`, повтор страницы, цену снапшота, лимит опроса, ошибку извлечения, `can_extract`, таймауты по маршрутам/env и закрытие PDF-страницы.

## Spec Change Log

## Review Triage Log

## Design Notes

- Таймауты gateway: `PROXY_TIMEOUT_SECONDS` (default 10) — по умолчанию; `PROXY_FILES_TIMEOUT_SECONDS` (default 120) — для методов, меняющих файлы (`POST /v1/files`, `POST /v1/files/{id}/extract`). Значения читаются в `wiring.py` и передаются в прокси.
- Клиент UI: загрузка и извлечение идут с таймаутом 120 с, остальные запросы — 15 с.
- Цепочка nginx: для `location /v1/` `proxy_read_timeout` и `proxy_send_timeout` — 130 с, чтобы nginx не обрывал файловые запросы раньше клиентского/gateway-бюджета 120 с.
- Цена снапшота: строка из API отображается как есть; `cost` и `total_cost` уже приходят с 2 знаками и не трогаются.
- Цель Epic 7: «UI показывает стадии обработки, запускает загрузку и извлечение; маппинг и расчёт доступны через API» — без обещания «довести до конца из интерфейса».

## Verification

**Commands (выполнено 2026-10-21):**
- `cd ui && npm run typecheck && npm run build && npm run test` → чисто, `104 passed`, сборка `dist/`
- `cd api-gateway && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `60 passed`
- `cd pdf-extract && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `83 passed`
- `cd deploy && docker compose config -q` → exit 0

**Живой стенд (образы `ui`, `api-gateway`, `pdf-extract` пересобраны):**
- в контейнере gateway `PROXY_TIMEOUT_SECONDS=10`, `PROXY_FILES_TIMEOUT_SECONDS=120`; в nginx UI `proxy_read_timeout 130s`, `proxy_send_timeout 130s` — цепочка клиент 120 с → nginx 130 с → gateway 120 с замкнута;
- через UI-origin (`:8080`): загрузка PDF `201`, чтение статуса файла `200`, извлечение `200` (1 страница, 31 символ).
