---
title: 'Story 7.3 — estimate: деградация вместо отказа и единый контракт внешних вызовов'
type: 'bugfix'
created: '2026-10-21'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
baseline_commit: '609b04c'
context:
  - '_bmad-output/implementation-artifacts/epic-7-context.md'
  - '_bmad-output/implementation-artifacts/epic-7-retro-10-21-2026.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 7.2 повесила обогащение справочником на общий маршрут `GET .../items`, которым пользуется и ручная проверка маппинка (5.3): недоступность `rate-catalog` даёт `502` на всей странице. Три HTTP-клиента `estimate` решают, что значит `404`/`422`, по-разному, а сессия БД удерживается во время сетевого вызова.

**Approach:** Перевести внешние вызовы на общий контракт: «ресурса нет» / «недоступность» / «нарушение контракта»; при недоступности внешнего сервиса отдавать данные с пустым обогащением и явным признаком вместо `502`; выносить сетевые вызовы за пределы сессии БД.

## Boundaries & Constraints

**Always:** чтение данных не зависит от доступности внешних сервисов — вместо отказа отдаём пустое обогащение с признаком; `404` без `application/problem+json` и `422` — недоступность/нарушение контракта, а не «ресурса нет»; сетевые вызовы выполняются при закрытой сессии БД; `rate_code`/`rate_name` остаются `null` в ответах импорта и ручной проверки; ошибки — `application/problem+json` без внутренних деталей.

**Never:** изменение правды ответа API позиций (значения приходят из сервиса как есть); запись в чужие сервисы; изменение `mapping-service`, `rate-catalog`, `pdf-extract`, `company-org`; UI-валидация и UI-diff (1.1); новые миграции; изменение формул расчёта (6.3).

**Решения (2026-10-20):** деградация вместо `502` на маршрутах чтения, где внешний сервис дополняет локальные данные (обогащение справочником, состояние файла); подтверждение скоупа этапа при отказе company-org — честный `502` (пустой список означал бы «смет нет» вместо «сервис недоступен»); отдельный доменный признак «внешний сервис недоступен» в ответе; общий хелпер HTTP-вызовов в `estimate/app/infrastructure`.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Справочник недоступен | `rate-catalog` отдаёт 5xx | `200`, позиции без `rate_code`/`rate_name`, признак недоступности | — |
| Справочник без расценки | `404` + problem+json | `200`, пустое обогащение, признак недоступности **не** выставлен | — |
| Нарушение контракта | `422` от внешнего сервиса | `502` с текстом «нарушение контракта», не «нет данных» | `502` |
| Неверный адрес сервиса | `404` без problem+json | `502`, а не «не найдено» | `502` |
| Файл недоступен | `pdf-extract` отдаёт 5xx | `200`, файловые стадии не выполнены, признак недоступности | — |
| Ручная проверка | `GET .../items` при живом справочнике | `200` с `rate_code`/`rate_name` | — |
| Импорт и правка | ответы импорта/проверки | `rate_code`/`rate_name` = `null` | — |
| Нет identity | без `X-Company-Id` | — | `401` |

</frozen-after-approval>

## Code Map

- `estimate/app/infrastructure/http.py` -- новый: общий вызов внешнего API с identity-заголовком, таймаутом и единым разбором ответа (`404`+problem+json → «нет ресурса», `422` → нарушение контракта, иное → недоступность).
- `estimate/app/domain/errors.py` -- `UpstreamContractViolation` (502) и переиспользование `*Unavailable` (502).
- `estimate/app/infrastructure/{company_org,pdf_extract,rate_catalog}_client.py` -- перевести на общий хелпер; убрать расхождение семантики `404`.
- `estimate/app/application/list_items.py` -- разделить чтение БД и обогащение: позиции читаются при открытой сессии, обогащение — после её закрытия; недоступность справочника → пустое обогащение с признаком.
- `estimate/app/application/processing_status.py` -- то же для файловых стадий: недоступность `pdf-extract` не гасит весь ответ, стадии помечаются неизвестными.
- `estimate/app/application/list_estimates.py` -- вызов `company-org` за пределами сессии.
- `estimate/app/domain/item.py` -- `ItemPage`/`ProcessingStatus` получают признак недоступности внешнего сервиса.
- `estimate/app/transport/schemas.py` -- `ItemsPageResponse` и ответ статуса получают поля-признаки.
- `estimate/app/wiring.py` -- последовательность «сессия закрыта → сетевой вызов → сборка ответа».
- `estimate/tests/` -- деградация, семантика кодов ответа, `null`-обогащение в ответах импорта/проверки, отсутствие сессии во время вызова.
- `ui/src/components/EstimateTable.vue`, `ui/src/views/EstimateView.ts`, `ui/src/components/ProcessingStatus.vue` -- показать предупреждение, когда сопоставления или файловые стадии недоступны.
- `ui/tests/` -- проверка предупреждения.
- Не трогать: `mapping-service/*`, `rate-catalog/*`, `pdf-extract/*`, `company-org/*`, `api-gateway/*`, миграции.

## Tasks & Acceptance

**Execution:**
- [x] `estimate/app/infrastructure/http.py`, `app/domain/errors.py` -- общий контракт внешних вызовов -- единая семантика кодов.
- [x] `estimate/app/infrastructure/*_client.py` -- перевод клиентов на общий хелпер -- 404/422 различаются.
- [x] `estimate/app/application/{list_items,processing_status,list_estimates}.py` -- сетевые вызовы вне сессии, деградация вместо 502 -- доступность чтения.
- [x] `estimate/app/domain/item.py`, `app/transport/schemas.py` -- поля-признаки недоступности -- клиент знает о деградации.
- [x] `estimate/app/wiring.py` -- последовательность сессия→сеть -- соединение освобождается до вызова.
- [x] `estimate/tests/` -- деградация, коды ответа, null-обогащение, отсутствие сессии -- регрессия.
- [x] `ui/src/components/*`, `src/views/EstimateView.ts` -- предупреждение о недоступности -- понятный экран.
- [x] `ui/tests/` -- проверка предупреждения -- регрессия UI.
- [x] `estimate` + `ui` -- ruff/pytest, typecheck/build/test -- верификация.

**Acceptance Criteria:**
- Given справочник недоступен, when `GET .../items`, then `200` с позициями, пустым обогащением и признаком недоступности — ручная проверка маппинга продолжает работать.
- Given внешний сервис отдаёт `422` или `404` без problem+json, when запрос, then `502` с текстом о сбое, а не «не найдено» и не пустые данные.
- Given `pdf-extract` недоступен, when запрошен статус обработки, then `200`: стадии маппинга и расчёта по своим данным, файловые — помечены как недоступные.
- Given позиции читаются, when выполняется запрос, then сетевой вызов происходит при закрытой сессии БД.
- Given ответы импорта позиций и ручной проверки, when они сформированы, then `rate_code`/`rate_name` равны `null`.
- Given внешний сервис недоступен, when UI получает ответ, then показано предупреждение, а позиции и итоги остаются видимыми.

## Implementation Notes

- **estimate (общий хелпер)**: `app/infrastructure/http.py` — базовый класс `UpstreamClient` (общий GET: identity-заголовок `X-Company-Id`, таймаут, разбор ответа) и перечисление `NotFound`: `absent` — «ресурса нет» (`404` + `application/problem+json` → `None`), `route_mismatch` — сервис не знает маршрут (любой `404` → нарушение контракта). Три исхода: «ресурса нет» (`None`), «недоступность» (`*Unavailable`: сеть, `5xx`, прочие статусы), «нарушение контракта» (`UpstreamContractViolation`: `422` от сервиса и `404` без problem+json). Ответ без problem+json не трактуется как «ресурса нет» никогда.
- **estimate (клиенты)**: `company_org_client`, `pdf_extract_client`, `rate_catalog_client` — наследники `UpstreamClient`, у каждого свой `service` и `unavailable_error`; собственные копии GET/таймаута/разбора удалены. Смысл `404` задаётся вызывающим: `NotFound.absent` — договор/этап/файл/расценка вне скоупа компании; `NotFound.route_mismatch` — пакетный маршрут `GET /v1/rate-catalog/rates?ids=` («ничего не разрешилось» приходит как `200 []`, поэтому `404` здесь — старая сборка сервиса, а не пустое обогащение).
- **estimate (домен)**: новая ошибка `UpstreamContractViolation` (502). `ItemPage.rates_unavailable` (default `false`), `ProcessingStatus.source_unavailable` (default `false`), `ProcessingStage.unknown` (default `false`) — стадия помечена как неизвестная, а не «не выполнена».
- **estimate (приложение)**: `list_items` — `ListItems` читает БД (`ItemsRead`), `EnrichItems` делает сетевой вызов (`RatesEnrichment`), `build_item_page` собирает ответ; недоступность справочника гасится в `EnrichItems` (пустое обогащение + признак), а `UpstreamContractViolation` проходит насквозь (`502`). `processing_status` — `GetProcessingStatus` читает БД (`EstimateProcessingState`), `GetSourceFileState` спрашивает pdf-extract (`SourceFileState`), `build_processing_status` собирает четыре стадии; при недоступности pdf-extract файловые стадии помечены `unknown`, маппинг и расчёт — по своим данным. `list_estimates` — `ConfirmStageScope` (сеть) и `ListStageEstimates` (БД).
- **estimate (wiring)**: `EstimateOperations.list_items`, `.processing_status`, `.list_estimates` — сетевой вызов выполняется при закрытой сессии БД: данные читаются внутри `with self._session_factory()`, сессия закрывается, затем уходит запрос к сервису. Слойность сохранена: application не знает про SQLAlchemy/FastAPI, инфраструктура — про use case'ы.
- **estimate (company-org)**: подтверждение скоупа этапа деградации не имеет: недоступность company-org — честный `502` (пустой список означал бы «смет нет» вместо «сервис недоступен» и размывал бы границу скоупа AD-013).
- **estimate (транспорт)**: `ItemsPageResponse.rates_unavailable` и `ProcessingStatusResponse.source_unavailable` — поля присутствуют всегда (`false`, когда сервис ответил); `ProcessingStageResponse.unknown`. Ошибка контракта — `application/problem+json` `502` «Upstream service contract was violated», без внутренних деталей.
- **ui (клиент)**: `ItemsPage.rates_unavailable`, `ProcessingStatus.source_unavailable`, `ProcessingStage.unknown` — типы в `src/api/client.ts`.
- **ui (экран сметы)**: `EstimateView.ts` хранит `rates_unavailable` и передаёт его в `EstimateTable`; `EstimateTable.vue` показывает предупреждение «Справочник расценок сейчас недоступен — коды и названия расценок не показаны. Позиции, стоимость и итог актуальны.» Позиции, стоимость позиций и итог версии остаются видимыми; UI по-прежнему говорит только с `estimate`. Признак **накапливается по страницам** (логическое «или»): если справочник был недоступен на первой странице, ранние строки остались без сопоставления, и предупреждение не исчезает после догрузки. Пустая ячейка сопоставления при недоступном справочнике помечена «сопоставление недоступно» — это не то же, что «не сопоставлена» (позиция не сопоставлена из-за недоступности сервиса).
- **ui (панель стадий)**: `ProcessingStatus.vue` — проп `sourceUnavailable`: предупреждение «Сервис обработки файлов недоступен — стадии загрузки и извлечения неизвестны. Остальные стадии показаны по данным сметы.»; стадия со `unknown` показывается как «неизвестно» со своим маркером («?»), а не «не выполнено» («○»). `HierarchyView.ts` передаёт признак из ответа статуса, считает неизвестную стадию **терминальной** (опрос статуса прекращается) и не предлагает кнопку «Извлечь текст», пока сервис извлечения недоступен.

## Spec Change Log

- 2026-10-21: реализация без изменения frozen-блока; уточнения контракта на уровне реализации:
  - `404` без `application/problem+json` и `404` у маршрута, которого нет у сервиса, отнесены к **нарушению контракта** (`UpstreamContractViolation` → `502`), а не к «ресурса нет» и не к деградируемой недоступности: данные в этом случае не подменяются пустыми (AC «422 или 404 без problem+json → 502»). Недоступность (сеть, `5xx`) остаётся деградируемой.
  - Смысл `404` задаёт вызывающий (`NotFound.absent` / `NotFound.route_mismatch`) — различие между «вне скоупа компании» и «сервис не знает маршрут» вынесено в API хелпера явно, а не спрятано в коде клиентов.
  - Признак недоступности добавлен на два уровня: флаг ответа (`rates_unavailable`, `source_unavailable`) и пометка отдельной стадии (`ProcessingStage.unknown`) — «неизвестно» отличается от «не выполнено».
  - Ответы импорта позиций и ручной проверки не изменились: `rate_code`/`rate_name` — `null` (закреплено тестами).
  - Доктрина деградации уточнена: деградация есть там, где внешний сервис дополняет локальные данные (обогащение справочником, состояние файла); подтверждение скоупа этапа при отказе company-org — `502`, а не пустой список (иначе «смет нет» вместо «сервис недоступен» и размывание границы скоупа AD-013).
  - Новых миграций БД не потребовалось; `mapping-service`, `rate-catalog`, `pdf-extract`, `company-org`, `api-gateway` не менялись.

## Review Triage Log

- **2026-10-21 · implementation · «недоступность» и «нарушение контракта» требовали одного класса ошибок, но деградация применяется только к первому** — verdict: medium (иначе `404` не от сервиса и `422` деградировали бы в пустые данные вопреки AC). evidence: `UpstreamContractViolation` вынесен в отдельный доменный класс и не перехватывается в `EnrichItems`/`GetSourceFileState`; тесты `test_external_contracts.py` фиксируют `502` на `422` и на `404` без problem+json. route: patch → два класса ошибок вместо одного.
- **2026-10-21 · implementation · третий HTTP-клиент скопировал логику GET/таймаута/разбора** — verdict: medium. evidence: три копии расходились в трактовке `404` (ретро F3). route: patch → общий `UpstreamClient` в `app/infrastructure/http.py`; клиенты задают только `service`, `unavailable_error` и смысл `404`.
- **2026-10-21 · implementation · «файла нет» и «сервис недоступен» для pdf-extract неразличимы на языке ответа** — verdict: low. evidence: при сбое pdf-extract стадии `uploaded`/`extracted` выглядели бы как «не выполнено». route: patch → `ProcessingStage.unknown` и флаг `source_unavailable`; UI показывает «неизвестно» и предупреждение.
- **2026-10-21 · implementation · `create_estimate`, `attach_source`, `review_item`, `calculate_cost` по-прежнему держат сессию БД во время сетевого вызова** — verdict: medium. evidence: вне области Code Map этой спеки (три use case'а из трёх), на путях записи деградация невозможна в принципе: внешний ответ участвует в решении о записи. route: defer → `deferred-work.md` (вынесено сетевой вызов; на мутациях нужен другой порядок — предварительный read, затем сеть, затем запись в новой транзакции).
- **2026-10-21 · implementation · `404` от company-org для дерева договоров трактуется как «дерева нет»** — verdict: low. evidence: раньше любой `404` (в том числе от неверного адреса) означал «этап не найден» (`404` клиенту). route: patch → `404` без problem+json — нарушение контракта (`502`), «дерева нет» — только problem+json.
- **2026-10-21 · implementation · UI не различает «сопоставления нет» и «справочник недоступен»** — verdict: medium. evidence: пустая ячейка сопоставления выглядит как «сметчик не проверил позицию». route: patch → флаг `rates_unavailable` в ответе и предупреждение в таблице; пустое обогащение при живом справочнике остаётся без предупреждения.

## Design Notes

- Признак недоступности: `rates_unavailable` у ответа списка позиций, `source_unavailable` у ответа статуса обработки. Значение `false` — внешний сервис отвечал, отсутствие обогащения — не ошибка.
- Общий хелпер возвращает один из трёх исходов: «ресурса нет» (`None`), «недоступность» (доменная ошибка `*Unavailable`), «нарушение контракта» (`UpstreamContractViolation`). Ответ без `application/problem+json` никогда не трактуется как «ресурса нет».
- Порядок в `EstimateOperations`: сначала чтение БД в сессии, затем её закрытие, затем сетевой вызов, затем сборка ответа.
- Границы деградации: `RateCatalogUnavailable`/`PdfExtractUnavailable` (сеть, `5xx`) — деградируются там, где внешний сервис дополняет локальные данные (обогащение позиций, состояние файла); `CompanyOrgUnavailable` на подтверждении скоупа этапа — честный `502` (пустой список означал бы «смет нет» вместо «сервис недоступен»); `UpstreamContractViolation` (`422`, `404` не от сервиса) — не деградируется, это `502`.
- Семантика `404` по маршрутам: `GET /v1/contracts/tree` и `GET /v1/contracts/{id}/stages` — «договора/дерева нет в скоупе»; `GET /v1/files/{id}` — «файла нет»; `GET /v1/rate-catalog/versions/current/rates` и `GET /v1/rate-catalog/rates/{id}` — «расценки/версии нет»; `GET /v1/rate-catalog/rates?ids=` — «сервис не знает маршрут» (пусто приходит как `200 []`).

## Verification

**Commands (выполнено 2026-10-21):**
- `cd estimate && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `280 passed`
- `cd ui && npm run typecheck && npm run build && npm run test` → без ошибок, `89 passed` (vitest), сборка `dist/` собрана

**Что закрыто тестами:**
- Контракт хелпера: `404`+problem+json → «нет»; `404` без problem+json → нарушение контракта; `404` у неизвестного маршрута → нарушение контракта; `422` → нарушение контракта; сеть/`5xx` → недоступность (`estimate/tests/test_external_contracts.py`).
- Сквозной путь через API с настоящим HTTP-клиентом справочника: `404` не от сервиса и `422` → `502`; `503` → `200` с `rates_unavailable: true`, позиции и стоимость на месте (`test_external_contracts.py`).
- Деградация: позиции без `rate_code`/`rate_name` и `rates_unavailable: true`; стоимость позиции и итог версии сохраняются; `source_unavailable: true` при сбое pdf-extract — стадии маппинга/расчёта по своим данным, файловые помечены неизвестными; при живом сервисе признак `false` (`test_item_enrichment.py`, `test_source_and_status.py`).
- Сессия БД: сетевой вызов (`rate-catalog`, `pdf-extract`, `company-org`) выполняется при нулевом числе открытых сессий, при этом сессия для чтения данных действительно открывалась (`test_session_discipline.py`).
- `rate_code`/`rate_name` в ответах импорта и ручной проверки — `null` (`test_item_enrichment.py`).
- UI: предупреждение при `rates_unavailable` (позиции, стоимость и итоги видны) и при `source_unavailable` (стадии на месте, файловые — «неизвестно»); без недоступности предупреждения нет (`ui/tests/estimate_view.test.ts`, `ui/tests/upload_status.test.ts`).

**Не покрыто автоматически:** UI не проверялся в реальном браузере (e2e-контура в репозитории нет — только vitest/jsdom); живой стенд (compose) для этой истории не поднимался, поведение подтверждено тестами с настоящими HTTP-клиентами и подменённым транспортом.
