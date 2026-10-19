---
title: 'Story 2.2 — Договоры произвольной глубины (adjacency list)'
type: 'feature'
created: '2026-10-19'
status: 'done'
route: 'dispatch'
baseline_commit: '1a94823d01375b1c3a22bb570252782ac6c05277'
review_loop_iteration: 1
context:
  - '_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** в `company-org` есть только компания (`POST /v1/companies`) — иерархии FR-002 нет, Stories 2.3/2.4 не на чем строить; identity-заголовки edge (AD-010) сервис ещё не читает, tenancy (AD-013) на доменных данных не доказана.

**Approach:** вертикаль `contract` (adjacency list): создание с опциональным `parent_id` + дерево компании целиком; tenancy из `X-Company-Id`; валидация иерархии в application-слое (AD-015); отдельная миграция Alembic.

## Boundaries & Constraints

**Always:** AD-011 (слои внутрь; FastAPI/SQLAlchemy не в `domain`/`application`). AD-013: `company_id` только из `X-Company-Id`, фильтр на чтение и запись, без заголовка — отказ. AD-015: подъём по `parent_id` до корня, отказ при самоссылке, цикле, чужом/несуществующем родителе. AD-006: ограничитель шагов подъёма — защита от зацикливания, не лимит вложенности. UUID; `application/problem+json` как в `auth-users`; пути `/v1/...`; `snake_case`; `company_id` клиент не задаёт. Роут-таблица edge получает один префикс `/v1/contracts` (Q1=B). `/health`, `POST /v1/companies`, их тесты, CI, `Dockerfile` — без изменений.

**Never:** Stories 2.3/2.4; контракт `POST /v1/companies` и `auth-users/app/infrastructure/company_client.py`; правки `api-gateway` сверх одного префикса; `deploy`, `auth-users`; update/move/delete, soft-delete, массовые операции, пагинация; роли >1, мультикомпанийность («вступить в компанию»); shared-lib; чужая БД; `Idempotency-Key` для договоров.

## I/O & Edge-Case Matrix

Решения ревью: цикл на create → 409; цикл при сборке дерева → 500; несуществующая компания в заголовке → 401.

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| CREATE_ROOT | `POST /v1/contracts`, `X-Company-Id: C1`, `{"name":"Договор А"}` | 201, плоский узел `{id, company_id: C1, parent_id: null, name, created_at}` | problem+json при невалидном теле |
| CREATE_CHILD | `parent_id` — договор компании C1 | 201, узел с этим `parent_id` (вложенность видна в дереве, не в ответе) | N/A |
| MISSING_PARENT | `parent_id` не существует | 404, `count(contract)` не меняется | problem+json |
| FOREIGN_PARENT | `parent_id` другой компании | 404, тело идентично MISSING_PARENT | problem+json |
| SPOOFED_PAYLOAD | тело с `company_id`/`id` другой компании | поля игнорируются, договор создан в C1 из заголовка | 422 при несовместимых типах |
| VALIDATION | `name` пустой/из пробелов | 422 | problem+json со списком полей |
| CYCLE_IN_DB | цикл в БД, создаётся узел с `parent_id` внутри него | 409, строк не добавлено | problem+json |
| TREE_HAPPY | `GET /v1/contracts/tree`, `X-Company-Id: C1` | 200, массив корней с рекурсивными `children`, порядок `created_at`, затем `id` | N/A |
| TREE_EMPTY | у компании нет договоров | 200 и ровно `[]` | N/A |
| SCOPE_ISOLATION | договоры C1 и C2, заголовок C1 | ни одного узла C2 в ответе | N/A |
| TREE_CYCLIC | цикл в БД, запрошено дерево | 500, узлы не отдаются частично | problem+json |
| SCOPE_UNKNOWN | `X-Company-Id` называет компанию, которой нет в `company` | отклонено до записи | 401 problem+json |
| MISSING_SCOPE | нет `X-Company-Id` | до репозитория не доходит | 401 problem+json |

</frozen-after-approval>

## Code Map

- `company-org/app/domain/company.py:7-18`, `errors.py:4-13` — образцы сущности/factory и доменных ошибок.
- `company-org/app/domain/ports.py:7-14` — `CompanyRepository` (существующие сигнатуры не менять).
- `company-org/app/application/create_company.py:9-31` — образец use-case.
- `company-org/app/infrastructure/models.py:9-19`, `repository.py:11-45`, `db.py:10-15` — `Base`/`CompanyRow`, мапперы `_to_domain`/`_to_row` + rollback на `IntegrityError`, `make_session_factory` (env `POSTGRES_DSN`, sqlite); не менять.
- `company-org/app/transport/schemas.py:8-26`, `routes.py:11-35`, `problem.py:7-49` — Pydantic/`field_validator`, `create_router`/`Header(alias=...)`/`ProblemException`, `problem_response`/`install_problem_handlers`.
- `company-org/app/wiring.py:10-26`, `main.py:12-21` — composition root; `main.py` не менять.
- `company-org/migrations/versions/0001_create_company.py:1-33` — формат ревизии → `0002_*`, `down_revision="0001"`.
- `company-org/tests/test_companies.py:19-41,80-102` — sqlite-DSN в `POSTGRES_DSN`, реальные миграции, `importlib.reload(main)`, problem+json-ассерты.
- `company-org/pyproject.toml:5-29`, `Dockerfile` (`alembic upgrade head && uvicorn`) — не менять: миграции применяет старт контейнера.
- `auth-users/app/domain/errors.py:27-31`, `infrastructure/repository.py:45-50`, `transport/problem.py:74-76` — scope-guard → 401.
- `api-gateway/app/application/routing.py:43,49-54` — + `("/v1/contracts", company_org_url)`; порядок 404/401 не менять.
- `api-gateway/app/infrastructure/proxy.py:13,51-56` — установка identity-заголовков (контекст).
- `api-gateway/tests/test_gateway.py:44-61,85-116,117-140,201-211,266-279` — образцы; ассерты `host`/`path` записанного запроса.

## Tasks & Acceptance

**Execution:**
- [x] `company-org/app/domain/contract.py` -- `Contract(id, company_id, parent_id, name, created_at)` + factory.
- [x] `company-org/app/domain/ports.py` -- `ContractRepository(add, get(contract_id, *, company_id), list_by_company(company_id))`; `CompanyRepository.get_by_id` (AD-013).
- [x] `company-org/app/domain/errors.py` -- `ContractParentNotFound`, `ContractCycleDetected`, `MissingCompanyScope`.
- [x] `company-org/app/application/create_contract.py` -- scope-guard → родитель той же компании → подъём по `parent_id` с ограничителем → `add`; ошибки `MissingCompanyScope`/`ContractParentNotFound`/`ContractCycleDetected`.
- [x] `company-org/app/application/list_contract_tree.py` -- выборка компании → `children` (порядок `created_at`, `id`); цикл → `ContractCycleDetected`; без SQL-рекурсии.
- [x] `company-org/app/infrastructure/models.py` -- `ContractRow`: FK `company_id`→`company.id` и self-FK `parent_id` (`RESTRICT`), индексы `company_id`, `parent_id`.
- [x] `company-org/app/infrastructure/repository.py` -- `SqlAlchemyContractRepository` (`company_id` обязателен, guard при `None`); `SqlAlchemyCompanyRepository.get_by_id`.
- [x] `company-org/app/transport/schemas.py` -- `CreateContractRequest` (`name` непустой после strip, `parent_id: UUID | None`, лишние поля игнорируются); схемы узла/дерева.
- [x] `company-org/app/transport/routes.py` -- `POST /v1/contracts`, `GET /v1/contracts/tree` + чтение `X-Company-Id`.
- [x] `company-org/app/transport/problem.py` -- 401 scope, 404 parent, 409 cycle (create), 500 cycle (tree).
- [x] `company-org/app/wiring.py` -- use case'ы по образцу `make_create_company_runner`, сессия на запрос.
- [x] `company-org/migrations/versions/0002_create_contract.py` -- `contract` (`down_revision="0001"`, обе FK, индексы).
- [x] `company-org/tests/test_contracts.py` -- матрица с точными ассертами: `count(contract)` до/после (MISSING_PARENT/FOREIGN_PARENT/CYCLE_IN_DB), тела MISSING_PARENT = FOREIGN_PARENT, игнор `company_id` в payload, изоляция компаний, порядок `children`, `[]` у TREE_EMPTY, spy без обращения к БД при MISSING_SCOPE, inspector констрейнтов/индексов.
- [x] `company-org/tests/test_migrations.py` -- `upgrade head` дважды идемпотентен; `upgrade 0001` → `company` → `head` сохраняет строку.
- [x] `api-gateway/app/application/routing.py` -- префикс `/v1/contracts` → `company_org_url` (Q1=B).
- [x] `api-gateway/tests/test_gateway.py` -- без токена 401; с токеном assert `host`/`path` (как `test_gateway.py:117-140`).

**Acceptance Criteria:**
- Given чистая БД, when `alembic upgrade head` дважды, then `0001`+`0002` применены, повтор схему не меняет
- Given БД с `0001` и данными `company`, when `upgrade head`, then строка `company` сохранена
- Given договоры C1 и C2, when дерево с `X-Company-Id: C1`, then узлов C2 нет ни на одном уровне
- Given `parent_id` чужой компании, when создание с `X-Company-Id: C1`, then 404, тело неотличимо от «родителя нет», строк не добавлено
- Given нет `X-Company-Id`, when запрос к договорам, then 401 `application/problem+json` и репозиторий не вызывается
- Given 9 существующих тестов `company-org`, when прогон после изменений, then зелёные и `POST /v1/companies` без изменений
- Given `/v1/contracts` в роут-таблице, when без токена, then 401 `application/problem+json`; when с токеном, then запрос уходит в `company-org` (проверяются хост и путь)

## Implementation Notes

- **main.py не изменён** (требование Code Map): `create_router` принимает контрактные раннеры опционально (инъекция в тестах), а при их отсутствии лениво докомпоновывает их из `wiring` — тот же паттерн «сессия на запрос», что у `make_create_company_runner`.
- Маппинг `ContractCycleDetected`: обработчик в `problem.py` отвечает 409 (путь создания), маршрут `tree` перехватывает ту же ошибку раньше и отвечает 500 — HTTP-контекст живёт в transport, домен об HTTP не знает.
- Ограничитель подъёма: `MAX_PARENT_CHAIN_STEPS = 10_000` + множество посещённых узлов (AD-006 — защита от зацикливания, не лимит вложенности).
- `uvx ruff` 0.17 включает B008 в дефолтный набор: два `Header(default=...)` с аннотацией `UUID | None` помечены `# noqa: B008` (стиль `Header(alias=...)` сохранён по образцу Story 2.1).
- Verification (2026-10-19): company-org `29 passed` (9 прежних + 18 контрактов + 2 миграции), api-gateway `26 passed`, `ruff check .` — `All checks passed!` в обоих сервисах, `docker compose config -q` — OK.
## Spec Change Log

- **iter 1 (2026-10-19, ревью спеки до утверждения, не step-04 loopback).** Триггеры: нет кода ответа для цикла; незафиксированная форма ответов; ложное «self-FK защищает от самоссылки»; `get` без обязательного `company_id`; несуществующая компания в заголовке; AC миграций никем не проверяются; `count(contract)`/spy-репозиторий/равенство тел/порядок `children` не ассертились; тест префикса мог пройти при неверном upstream. Правки: +5 сценариев матрицы (SPOOFED_PAYLOAD, CYCLE_IN_DB, TREE_CYCLIC, SCOPE_UNKNOWN, MISSING_SCOPE) и точные ассерты; форма ответов (201 — плоский узел, `tree` — корни с `children`, пусто `[]`, порядок `created_at`,`id`); `get(contract_id, *, company_id)`; честный self-FK; задачи `tests/test_migrations.py` и инспекция констрейнтов. Решения человека: цикл на create → 409, цикл в дереве → 500, несуществующая компания в заголовке → 401 (guard явным чтением: на sqlite FK по умолчанию выключены). Предотвращённое плохое состояние: молча потерянные узлы цикла, green-тесты при невалидируемом родителе/неверном upstream, data-loss в `0002`.
- **Сжатие (2026-10-19, запрос владельца спеки).** Сокращены формулировки в Code Map, задачах, Design Notes, Verification и этом логе; совокупность требований, сценариев матрицы (13), решений (409/500/401), AC (7) и задач (16) не менялась.

## Review Triage Log

Ревью прогона (blind-hunter, edge-case-hunter, verification-gap). Все находки проверены по фактическому коду; severity ревьюеров не учитывалась.

| # | Находка (слой) | Вердикт | Маршрут | Доказательство |
|---|----------------|---------|---------|----------------|
| 1 | diff-артефакт: абсолютные пути новых файлов, строки-разделители `===== repo @` (BH) | false | — | `diff_file` — временный ревью-артефакт, никогда не применяется; код лежит в трёх репозиториях и проверялся на месте. |
| 2 | `sprint-status.last_updated` откатился (`21:05` → `18:22`) (BH, EH) | low | patch | Предыдущее значение было из будущего относительно записи; привёл к текущему локальному времени в том же формате. |
| 3 | `ContractCycleDetected`: глобальный хендлер 409 против 500 в маршруте дерева (BH) | low | reject | Роутинг статусов зафиксирован матрицей (create→409, tree→500); хендлер — страховка, недостижимая на текущих путях; названный вред только у гипотетических будущих маршрутов, правка структурная. |
| 4 | 401-детали у MISSING_SCOPE и SCOPE_UNKNOWN различаются (BH) | false | — | Детали как раз различаются (`X-Company-Id header is required` vs `company_id is required`), клиент их отличает; идентичность тел спека требует только для MISSING/FOREIGN parent. |
| 5 | Текст 401 `company_id is required` вводит в заблуждение при «компания не найдена» (BH) | low | patch | `MissingCompanyScope` бросается в use case'ах при неизвестной компании, а не только при отсутствии заголовка; detail следует сделать нейтральным. |
| 6 | Владение транзакцией: `SqlAlchemyContractRepository.add` коммитит сам (BH) | false | — | Посылка неверна: `SqlAlchemyCompanyRepository.add` тоже коммитит (repository.py:33); паттерн консистентен, вреда нет. |
| 7 | Дрейф nullable port↔impl и мёртвый guard `company_id is None` (BH) | false | — | Ни один вызывающий не может передать `None` (X-Company-Id обязателен, `Contract.company_id: UUID`); guard недостижим, спек-задача прямо требует «guard при None». |
| 8 | Нет составного FK «parent той же компании»; чужой parent → узел-сирота → `ContractCycleDetected`/500 (BH) | low | reject | Достижимо только прямой записью в БД (тот же класс порчи данных, что CYCLE_IN_DB → 500 по матрице); неточно только имя ошибки. |
| 9 | Глубокое дерево (~1000) → `RecursionError` при рекурсивной сериализации (BH, EH) | low | reject | Практически недостижимо на объёмах MVP; итеративная сериализация — нетривиальная правка, добавляет поверхность. |
| 10 | Ограничитель `MAX_PARENT_CHAIN_STEPS` метит легитимную цепочку >10 000 как цикл (BH, EH, VG) | low | reject | Требует >10 000 последовательных созданий; неверная категория ошибки практического вреда не даёт. |
| 11 | Нет `Idempotency-Key` у `POST /v1/contracts` (BH) | low | reject | Frozen intent прямо исключает («`Idempotency-Key` для договоров» в Never) — вне области. |
| 12 | Нет `GET /v1/contracts/{id}` и `DELETE` (BH) | low | reject | Frozen intent исключает update/move/delete; получение одного договора не входит в Intent. |
| 13 | Трассировка AC→тесты из диффа не проверяется, claims-файл исключён (BH) | false | — | Слой verification-gap независимо подтвердил покрытие всех 13 строк матрицы тестами; исключение claims-файла из диффа предписано step-04. |
| 14 | Тест-пробелы: не-UUID заголовок, ассерт strip, `downgrade()`, FK-enforced/dispose (BH) | low | reject | Вне матрицы/спеки; правки добавляют поверхность; выключенные sqlite-FK сознательно зафиксированы в Design Notes. |
| 15 | `json.loads` в api-gateway тесте без импорта; шлюз не проверяет перезапись identity-заголовков для `/v1/contracts` (BH) | false | — | `import json` есть (test_gateway.py:7); перезапись покрыта `test_spoofed_identity_overwritten_without_duplicates` (логика прокси не зависит от префикса). |
| 16 | `create_router` лениво импортирует wiring и создаёт session factory на сборке роутера (BH) | false | — | Сессия создаётся на запрос внутри runner (wiring.py); паттерн идентичен Story 2.1; ленивый импорт — следствие запрета менять `main.py`. |
| 17 | `ContractResponse` и `ContractNodeResponse` дублируют поля без базового класса (BH) | false | — | Стилистика без названного вреда (grading rule: «messy» без именованного вреда → false). |
| 18 | `IntegrityError` при FK-нарушении (компания/родитель удалены до commit) → 500 без problem+json (EH) | false | — | Ни одной операции удаления договора/компании в системе нет, поэтому FK-нарушение недостижимо. |
| 19 | `epic-2-context.md`: потеря упоминания «один контейнер PostgreSQL в compose» и противоречие про edge-слой (BH) | low | defer | Файл изменён до baseline (docs-работа прошлой сессии), не этой историей. |

Итог: intent_gap/bad_spec отсутствуют; применены patch #2 и #5, #19 вынесена в deferred-work, остальные отклонены.

## Design Notes

- insert-only: новый узел не станет предком существующего → реально возможны только самоссылка (недостижима, `id` даёт сервер) и чужой цикл в БД; подъём с ограничителем закрывает оба и готовит почву для `move`. Ограничитель ≠ лимит вложенности (AD-006).
- Тот же обход нужен при сборке дерева, иначе «сирота цикла» даёт бесконечную рекурсию; при цикле — 500 (порча данных), а не частичное дерево: молча потерянные узлы опаснее явной ошибки.
- Scope-guard — явным чтением `CompanyRepository.get_by_id`, а не через FK: на sqlite `PRAGMA foreign_keys` по умолчанию выключен, FK-нарушение в тестах не возникло бы; FK в схеме — страховка на уровне БД.
- Q1=B: арендатор уже доказан заголовком (AD-013), дублировать его в пути — второй источник истины. Цена — один префикс в `api-gateway/app/application/routing.py:43`. Форма ответов (201 — плоский узел, `tree` — корни с `children`, пусто `[]`) зафиксирована в матрице.

## Verification

**Commands:**
- `uv run --extra dev --directory /home/shved/projects/smeta/company-org pytest -q` → 9 прежних + новые (вкл. `test_migrations.py`)
- `cd /home/shved/projects/smeta/company-org && uvx ruff check .` → `All checks passed!`
- `uv run --extra dev --directory /home/shved/projects/smeta/api-gateway pytest -q` → вкл. тест префикса `/v1/contracts`
- `cd /home/shved/projects/smeta/api-gateway && uvx ruff check .` → `All checks passed!`
- `docker compose -f /home/shved/projects/smeta/deploy/docker-compose.yml config -q` → OK (deploy не менялся)

**Manual checks (if no CLI):**
- Дерево без узлов чужих компаний; без `X-Company-Id` — `application/problem+json` с `title`/`status`/`detail`; коды 401/404/409/500 по матрице.
