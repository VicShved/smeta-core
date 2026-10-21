---
title: 'Story 7.1a — UI: вход и дерево «Компания → Договор → Этап → Сметы»'
type: 'feature'
created: '2026-10-21'
status: 'done'
route: 'dispatch'
baseline_commit: 'fb3b6a4'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-7-context.md'
  - '_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md'
  - '_bmad-output/implementation-artifacts/spec-6-1-создание-сметы-и-версий-ровно-одна-активная.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Система уже умеет хранить и считать сметы, но пользователь не может до них дойти: `ui` — каркас без входа и без навигации, а API не отдаёт дерево «Компания → Договор → Этап → Сметы» (нет списка смет по этапу и этапов в дереве договоров).

**Approach:** Довести API до навигации (дерево договоров со вложенными этапами, список смет этапа) и собрать в `ui` рабочее место сметчика: экран входа по email/паролю с регистрацией, дерево с раскрываемыми узлами и списком смет выбранного этапа. Загрузка PDF и статусы обработки — отдельная история 7.1b.

## Boundaries & Constraints

**Always:** UI — read-only проекция домена: не считает стоимость, не применяет маппинг, не ходит в LLM; единственная точка входа — `api-gateway`, JWT не разбирается, identity-заголовки не подставляются; `company_id` берётся из ответа на вход, переключателя компаний нет; дерево рендерится рекурсивно (произвольная глубина договоров); ошибки API показываются текстом без внутренних деталей; контракты API — `/v1/...`, `snake_case`, ошибки `application/problem+json`; UI в nginx отдаётся с того же origin и проксирует `/v1` в `api-gateway` (CORS не нужен и не реализуется).

**Never:** загрузка PDF, статусы обработки, просмотр позиций/стоимости/маппинга (7.1b/7.2); UI diff и валидация маппинга (1.1); правка смет, договоров, этапов; хранение бизнес-данных на клиенте, кроме access-токена; новые роли/компании; собственный бэкенд для UI.

**Решения (2026-10-20):** история разделена на 7.1a (навигация и вход) и 7.1b (загрузка PDF и статусы обработки); недостающие API дорабатываются (список смет по этапу, этапы в дереве договоров, связь `file_id` со сметой — последнее в 7.1b); вход и регистрация — отдельные формы (email/пароль), токен в `localStorage`; `ui` подключается к compose как сервис на nginx с прокси `/v1` → `api-gateway` (единый origin, CORS не нужен).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Вход | верные email/пароль | `200`, токен сохранён, открыто дерево | — |
| Неверный пароль | неверные данные | сообщение пользователю, без деталей | `401` от API → текст |
| Регистрация | новый email/пароль | аккаунт создан, выполнен вход | — |
| Регистрация: email занят | существующий email | сообщение «уже зарегистрирован» | `409` от API → текст |
| Нет токена | открыт UI без токена | показан экран входа | — |
| Истёкший токен | `401` на любом запросе | возврат к экрану входа | `401` → экран входа |
| Пустое дерево | нет договоров | пустое состояние, без ошибки | — |
| Глубокая вложенность | договоры 3+ уровней | дерево раскрывается рекурсивно | — |
| Список смет этапа | этап со сметами | список смет с активной версией | — |
| Пустой этап | нет смет | «пока нет смет» | — |
| Чужая компания | `company_id` не совпадает | `404`, не пустой список | — |
| Сеть недоступна | gateway не отвечает | текст ошибки, кнопка повтора | — |
| company-org недоступен | список смет этапа, company-org не отвечает | — | `502` |
| Пагинация вне границ | `limit`/`offset` вне допустимых значений | — | `422` |
| Сеть недоступна (дерево) | загрузка дерева смет, gateway не отвечает | текст ошибки, кнопка повтора | — |
| Смета без активной версии | у сметы нет активной версии | в списке — «нет активной версии» | — |
| Регистрация: имя компании | email, из локальной части выводится имя | компания создаётся с именем по умолчанию из email | — |

</frozen-after-approval>

## Code Map

- `company-org/app/domain/contract.py`, `app/application/list_contract_tree.py` -- вложить этапы в узлы дерева: один запрос всех этапов компании + группировка по `contract_id` (без N+1).
- `company-org/app/domain/ports.py`, `app/infrastructure/repository.py` -- выборка всех этапов компании (`list_by_company`).
- `company-org/app/transport/schemas.py`, `routes.py` -- `stages` в `ContractNodeResponse` ответа `GET /v1/contracts/tree`; порядок узлов и этапов — `created_at`, затем id.
- `company-org/tests/` -- дерево с этапами, пустое дерево, скоуп компании.
- `estimate/app/domain/ports.py`, `app/infrastructure/repository.py` -- список смет этапа в скоупе компании.
- `estimate/app/application/list_estimates.py` -- use case списка смет этапа.
- `estimate/app/transport/{schemas,routes}.py` -- `GET /v1/estimates?stage_id=...` с `limit`/`offset`; в ответе — сводка сметы и активная версия.
- `estimate/tests/` -- список смет этапа, скоуп, пустой этап, пагинация.
- `ui/src/api/client.ts` -- HTTP-клиент к `api-gateway`: общий `fetch` с `Authorization: Bearer`, разбор `application/problem+json` в текст, перехват `401`.
- `ui/src/auth/{store,views}.ts` -- хранение и загрузка access-токена (`localStorage`), состояние «вошёл/не вошёл».
- `ui/src/views/{LoginView,RegisterView}.ts` -- формы входа и регистрации по email/паролю.
- `ui/src/views/HierarchyView.ts` -- дерево «Договор → Этап → Сметы» с раскрытием узлов и подгрузкой смет по этапу.
- `ui/src/components/{TreeNode,EstimateList}.vue` -- рекурсивный узел дерева, список смет этапа.
- `ui/src/router.ts` -- маршруты: вход/регистрация → дерево; guard по наличию токена.
- `ui/vite.config.ts` -- dev-proxy `/v1` → `http://localhost:8000`.
- `ui/nginx.conf` -- прокси `/v1` → `api-gateway` (в compose), SPA-fallback.
- `ui/Dockerfile`, `ui/package.json` -- сборка, скрипты, `vue-router` в зависимостях.
- `ui/tests/` -- vitest: клиент (заголовки, 401, problem+json), форма входа, дерево (раскрытие, подгрузка смет).
- `deploy/docker-compose.yml`, `.env.example`, `README.md` -- сервис `ui`, прокси к `api-gateway`, порт 8080.
- Не трогать: `pdf-extract/*` (загрузка — 7.1b), `rate-catalog/*`, `mapping-service/*`, `llm-gateway/*`, `calc-service/*`, `auth-users/*` (регистрация уже есть).

## Tasks & Acceptance

**Execution:**
- [x] `company-org/app/domain/*`, `application/list_contract_tree.py`, `infrastructure/repository.py` -- выборка этапов компании и вложение в узлы дерева -- один запрос вместо N+1.
- [x] `company-org/app/transport/schemas.py`, `routes.py`, `tests/` -- `stages` в ответе дерева -- контракт навигации.
- [x] `estimate/app/domain/ports.py`, `application/list_estimates.py`, `infrastructure/repository.py` -- список смет этапа -- выборка в скоупе компании.
- [x] `estimate/app/transport/{schemas,routes}.py`, `tests/` -- `GET /v1/estimates?stage_id=` -- API навигации.
- [x] `ui/src/api/client.ts`, `ui/src/auth/*` -- HTTP-клиент и состояние токена -- единая точка обращения к gateway.
- [x] `ui/src/views/*`, `ui/src/components/*`, `ui/src/router.ts` -- вход, регистрация, дерево -- пользовательский сценарий.
- [x] `ui/vite.config.ts`, `ui/nginx.conf`, `ui/Dockerfile`, `ui/package.json` -- dev-proxy и прокси `/v1` -- один origin без CORS.
- [x] `ui/tests/*` -- vitest: клиент, вход, дерево -- регрессия UI.
- [x] `deploy/*` -- сервис `ui` в стенде -- локальный запуск.
- [x] `company-org` + `estimate` -- ruff/pytest; `ui` -- typecheck/build/test -- верификация.

**Acceptance Criteria:**
- Given неавторизованный пользователь, when открыт UI, then показан экран входа с полями email/пароль.
- Given верные учётные данные, when выполнен вход, then получен access-токен, сохранён в браузере и открыто дерево смет.
- Given новый email, when заполнена форма регистрации, then аккаунт создан и выполнен вход.
- Given договоры с этапами, when открыто дерево, then видна иерархия «Договор → Этап → Сметы» с произвольной вложенностью.
- Given раскрытый этап, when загружены его сметы, then показан список смет с активной версией.
- Given `401` от API, when любой запрос, then UI возвращает к экрану входа.
- Given ошибка API, when она показана, then это текст без внутренних деталей.

## Implementation Notes

- **company-org**: `StageRepository.list_by_company(company_id)` + `ListContractTree._stages_by_contract()` -- этапы компании выбираются одним запросом и группируются по `contract_id` (без N+1, тест считает SQL-запросы к `stage`). Узел ответа `GET /v1/contracts/tree` получил `stages[]` (`StageResponse`); порядок узлов и этапов -- `created_at`, затем `id`.
- **estimate**: `GET /v1/estimates?stage_id=&limit=&offset=` -> `200 {items, total}`; элемент -- сводка сметы `{id, contract_id, stage_id, name, created_at, active_version}`, где `active_version` -- версия с `is_active=true` (`null`, если её нет). Выборка -- `outerjoin` активной версии, скоуп по `company_id` в репозитории (AD-013), порядок `created_at`, затем `id`.
- **estimate x company-org**: чтобы отличить «пустой этап» (`200`, `items: []`) от «чужой/неизвестный этап» (`404`, а не пустой список -- матрица I/O), скоуп этапа подтверждается через company-org: новый порт `CompanyOrgPort.list_company_stages` читает `GET /v1/contracts/tree` (дерево уже содержит этапы) и собирает этапы компании. `401/404` от company-org -- «компания вне скоупа» (`None` -> `404`), сбой/5xx -- `CompanyOrgUnavailable` -> `502`.
- **ui**: `src/api/client.ts` -- единственная точка обращения к gateway: `fetch` с `Authorization: Bearer`, разбор `application/problem+json`, перехват `401` (сессия сбрасывается, вызывается обработчик -> возврат к экрану входа). Тексты ошибок -- заранее заданные русские формулировки по статусу, внутренние детали сервисов не показываются; текст проблемы остаётся в `ApiError.problem`.
- **ui**: `src/auth/store.ts` -- access-токен в `localStorage` (единственные бизнес-данные на клиенте); `src/auth/views.ts` -- реактивное состояние сессии «вошёл/не вошёл» (`start_session`/`end_session`/`is_authenticated`). Компания берётся из ответа на вход (`TokenResponse.company_id`), переключателя нет.
- **ui**: `views/LoginView.ts` / `views/RegisterView.ts` -- отдельные формы email/пароль (компоненты на `defineComponent` + `h()`, файлы `.ts` -- как в Code Map). Регистрация создаёт компанию по умолчанию: `company_name` берётся из локальной части email (auth-users требует `company_name` в теле) и показывается в форме.
- **ui**: `views/HierarchyView.ts` -- дерево; `components/TreeNode.vue` -- рекурсивный узел (произвольная глубина), при раскрытии этапа подгружает `GET /v1/estimates?stage_id=`; `components/EstimateList.vue` -- список смет этапа с номером активной версии (позиции и стоимость не показываются -- это 7.1b/7.2).
- **ui**: `router.ts` -- маршруты `/login`, `/register`, `/`; guard по наличию токена (без токена -- экран входа, с токеном -- дерево). `vite.config.ts` -- dev-proxy `/v1` -> `http://localhost:8000`; `nginx.conf` -- прокси `/v1` -> `api-gateway` (резолв имени в рантайме, чтобы nginx стартовал независимо от api-gateway) и SPA-fallback.
- **deploy**: сервис `ui` (nginx, порт `${UI_PORT:-8080}`), `depends_on: api-gateway`; наружу выходят `ui` (8080) и `api-gateway` (8000).
- **ui (обработка ошибок)**: `401` на публичных путях `/auth/login` и `/auth/register` не сбрасывает сессию и не вызывает обработчик истечения -- это «неверные учётные данные», а не истёкшая сессия. Запросы ограничены таймаутом 15 с (`AbortSignal.timeout`): обрыв по таймауту -- «Сервер не ответил вовремя…», сетевая ошибка -- «Нет связи с сервером…». Для статусов 429/502/503/504 заданы отдельные тексты.
- **ui (хранение)**: в `localStorage` лежит только access-токен; `company_id` живёт в состоянии сессии (в памяти вкладки) и в браузере не сохраняется. Если `localStorage` недоступен (приватный режим, квота, запрет записи), токен хранится в памяти вкладки -- вход при этом не выглядит неудачным.
- **ui (список смет этапа)**: запрос идёт с `limit=500`; если `total` больше показанного -- выводится «Показано N из M». Смета без активной версии помечается «нет активной версии». Узел без полей `stages`/`children` не роняет рендер; переключатели имеют `aria-expanded`, узлы -- роли дерева.
- **ui (перезагрузка дерева)**: узлы дерева пересоздаются при каждой загрузке (счётчик в ключе), поэтому кэш смет предыдущего дерева не сохраняется; перезагрузка доступна кнопкой «Обновить».
- **ui/nginx**: `client_max_body_size 5m` (загрузка PDF -- Story 7.1b), точный путь `/v1` отдаёт `404 application/problem+json` (не SPA), добавлены `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: same-origin` (CSP -- отдельная задача).
- **estimate**: `401` от company-org трактуется как недоступность сервиса (`502`), так же, как в `list_contract_stages`; `404` -- «дерева/компании нет» (`None` -> `404 Stage not found`).
- **company-org**: сигнатуры `list_by_company` согласованы между портами и обеими реализациями: `company_id` обязателен (AD-013), `None` -- отказ `MissingCompanyScope`, а не выборка всех строк.

## Spec Change Log

- 2026-10-21: реализация без изменения frozen-блока; уточнения контракта на уровне реализации:
  - `GET /v1/estimates?stage_id=` добавлен в `estimate`; маршрутизация в api-gateway уже покрыта префиксом `/v1/estimates` -- правки api-gateway не потребовались.
  - Ответ списка смет -- `{items, total}` (как `ItemsPageResponse` в estimate); `limit` 1–500, `offset` ≥ 0.
  - Скоуп этапа подтверждается вызовом company-org (`GET /v1/contracts/tree`); это дополнение к `CompanyOrgPort` estimate, не меняющее публичный контракт api-gateway.
  - Форма регистрации UI -- только email/пароль (frozen-блок); `company_name` для auth-users формируется из локальной части email и показывается пользователю в форме.
  - UI-файлы по Code Map: представления -- `.ts` (LoginView/RegisterView/HierarchyView), компоненты дерева -- `.vue` (TreeNode/EstimateList).
- 2026-10-21 (по итогам ревью реализации): в I/O-матрицу добавлены строки про недоступность company-org (`502`), границы пагинации (`422`), сетевую ошибку загрузки дерева, смету без активной версии и вывод имени компании из email при регистрации; frozen-блок в части Intent/Boundaries не изменялся. Зафиксированные решения:
  - `401` на публичных путях UI (`/auth/login`, `/auth/register`) не сбрасывает сессию и не возвращает к экрану входа -- это «неверные учётные данные».
  - В браузере сохраняется только access-токен; `company_id` -- в памяти сессии. При недоступном `localStorage` токен остаётся в памяти вкладки.
  - Список смет этапа запрашивается с `limit=500`; при `total` больше показанного выводится «Показано N из M».
  - Таймаут запроса к gateway -- 15 с; статусы 429/502/503/504 имеют собственные тексты.
  - nginx: `client_max_body_size 5m` под загрузку PDF следующей истории (7.1b); `/v1` без префикса -- `404 problem+json`; добавлены базовые защитные заголовки (без CSP).
  - `401` от company-org в `estimate` трактуется как недоступность сервиса (`502`) -- как в соседнем `list_contract_stages`.

## Review Triage Log

- **2026-10-21 · blind-hunter + edge-case-hunter · неверный пароль показывал и «Сессия истекла», и «Неверный email или пароль»** — verdict: medium. evidence: клиент сбрасывал сессию на любой 401, включая `/v1/auth/login`. route: patch → публичные пути `/auth/*` не запускают обработчик истечения сессии.
- **2026-10-21 · edge-case-hunter · у запросов нет таймаута — спиннер висит вечно** — verdict: medium. evidence: `fetch` без `signal`. route: patch → `AbortSignal.timeout(15 с)` и отдельный текст таймаута.
- **2026-10-21 · blind-hunter · статусы 429/504 без своих текстов** — verdict: low. evidence: пользователь видит общий текст вместо «слишком много запросов»/«сервис не ответил». route: patch → отдельные сообщения.
- **2026-10-21 · blind-hunter · `company_id` сохранялся в `localStorage` вопреки frozen-правилу «только access-токен»** — verdict: medium. evidence: значение нигде не читалось, но нарушало границу хранения. route: patch → `company_id` только в памяти сессии.
- **2026-10-21 · edge-case-hunter · `localStorage.setItem` может бросить — успешный вход выглядит неудачным** — verdict: medium. evidence: квота/приватный режим. route: patch → `try/catch` с откатом на память.
- **2026-10-21 · edge-case-hunter · `safe_redirect` пропускал `/\evil.com`** — verdict: medium. evidence: браузер нормализует в protocol-relative URL, `pushState` бросает. route: patch → отклонять обратный слэш.
- **2026-10-21 · blind-hunter + edge-case-hunter · список смет этапа обрезался до 50 смет, `total` игнорировался** — verdict: medium. evidence: пользователь видел неполный список без признака усечения. route: patch → `limit=500` и «показано N из M».
- **2026-10-21 · edge-case-hunter · узел без `stages`/`children` ронял рендер** — verdict: medium. evidence: `undefined` в `v-for`. route: patch → `?? []`.
- **2026-10-21 · edge-case-hunter · не-Array ответ дерева тихо рисовался как пустое дерево** — verdict: medium. evidence: пользователь думал, что договоров нет. route: patch → проверка формы ответа.
- **2026-10-21 · blind-hunter · после перезагрузки дерева оставался кэш смет** — verdict: medium. evidence: Vue переиспользовал узлы с прежним `estimates_by_stage`. route: patch → сброс кэша при перезагрузке.
- **2026-10-21 · blind-hunter · смета без активной версии не отличалась от обычной строки** — verdict: low. evidence: `active_version` допускает `null`. route: patch → «нет активной версии».
- **2026-10-21 · blind-hunter · у дерева не было состояния доступности** — verdict: low. evidence: раскрытие обозначено только глифом `aria-hidden`. route: patch → `aria-expanded` и роли узлов.
- **2026-10-21 · blind-hunter · nginx без `client_max_body_size` отверг бы PDF следующей истории (1 МБ по умолчанию)** — verdict: medium. evidence: лимит PDF в эпике — 5 МБ. route: patch → `client_max_body_size 5m`.
- **2026-10-21 · blind-hunter · точный `/v1` отдавал `index.html` с кодом 200** — verdict: low. evidence: SPA-fallback перехватывал. route: patch → явный 404 problem+json.
- **2026-10-21 · blind-hunter · нет защитных заголовков при токене в `localStorage`** — verdict: low. evidence: `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy` отсутствовали. route: patch → базовые заголовки добавлены; CSP → defer.
- **2026-10-21 · blind-hunter + edge-case-hunter · 401 от company-org трактовался по-разному в двух методах клиента** — verdict: medium. evidence: `list_company_stages` давал 404 вместо 502, что маскировало сбой авторизации под «этап не найден». route: patch → единое поведение: 401 → `CompanyOrgUnavailable`.
- **2026-10-21 · blind-hunter · порт `list_by_company` расходился с реализацией по типу `company_id`** — verdict: low. evidence: порт объявлял `UUID`, реализация принимала `UUID | None` и бросала `MissingCompanyScope`. route: patch → сигнатуры согласованы.
- **2026-10-21 · verification-gap · production-обработчик 401 не был покрыт тестом** — verdict: medium. evidence: удаление регистрации обработчика не ломало ни один тест — пользователь оставался на дереве вместо экрана входа. route: patch → обработчик вынесен в экспортируемую функцию, добавлен тест перехода на `login`.
- **2026-10-21 · verification-gap · кнопка «Выйти» в `App.vue` не была покрыта** — verdict: medium. evidence: потеря `end_session()` не ломала suite — токен оставался в браузере. route: patch → тест компонента.
- **2026-10-21 · verification-gap · порядок «created_at, затем id» не был закреплён ни в дереве этапов, ни в списке смет** — verdict: medium. evidence: удаление `.id.asc()` не ломало тесты; при общем `created_at` порядок становился случайным. route: patch → кейсы с одинаковым временем.
- **2026-10-21 · verification-gap · tz-нормализация `created_at` договоров и компаний не была закреплена** — verdict: medium. evidence: ответы создания и чтения могли различаться по часовому поясу без падения тестов. route: patch → проверки tz-aware и равенства с ответом создания.
- **2026-10-21 · verification-gap · не покрыты `active_version: null` и смещение за конец выборки** — verdict: medium. evidence: ветки outerjoin и пустой страницы не исполнялись. route: patch → добавлены кейсы.
- **2026-10-21 · blind-hunter · отсутствовали тесты UI: `main.ts`, `App.vue`, `safe_redirect`, `default_company_name`, fallback `storage()`, пустое состояние** — verdict: medium. evidence: перечисленные сценарии не покрыты. route: patch → добавлены тесты.
- **2026-10-21 · verification-gap · `ui/tsconfig.json` не включал `tests/**` — CI не проверял типы тестов** — verdict: low. evidence: `vue-tsc` не видел ни одного файла из `tests/`. route: patch → `include` дополнен.
- **2026-10-21 · blind-hunter + verification-gap · в diff попал не связанный перенос healthcheck `llm-gateway` (10s → 5s)** — verdict: medium. evidence: изменение не относится к Story 7.1a и не зафиксировано в change log. route: patch → возврат к `10s`; сервису `ui` добавлен healthcheck в compose.
- **2026-10-21 · blind-hunter + edge-case-hunter · разделение 7.1a/7.1b не отражено в epics-документах и контексте эпика** — verdict: medium. evidence: трекер знает `7-1a`/`7-1b`, а в epics осталась одна «Story 7.1». route: patch → epics и `epic-7-context.md` приведены к разделению; «Vite 8» исправлено на фактический Vite 5.
- **2026-10-21 · blind-hunter · в I/O-матрице спеки не хватало строк** — verdict: low. evidence: не отражены сбой company-org (502), границы пагинации (422), сетевая ошибка дерева, смета без активной версии, вывод имени компании из email. route: patch → строки добавлены.
- **2026-10-21 · blind-hunter · README estimate не перечислял поля `active_version`** — verdict: low. evidence: потребитель опирался на память о модели. route: patch → поля перечислены.
- **2026-10-21 · edge-case-hunter · `total` и `items` считались двумя запросами и могли разойтись** — verdict: low. evidence: гонка записи между count и выборкой. route: defer → единый запрос с `COUNT(*) OVER ()` при росте нагрузки.
- **2026-10-21 · edge-case-hunter · nginx-прокси жёстко привязан к `resolver 127.0.0.11` и `api-gateway:8000`** — verdict: low. evidence: работает только в docker-сети; отдельный хост не настроить. route: defer → параметризация при необходимости внешнего стенда.
- **2026-10-21 · blind-hunter · `StageRef.contract_id` не читается** — verdict: low. evidence: поле заполняется, но не используется. route: reject.
- **2026-10-21 · blind-hunter · проверка дерева на цикл выполнялась после выборки этапов** — verdict: low. evidence: лишний запрос только при цикле, результат тот же. route: reject.
- **2026-10-21 · blind-hunter · `depth` у узла дерева не влиял на отображение** — verdict: low. evidence: отступ задаётся CSS. route: reject.
- **2026-10-21 · blind-hunter · N+1-guard в тесте считает подстроки в SQL** — verdict: low. evidence: проверка не за detecting N+1 через join и не считает запросы по договорам. route: reject (тест подтверждает одну выборку этапов; при смене реализации потребуется переписать).
- **2026-10-21 · blind-hunter · в UI нет `aria`-состояния и «Этапов нет»** — verdict: low. evidence: см. выше; закрыто patch-правками. route: reject.
- **2026-10-21 · blind-hunter · регистрация выводит имя компании из локальной части email** — verdict: low. evidence: `auth-users` требует `company_name`, а frozen-форма — только email/пароль. route: defer → запись в deferred-work (имя компании при регистрации).

- _Пусто: ревью отдельным проходом (bmad-review) не выполнялось в рамках этой реализации._

## Design Notes

- Дерево договоров уже отдаётся на `GET /v1/contracts/tree`; в 7.1a в ответ добавляются вложенные `stages[]`. Новый endpoint: `GET /v1/estimates?stage_id=&limit=&offset=` — сметы этапа.
- UI не знает ни про `company_id` из URL, ни про внутренние идентификаторы сверх того, что вернуло API: компания берётся из ответа на вход (`TokenResponse.company_id`).
- Порядок в дереве и списках — `created_at`, затем id (как в company-org).
- Токен хранится в `localStorage` под отдельным ключом; при `401` очищается и показывается экран входа.
- Формы: email + пароль; регистрация создаёт компанию по умолчанию (поведение `auth-users`), отдельного выбора компании нет.
- `ui` — Vue 3 + Vite + TypeScript; добавляется `vue-router` и `vitest` для тестов.
- Dev-режим: `vite` проксирует `/v1` на `http://localhost:8000`; в compose — nginx с тем же прокси на `api-gateway`.
- Спека разделена: 7.1b (загрузка PDF, связь `file_id` со сметой, статусы «загрузка → извлечение → маппинг → расчёт») оформлена отдельной историей в `sprint-status.yaml`.

## Verification

**Commands (выполнено 2026-10-21):**
- `cd company-org && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `59 passed`
- `cd estimate && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `175 passed`
- `cd ui && npm ci && npm run typecheck && npm run build && npm run test` → без ошибок, `42 passed` (vitest), сборка `dist/` собрана
- `cd deploy && docker compose config -q` → exit 0
- `cd ui && docker build -t smeta-ui:7.1a .` → образ собран; `docker run --rm smeta-ui:7.1a nginx -t` → `configuration file /etc/nginx/nginx.conf test is successful`

**Живой стенд (compose, порт 8080 → nginx UI → api-gateway):**
- `POST /v1/auth/register` → токен; `POST /v1/contracts` → `POST /v1/contracts/{id}/stages` → `POST /v1/estimates` — отработали через nginx UI.
- `GET /v1/contracts/tree` → узел содержит `stages[]` (этап вложен в свой договор).
- `GET /v1/estimates?stage_id={id}` → `200 {items:[{... active_version}], total:1}`; пустой этап → `200 {items:[], total:0}`; несуществующий этап → `404` («Stage not found»); без `stage_id` → `422`; без токена → `401`.
- Проверка nginx-прокси отдельно (mock upstream): путь `/v1/contracts/tree` и заголовки `Authorization`/`X-Company-Id` доходят до api-gateway без изменений; SPA-fallback отдаёт `index.html`.

**Не покрыто автоматически:** UI не проверялся в реальном браузере (нет e2e-стенда — только vitest/jsdom); живой Postgres покрыт compose-стендом, но миграции по-прежнему проверяются на SQLite в CI.
