---
title: 'api-gateway: проверка JWT и identity-заголовки на edge'
type: 'feature'
created: '2026-10-19'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
baseline_commit: '1fc3f6a73a084a57a0944cc67ef390f86c5e18fe'
baseline_repos:
  core: '1fc3f6a73a084a57a0944cc67ef390f86c5e18fe'
  api-gateway: 'ef3b025519037fdc096d21858f6eec5222211c18'
  deploy: '903fa11fc4cbd237f3ffe628185b4bd75d2a467c'
context:
  - '_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** AD-010 (JWT проверяется только на edge, внутрь идёт identity заголовками) не реализован: `auth-users`/`company-org` доступны только внутри compose, проверять токен некому, а Story 2.2+ неоткуда взять `X-Company-Id` — tenancy AD-013 снаружи недоказуема.

**Approach:** Наполнить скаффолд `api-gateway` проверкой JWT HS256 и обратным прокси: `/health` и `POST /v1/auth/register|login` проксируются без токена, остальные `/v1/**` — только с валидным токеном; из проверенных claims формируются `X-User-Id`/`X-Company-Id`/`X-Role`, клиентские одноимённые заголовки всегда удаляются. В `deploy` api-gateway становится единственным сервисом с публикуемым app-портом (`8000:8000`, решение от 2026-10-19), `llm-gateway` переводится в expose-only.

## Boundaries & Constraints

**Always:** токен приходит в `Authorization: Bearer <jwt>`. HS256-секрет знают только `auth-users` (выпуск) и `api-gateway` (проверка), из env `JWT_SECRET`, значение которого обязано совпадать со значением у `auth-users` (в compose — один и тот же `${JWT_SECRET:-...}`); gateway токенов не выпускает. Верификация: `algorithms=["HS256"]`, обязательны `exp`, `sub`, `role`; `company_id` присутствует, но может быть `null`; отсутствие/битая подпись/просрочка/нехватка claims → 401. Identity-заголовки формируются исключительно из проверенных claims; клиентские `X-User-Id`/`X-Company-Id`/`X-Role` удаляются на всех путях, включая публичные. Слои AD-011: `domain` → `application` → `infrastructure` → `transport`, зависимости только внутрь; `domain`/`application` не импортируют FastAPI, httpx, PyJWT. Ошибки — RFC 7807 `application/problem+json` в формате `auth-users`/`company-org`. Прокси: путь и query string сохраняются без переписывания; пробрасываются заголовки клиента, кроме identity-трио, `Host`, `Content-Length` и hop-by-hop (`Connection`, `Transfer-Encoding`, `Keep-Alive`, `Upgrade`); входящий `Authorization` внутрь не пробрасывается (сервисы токен не парсят, AD-010). Порядок отказа: неизвестный префикс → 404 без проверки токена; известный доменный путь без валидного токена → 401 до вызова upstream.

**Never:** другие сервисы не парсят и не проверяют токен. Не в этой истории: refresh/rotation/JWKS, rate-limit, аудит/метрики/трейсинг, CORS и UI-прокси. Не проксировать upstream `/docs`, `/openapi.json` и прочие не перечисленные пути. Не менять контракты, роуты, миграции и БД `auth-users`/`company-org` (в том числе не добавлять им echo-эндпоинты для тестов).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| PUBLIC_ROUTE | `POST /v1/auth/register` (или `login`) без токена | проксируется на auth-users, 201/200 + token | `problem+json` upstream отдаётся как есть |
| AUTHED_PROXY | `POST /v1/companies` с валидным токеном | проксируется; upstream получает `X-User-Id`, `X-Company-Id`, `X-Role` из claims | 401 при отсутствии/битой подписи/просрочке/нехватке `exp`·`sub`·`role`; 502 при таймауте или недоступном upstream |
| SPOOFED_IDENTITY | тот же запрос + клиентские `X-User-Id`/`X-Company-Id`/`X-Role` | upstream получает только значения из claims токена | клиентские заголовки удалены, дублей нет |
| NO_TOKEN_DOMAIN | известный доменный путь без `Authorization` | запрос до upstream не доходит | 401 problem+json |
| COMPANY_SCOPE_MISSING | валидный токен с `company_id: null` | `X-Company-Id` не выставляется, запрос проксируется | отказ формирует upstream (AD-013) |
| UNKNOWN_PREFIX | путь вне `/health` и известных префиксов (`/v1/auth/`, `/v1/companies`) | 404 problem+json | токен не проверяется, upstream не вызывается |
| HEALTH | `GET /health` без токена | 200 `{"status":"ok","service":"api-gateway"}` | N/A |

</frozen-after-approval>

## Code Map

- `api-gateway/app/main.py:1-16` — модульная сборка app (`SERVICE_NAME`, `FastAPI`, `/health`); форму `/health` не менять, здесь подключаются слои.
- `auth-users/app/transport/problem.py:9-31,43-93` — `problem_response`/`ProblemException`/`install_problem_handlers`; копировать форму и media_type (не импортировать: shared-lib нет, AD-009).
- `auth-users/app/main.py`, `company-org/app/main.py` + `app/wiring.py` — эталон сборки app и composition root (модульный `app`, фабрики-раннеры).
- `auth-users/app/infrastructure/tokens.py:29-51` — форма claims (`sub`/`company_id`/`role`/`iat`/`exp`), контракт верификации; не менять.
- `company-org/app/transport/routes.py:19-25` — единственный доменный роут сегодня (`POST /v1/companies`, `Idempotency-Key` обязателен); identity-заголовки пока никто не читает.
- `api-gateway/pyproject.toml` — добавить runtime `pyjwt>=2.8`, `httpx>=0.27`; `uv.lock` трекается → перегенерировать; прочие секции ruff (line-length 100) не трогать.
- `api-gateway/Dockerfile`, `api-gateway/.github/workflows/ci.yml`, `api-gateway/tests/test_health.py` — менять не нужно; CI гоняет `pip install -e ".[dev]"`, `ruff check .`, `pytest -q`, и прежние тесты (в т.ч. старт без env) должны остаться зелёными.
- `deploy/docker-compose.yml` — `expose`-паттерн `company-org:28-42`/`auth-users:44-60`; `llm-gateway:62-90` (`ports:79-80` снять); `postgres:15-18` (`127.0.0.1:5432` остаётся служебным).
- `deploy/.env.example:27`, `deploy/README.md:25,32,40` — `JWT_SECRET` и таблица портов/smoke-команда; привести к факту (README уже (неверно) обещает `localhost:8000 → api-gateway`).
- Новые env gateway: `JWT_SECRET`, `AUTH_USERS_URL` (default `http://auth-users:8000`), `COMPANY_ORG_URL` (default `http://company-org:8000` — compose DNS, как в `auth-users/app/infrastructure/company_client.py:13,21`).

## Tasks & Acceptance

**Execution:**
- [x] `api-gateway/app/transport/problem.py` -- `problem_response`/`ProblemException`/`install_problem_handlers` по образцу auth-users -- единый контракт ошибок edge.
- [x] `api-gateway/app/domain/ports.py` -- порты `TokenVerifier` (verify → `Identity`) и `UpstreamProxy` -- изоляция domain/application от PyJWT/httpx.
- [x] `api-gateway/app/application/routing.py` -- таблица маршрутов: публичные пути, префикс → upstream, порядок 404/401 -- одна точка решения.
- [x] `api-gateway/app/infrastructure/jwt.py` -- HS256-верификация (`algorithms=["HS256"]`, `exp`/`sub`/`role`) → `Identity` -- AD-010.
- [x] `api-gateway/app/infrastructure/proxy.py` -- прокси на `httpx.AsyncClient` (transport инъектируется для тестов): удаление identity-трио и hop-by-hop, установка заголовков из `Identity`, проброс `Idempotency-Key`, тела, статуса и `problem+json`, таймаут → 502.
- [x] `api-gateway/app/transport/routes.py`, `app/wiring.py`, `app/main.py` -- catch-all роут, wiring из env, сохранённый `/health`.
- [x] `api-gateway/pyproject.toml` + `api-gateway/uv.lock` -- runtime-зависимости и lock -- иначе CI/реализация не воспроизводимы.
- [x] `api-gateway/tests/test_gateway.py` -- unit-тесты каждой строки I/O Matrix со stub-upstream, записывающим пришедшие заголовки -- покрытие матрицы.
- [x] `deploy/docker-compose.yml`, `deploy/.env.example`, `deploy/README.md` -- сервис api-gateway с портом `8000:8000`, llm-gateway без `ports`, env по факту.

**Acceptance Criteria:**
- Given валидный токен с `company_id=A`, when клиент шлёт `POST /v1/companies` с `X-Company-Id: B`, then stub-upstream получает ровно `A` (unit-тест; прямое end-to-end наблюдение появится в Story 2.2, когда upstream начнёт читать заголовки).
- Given известный доменный путь без валидного токена, when запрос доходит до gateway, then 401 `application/problem+json` и upstream не вызывается.
- Given `docker compose up`, when клиент делает `curl localhost:8000/health`, then 200 `{"service":"api-gateway"}`, и `llm-gateway` с хоста недоступен.
- Given регистрация через gateway, when клиент регистрируется и вызывает защищённый путь с выданным токеном, then 201, затем успешный проксируемый ответ.

## Implementation Notes

- Реализовано в `api-gateway` (слои AD-011: `domain/ports.py`, `application/routing.py`, `infrastructure/jwt.py`, `infrastructure/proxy.py`, `transport/problem.py`, `transport/routes.py`, `wiring.py`, `main.py`) и в `deploy` (сервис api-gateway `8000:8000`, `llm-gateway` → `expose`, `.env.example`, `README.md`). В `auth-users`/`company-org` изменений нет.
- Отклонения от буквального текста спеки: `httpx` перенесён из `dev`-extra в runtime-зависимости (нужен прокси), `uv.lock` перегенерирован; входящий `Authorization` внутрь не пробрасывается (как решено в спеке); ответ upstream собирается из `response.headers.items()`, поэтому дублирующиеся заголовки ответа (например, несколько `Set-Cookie`) схлопываются — для текущих JSON-контрактов неактуально.
- Тесты: `tests/test_gateway.py` (17) + прежний `tests/test_health.py` (2) = 19 passed; каждая строка I/O Matrix покрыта тестом со stub-upstream, записывающим пришедшие заголовки/путь/query/тело.
- Сюрприз харнесса (не дефект кода): первый `docker compose up -d --build api-gateway` упал на конфликте порта 8000 (его держал прежний `llm-gateway`) и оставил полусозданный контейнер без проброса портов; лечение — `docker compose up -d` (освобождает 8000 у llm-gateway) + `docker compose up -d --force-recreate api-gateway`.
- Смоук на реальном стеке: `/health` → `api-gateway`; `register` 201 + токен; `login` 200; `POST /v1/companies` с токеном и подложенными `X-User-Id/X-Company-Id/X-Role` → 201 от реального company-org; без токена → 401 `application/problem+json`; неизвестный префикс → 404; битый токен → 401; `llm-gateway` с хоста недоступен (порт не публикуется).

## Spec Change Log

## Review Triage Log

Ревью-пасс 1 (слои: blind-hunter, edge-case-hunter, verification-gap). Диф: `/tmp/opencode/gateway-review-final-1792407874.diff`.

**patch (группы):**
- [low] `api-gateway/app/infrastructure/proxy.py` — неполная гигиена hop-by-hop: в запросе не удаляются токены, перечисленные в `Connection`, а также `TE`, `Trailer`, `Proxy-Connection`; в ответе не удаляются `Keep-Alive`, `Upgrade`, `Proxy-Authenticate`/`-Authorization`. Наименьший фикс — расширить два frozenset в `proxy.py`. (blind-hunter BT5; edge-case-hunter EC1, EC2)
- [low] `api-gateway/app/infrastructure/proxy.py` — `httpx.InvalidURL` не подкласс `httpx.HTTPError`, поэтому кривой `AUTH_USERS_URL`/`COMPANY_ORG_URL` даёт 500 вместо 502; наименьший фикс — добавить тип в `except`. (edge-case-hunter EC5)
- [medium] `api-gateway/tests/` — состав production-приложения (подключение роутера и problem-обработчиков в `app/main.py`) и env-контракт (`JWT_SECRET`, `AUTH_USERS_URL`, `COMPANY_ORG_URL` в `wiring.py`) не наблюдаются ни одним тестом: слой выполнил мутации (убрать `include_router`/`install_problem_handlers`, опечатка в имени env) — 19 тестов остаются зелёными, хотя в проде это 404 на всех `/v1/**` или 401/502 на всех запросах. Наименьший фикс — тест через `main.app` с env + `importlib.reload` (паттерн `auth-users/tests/test_auth.py:42-51`). (verification-gap VG1, VG2)
- [low] `deploy/README.md` — добавленное утверждение «Наружу публикуется только api-gateway» неверно: `postgres` публикует `127.0.0.1:5432` (`docker-compose.yml:15-18`), а таблица рядом перечисляет `ui`/MinIO, которых в compose нет; наименьший фикс — прямая правка формулировки. (edge-case-hunter EC8; verification-gap «Other findings»)

**rejected — false:**
- Нет assertion, что `Authorization` не пробрасывается upstream (BT2): `tests/test_gateway.py` проверяет `"authorization" not in request.headers` и в PUBLIC_ROUTE, и в AUTHED_PROXY — утверждение неверно.
- «502-ответ не описан как problem+json (type/title)» (BT3): обработчик `UpstreamUnavailable` отдаёт `502 Bad Gateway` с media_type `application/problem+json`, это проверяет `test_upstream_unreachable_returns_502`; ущерба нет.
- Нет теста на отсутствие `X-Company-Id` при `company_id=null` (BT4): `test_company_scope_missing_omits_company_header` параметризован `None`/«absent» и ассертит `get_list("x-company-id") == []`.
- Порядок маршрутизации не покрыт тестом (BT8): `test_unknown_prefix_returns_404_without_token_check` шлёт запрос без токена и получает 404 (а не 401) — именно это и есть проверка порядка.
- `/health` не агрегирует upstream, вопреки intent (EC9): `I/O Matrix` (HEALTH) и AC задают локальный `/health` отдаваемого приложения; агрегации в интенте нет.

**rejected — low:**
- Нет `leeway` для `exp` (BT1): дрейф часов между контейнерами одного хоста пренебрежим при TTL 168ч; фикс добавляет параметр.
- Поведение других методов/подпутей `/v1/auth/*` не задано (BT6): спека фиксирует публичными ровно `POST /v1/auth/register|login`; не бытовой случай, фикс добавляет логику матчинга.
- Сырой не-ASCII в query после `decode("latin-1")` даёт двойное кодирование (EC3): клиенты шлют percent-encoded; фикс — переупаковка кодирования, не прямая правка.
- Дублирующиеся заголовки ответа схлопываются `dict(...)` (BT7, EC4): в текущих JSON-контрактах нет cookie/дублей; фикс — перестройка `Response` на raw-пары.
- Глобальный `httpx.AsyncClient` не закрывается (EC6): предупреждения при остановке; фикс — lifespan, новая инфраструктура.
- `POST /v1/auth/register/` со слэшем получает 401 (EC7): не бытовой случай; фикс — нормализация пути.
- Compose-контракт проверяется только `docker compose config -q` (VG3): в `deploy` нет тестовой инфраструктуры, проводка покрыта ручным smoke из Verification; фикс — новые CI-ассерты, не прямая правка.

**follow-up (walkthrough 2026-10-19):**
- [patch] `deploy/docker-compose.yml` — dev-дефолт `JWT_SECRET` устранён: `${JWT_SECRET:?...}` для `auth-users` и `api-gateway`, `.env.example` без значения, README переписан (`.env` теперь обязателен). (walkthrough, решение Shved) — `deploy@1090621`.
- [tracking] Edge-слой внесён в трекинг эпика: `epics.md` Story 2.0 (внеплановая) и `sprint-status.yaml` `2-0-edge-jwt-и-identity-заголовки: done`.
- Оставлено намеренно (по обсуждению): порядок «404 раньше 401» с 401 на не-POST `/v1/auth/*`; `/health` не агрегирует upstream; все остальные low-находки без изменений.

**rejected — правило 86 (фикс = правка этой спеки):**
- Утверждение спеки «/health проксируется без токена» vs локальный `/health` (уже покрыто `rejected — false` EC9).
- Дублирующийся заголовок `## Implementation Notes` в спеке (verification-gap «Other findings»): фикс — правка этой же спеки; дубль удалён попутно как исправление собственной ошибки триажа.


## Design Notes

Копирование `problem.py` в gateway, а не общий пакет, — следствие AD-009 (shared-lib не вводится, контракты только по API). Gateway не знает про сущности: `application`-слой содержит лишь таблицу маршрутов и правила отказа, поэтому прокси и верификация тестируются без БД и без upstream'ов.

## Verification

**Commands:**
- `cd /home/shved/projects/smeta/api-gateway && uv run --extra dev pytest -q` -- зелёные все тесты, включая прежний `tests/test_health.py`.
- `cd /home/shved/projects/smeta/api-gateway && uvx ruff check .` -- `All checks passed!`
- `docker compose -f /home/shved/projects/smeta/deploy/docker-compose.yml config -q` -- OK.
- `docker compose up -d --build` + smoke: `register` 201, `login` 200, защищённый `POST /v1/companies` с токеном → ответ upstream, без токена → 401, неизвестный префикс → 404, `curl localhost:8000/health` → api-gateway, llm-gateway с хоста недоступен.

**Manual checks (if no CLI):**
- В логах gateway нет проброса клиентских identity-заголовков; страницы ошибок идут с `content-type: application/problem+json`.

### Post-patch (после триажа ревью, диф `/tmp/opencode/gateway-review-final-1792410067.diff`)

Исправлено 4 группы находок (hop-by-hop в обе стороны; `httpx.InvalidURL` → 502; тест состава `app/main.py` + env; формулировка `deploy/README.md`); отклонённые находки и обоснования — в `## Review Triage Log`.

- `cd /home/shved/projects/smeta/api-gateway && uv run --extra dev pytest -q` -- **23 passed** (было 19).
- `cd /home/shved/projects/smeta/api-gateway && uvx ruff check .` -- `All checks passed!`; `uv lock --check` -- OK (32 packages).
- `cd /home/shved/projects/smeta/deploy && docker compose config -q` -- COMPOSE_OK; `docker compose up -d --build --force-recreate api-gateway` → `smeta1-api-gateway | Up (healthy) | 0.0.0.0:8000->8000/tcp`; `llm-gateway` -- `8000/tcp` без host-публикации; `postgres` -- `127.0.0.1:5432`.
- Smoke после патчей: `/health` → `{"status":"ok","service":"api-gateway"}`; `register` 201 + `login` 200; `POST /v1/companies` с токеном и подложенными `X-Company-Id: spoof`/`X-Role: admin` → 201; без токена → 401 `application/problem+json`; неизвестный префикс → 404 `application/problem+json`.
