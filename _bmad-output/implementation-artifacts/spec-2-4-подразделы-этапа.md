---
title: 'Story 2.4 — Подразделы этапа (stage_subdivision)'
type: 'feature'
created: '2026-10-19'
status: 'cancelled'
route: 'dispatch'
baseline_commit: '30b9078beecc2cb5b823b9b70b18e26030317be3'
review_loop_iteration: 1
context:
  - '_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** `company-org` знает компании (2.1), договоры (2.2) и этапы (2.3), но подразделов нет: FR-002 не завершён, сметы не на что вешать внутри этапа, Epic 3–7 ждут.

**Approach:** вертикаль `stage_subdivision` в `company-org`: создание подраздела под этапом и листинг подразделов этапа; отдельная миграция Alembic. Арендатор — из `X-Company-Id` (AD-013), этап обязан принадлежать компании. Эндпоинты — `POST`/`GET /v1/contracts/{contract_id}/stages/{stage_id}/subdivisions` (решение: вариант A — вписывается в префикс `/v1/contracts`, правок `api-gateway`/`deploy` нет).

## Boundaries & Constraints

**Always:** AD-011 (слои внутрь; FastAPI/SQLAlchemy не в `domain`/`application`). AD-013: `company_id` только из `X-Company-Id`, фильтр на чтение и запись, без заголовка — отказ; этап чужой/несуществующей компании невидим. UUID; `application/problem+json` как в `contract`/`stage`; пути `/v1/...`; `snake_case`; `company_id`/`stage_id` клиент не задаёт. Подраздел адресуется парой (`stage_id`, `company_id`). Порядок — `created_at`, затем `id`; имена повторяемы (уникальности нет). `/health`, прежние роуты и их тесты, CI, `Dockerfile` — без изменений. `main.py` не менять (ленивая компоновка — как в 2.2/2.3).

**Never:** встраивание подразделов в листинг этапов или `GET /v1/contracts/tree`; update/move/delete, soft-delete, пагинация; правки контрактов компаний/договоров/этапов, `auth-users`; shared-lib; чужая БД; `Idempotency-Key` для подразделов; явное поле порядка (`position`).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| CREATE_SUBDIVISION | `POST …/stages/{sid}/subdivisions`, `X-Company-Id: C1`, `{"name":"Подраздел 1"}`, `sid`∈C1 | 201, `{id, company_id: C1, stage_id: sid, name, created_at}` | problem+json при невалидном теле |
| MULTIPLE_SUBDIVISIONS | второй POST по тому же `sid` | 201; у этапа два подраздела | N/A |
| LIST_SUBDIVISIONS | `GET …/stages/{sid}/subdivisions`, C1, у этапа подразделы | 200, массив только подразделов этого этапа, порядок `created_at`, затем `id` | N/A |
| LIST_EMPTY | этап без подразделов | 200, `[]` | N/A |
| MISSING_STAGE | `sid` не существует | 404, `count(stage_subdivision)` без изменений | problem+json |
| FOREIGN_STAGE | `sid`∈C2, заголовок C1 | 404, тело неотличимо от MISSING_STAGE | problem+json |
| CONTRACT_MISMATCH | `sid`∈C1, но `contract_id` в пути — другой договор C1 (этап не принадлежит договору пути) | 404, тело неотличимо от MISSING_STAGE; вставки нет | problem+json |
| MISSING_CONTRACT_PATH | `contract_id` в пути не существует | 404 (create и list) | problem+json |
| SCOPE_ISOLATION | у C2 есть подраздел своего этапа | листинг C1 его не содержит | N/A |
| SPOOFED_PAYLOAD | payload с `company_id`/`stage_id`/`id` | игнорируются; подраздел в C1 и этапе из пути | N/A |
| VALIDATION | `name` пустой/пробельный или отсутствует | 422 problem+json с полем | N/A |
| BAD_PATH_UUID | `sid` не UUID | 422 | problem+json |
| SCOPE_UNKNOWN | `X-Company-Id` неизвестна | 401 (create и list) | problem+json |
| MISSING_SCOPE | нет `X-Company-Id` | 401 до репозитория | problem+json |

</frozen-after-approval>

## Code Map

- `company-org/app/domain/stage.py:7-19` — образец сущности + factory → `subdivision.py`.
- `company-org/app/domain/errors.py:32-37` — `ContractNotFound` → аналог `StageNotFound` (404).
- `company-org/app/domain/ports.py:36-43` — `StageRepository` → `StageSubdivisionRepository(add, list_by_stage(stage_id, *, company_id))` (существующие сигнатуры не менять).
- `company-org/app/application/create_stage.py` / `list_stages.py` — паттерн scope-guard компании → проверка этапа в компании → `create_subdivision`/`list_subdivisions`.
- `company-org/app/infrastructure/models.py:36-47` — `StageRow` → `StageSubdivisionRow` (FK `company.id`, `stage.id` `RESTRICT`, оба index).
- `company-org/app/infrastructure/repository.py:73-94,135-157` — `SqlAlchemyStageRepository` и `_stage_to_*` → репозиторий/мапперы подразделов (с UTC-нормализацией `created_at`).
- `company-org/app/transport/schemas.py:65-84` — `CreateStageRequest`/`StageResponse` → `CreateSubdivisionRequest`/`SubdivisionResponse`.
- `company-org/app/transport/routes.py:42-51,153-181` — `CreateStageFn`/`ListStagesFn`, этапные роуты, ленивая компоновка → роуты подразделов.
- `company-org/app/transport/problem.py:57-70` — хендлеры 404 → 404 `StageNotFound`.
- `company-org/app/wiring.py:80-119` — раннеры этапов → `make_create_subdivision_runner`, `make_list_subdivisions_runner`.
- `company-org/migrations/versions/0003_create_stage.py` — формат ревизии → `0004_create_stage_subdivision`, `down_revision="0003"`.
- `company-org/tests/test_stages.py` — образец матрицы (sqlite-DSN, реальные миграции, `importlib.reload(main)`, `count`, spy, inspector) → `test_subdivisions.py`.
- `company-org/tests/test_migrations.py:34-72,109-123` — сигнатура до `0004`+`stage_subdivision`; `upgrade 0003` → `head`.
- `company-org/tests/test_contracts.py:397-420` — spy-тест `create_router`; учитывать новые параметры.
- `api-gateway/app/application/routing.py:42-47` — префикс `/v1/contracts` покрывает вложенный путь (для варианта A правок нет).
- `api-gateway/tests/test_gateway.py:260-300` — образцы тестов вложенного пути → добавить `…/subdivisions`.

## Tasks & Acceptance

**Execution:**
- [x] `company-org/app/domain/subdivision.py` -- `StageSubdivision(id, company_id, stage_id, name, created_at)` + factory.
- [x] `company-org/app/domain/ports.py` -- `StageSubdivisionRepository(add, list_by_stage(stage_id, *, company_id))`.
- [x] `company-org/app/domain/errors.py` -- `StageNotFound(stage_id)` (404).
- [x] `company-org/app/application/create_subdivision.py` -- scope-guard компании → этап существует в компании **и принадлежит договору из пути** (`stage.contract_id == contract_id`) → add; чужой/несуществующий этап или этап другого договора → `StageNotFound`.
- [x] `company-org/app/application/list_subdivisions.py` -- scope-guard → та же проверка договора → подразделы этапа, порядок `created_at`, `id`.
- [x] `company-org/app/infrastructure/models.py` -- `StageSubdivisionRow`: FK `company_id`→`company.id`, `stage_id`→`stage.id` (`RESTRICT`), оба index.
- [x] `company-org/app/infrastructure/repository.py` -- `SqlAlchemyStageSubdivisionRepository` + мапперы; `company_id` обязателен.
- [x] `company-org/app/transport/schemas.py` -- `CreateSubdivisionRequest` (`name` непустой после strip), `SubdivisionResponse`.
- [x] `company-org/app/transport/routes.py` -- `POST`/`GET` подразделов: проброс `contract_id` из пути в use case + чтение `X-Company-Id`; ленивая компоновка.
- [x] `company-org/app/transport/problem.py` -- 404 `StageNotFound`.
- [x] `company-org/app/wiring.py` -- раннеры подразделов (сессия на запрос).
- [x] `company-org/migrations/versions/0004_create_stage_subdivision.py` -- таблица (`down_revision="0003"`, обе FK, индексы).
- [x] `company-org/tests/test_subdivisions.py` -- матрица: `count` до/после (MISSING/FOREIGN), тела MISSING = FOREIGN, игнор `company_id`/`stage_id` в payload, изоляция компаний, порядок `created_at`,`id`, `[]`, **CONTRACT_MISMATCH и MISSING_CONTRACT_PATH → 404 = MISSING_STAGE без вставки**, spy без БД при MISSING_SCOPE, inspector FK/индексов, UTC-`created_at`.
- [x] `company-org/tests/test_migrations.py` -- сигнатура до `0004`+`stage_subdivision`; `upgrade 0003` → `head` сохраняет строку.
- [x] `api-gateway/tests/test_gateway.py` -- вложенный путь `…/subdivisions`: без токена 401 без upstream (POST и GET); с токеном assert `host`/`path` (201 create, 200 list).

**Acceptance Criteria:**
- Given существующий этап C1, when создаёт подраздел, then `stage_id` = путь, `company_id` = заголовок; несколько подразделов на этап возможны.
- Given у этапа C1 два подраздела, when `GET …/subdivisions` с `X-Company-Id: C1`, then только подразделы этапа в порядке `created_at`, `id`.
- Given подраздел существует у C2, when создание/чтение под заголовком C1, then 404 либо пусто, данные C2 не утекают.
- Given этап принадлежит договору A, when создание/чтение по пути с другим договором или несуществующим `contract_id` (та же компания), then 404, тело как у MISSING_STAGE, вставки нет.
- Given чистая БД, when `alembic upgrade head` дважды, then применены `0001`–`0004`, повтор идемпотентен.
- Given нет `X-Company-Id`, when любой эндпоинт подразделов, then 401 problem+json до обращения к БД.
- Given прежние тесты, when полный прогон, then зелёные; `company`/`contract`/`stage` контракты и `/health` не изменены.

## Implementation Notes

## Spec Change Log

### 2026-10-20: история отменена человеком (не loopback)

- **Решение владельца:** сущность `stage_subdivision` упразднена целиком — сметы привязываются напрямую к этапу (`estimate.stage_id`), уровень «Подраздел этапа» убран из доменной иерархии.
- **Что сделано:** revert коммитов истории в `company-org` (`703ec4f`) и `api-gateway` (`b2b0a4d`); дерево вернулось к baseline (47/28 passed, ruff чисто). Статус истории — `cancelled`, ключ удалён из `sprint-status.yaml`, Story 2.4 удалён из `epics.md`, FR-002 и `ARCHITECTURE-SPINE`/`OVERVIEW` поправлены (`estimate.stage_subdivision_id` → `stage_id`).
- **Не откатывается (история):** `<frozen-after-approval>` и Review Triage Log остаются записью о том, что была реализована и прошла 3-слойное ревью с loopback. Итерация `review_loop_iteration: 1` сохранена.
- **Хранить для будущего:** если сущность подразделов вернётся (решение по гранулярности привязки смет — см. `docs/brief.md`, открытый вопрос), откатывать отсюда, а не переизобретать: вся матрица, Keep-инструкции и scoping-паттерн (`contract_id` из пути) описаны ниже в логе итераций 1–2.

### Итерация 1 → 2 (loopback `bad_spec`)

- **Триггерная находка:** G1 {B1, E1, V1, B2, B3} — `contract_id` из пути принимался, но не сверялся с договором этапа; `routes.py` не пробрасывал его в use case, матрица/тесты не варьировали договор независимо от этапа. Эмпирически: `POST /v1/contracts/{несуществующий}/stages/{sid}/subdivisions` → 201, GET → 200 (known-bad).
- **Корень:** спецификация не задавала scoping `contract_id` из пути. Откат к baseline-деревьям `company-org@27a7d7e`, `api-gateway@6d97d68` до аменда.
- **Аменд (non-frozen):** в I/O-матрицу добавлены строки `CONTRACT_MISMATCH` и `MISSING_CONTRACT_PATH`; задача на сверку `stage.contract_id == contract_id` (404 `StageNotFound`); проброс `contract_id` в `routes.py`; тест-кейсы на оба случая; AC и Design Notes; счётчик `review_loop_iteration` 1.
- **KEEP при пере-выводе:** вертикаль `stage_subdivision` (слои/порты/ошибки), миграция `0004`, UTC-маппер, матрица тестов и teardown-фикстура на реальных миграциях — сохранить, пере-выводить по спеке.

## Review Triage Log

Итерация 1. 16 находок (blind-hunter 14, edge-case-hunter 1, verification-gap 1 «Other findings»); gaps-находок слой verification-gap не дал.

| # | Finding (слой) | Verdict | Evidence |
|---|----------------|---------|----------|
| 1 | B1: `contract_id` из пути принимается, но не сверяется с договором этапа (blind) | medium | `routes.py` `create/list_subdivision_endpoint` не передают `contract_id` в use case; `create_stage.py:24` (соседний роут) путь-guard делает. Реальный пробел scoping'а пути. |
| 2 | E1: тот же факт — этап чужого договора той же компании принимается (edge) | medium | Совпадает с B1; подтверждено поведением. |
| 3 | V1: тот же факт; слой подтвердил эмпирически 201/200 для несуществующего договора (verification-gap) | medium | Проверено на реальных миграциях и `main.app`: `POST /v1/contracts/{несуществующий}/stages/{sid}/subdivisions` → 201, GET → 200. Явно неверно. |
| 4 | B2: в I/O-матрице нет строки для этапа из другого договора / чужого `contract_id` в пути (blind) | medium | Матрица (`spec-2-4`, строки 29–42) обрывается на SPOOFED_PAYLOAD; корень пробела — тот же. |
| 5 | B3: тесты никогда не варьируют `contract_id` независимо от `stage_id` (blind) | medium | `tests/test_subdivisions.py:961,988` меняют только компанию, поэтому scoping договора не поймали бы. |
| 6 | B4: расхождение `sprint-status.yaml` (`in-progress`) и статуса спеки (`in-review`) (blind) | false | Ведение спринта опаздывает на шаг; синхронизирует step-05. Состояния 2-1…2-3 `review` намеренны. `reject` |
| 7 | B5: диф не содержит саму спеку и даёт абсолютные пути для новых файлов (blind) | low | Артефакт стейджинга диффа (`git diff --no-index /dev/null <abs>`); спека исключена как claims-файл по правилу шага. Правок продукта не требует. `reject` |
| 8 | B6: нет композитного FK `(stage_id, company_id)`, tenant-инвариант держит приложение (blind) | low | Как в 2.3 (B3): через API недостижимо (`create_subdivision.py:24-26`), фикс — FK+миграция → сложнее прямой правки. `reject` |
| 9 | B7: пустое имя защищено только в `CreateSubdivisionRequest`, не в домене/БД (blind) | low | Валидация — на transport, как у `company`/`contract`/`stage` (2.1/2.2/2.3); не-HTTP вызывающих нет. `reject` |
| 10 | B8: не проверена граница `max_length` (255/256) (blind) | low | Дефекта нет; аддитивный тест. `reject` |
| 11 | B9: нет индекса под фильтр+сортировку (`stage_id`,`company_id` + `created_at`,`id`) (blind) | low | Объёмы MVP малы; тот же паттерн, что у списка этапов. `reject` |
| 12 | B10: раннеры в `wiring.py` — шестая/седьмая копия; `add` коммитит дважды (blind) | false | Зеркалит принятый паттерн 2.1–2.3; наблюдаемого вреда нет (B6/B9 итерации 2.3). `reject` |
| 13 | B11: docstring порта обещает guard при `None`, но `company_id` не optional (blind) | false | Как `ContractRepository`/`StageRepository`; ветка недостижима, вреда нет. `reject` |
| 14 | B12: тестовый каркас скопирован, а не вынесен в conftest (blind) | low | Только тесты; в репозитории нет конвенции conftest (2.3 так же). `reject` |
| 15 | B13: `line-length=100` не форсится, `ruff format` не запускается (blind) | false | Конвенция репозитория — `ruff check` (E501 не в конфиге правил по факту прохождения). Не дефект. `reject` |
| 16 | B14: регрессия прозы — «докомпонуются» вместо «докомпоновываются» (blind) | low | Косметика docstring; код пере-выводится в loopback. `moot` |

**Группы и маршрутизация:**
- G1 {B1, E1, V1, B2, B3} — корень: в спецификации не задан scoping `contract_id` из пути, поэтому код его игнорирует → **bad_spec** → loopback (пере-вывод кода по уточнённой спеке).
- Отклонены: B4 `false`; B5, B6, B7, B8, B9, B12 `low`; B10, B11, B13 `false`; B14 `low` (moot после loopback).

Loopback: `bad_spec` (корень вне `<frozen-after-approval>`). KEEP: вертикаль `stage_subdivision` (слои, порты, миграция `0004`, UTC-маппер, матрица тестов, teardown-фикстура на реальных миграциях) сохранена как есть. Код откачен; в спеку добавлены scoping договора и недостающие строки/задачи.

Итерация 2 (после loopback). 10 находок (blind-hunter 15, edge-case-hunter 1, verification-gap 1 regression gap); совпавшие с итерацией 1 несут `carried`.

| # | Finding (слой) | Verdict | Evidence |
|---|----------------|---------|----------|
| 17 | BH1: нет композитного FK `(stage_id, company_id)`, tenant-инвариант держит приложение (blind) | low | `carried` — тот же факт и место, что строка 8 (B6); код читается так же. `reject` |
| 18 | BH2/E1: пустое/пробельное имя защищено только в `CreateSubdivisionRequest`, домен/БД принимают (blind/edge) | low | `carried` — строка 9 (B7); не-HTTP вызывающих нет. `reject` |
| 19 | BH3: нет теста границы `max_length` (255/256) (blind) | low | `carried` — строка 10 (B8); аддитивный тест. `reject` |
| 20 | BH4: индекс не под фильтр+сортировку (blind) | low | `carried` — строка 11 (B9); объёмы MVP малы. `reject` |
| 21 | BH5: репозиторий не задаёт `ORDER BY`; порядок навязан в Python; «порт обещает порядок» (blind) | false | Посылка неверна: docstring `list_by_stage` (`ports.py:44-45`) порядка не обещает; порядок гарантирует use case (`list_subdivisions.py:36`), и только он — потребитель. `reject` |
| 22 | BH6: `_assert_stage_belongs_to_contract` продублирован дословно в create/list (blind) | low | Наблюдаемого расхождения нет; фикс — новый общий хелпер/модуль (сложнее прямой правки). `reject` |
| 23 | BH7: владение этапом проверяется полным `list_by_contract` + `any` (O(N)) (blind) | low | Несущественно при MVP-объёмах; точечный lookup — новый метод порта (публичная поверхность). `reject` |
| 24 | BH8: guard `company_id is None` в `add` недостижим; порт `UUID` vs реализация `UUID \| None` (blind) | false | `carried` — строка 13 (B11); ветка недостижима, вреда нет. `reject` |
| 25 | BH9/BH15: дубль коммита и шестая/седьмая копия раннера в `wiring.py` (blind) | false | `carried` — строка 12 (B10); зеркалит принятый паттерн. `reject` |
| 26 | BH10: docstring `test_subdivisions.py` обещает «15 сценариев», а тестов 17 при 14 строках матрицы (blind) | low | Реальная неточность, введённая этим изменением; фикс — прямая правка. **patch** |
| 27 | BH11: naive-ветка `_subdivision_to_domain` не покрыта тестами (blind) | false | sqlite отдаёт naive `created_at`, поэтому `test_created_at_is_tz_aware_and_stable_between_create_and_list:185-197` проходит лишь через `replace(tzinfo=UTC)`. `reject` |
| 28 | BH12: нет теста `downgrade` `0004`; upgrade-тест не проверяет колонки/FK/индексы (blind) | low | Колонки/FK/индексы покрыты `test_stage_subdivision_table_constraints_and_indexes` и сигнатурой миграций; непокрыт лишь `downgrade` — аддитивно, не повседневно. `reject` |
| 29 | BH13: `SubdivisionResponse` без `contract_id`, в отличие от соседних ответов (blind) | false | Форма ответа задана intent/матрицей (`{id, company_id, stage_id, name, created_at}`); отклонение от соседей не дефект. `reject` |
| 30 | BH14: диф не применяется одним патчем (смешаны абсолютные/относительные пути и два репо) (blind) | false | Артефакт стейджинга дифа (`git diff --no-index /dev/null <abs>`), не продукт. `reject` |
| 31 | VG1: эквивалентность 404-тела на read-пути не проверена — GET-ассерты сравнивают только `status_code` (verification-gap, regression gap) | medium | Пре-верифицировано слоем: дивергенция (`ContractNotFound` вместо `StageNotFound` в `ListSubdivisions`) прошла бы; подтверждает строки 295/320/352 против POST-эквивалентности 268/296/321. **patch** |

**Группы и маршрутизация (итерация 2):**
- Loopback не требуется: `intent_gap`/`bad_spec` нет. `review_loop_iteration` остаётся 1.
- patch: BH10 (docstring), VG1 (read-тесты) — обрабатываются авто-фиксом.
- Отклонены: BH5/BH8/BH9/BH11/BH13/BH14/BH15 `false`; BH1/BH2/BH3/BH4/BH6/BH7/BH12/E1 `low` (`carried` там, где совпало с итерацией 1).

## Design Notes

- Форма пути (вариант A) сохраняет неизменным edge-префикс `/v1/contracts`, поэтому `api-gateway` и `deploy` правок не требуют; сам этап читается из `X-Company-Id` (AD-013), не из пути как tenant.
- 404 для чужого этапа тем же телом, что для несуществующего, — чтобы не утекало существование чужих id (как `contract`/`stage`).
- Порядок `created_at`, затем `id` (без `position`), без уникальности имён — те же решения, что в 2.2/2.3; переупорядочивание и ограничения вне MVP.
- Листинг этапов и `GET /v1/contracts/tree` подразделами не расширяются: подразделы читаются своим эндпоинтом.
- `contract_id` из пути не декоративен: этап обязан принадлежать этому договору (и компании из заголовка), иначе 404 телом MISSING_STAGE — тем же, что для чужого/несуществующего этапа. Это зеркалит путь-guard соседнего `create_stage` (проверка родителя из пути), поэтому договор, указанный в URL, не может молча не совпасть с этапом.

## Verification

**Commands:**
- `uv run --extra dev --directory /home/shved/projects/smeta/company-org pytest -q` -- прежние + новые (вкл. `test_migrations.py`).
- `cd /home/shved/projects/smeta/company-org && uvx ruff check .` → `All checks passed!`
- `uv run --extra dev --directory /home/shved/projects/smeta/api-gateway pytest -q` -- вкл. тесты вложенного пути.
- `cd /home/shved/projects/smeta/api-gateway && uvx ruff check .` → `All checks passed!`
- `docker compose -f /home/shved/projects/smeta/deploy/docker-compose.yml config -q` → OK (deploy не менялся).

**Manual checks (if no CLI):**
- Без заголовка — `application/problem+json` 401; чужой этап даёт тело, равное несуществующему; `count(stage_subdivision)` по матрице.
