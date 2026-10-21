---
title: 'Story 6.3 — Детерминированный расчёт стоимости'
type: 'feature'
created: '2026-10-21'
status: 'done'
route: 'dispatch'
baseline_commit: '63320f0'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-6-context.md'
  - '_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md'
  - '_bmad-output/implementation-artifacts/spec-6-1-создание-сметы-и-версий-ровно-одна-активная.md'
  - '_bmad-output/implementation-artifacts/spec-5-3-api-ручной-проверки-маппинга.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** В смете есть проверенные сопоставления, но нет стоимости: ни детальной по позициям, ни итоговой. Позиции не хранят снапшоты расценок, поэтому цена из справочника не «замораживается» на момент расчёта, а пересчёт невоспроизводим.

**Approach:** Добавить в `estimate` расчёт версии: позиции с подтверждённой расценкой получают снапшот (id, ЕИ, цена) и стоимость, версия — итоговую стоимость и отметку времени расчёта. Расчёт детерминирован (Decimal, фиксированные формулы), LLM не участвует, справочник не меняется.

## Boundaries & Constraints

**Always:** `company_id` из `X-Company-Id`; расчёт только активной и ещё не рассчитанной версии; стоимость позиции = `qty × rate_amount` с округлением до 2 знаков (`ROUND_HALF_UP`), итог = сумма стоимостей позиций; снапшот (`rate_snapshot_id`, `unit_snapshot`, `rate_amount_snapshot`) фиксируется при расчёте и далее неизменен; позиции без подтверждённого статуса (`confirmed`/`corrected`) или без расценки не попадают в итог и перечисляются отдельным списком; Decimal, никакого float; деньги в API — строкой с 2 знаками; ошибки — `application/problem+json`; слои AD-011; расценки — только через API rate-catalog (AD-014).

**Never:** LLM и участие в арифметике (AD-001); изменение справочника, позиций и их статусов; повторный расчёт рассчитанной версии (снапшоты и стоимость неизменны); перезапись снапшота при неизменной расценке; UI; очередь; хранение истории расчётов (только текущие значения).

**Решения (2026-10-20):**
- Формулы: `cost = qty × rate_amount`, округление 2 знака (`ROUND_HALF_UP`) на позиции и итоге; итог — сумма округлённых стоимостей позиций.
- Частичный расчёт: непокрытые позиции исключаются из итога и возвращаются списком с причиной; ошибки не выбрасываются.
- Расчёт выполняется внутри `estimate` (решение владельца): `calc-service` остаётся каркасом. Это отклонение от AD-001/спайна (`calc-service` — владелец расчёта) — требует правки архитектурных документов.
- История 3.3 (snapshot) закрывается этой же историей: снапшот пишется при расчёте, повторный расчёт версии — `409`.
- В расчёт входят только позиции со статусом `confirmed`/`corrected`; остальные — в `uncovered` с причиной `not_reviewed` (AC «активная версия сметы с подтверждёнными сопоставлениями»).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Расчёт | активная версия с подтверждёнными позициями | `200`: `items` со стоимостью, `total_cost`, `calculated_at` | — |
| Частичный расчёт | часть позиций без подтверждения/расценки | `200`: итог по покрытым, `uncovered` с причинами | — |
| Нет покрытых позиций | все позиции непокрыты | `200`: `total_cost="0.00"`, `uncovered` — все | — |
| Пустая версия | нет позиций | `200`: `total_cost="0.00"`, пустые списки | — |
| Повторный расчёт | версия уже рассчитана | — | `409` |
| Неактивная версия | расчёт неактивной версии | — | `409` |
| Расценка не найдена | `rate_catalog_id` неизвестен справочнику | позиция в `uncovered` (`rate_not_found`) | — |
| rate-catalog недоступен | сеть/5xx | — | `502` |
| Чужая/неизвестная смета или версия | другой `company_id` | — | `404` |
| Нет identity | без `X-Company-Id` | — | `401` |
| Детерминизм | одинаковые входные данные | одинаковые стоимости и итог | — |

</frozen-after-approval>

## Code Map

- `estimate/migrations/versions/0002_add_costs.py` -- новый: `estimate_item.cost Numeric(18,2)`, `estimate_version.total_cost Numeric(18,2)` (nullable); `calculated_at` уже есть.
- `estimate/app/domain/cost.py` -- новый: `LineCost`, `UncoveredItem`, `CalculationResult`, округление 2 знака `ROUND_HALF_UP`, чистая функция расчёта по `(qty, rate_amount)`.
- `estimate/app/application/calculate_cost.py` -- новый: `CalculateCost` — проверка версии, выбор позиций, разрешение расценок, снапшоты, запись стоимостей и итога.
- `estimate/app/domain/errors.py` -- `VersionAlreadyCalculated` (409).
- `estimate/app/domain/ports.py` -- `EstimateRepository`: `update_item_snapshot_and_cost`, `update_version_calculation`.
- `estimate/app/infrastructure/repository.py` -- сохранение снапшота/стоимости позиции и `total_cost`/`calculated_at` версии.
- `estimate/app/infrastructure/rate_catalog_client.py` -- `find_rate_by_id` через новый `GET /v1/rate-catalog/rates/{id}`; `404` → `None`, сеть/5xx → `RateCatalogUnavailable`.
- `estimate/app/transport/{schemas,routes,problem}.py` -- `POST /v1/estimates/{id}/versions/{vid}/calculate`, схемы ответа, `409` для рассчитанной/неактивной версии; в `ItemResponse` добавить `cost`, в `EstimateVersionResponse` — `total_cost`.
- `estimate/app/wiring.py` -- метод `calculate_cost` в `EstimateOperations`.
- `estimate/tests/test_calculate_cost.py` -- новый: матрица, формулы, округление, снапшоты, частичный расчёт, tenancy, 409/502, детерминизм.
- `rate-catalog/app/domain/{ports,rate}.py`, `app/application/get_rate.py`, `app/infrastructure/repository.py`, `app/transport/{routes,schemas}.py`, `app/wiring.py`, `tests/test_get_rate.py` -- `GET /v1/rate-catalog/rates/{rate_id}`: неизменяемая строка расценки в скоупе компании, `404` для чужой/неизвестной.
- `estimate/README.md`, `rate-catalog/README.md` -- контракт расчёта и новый endpoint.
- Не трогать: `calc-service/*` (остаётся каркас), `api-gateway/*` (префикс `/v1/estimates` уже есть), `deploy/*` (`RATE_CATALOG_URL` у estimate уже задан), миграции rate-catalog.

## Tasks & Acceptance

**Execution:**
- [x] `rate-catalog/app/domain/*`, `application/get_rate.py`, `infrastructure/repository.py`, `transport/*`, `wiring.py` -- чтение расценки по id -- снапшот из неизменяемой строки.
- [x] `rate-catalog/tests/test_get_rate.py` -- 200/404/tenancy -- контракт снапшота.
- [x] `estimate/migrations/versions/0002_add_costs.py` -- колонки стоимости -- хранение результата.
- [x] `estimate/app/domain/cost.py`, `application/calculate_cost.py` -- формулы и use case -- детерминированный расчёт.
- [x] `estimate/app/domain/ports.py`, `infrastructure/{repository,rate_catalog_client}.py` -- запись снапшотов/стоимостей, чтение расценки по id -- персистентность.
- [x] `estimate/app/transport/*`, `wiring.py` -- HTTP-контракт -- 200/401/404/409/502.
- [x] `estimate/tests/test_calculate_cost.py` -- матрица, формулы, детерминизм -- регрессия.
- [x] `estimate/README.md`, `rate-catalog/README.md` -- документация -- потребители 6.2/7.2.
- [x] `estimate` + `rate-catalog` -- ruff/pytest -- верификация.

**Acceptance Criteria:**
- Given активная нерассчитанная версия с подтверждёнными сопоставлениями, when `POST .../calculate`, then `200`: по каждой покрытой позиции `cost`, `total_cost` и `calculated_at` сохранены в позиции/версии.
- Given позиция без подтверждения или без расценки, when расчёт, then она отсутствует в итоге и присутствует в `uncovered` с причиной; `total_cost` равен сумме покрытых.
- Given цена и количество, when считается стоимость, then `cost = qty × rate_amount`, округлено до 2 знаков, итог — сумма этих стоимостей.
- Given расчёт выполнен, when позиции проверяются повторно, then снапшот и стоимость неизменны, а повторный расчёт версии даёт `409`.
- Given чужая смета/версия, неактивная версия или недоступный rate-catalog, when расчёт, then `404`/`409`/`502`; без `X-Company-Id` — `401`.
- Given одинаковые входные данные, when расчёт повторяется на другой версии, then стоимости и итог совпадают.

## Implementation Notes

- Расчёт живёт в `estimate` (решение владельца 2026-10-20): `app/application/calculate_cost.py` + `app/domain/cost.py` (чистые `calculate_line_cost`/`sum_costs` на Decimal). `calc-service` не трогали.
- `rate-catalog`: `GET /v1/rate-catalog/rates/{rate_id}` — строка расценки любой версии справочника компании (`get_rate` c JOIN `rate_catalog_rate → rate_catalog_version → rate_catalog` по `company_id`); 404 для чужой/неизвестной, миграций не добавляли (схема не менялась).
- `estimate`: миграция `0002_add_costs.py` — `estimate_item.cost Numeric(18,2)` и `estimate_version.total_cost Numeric(18,2)` (nullable), downgrade через `batch_alter_table` (SQLite).
- Порядок проверок в use case: версия не найдена → `404`; уже рассчитана → `409`; не активна → `409`.
- Причины `uncovered` проверяются по приоритету: статус не `confirmed`/`corrected` → `not_reviewed`; нет `rate_catalog_id` → `no_rate`; расценка не найдена → `rate_not_found`.
- Расценки позиций читаются одним портом `RateCatalogPort.find_rate_by_id` (новая ручка клиента), не переиспользуется `find_rate` (активная версия) — семантика 5.3 не меняется. В рамках одного расчёта id кэшируются (один HTTP-вызов на уникальную расценку).
- Снапшот пишется вместе со стоимостью: `rate_snapshot_id`, `unit_snapshot`, `rate_amount_snapshot`, `cost`; `updated_at` позиции обновляется отметкой расчёта. `rate_catalog_id` остаётся провенансом.
- Рассчитанная версия заморожена: `PATCH` позиции и импорт в неё → `409` (`VersionAlreadyCalculated`), иначе стоимость разошлась бы с сопоставлениями (Design Notes).
- Денежный формат: `cost`, `total_cost` и `rate_amount` в ответе — строки с 2 знаками (решение владельца). Стоимость при этом посчитана по **точной** цене снапшота (`rate_amount_snapshot`, до 4 знаков), поэтому пересчёт по показанным двум знакам может отличаться на копейку; это зафиксировано в `estimate/README.md`.
- Границы денег: стоимость позиции и итог проверяются на диапазон `Numeric(18,2)` (`±9999999999999999.99`) до записи; выход за диапазон — доменная ошибка `CostOutOfRange` → `422` problem+json, версия остаётся нерассчитанной (транзакция откатывается, запись начинается только после проверки итога).
- Конкурентный расчёт: `update_version_calculation` пишет условно (`UPDATE ... WHERE calculated_at IS NULL`); при `rowcount == 0` — `VersionAlreadyCalculated` (`409`), поэтому второй параллельный расчёт не перезаписывает снапшоты и итог.
- Клиент rate-catalog дополнительно проверяет, что ответ содержит именно запрошенный id расценки (иначе `RateCatalogUnavailable` → `502`), и что цена имеет не более 4 знаков после запятой (иначе она округлилась бы в `Numeric(18,4)` и разорвала `cost = qty × rate_amount`).
- Пустая версия считается как есть: `total_cost="0.00"`, `calculated_at` проставлен, версия заморожена (повторный расчёт — `409`).
- `ItemResponse` получил `cost`, `EstimateVersionResponse` — `total_cost` (в API версии они nullable до расчёта).
- Отклонение от AD-001 зафиксировано в архитектурных документах: `docs/architecture-spine.md` и `_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md` (AD-001, список сервисов, конвейер, карта capability), `docs/architecture-overview.md` и парная копия (контейнеры и таблица сервисов), плюс `epic-6-context.md`.

## Spec Change Log

- 2026-10-21: реализация без изменения frozen-блока; уточнения контракта на уровне реализации:
  - `rate_amount` в ответе расчёта отдаётся с 2 знаками (решение владельца), как и `cost`/`total_cost`. Стоимость посчитана по точной цене снапшота (`rate_amount_snapshot`, до 4 знаков), поэтому пересчёт по показанным двум знакам может отличаться на копейку — это зафиксировано в `estimate/README.md`.
  - Пустая версия считается и замораживается как есть: `total_cost="0.00"`, `calculated_at` проставлен, повторный расчёт — `409`.
  - Приоритет причин `uncovered`: сначала статус (`not_reviewed`), затем `no_rate`, затем `rate_not_found` — следует из решения frozen-блока «остальные — в `uncovered` с причиной `not_reviewed`».
  - Импорт позиций в рассчитанную версию запрещён (`409`), как и `PATCH`: frozen-блок требует заморозки рассчитанной версии, иначе итог разошёлся бы с позициями.
  - Суммы вне `Numeric(18,2)` — `422` (`CostOutOfRange`), а не `500`; конкурентный расчёт — `409` через условный `UPDATE`.
  - Расчёт выполняет `estimate` (решение владельца 2026-10-20 из frozen-блока); отклонение от AD-001 отражено в `ARCHITECTURE-SPINE`/`ARCHITECTURE-OVERVIEW` и `epic-6-context.md`.

## Review Triage Log

- **2026-10-21 · blind-hunter · реализация противоречит AD-002 (`rate_catalog_id` не участвует в расчёте)** — verdict: medium. evidence: цена берётся по `rate_catalog_id` живым вызовом, `rate_amount_snapshot` на момент расчёта ещё NULL. route: patch → AD-002 уточнён в обеих копиях спайна: `rate_catalog_id` — провенанс и источник захвата снапшота, в формулах участвует `rate_amount_snapshot`.
- **2026-10-21 · blind-hunter · причина `no_rate` недостижима через публичный API** — verdict: low. evidence: 5.3 не даёт подтвердить позицию без расценки; ветка достижима только при прямой записи в БД. route: reject (защитная ветка, причина остаётся в контракте).
- **2026-10-21 · blind-hunter + edge-case-hunter · id расценки в ответе rate-catalog не сверяется с запрошенным** — verdict: medium. evidence: клиент берёт id из тела ответа — чужой id дал бы снапшот с чужой ценой. route: patch → сверка id, расхождение — `RateCatalogUnavailable` (502).
- **2026-10-21 · blind-hunter + edge-case-hunter · инвариант «повторный расчёт → 409» не защищён от гонки** — verdict: medium. evidence: проверка `calculated_at is not None` идёт до безусловного UPDATE — два параллельных расчёта оба записали бы. route: patch → условный `UPDATE ... WHERE calculated_at IS NULL` + `rowcount == 0` → 409.
- **2026-10-21 · blind-hunter + edge-case-hunter · до 1000 последовательных HTTP-вызовов к rate-catalog внутри транзакции** — verdict: medium. evidence: по вызову на уникальную расценку, без батча и без общего дедлайна; таймаут клиента 10 с на запрос. route: defer → батч-endpoint/дедлайн — отдельная задача (объёмы MVP: 100–500 позиций).
- **2026-10-21 · blind-hunter · N+1 к БД при записи результата (SELECT+UPDATE на позицию)** — verdict: low. evidence: до 1000 итераций с flush; на объёмах MVP терпимо, PostgreSQL вручную проверен. route: defer → bulk-обновление при росте объёмов.
- **2026-10-21 · blind-hunter + edge-case-hunter + verification-gap · нет ограничения денежных величин → 500 вместо доменной ошибки** — verdict: medium. evidence: `qty` и цена до 1e12, произведение выходит за `Numeric(18,2)` → `numeric field overflow` на Postgres. route: patch → доменная проверка диапазона `±9999999999999999.99` с `422` до записи.
- **2026-10-21 · blind-hunter · масштаб цены расценки не валидируется (округление в `Numeric(18,4)`)** — verdict: medium. evidence: цена с >4 знаками округлится в БД и разорвёт равенство `cost = qty × rate_amount`. route: patch → клиент отклоняет цены с >4 знаками.
- **2026-10-21 · blind-hunter · ответ расчёта не пагинируется** — verdict: low. evidence: до 1000 позиций одним ответом; соседний diff (6.2) устроен так же, лимита в AC нет. route: reject.
- **2026-10-21 · blind-hunter · README estimate не отражает изменившийся контракт версии и позиции** — verdict: low. evidence: в ответе версии появился `total_cost`, у позиции — `cost`; перечисления полей устарели. route: patch → README.
- **2026-10-21 · blind-hunter · brief и epics-stories не приведены к решению владельца (calc-service)** — verdict: medium. evidence: источники требований противоречат спайну: AC 6.3 и brief всё ещё упоминают calc-service. route: patch → epics.md, docs/epics-stories.md, docs/brief.md.
- **2026-10-21 · blind-hunter · review-loop не прогонялся, а Story 3.3 уже помечена resolved** — verdict: false. evidence: review-loop — текущий шаг workflow; 3.3 закрывается по его итогам, запись в deferred-work сделана в рамках той же истории. route: reject.
- **2026-10-21 · blind-hunter · diff не показывает разницу стоимости** — verdict: low. evidence: стоимость производна от quantity/unit/rate, которые diff уже сравнивает; контракт 6.2 не меняем. route: reject.
- **2026-10-21 · blind-hunter · расчёт пустой версии замораживает её навсегда** — verdict: medium. evidence: `total_cost="0.00"`, дальше импорт/правка — 409, разморозки нет. route: решение владельца (2026-10-21): оставить как есть (вариант C) → поведение зафиксировано в README.
- **2026-10-21 · blind-hunter · нет пути переноса подтверждённых позиций в новую версию** — verdict: medium. evidence: импорт 5.3 создаёт позиции в статусе `proposed` — подтверждать заново. route: patch (документация) + defer (функциональность копирования — отдельная задача).
- **2026-10-21 · blind-hunter · нет аудита актора расчёта** — verdict: low. evidence: `X-User-Id` не читается, у версии нет `calculated_by`; заморозка не оставляет след об авторе. route: defer → `calculated_by` отдельной задачей.
- **2026-10-21 · blind-hunter + verification-gap · пробелы покрытия (версия другой сметы той же компании, порядок причин `uncovered`, keyless-сортировка, неизменность снапшотов при повторе, различение «активная версия»/«любая версия»)** — verdict: medium. evidence: перечисленные сценарии не проверялись. route: patch → тесты.
- **2026-10-21 · verification-gap · масштаб `rate_amount` в ответе расчёта не закреплён** — verdict: medium. evidence: замена на 2 знака не ломала suite. route: patch → assert на точное значение + решение владельца: `rate_amount` отдаётся с 2 знаками, стоимость посчитана по точной цене снапшота (документировано).
- **2026-10-21 · verification-gap · приоритет причин `uncovered` (`not_reviewed` раньше `no_rate`) не закреплён** — verdict: medium. evidence: перестановка проверок не ломала suite. route: patch → тест.
- **2026-10-21 · verification-gap · кэш расценок (один вызов на уникальную расценку) не закреплён** — verdict: medium. evidence: удаление кэша не ломало suite. route: patch → тест с двумя позициями одной расценки.
- **2026-10-21 · verification-gap · `updated_at` позиции после расчёта не закреплён** — verdict: medium. evidence: удаление строки `item.updated_at = calculated_at` не ломало suite. route: patch → тест.
- **2026-10-21 · verification-gap · связка «расчёт → diff» не закреплена** — verdict: medium. evidence: расчёт пишет `unit_snapshot` расценки, поэтому diff рассчитанной версии против идентичной нерассчитанной показывает `changes: ["unit"]`. route: patch → тест, фиксирующий поведение, + описание в README.
- **2026-10-21 · edge-case-hunter · позиция исчезает из ответа, если `_resolve` не вернёт ни строки, ни причины** — verdict: false. evidence: все ветки `_resolve` возвращают строку или причину; ветка недостижима. route: reject.
- **2026-10-21 · edge-case-hunter · PATCH неизвестного item_id рассчитанной версии даёт 409 вместо 404** — verdict: low. evidence: версия заморожена, 409 корректен и не раскрывает существование позиции. route: reject.
- **2026-10-21 · edge-case-hunter · правка позиции во время расчёта даёт снапшот устаревшей расценки** — verdict: low. evidence: гонка соседних use case'ов; расчёт и правка в одной версии исключены заморозкой. route: reject.
- **2026-10-21 · edge-case-hunter · цена ≥1e13 приводит к 502 (клиент жёстче справочника)** — verdict: low. evidence: нереалистичная цена; fail-loud вместо молчаливого округления. route: reject.
- **2026-10-21 · edge-case-hunter · сумма превышает 28 значащих цифр Decimal** — verdict: medium. evidence: при экстремальных величинах `sum_costs` может выйти за точность контекста. route: patch → закрыто проверкой денежного диапазона до записи.
- **2026-10-21 · edge-case-hunter · `rate_amount` в ответе расчёта отдаётся не в 2 знака, а в масштабе справочника** — verdict: medium. evidence: frozen-блок требует «деньги в API — строкой с 2 знаками». route: patch → решение владельца (2026-10-21): 2 знака; расхождение с пересчётом по показанным значениям задокументировано.
- **2026-10-21 · verification-gap · 404 от rate-catalog трактуется как «расценка не найдена» и версия фиксируется с итогом 0.00** — verdict: low. evidence: поведение закреплено контрактом (`rate_not_found`), клиент видит причины в `uncovered`; риск — развёрнутая без маршрута версия rate-catalog. route: reject (задокументировано как риск развёртывания).
- **2026-10-21 · blind-hunter · `RateNotFound.__init__` с необязательным параметром** — verdict: low. evidence: вызов всегда с id; сообщение «rate None» недостижимо. route: reject.

## Design Notes

- `POST /v1/estimates/{estimate_id}/versions/{version_id}/calculate` → `200 {version_id, calculated_at, total_cost, items:[{item_id,row_ref,name,qty,unit,rate_id,rate_amount,cost}], uncovered:[{item_id,row_ref,name,reason}]}`; деньги — строкой с 2 знаками.
- Причины `uncovered`: `not_reviewed` (статус `proposed`), `no_rate` (нет `rate_catalog_id`), `rate_not_found` (id не найден в справочнике). В итог входят только позиции со статусом `confirmed`/`corrected`.
- Снапшот пишется при первом расчёте: `rate_snapshot_id` = id выбранной расценки, `unit_snapshot` = ЕИ расценки, `rate_amount_snapshot` = её цена; `rate_catalog_id` остаётся провенансом (AD-002).
- Расценка позиции разрешается новым `GET /v1/rate-catalog/rates/{id}` (неизменяемая строка любой версии справочника) — поэтому переимпорт справочника не меняет стоимость уже рассчитанных смет.
- Повторный расчёт версии — `409`; чтобы получить новые цены, создаётся новая версия сметы (6.1) и в неё импортируются позиции с новой расценкой (5.3).
- Правка позиции рассчитанной версии запрещается: 5.3 `PATCH` для версии с `calculated_at` → `409`, иначе стоимость версии разошлась бы с сопоставлениями.
- Порядок в ответе: `items` и `uncovered` — по `row_ref` (пустые в конце), затем `id`; как в diff (6.2).
- Расчёт детерминирован: только Decimal, никаких float и внешних вызовов кроме чтения расценок; LLM не участвует.

## Verification

**Commands (выполнено 2026-10-21):**
- `cd estimate && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `150 passed`
- `cd rate-catalog && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `66 passed`
- `cd api-gateway && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `48 passed` (edge не менялся)
- `cd deploy && docker compose config -q` → exit 0 (compose не менялся)

**Дополнительно (вручную, вне CI):**
- Миграции estimate на живой PostgreSQL 18.6: `upgrade head` → `downgrade 0001` → `upgrade head` — без ошибок (SQLite покрыт тестом миграций).
- Сквозной расчёт на PostgreSQL: `2.5 × 1234.5678 = 3086.4195 → 3086.42`; в БД `cost=3086.42`, `rate_amount_snapshot=1234.5678`, `total_cost=3086.42`.
- Живой стек (`deploy/docker-compose.yml`, Postgres 18.6 + MinIO + контейнеры с новым кодом): импорт справочника → `estimate` прочитал расценку через `GET /v1/rate-catalog/rates/{id}` → расчёт версии `200` (`total_cost=3086.42`, `items[0].cost=3086.42`, `uncovered[0].reason=not_reviewed`) → повторный расчёт `409`; `GET /v1/rate-catalog/rates/{id}`: `200`, `404` (неизвестная и чужая компания), `401` (без identity).
- Образы сервисов собираются: `docker build` для `estimate` и `rate-catalog` — OK.

**Не покрыто автоматически:** review-loop (blind-hunter/edge-case-hunter/verification-gap) не запускался — `Review Triage Log` пуст, статус `review`. Тесты идут на SQLite (CI без Postgres) — общая инфра-задача ретро Epic 2/4.
