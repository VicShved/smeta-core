---
title: 'Story 5.3 — API ручной проверки маппинга'
type: 'feature'
created: '2026-10-20'
status: 'done'
route: 'dispatch'
baseline_commit: '4e4730f'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-5-context.md'
  - '_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md'
  - '_bmad-output/implementation-artifacts/spec-6-1-создание-сметы-и-версий-ровно-одна-активная.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 5.2 возвращает предложения маппинга, но в `estimate` нет API, чтобы сохранить их в позиции версии и подтвердить/исправить: сметчик не может закрыть проверку, а расчёт 6.3 не получает проверенных данных.

**Approach:** Добавить в `estimate` импорт позиций из предложений 5.2 в активную версию, выдачу позиций с выделением `confidence < 0.70` и подтверждение/исправление расценки и ЕИ. Выбранные значения и статус сохраняются в `estimate_item`; снапшоты и расчёт остаются 6.3.

## Boundaries & Constraints

**Always:** `company_id` из `X-Company-Id`; смета, версия и позиция проверяются на принадлежность компании (`404`); импорт и правки — только в активную версию; статусы `proposed|confirmed|corrected`; `confidence` в `[0,1]`, `needs_review = confidence < 0.70`; выбранная при исправлении расценка проверяется по API rate-catalog (AD-014); ошибки — `application/problem+json`; Decimal — строкой; лимит 1000 позиций; слои AD-011; одна транзакция на импорт/правку.

**Never:** UI; LLM и пересчёт confidence/маппинга (5.2); снапшоты и расчёт стоимости (6.3); редактирование позиций неактивной версии; изменение mapping-service/rate-catalog/company-org; прямое чтение чужой БД; очередь; миграции БД.

**Решения (2026-10-20):** повторный импорт в непустую версию — `409` (для нового PDF создаётся новая версия); импорт и confirm/correct разрешены только для активной версии, исторические версии read-only.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Импорт | пустая активная версия, валидные items | `201`, позиции `proposed`, `rate_catalog_id`, `confidence` | — |
| Пустой импорт | `items=[]` | — | `422` |
| Слишком много | > 1000 позиций | — | `422` |
| Версия не пуста | в версии уже есть позиции | — | `409` |
| Неактивная версия | импорт/правка | — | `409` |
| Просмотр | позиции есть | `200` `items`, `total`, `needs_review_count` | — |
| Низкая уверенность | `confidence=0.69` | `needs_review=true` | — |
| Подтверждение | `proposed`, `rate_catalog_id` задан | `200`, `status=confirmed` | — |
| Подтверждение без расценки | `rate_catalog_id=null` | — | `422` |
| Исправление | `rate_catalog_id` из активного справочника | `200`, `status=corrected`, новая ЕИ | — |
| Неизвестная расценка | `rate_catalog_id` не найден | — | `422` |
| rate-catalog недоступен | сеть/5xx | — | `502` |
| Чужая смета/версия/позиция | другой `company_id` | — | `404` |
| Нет identity | без `X-Company-Id` | — | `401` |
| Невалидное тело | пустое имя, ЕИ, confidence вне `[0,1]` | — | `422` |

</frozen-after-approval>

## Code Map

- `estimate/app/domain/rate.py` -- новый: `RateRef` (id, code, name, unit, rate_amount) и порт чтения активных расценок.
- `estimate/app/domain/errors.py` -- `VersionNotActive` (409), `ItemsAlreadyImported` (409), `RateNotFound` (422), `RateCatalogUnavailable` (502).
- `estimate/app/domain/ports.py` -- `EstimateRepository`: `add_items`, `count_items`, `get_item`, `update_item` (все company-scoped); `RateCatalogPort.find_rate`.
- `estimate/app/application/import_items.py` -- проверка активной версии и пустоты, построение `EstimateItem` из предложений, bulk-вставка.
- `estimate/app/application/list_items.py` -- выборка с `limit`/`offset`, фильтр `needs_review`, `total`, `needs_review_count`.
- `estimate/app/application/review_item.py` -- `confirm`/`correct`; для `correct` — валидация расценки через rate-catalog.
- `estimate/app/infrastructure/rate_catalog_client.py` -- `GET /v1/rate-catalog/versions/current/rates` с `X-Company-Id`; 404/нет ставки → `RateNotFound`, сеть/5xx → `RateCatalogUnavailable`.
- `estimate/app/infrastructure/repository.py` -- bulk add, count, get/update item с JOIN к `estimate.company_id`.
- `estimate/app/transport/{schemas,routes,problem}.py` -- `POST/GET .../versions/{version_id}/items`, `PATCH .../items/{item_id}`; problem+json.
- `estimate/app/wiring.py` -- `RATE_CATALOG_URL`, клиент, методы операций.
- `estimate/tests/{test_import_items,test_list_items,test_review_items,test_rate_catalog_client}.py` -- матрица, tenancy, rate-catalog на MockTransport.
- `estimate/README.md` -- контракт импорта, просмотра, confirm/correct и лимиты.
- `deploy/docker-compose.yml`, `.env.example`, `README.md` -- `RATE_CATALOG_URL` для estimate, `depends_on` rate-catalog.
- Не трогать: `api-gateway/*` (префикс `/v1/estimates` уже есть), `mapping-service/*`, `rate-catalog/*`, `company-org/*`, `llm-gateway/*`, `calc-service/*`, `ui/*`, миграции.

## Tasks & Acceptance

**Execution:**
- [x] `estimate/app/domain/{rate,errors,ports}.py` -- типы, ошибки, порты -- контракт проверки.
- [x] `estimate/app/infrastructure/{repository,rate_catalog_client}.py` -- bulk/update/чтение item и клиент расценок -- tenancy и валидация.
- [x] `estimate/app/application/*.py` -- импорт, список, confirm/correct -- бизнес-правила.
- [x] `estimate/app/transport/*`, `wiring.py` -- HTTP-контракт -- 201/200/401/404/409/422/502.
- [x] `estimate/tests/*` -- матрица, tenancy, rate-catalog -- регрессия.
- [x] `estimate/README.md`, `deploy/*` -- документация и env -- запуск в стеке.
- [x] `estimate` -- ruff/pytest; `deploy` -- compose config -- верификация.

**Acceptance Criteria:**
- Given пустая активная версия и предложения 5.2, when `POST .../items`, then `201`, позиции сохранены со `status=proposed`, расценкой, ЕИ и `confidence`.
- Given позиции версии, when `GET .../items`, then `200` с `items`, `total`, `needs_review_count`; позиции с `confidence < 0.70` помечены `needs_review=true`.
- Given позиция `proposed` с расценкой, when `PATCH decision=confirm`, then `status=confirmed`; при `rate_catalog_id=null` → `422`.
- Given позиция и расценка активного справочника, when `PATCH decision=correct`, then `status=corrected` и сохранены расценка/ЕИ; неизвестная расценка → `422`, rate-catalog недоступен → `502`.
- Given неактивная версия, чужой объект или непустая версия при импорте, when запрос, then `409`/`404`; без `X-Company-Id` → `401`.

## Implementation Notes

## Spec Change Log

## Review Triage Log

- **2026-10-20 · blind-hunter · diff не содержит тело спеки 5.3** — verdict: false. evidence: `git ls-files --others` вернул Cyrillic-путь в кавычках, `git diff --no-index` не открыл файл; это дефект генерации review-артефакта, не кода. Спека доступна как claims_file и на диске.
- **2026-10-20 · blind-hunter · рассинхрон статусов: spec in-review, sprint in-progress, docker build не завершён** — verdict: low. evidence: `sprint-status` обновляется на шаге present; сборка прервана пользователем и отмечена в Verification. route: patch → синхронизировать sprint-status на `review`.
- **2026-10-20 · blind-hunter + edge-case-hunter · confidence с >4 знаками округляется в `Numeric(5,4)` и перескакивает порог 0.70** — verdict: medium. evidence: `0.69995` сохранится как `0.7000`; POST вернёт `needs_review=true`, GET — `false`. route: patch → валидировать/квантовать scale ≤4 и тест на 0.70/0.69995.
- **2026-10-20 · blind-hunter + edge-case-hunter · qty с >4 знаками молча округляется** — verdict: medium. evidence: `Numeric(18,4)`; `0.00005` станет `0.0000`; import response отличается от GET. route: patch → тот же scale-валидатор и квантование qty.
- **2026-10-20 · blind-hunter + edge-case-hunter · qty принимает 0/отрицательные/1e-12** — verdict: low. evidence: отрицательные количества допустимы в сметах (сторно); `1e-12` отсекается scale-валидатором; требование `gt=0` в AC отсутствует. route: reject.
- **2026-10-20 · blind-hunter · список выгружает все позиции и пагинирует в Python** — verdict: low. evidence: максимум 1000 позиций на версию; SQL-пагинация — оптимизация, не требование AC. route: reject.
- **2026-10-20 · blind-hunter + edge-case-hunter · гонка импорта/активации между проверкой и записью** — verdict: low. evidence: требуется одновременный импорт/активация одной сметы; MVP — один сметчик, 20–50 смет/мес. route: reject (блокировки — отдельная задача).
- **2026-10-20 · blind-hunter + verification-gap · `add_items` проверяет компанию только у первого элемента** — verdict: low. evidence: текущий `ImportItems` строит все позиции с одним `version_id`, смешанная пачка недостижима; но контракт порта допускает ошибку. route: patch → guard на единый `version_id`.
- **2026-10-20 · blind-hunter · на каждый correct скачивается весь справочник** — verdict: low. evidence: объём MVP и частота правок невелики; кеш/точечное чтение — оптимизация. route: reject.
- **2026-10-20 · blind-hunter · одна битая строка каталога обрушивает весь разбор** — verdict: low. evidence: rate-catalog — наш сервис; битая строка означает порчу данных, fail-loud корректнее тихого пропуска. route: reject.
- **2026-10-20 · blind-hunter · ЕИ расценки не проверяется на пустоту/длину** — verdict: medium. evidence: пустая/слишком длинная ЕИ из каталога попадёт в `unit_normalized` и нарушит контракт. route: patch → валидировать unit в клиенте и тест.
- **2026-10-20 · blind-hunter · повторный идентичный PATCH меняет `updated_at`** — verdict: medium. evidence: спека обещает идемпотентность; тест сравнивает только статус/расценку. route: patch → no-op без изменения `updated_at` и тест.
- **2026-10-20 · blind-hunter · нет правил переходов статусов и истории проверки** — verdict: low. evidence: спека фиксирует «статус отражает последнее решение»; audit/переходы не входят в AC. route: reject.
- **2026-10-20 · blind-hunter + verification-gap · README: «все операции только для активной версии», хотя GET доступен историческим** — verdict: low. evidence: реализация и решения разрешают GET исторической версии; ограничение относится к мутациям. route: patch → уточнить README.
- **2026-10-20 · blind-hunter · README: пример `needs_review=` пустой, не описаны limit/total** — verdict: low. evidence: документация неполна, контракт в коде/спеке корректен. route: patch → исправить пример и описать параметры/счётчики.
- **2026-10-20 · blind-hunter · README: для PATCH не описаны 401/404/409 и идемпотентность** — verdict: low. evidence: контракт реализован, но не документирован. route: patch → дополнить README.
- **2026-10-20 · blind-hunter · fallback `row_ref=str(index)` может совпасть с явным row_ref** — verdict: low. evidence: дубликаты `row_ref` допускаются и паруются по порядку в решении 6.2; provenance-коллизия не ломает контракт. route: reject.
- **2026-10-20 · blind-hunter · порядок `created_at,id` не гарантирует порядок входа** — verdict: low. evidence: порядок входа гарантируется только в ответе импорта; список сортируется детерминированно, 6.2 использует `row_ref`. route: reject.
- **2026-10-20 · blind-hunter · тест foreign item падает на версии и не проверяет `get_item`; FakeRateCatalog глобальный; SQLite не воспроизводит Numeric** — verdict: medium. evidence: item-level tenancy не покрыт; scale-баг закрывается валидацией. route: patch → тест PATCH с неизвестным item_id в своей версии; глобальность fake — приемлема для unit-тестов; SQLite-округление закрыто scale-валидатором.
- **2026-10-20 · blind-hunter · нет тестов 0.70/None, `needs_review=false`, ровно 1000, qty<=0, коллизии row_ref, PATCH неизвестных сущностей, GET истории, correct с rate+unit** — verdict: medium. evidence: часть кейсов закрывается verification-gap; qty<=0 и коллизия row_ref — осознанно допустимы. route: patch → добавить 0.70, false-фильтр, 1000, неизвестный item, историческую версию, rate+unit.
- **2026-10-20 · blind-hunter · `needs_review_count` не уменьшается после проверки, нет фильтра по статусу** — verdict: low. evidence: спека определяет счётчик по confidence, а статус — отдельное поле; фильтр по статусу не входит в AC. route: reject.
- **2026-10-20 · verification-gap · сохранение confirm/correct в БД не проверяется** — verdict: medium. evidence: PATCH возвращает in-memory объект; удаление присваиваний в `update_item` не ломает 12 тестов. route: patch → GET после PATCH и проверка полей из БД.
- **2026-10-20 · verification-gap · `RATE_CATALOG_URL` и клиент в production composition не проверяются** — verdict: medium. evidence: env можно переименовать, suite зелёный. route: patch → composition-тест с correct и MockTransport для rate-catalog.
- **2026-10-20 · verification-gap · 404 неизвестной позиции не исполняется** — verdict: medium. evidence: foreign-тест отсекается на версии; случайный item_id в своей версии не покрыт. route: patch → тест неизвестного item_id.
- **2026-10-20 · verification-gap · фильтр `needs_review=false` не проверяется** — verdict: medium. evidence: только `true` и без фильтра; инверсия условия проходит suite. route: patch → тест false.
- **2026-10-20 · verification-gap · порог 0.70 не закреплён** — verdict: medium. evidence: `<` можно заменить на `<=` без падения. route: patch → тест ровно 0.70.
- **2026-10-20 · verification-gap · GET исторической версии не проверяется** — verdict: medium. evidence: ограничение списка активной версией прошло бы suite, хотя read-only история нужна 6.2. route: patch → тест GET после активации другой версии.
- **2026-10-20 · edge-case-hunter · correct с unit-only на позиции без расценки оставляет её без rate** — verdict: medium. evidence: такая позиция не пригодна для расчёта; confirm для неё невозможен. route: patch → требовать расценку (в теле или уже предложенную).

## Design Notes

- `POST /v1/estimates/{estimate_id}/versions/{version_id}/items`: `{items:[{row_ref?, name, material_work, unit_orig, unit_normalized, qty, section, rate_catalog_id?, confidence}]}` → `201 {items, total}`. `row_ref` по умолчанию — строка индекса в списке; `rate_catalog_id` берётся из `match.rate_id` 5.2, `unit_normalized`/`confidence` — из предложения. Статус — `proposed`.
- `GET /v1/estimates/{estimate_id}/versions/{version_id}/items?limit=50&offset=0&needs_review=` → `200 {items,total,needs_review_count}`; `limit` 1–500. `total` — после фильтра, `needs_review_count` — по всей версии.
- `PATCH /v1/estimates/{estimate_id}/versions/{version_id}/items/{item_id}`: `{decision: confirm|correct, rate_catalog_id?, unit?}`. `confirm` требует непустой `rate_catalog_id` и не меняет значения; `correct` требует непустую расценку (в теле или уже предложенную) и опциональный `unit`, при новой расценке проверяет её в активном справочнике, при отсутствии `unit` берёт ЕИ расценки. Повторный идентичный вызов — no-op без изменения `updated_at`.
- Элемент ответа: `{id,row_ref,name,material_work,unit_orig,unit_normalized,qty,section,rate_catalog_id,confidence,needs_review,status,created_at,updated_at}`; Decimal — строкой. `needs_review = confidence is not None and confidence < 0.70` (не зависит от статуса).
- Импорт не валидирует предложенный `rate_catalog_id` (он пришёл из 5.2 и мог устареть); проверка выполняется только при `correct`. Снапшоты не трогаются (6.3).
- Валидация импорта: `confidence` конечный в `[0,1]`, `qty` конечный Decimal с ограниченным exponent; оба квантуются до 4 знаков (`Numeric(5,4)`/`Numeric(18,4)`), значения с большей точностью отклоняются, чтобы `needs_review` не перескакивал порог при округлении; непустые `name`/`unit_orig`/`unit_normalized`; длины соответствуют колонкам БД (`name` ≤ 1000, `unit_*`/`section`/`row_ref` ≤ 100/255). `confirm` с `rate_catalog_id`/`unit` в теле и `correct` без расценки — `422`. `update_item` обновляет `updated_at` только при реальном изменении.
- Сортировка списка — `created_at`, `id`; при импорте порядок входа сохраняется порядком вставки. `needs_review` — строго `confidence < 0.70` (0.70 не требует проверки).
- Ошибки: неактивная версия — `409`, непустая версия при импорте — `409`, чужие/неизвестные сущности — `404`, неизвестная расценка — `422`, недоступный rate-catalog — `502`.

## Verification

**Commands (выполнено 2026-10-20):**
- `cd estimate && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `91 passed`
- `cd deploy && docker compose config -q` → exit 0
- `cd estimate && docker build -q -t estimate:ci .` → не завершено: команда прервана пользователем; образ не подтверждён.

**Walkthrough (2026-10-21):** approved — импорт в активную версию, confirm/correct и деградация при недоступном справочнике приняты; статус истории `done`.
