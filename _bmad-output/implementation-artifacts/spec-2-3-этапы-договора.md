---
title: 'Story 2.3 — Этапы договора (stage)'
type: 'feature'
created: '2026-10-19'
status: 'done'
route: 'dispatch'
baseline_commit: '96d21793fcd988d97ed466aa259fa42504c91ecb'
review_loop_iteration: 1
context:
  - '_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** `company-org` знает компании и договоры (Story 2.2), но этапов нет: FR-002 не продолжается, Stories 2.4 и сметы не на что вешать; договор нельзя сгруппировать по этапам работ.

**Approach:** вертикаль `stage` в `company-org`: создание этапа под договором (tenant из `X-Company-Id`) и листинг этапов договора; отдельная миграция Alembic. Арендатор доказывается заголовком (AD-013), договор обязан принадлежать компании.

## Boundaries & Constraints

**Always:** AD-011 (слои внутрь; FastAPI/SQLAlchemy не в `domain`/`application`). AD-013: `company_id` только из `X-Company-Id`, фильтр на чтение и запись, без заголовка — отказ; договор чужой/несуществующей компании невидим. UUID; `application/problem+json` как в `contract`; пути `/v1/...`; `snake_case`; `company_id` клиент не задаёт; этап адресуется парой (`contract_id`, `company_id`). Порядок этапов — `created_at`, затем `id`; имена этапов повторяемы (уникальности нет) — это закрывает deferred-пункт «Порядок этапов и уникальность имён» из ARCHITECTURE-SPINE. `/health`, прежние роуты и их тесты, CI, `Dockerfile` — без изменений. `main.py` не менять (компоновка — как в 2.2).

**Never:** Story 2.4 (подразделы этапа); встраивание этапов в `GET /v1/contracts/tree`; update/move/delete, soft-delete, пагинация; правки `contract`-контракта, `auth-users`, `api-gateway`, `deploy`; shared-lib; чужая БД; `Idempotency-Key` для этапов; явное поле порядка (`position`).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| CREATE_STAGE | `POST /v1/contracts/{cid}/stages`, `X-Company-Id: C1`, `{"name":"Этап 1"}`, `cid` принадлежит C1 | 201, `{id, company_id: C1, contract_id: cid, name, created_at}` | problem+json при невалидном теле |
| MULTIPLE_STAGES | второй POST по тому же `cid` | 201; у договора два этапа | N/A |
| LIST_STAGES | `GET /v1/contracts/{cid}/stages`, C1, у договора этапы | 200, массив только этапов этого договора, порядок `created_at`, затем `id` | N/A |
| LIST_EMPTY | договор без этапов | 200, `[]` | N/A |
| MISSING_CONTRACT | `cid` не существует | 404, `count(stage)` без изменений | problem+json |
| FOREIGN_CONTRACT | `cid` принадлежит C2, заголовок C1 | 404, тело неотличимо от MISSING_CONTRACT | problem+json |
| SCOPE_ISOLATION | у C2 есть этап своего договора | листинг C1 его не содержит | N/A |
| SPOOFED_PAYLOAD | payload с `company_id`/`contract_id`/`id` | игнорируются; этап в C1 и договоре из пути | N/A |
| VALIDATION | `name` пустой/пробельный или отсутствует | 422 problem+json с полем | N/A |
| BAD_PATH_UUID | `cid` не UUID | 422 | problem+json |
| SCOPE_UNKNOWN | `X-Company-Id` неизвестна | 401 (create и list) | problem+json |
| MISSING_SCOPE | нет `X-Company-Id` | 401 до репозитория | problem+json |

</frozen-after-approval>

## Code Map

- `company-org/app/domain/contract.py:7-19` — образец сущности + factory → `stage.py`.
- `company-org/app/domain/errors.py:24-37` — образец доменных ошибок → `ContractNotFound` (404).
- `company-org/app/domain/ports.py:22-31` — `ContractRepository` → `StageRepository(add, list_by_contract(contract_id, *, company_id))` (существующие сигнатуры не менять).
- `company-org/app/application/create_contract.py:25-35` — use case со scope-guard договора → `create_stage` повторяет паттерн.
- `company-org/app/application/list_contract_tree.py:35-39` — scope-guard + сортировка `created_at`, `id`.
- `company-org/app/infrastructure/models.py:13-33` — `Base`/`CompanyRow`/`ContractRow` → `StageRow` (FK `company.id` и `contract.id`, индексы).
- `company-org/app/infrastructure/repository.py:39-68` — `SqlAlchemyContractRepository` и `_contract_to_*` → `SqlAlchemyStageRepository`, `_stage_to_domain`/`_stage_to_row`.
- `company-org/app/transport/schemas.py:29-62` — `field_validator`/`from_attributes` → `CreateStageRequest`/`StageResponse`.
- `company-org/app/transport/routes.py:39-122` — `_require_company_id`, ленивая компоновка, контрактные роуты → этапные роуты.
- `company-org/app/transport/problem.py:57-70` — хендлеры доменных ошибок → 404 `ContractNotFound`.
- `company-org/app/wiring.py:36-73` — раннеры → `make_create_stage_runner`, `make_list_stages_runner`.
- `company-org/migrations/versions/0002_create_contract.py` — формат ревизии → `0003_create_stage`, `down_revision="0002"`.
- `company-org/tests/test_contracts.py:19-43,80-102` — sqlite-DSN, реальные миграции, `importlib.reload(main)`, `count`-хелперы.
- `company-org/tests/test_migrations.py:34-72` — сигнатура схемы: расширить до `0003` + `stage`.
- `api-gateway/app/application/routing.py:43-47` — префикс `/v1/contracts` уже покрывает вложенный путь; правок нет.
- `api-gateway/tests/test_gateway.py:212-259` — образцы тестов префикса; добавить `.../{cid}/stages`.

## Tasks & Acceptance

**Execution:**
- [x] `company-org/app/domain/stage.py` -- `Stage(id, company_id, contract_id, name, created_at)` + factory.
- [x] `company-org/app/domain/ports.py` -- `StageRepository(add, list_by_contract(contract_id, *, company_id))`.
- [x] `company-org/app/domain/errors.py` -- `ContractNotFound(contract_id)` (404).
- [x] `company-org/app/application/create_stage.py` -- scope-guard компании → договор существует в компании → add; чужой/несуществующий → `ContractNotFound`.
- [x] `company-org/app/application/list_stages.py` -- scope-guard → этапы договора, порядок `created_at`, `id`.
- [x] `company-org/app/infrastructure/models.py` -- `StageRow`: FK `company_id`→`company.id`, `contract_id`→`contract.id` (`RESTRICT`), оба index.
- [x] `company-org/app/infrastructure/repository.py` -- `SqlAlchemyStageRepository` + мапперы; `company_id` обязателен.
- [x] `company-org/app/transport/schemas.py` -- `CreateStageRequest` (`name` непустой после strip, лишние поля игнорируются), `StageResponse`.
- [x] `company-org/app/transport/routes.py` -- `POST`/`GET` `/v1/contracts/{contract_id}/stages` + чтение `X-Company-Id`.
- [x] `company-org/app/transport/problem.py` -- 404 `ContractNotFound`.
- [x] `company-org/app/wiring.py` -- раннеры этапов (сессия на запрос).
- [x] `company-org/migrations/versions/0003_create_stage.py` -- `stage` (`down_revision="0002"`, обе FK, индексы).
- [x] `company-org/tests/test_stages.py` -- матрица с точными ассертами: `count(stage)` до/после (MISSING/FOREIGN), тела MISSING = FOREIGN, игнор `company_id`/`contract_id` в payload, изоляция компаний, порядок `created_at`,`id`, `[]` у LIST_EMPTY, spy без обращения к БД при MISSING_SCOPE, inspector FK/индексов.
- [x] `company-org/tests/test_migrations.py` -- сигнатура до `0003`+`stage`; `upgrade 0002` → `contract` → `head` сохраняет строку.
- [x] `api-gateway/tests/test_gateway.py` -- вложенный путь `.../{cid}/stages`: без токена 401 без upstream; с токеном assert `host`/`path`.

**Acceptance Criteria:**
- Given чистая БД, when `alembic upgrade head` дважды, then применены `0001`–`0003`, повтор идемпотентен.
- Given у договора C1 два этапа, when `GET .../stages` с `X-Company-Id: C1`, then вернулись только этапы этого договора в порядке `created_at`, `id`.
- Given этап существует у C2, when создание/чтение под заголовком C1, then 404 либо пусто, этапы C2 не утекают.
- Given существующий договор, when создаёт этап, then `contract_id` = путь, `company_id` = заголовок; несколько этапов на договор возможны.
- Given нет `X-Company-Id`, when любой эндпоинт этапов, then 401 problem+json до обращения к БД.
- Given прежние тесты, when полный прогон, then зелёные, `company`/`contract` контракты и `/health` не изменены.

## Implementation Notes

## Spec Change Log

## Review Triage Log

Итерация 1. 15 находок (blind-hunter 13, edge-case-hunter 1, verification-gap 1 «Other findings»); verification-gap gaps-находок не дал.

| # | Finding (слой) | Verdict | Evidence |
|---|----------------|---------|----------|
| 1 | B1: `test_contract_stages_nested_path_proxied_to_company_org` одним стабом `httpx.Response(201,…)` ассертит GET-листинг как 201 (blind) | low | `api-gateway/tests/test_gateway.py:272-296`: стаб один на все методы, `list_resp.status_code == 201`; реальный GET листинга — 200 (ср. `:223-237`). Вред продуктовый нулевой, но тест закрепляет неверный статус и не отличает листинг. `patch` |
| 2 | V1: тот же факт, подан как «не verification gap» (verification-gap) | low | Тот же локус `test_gateway.py:272-296`; слой сам подтвердил, что шлюз форвардит статус апстрима как есть — дефекта компании-org нет, это фиктивность теста. `patch` (в группе с B1) |
| 3 | B11: вложенный путь без токена покрыт только GET, POST не проверен (blind) | low | `test_gateway.py:260-269` — только `client.get`; auth-правило префиксное (`/v1/…` кроме auth), POST `…/stages` должен так же давать 401. Пробел теста. `patch` (в группе с B1) |
| 4 | B2: `test_missing_scope_rejected_before_repositories` не обновлён под новую сигнатуру (blind) | low | `tests/test_contracts.py:416` вызывает `create_router(spy_create_company, spy_create_contract, spy_list_tree)` → `create_stage`/`list_stages = None` → ленивая компоновка реальных раннеров из `wiring` (`make_session_factory()`). Ассерт `calls == []` и 401 проходит (заголовка нет), но изоляция теста ослабла. `patch` |
| 5 | B5: `created_at` POST (aware) ≠ GET (naive), не тестируется (blind) | low | Измерено: POST `'…736636Z'`, GET `'…736636'` (sqlite + `DateTime(timezone=True)` отдаёт naive). Реальная нестыковка сериализации между двумя эндпоинтами одной сущности. `patch` |
| 6 | B3: нет структурного tenant-инварианта (два независимых FK) (blind) | low | `models.py:78-89` / миграция `0003`: FK на `company.id` и `contract.id` отдельно, композитного `(contract_id, company_id)→contract(id, company_id)` нет. Через API недостижимо (`create_stage.py:24`), но инвариант держится только приложением. Фикс — композитный FK+миграция → сложнее прямой правки. `reject` |
| 7 | E1: `repository.add` не блокирует `contract_id=None`/чужого тенанта, FK-нарушение → 500 (edge) | false | Гвард выше по стеку: `create_stage.py:24` (`ContractNotFound` до `add`), `contract_id` всегда из пути-UUID; эндпоинта удаления договора нет → состояние недостижимо. `reject` на опровержении |
| 8 | B4: в репозитории нет `ORDER BY`, порядок только in-memory (blind) | false | `list_stages.py:25` сортирует `(created_at, id)` до возврата — клиент всегда получает заданный порядок; «неотсортированность» наружу не наблюдаема. `reject` |
| 9 | B6: `repository.add` коммитит, раннер коммитит/роллбэчит ещё раз (blind) | false | Зеркалит `SqlAlchemyCompanyRepository`/`ContractRepository` (2.1/2.2). После `repository.add` в use case ничто не падает (`create_stage.py:27-28`), поэтому «потерянный откат» недостижим. `reject` |
| 10 | B7: нет теста downgrade `0003`→`0002` (blind) | low | `test_migrations.py` покрывает только `upgrade head` (идемпотентность, сохранность) — как и для `0001/0002`. AC downgrade не требует; фикс — новый тест. `reject` |
| 11 | B8: не тестируются trim и граница `max_length=255` (blind) | low | Проверено: `"  Этап  "`→`"Этап"` работает (`schemas.py` validator), но отдельного ассерта нет; поведение корректно, дефекта нет. `reject` |
| 12 | B9: `extra="ignore"` не задокументирован, чужие ключи не форбид (blind) | false | Pydantic по умолчанию `extra="ignore"` — спека прямо требует «лишние поля игнорируются»; поведение соблюдено, вреда нет. `reject` |
| 13 | B10: нет `IntegrityError`-обработки в `add` при гонке с удалением договора (blind) | false | Удаление договора не реализовано (нет эндпоинта) → гонка недостижима; апстрим-гвард `create_stage.py:24` отрабатывает. `reject` |
| 14 | B12: README сервисов не обновлён под новый эндпоинт (blind) | false | `company-org/README.md` и `api-gateway/README.md` не перечисляют эндпоинты (`v1/contracts`/`v1/companies` не встречаются) — конвенции документации API нет. `reject` |
| 15 | B13: `sprint-status.yaml` (2-2 `review`, формат даты) (blind) | false | Ведение спринта вне периметра диффа; `2-2: review` — намеренное состояние, формат `MM-DD-YYYY` — конвенция проекта (`generated: '2026-10-17'`). `reject` |

**Группы и маршрутизация:**
- G1 {B1, V1, B11} — корень: тест вложенного пути неполон/нестрог → low → **patch**.
- G2 {B2} — корень: не обновлённый spy-тест под новую сигнатуру роутера → low → **patch**.
- G3 {B5} — корень: naive `created_at` у stage при чтении из БД → low → **patch**.
- G4 {B3, E1} — корень: нет структурного tenant-инварианта на персистентности → low → **reject**.
- B4, B6, B9, B10, B12, B13 → `false`, отклонены на опровержении.
- B7, B8 → low без дефекта, отклонены.

Loopback не требуется (нет intent_gap/bad_spec). Патчи G1–G3 отправлены тому же реализационному субагенту (`ses_eaa9445feffeNTGqcxbJQ5IfZ4`).

## Design Notes

- Вложенный путь `/v1/contracts/{cid}/stages` выбран, чтобы edge-префикс `/v1/contracts` уже маршрутизировал эндпоинт (правок `api-gateway` нет) и связь читалась из пути; `company_id` при этом — только из заголовка (AD-013), не из пути.
- 404 для чужого договора тем же телом, что для несуществующего, — чтобы не утекало существование чужих id (как FOREIGN/MISSING parent в 2.2).
- Порядок `created_at`, затем `id` (без поля `position`) — как дерево договоров 2.2; переупорядочивание вне MVP. Уникальность имён не вводится: дубликаты допустимы, ограничение — отдельной историей. Оба решения закрывают deferred-пункт ARCHITECTURE-SPINE.
- Расширение `GET /v1/contracts/tree` этапами отклонено: контракт дерева (и его тесты) 2.2 не меняется; этапы читаются своим эндпоинтом.

## Verification

**Commands:**
- `uv run --extra dev --directory /home/shved/projects/smeta/company-org pytest -q` — прежние + новые (вкл. `test_migrations.py`).
- `cd /home/shved/projects/smeta/company-org && uvx ruff check .` → `All checks passed!`
- `uv run --extra dev --directory /home/shved/projects/smeta/api-gateway pytest -q` -- вкл. тесты вложенного пути.
- `cd /home/shved/projects/smeta/api-gateway && uvx ruff check .` → `All checks passed!`
- `docker compose -f /home/shved/projects/smeta/deploy/docker-compose.yml config -q` → OK (deploy не менялся).

**Manual checks (if no CLI):**
- Без заголовка — `application/problem+json` 401; чужой договор даёт тело, равное несуществующему; `count(stage)` по матрице.
</content>
