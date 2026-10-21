---
title: 'Story 7.8 — браузерный e2e ключевого пути UI'
type: 'feature'
created: '2026-10-21'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-7-retro-10-21-2026.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** UI проверяется только в jsdom (`vitest`), поэтому не ловятся поломки реальной сборки: nginx-прокси `/v1`, JWT через api-gateway, multipart-загрузка, опрос `processing-status`, переходы роутера. Пункт `epic-7-retro-item-12` требует браузерного прогона ключевого пути.

**Approach:** Python-стек `pytest` + `pytest-playwright` в `deploy/e2e`. Данные готовятся скриптом сидинга через публичные API (регистрация → договор → этап → импорт тарифов → смета → PDF → извлечение → маппинг → позиции → подтверждение → расчёт). Браузерный тест входит реальной формой, открывает дерево, проверяет рассчитанную смету и проходит загрузку + извлечение второго PDF через UI. Негативные кейсы: неверный пароль и не-PDF.

**Важный факт:** UI сейчас не запускает маппинг и расчёт — браузерный путь заканчивается извлечением (`ui/src/views/HierarchyView.ts`, `ui/src/components/EstimateUpload.vue`). Маппинг и расчёт в e2e запускаются через публичный API gateway внутри того же теста, а их результат проверяется в браузере (итог и строки сметы). Кнопки запуска маппинга/расчёта в UI — вне объёма этой истории.

## Boundaries & Constraints

**Always:** тест ходит только через опубликованные точки входа (`http://localhost:8080`, путь `/v1/...` через api-gateway); сидинг использует публичные API, не прямые SQL-вставки; каждый прогон создаёт уникального пользователя (новый email) и свои данные; e2e-код живёт в `deploy/e2e` и не попадает в образы сервисов; артефакты Playwright (trace, screenshot) сохраняются при падении.

**Never:** не менять прод-код сервисов и UI ради тестов; не подменять сервисы заглушками (маппинг детерминирован в `mapping-service`, `llm-gateway` в пути не участвует); не добавлять `llm-gateway` и `calc-service` в стенд; не делать e2e блокирующим чеком на первом этапе; не трогать `llm-gateway`, `calc-service`, схемы и миграции.

**Решения (2026-10-21, владелец):**
- Путь: вход → дерево → загрузка PDF → извлечение → маппинг → расчёт, happy path + неверный пароль + не-PDF.
- Сидинг — через публичные API.
- Стенд в CI — облегчённый: `postgres`, `minio`, `auth-users`, `company-org`, `rate-catalog`, `pdf-extract`, `mapping-service`, `estimate`, `api-gateway`, `ui`.
- Стек — Python `pytest-playwright`, размещение — `deploy/e2e`.
- Вход — реальная форма; PDF — синтетический (reportlab).
- CI — неблокирующий (`continue-on-error: true`), позже перевести в блокирующий.
- Сначала спека, затем реализация.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Вход | email/пароль сидированного пользователя | переход на `/`, заголовок «Сметы», дерево с договором | при ошибке — сообщение `role=alert` |
| Неверный пароль | валидный email, неверный пароль | остаёмся на `/login`, «Неверный email или пароль.» | — |
| Дерево | есть договор и этап | видны названия договора и этапа, список смет | — |
| Смета | рассчитанная версия | «Итог» = `250.00`, строка «Бетон М200», стоимость `250.00` | — |
| Загрузка | синтетический PDF через UI | смета создана, панель «Обработка сметы» | ошибка API → `role=alert` |
| Извлечение | нажатие «Извлечь текст» | `[data-stage="extracted"]` → «выполнено» | таймаут 60 с → падение с trace |
| Не-PDF | файл `.txt` | клиентская ошибка «Загрузите PDF-файл.» | запрос не уходит |
| Повторный прогон | новые email/имена | данные не конфликтуют, тест идемпотентен | — |

</frozen-after-approval>

## Code Map

- `deploy/e2e/pyproject.toml` (новый) -- зависимости: `pytest`, `pytest-playwright`, `httpx`, `reportlab`, `openpyxl`; Python 3.12.
- `deploy/e2e/smeta_e2e/seed.py` (новый) -- сидинг через API: регистрация, договор, этап, XLSX-тарифы, PDF, смета, source, извлечение, маппинг, позиции, подтверждение, расчёт.
- `deploy/e2e/smeta_e2e/fixtures.py` (новый) -- генерация XLSX/PDF, HTTP-клиент с JWT, уникальные имена.
- `deploy/e2e/tests/test_key_path.py` (новый) -- браузерные сценарии.
- `deploy/e2e/pytest.ini` / `playwright` конфиг -- `trace: on-first-retry`, `screenshot: only-on-failure`, `video: off`, `base_url`.
- `deploy/.github/workflows/ci.yml` -- job `e2e` с compose-стендом, установкой Chromium и артефактами.
- `deploy/README.md` -- как запустить e2e локально и в CI.
- Не трогать: сервисы, UI, миграции, `docker-compose.yml` (кроме, при необходимости, healthcheck-ожиданий в CI-скрипте).

## Tasks & Acceptance

**Execution:**
- [x] `deploy/e2e` -- каркас pytest-playwright + утилиты (HTTP, XLSX, PDF, уникальные имена) -- основа тестов.
- [x] `deploy/e2e/smeta_e2e/seed.py` -- полный сидинг через публичные API -- данные для сметы.
- [x] `deploy/e2e/tests/test_key_path.py` -- happy path + неверный пароль + не-PDF -- браузерная проверка.
- [x] `deploy/.github/workflows/ci.yml` -- job `e2e` (стенд, Chromium, артефакты, неблокирующий) -- автоматизация.
- [x] `deploy/README.md` -- запуск e2e -- документация.
- [x] `deploy/e2e` -- прогон против локального compose -- верификация.

**Acceptance Criteria:**
- Given поднятый облегчённый стенд, when e2e прогоняется, then вход, дерево, смета с итогом `250.00`, загрузка и извлечение PDF проходят в реальном Chromium.
- Given неверный пароль, when форма отправлена, then показывается «Неверный email или пароль.» и пользователь остаётся на `/login`.
- Given не-PDF файл, when нажата загрузка, then показывается «Загрузите PDF-файл.» и запрос не отправляется.
- Given падение теста в CI, when job завершается, then trace и screenshot доступны артефактом, а job не блокирует merge.
- Given повторный прогон, when сидинг выполняется, then создаются новые пользователь и данные без конфликтов.

## Implementation Notes

- `deploy/e2e`: `pytest` + `pytest-playwright`, `httpx`, `reportlab`, `openpyxl`; `[tool.uv] package = false`, `pythonpath = ["."]`.
- `smeta_e2e/fixtures.py`: ASCII-текст PDF (Helvetica без кириллицы), тариф `C-1 / Beton M200 / m3 / 100.0000`, уникальный суффикс прогона.
- `smeta_e2e/seed.py`: регистрация → договор → этап → XLSX-импорт (становится активной версией) → PDF → extract → proposals → смета → source → items → confirm → calculate; проверяет итог `250.00`.
- `tests/test_key_path.py`: вход формой, дерево, итог/строки сметы, загрузка второго PDF и ожидание `[data-stage="extracted"]` = «выполнено»; негативные — неверный пароль и не-PDF (дополнительно проверяется, что `POST /v1/files` не ушёл).
- CI `deploy`: job `e2e` с `continue-on-error`, `timeout-minutes: 40`, поднимает 10 контейнеров, ждёт `/health` UI, ставит Chromium, артефакты trace/screenshot при падении.

## Spec Change Log

## Review Triage Log

## Design Notes

- **Стенд:** `docker compose up -d --build postgres minio auth-users company-org rate-catalog pdf-extract mapping-service estimate api-gateway ui`. `llm-gateway` и `calc-service` не нужны: маппинг детерминированный (`mapping-service`), расчёт — в `estimate`.
- **Почему не mock LLM:** `llm-gateway` отсутствует в пути UI и не вызывается `mapping-service`/`estimate`; подмена не требуется.
- **Данные для расчёта:** тариф `C-1 / Бетон М200 / м3 / 100.0000` (XLSX через `POST /v1/rate-catalog/imports`, первый импорт становится активной версией); PDF с текстом «Бетон М200 2.5 м3»; позиция `qty=2.5`, `rate_catalog_id` из ответа маппинга, `confidence=1.0`; подтверждение `PATCH .../items/{id}` `{"decision":"confirm"}`; итог `2.5 × 100.00 = 250.00`.
- **Уникальность:** email вида `e2e+<uuid>@example.test`, имена договора/этапа/сметы с суффиксом прогона.
- **Локаторы:** `get_by_label("Email")`, `get_by_label("Пароль")`, `get_by_role("button", name="Войти")`, `get_by_role("button", name="Открыть смету «...»")` (aria-label), `[data-stage="extracted"]`, `.estimate-summary__value--total`, `.estimate-table__cost`; кнопки «Загрузить PDF-смету» скоупить контейнерами `.tree-node__upload` / `.estimate-upload`.
- **Ожидание:** `expect(...).to_have_text`/`to_contain_text` с таймаутом 60 с для извлечения; статус опрашивается UI каждые 3 с.
- **CI:** отдельный job в `deploy`; `.env` с `JWT_SECRET` создаётся в job; ожидание `http://localhost:8080/health`; `playwright install --with-deps chromium`; `continue-on-error: true`; `actions/upload-artifact` при `failure()`; `timeout-minutes: 40`.
- **Риски:** полная сборка 9 образов в CI — долго; первый прогон может быть нестабильным (тогда job остаётся неблокирующим); при недоступности registry сборка упадёт — это сигнал, а не регрессия e2e.

## Verification

**Результат (2026-10-21):** `uv run pytest -q` против локального compose — `3 passed`; повторный прогон — `3 passed` (идемпотентность); YAML CI валиден; `docker compose config -q` — OK.


**Commands:**
- `cd deploy && docker compose up -d --build postgres minio auth-users company-org rate-catalog pdf-extract mapping-service estimate api-gateway ui` -- стенд.
- `cd deploy/e2e && uv sync && uv run playwright install chromium` -- зависимости и браузер.
- `cd deploy/e2e && uv run pytest -q` -- expected: все сценарии зелёные.
- `cd deploy && docker compose config -q` -- expected: OK.
- `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))"` -- expected: YAML валиден.
