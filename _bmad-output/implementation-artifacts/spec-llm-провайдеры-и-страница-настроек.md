---
title: 'Провайдеры LLM и страница настроек моделей'
type: 'feature'
created: '2026-10-22'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
baseline_commit: 'polyrepo: spec-service=a1c14e1232b2c76b197b48cd5b74b9060d171699 llm-proxy=b715d4013231708b7aa07068d549be0b2cffb853 deploy=622738234aa6672ba63d961fd9f2253205d81db1 ui=ccab340e8c28a269cc9191a71179534df99a99e1 core=e16ac1ea10ae37521ea28362b341d0423aca5fe8'
context:
  - '{project-root}/docs/architecture-spine.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Провайдеры LLM и их ключи сейчас задаются только в YAML/env `llm-proxy`; в UI нельзя ни добавить провайдера, ни изменить `base_url`, `api_key_env` или параметры модели, ни сразу применить это к proxy.

**Approach:** Добавить в `spec_service` глобальный каталог провайдеров и моделей и страницу `/settings/llm` для всех авторизованных пользователей. В БД хранится только имя env-переменной ключа. После изменения `spec-service` передаёт эффективную конфигурацию в `llm-proxy` по внутреннему API; proxy применяет её без рестарта, а при старте подтягивает из `spec-service`. Активная модель остаётся per-company.

## Boundaries & Constraints

**Always:** Все авторизованные пользователи могут открывать страницу и менять общий каталог. Ключи не попадают в БД, API и UI — только `api_key_env`. Внутренний API конфигурации защищён `LLM_INTERNAL_TOKEN`. Все изменения аудируются с `user_id`, без секретов. Единственный источник конфигурации proxy — БД `spec_service`; статического YAML нет. Активная модель — per-company.

**Never:** Не назначать роли через SQL и не требовать admin. Не хранить значения ключей. Не менять `auth-users`. Не ломать существующие `/v1/spec/extract` и cloud consent.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|---|---|---|---|
| Создание провайдера | все поля валидны | запись в БД + push в proxy | 201/200 |
| Невалидный env | `api-key` | отказ | 422 |
| Изменение `api_key_env` | `DEEPSEEK_API_KEY` | proxy использует новое имя | 200 |
| Proxy недоступен | push не удался | БД сохранена, фоновая повторная синхронизация | 202/200 |
| Старт proxy | spec-service доступен | конфиг подтянут по API | 200 |
| Старт proxy | spec-service недоступен | proxy not_ready, повторный pull | 503 до конфига |
| Дубликат proxy_alias | две модели | отказ | 422 |
| Любая роль | estimator/admin | управление разрешено | 200 |
| Секрет в запросе | поле `api_key` | отказ | 422 |

</frozen-after-approval>

## Code Map

- `spec-service/migrations/versions/0003_llm_providers.py` — таблицы и поля.
- `spec-service/app/infrastructure/models.py` — `LlmProviderRow`, `LlmConfigAuditRow`, поля `LlmModelRow`.
- `spec-service/app/infrastructure/llm_provider_repository.py` — CRUD провайдеров.
- `spec-service/app/infrastructure/llm_model_repository.py` — модели, активная, конфиг.
- `spec-service/app/infrastructure/llm_config_audit_repository.py` — аудит.
- `spec-service/app/infrastructure/llm_config_publisher.py` — push в proxy.
- `spec-service/app/application/llm_settings.py` — CRUD, сборка конфига, валидация.
- `spec-service/app/transport/{schemas,routes,internal_routes}.py` — API и internal routes.
- `spec-service/app/{config,wiring,main}.py` — env, зависимости, sync task.
- `llm-proxy/app/config.py` — парсинг конфига из mapping.
- `llm-proxy/app/proxy.py` — атомарная замена runtime config.
- `llm-proxy/app/internal_routes.py` — `PUT /internal/config`.
- `llm-proxy/app/config_sync.py` — startup pull.
- `llm-proxy/app/main.py` — lifespan/ready.
- `deploy/{docker-compose.yml,.env.example,README.md}` — internal token/URL.
- `ui/src/api/client.ts` — типы/вызовы провайдеров и моделей.
- `ui/src/views/LlmSettingsView.vue` — страница.
- `ui/src/router.ts`, `ui/src/App.vue`, `ui/tests/*` — маршрут, ссылка, тесты.

## Tasks & Acceptance

**Execution:**
- [x] `spec-service/migrations/versions/0003_llm_providers.py` — создать `llm_provider`, поля модели, аудит, seed.
- [x] `spec-service/app/infrastructure/models.py` — ORM.
- [x] `spec-service/app/infrastructure/llm_provider_repository.py` — CRUD.
- [x] `spec-service/app/infrastructure/llm_model_repository.py` — provider/upstream, конфиг.
- [x] `spec-service/app/infrastructure/llm_config_audit_repository.py` — аудит.
- [x] `spec-service/app/infrastructure/llm_config_publisher.py` — push.
- [x] `spec-service/app/application/llm_settings.py` — CRUD/валидация/конфиг.
- [x] `spec-service/app/transport/*` — API без ролевых ограничений.
- [x] `llm-proxy/app/config.py` — mapping.
- [x] `llm-proxy/app/proxy.py` — runtime config.
- [x] `llm-proxy/app/internal_routes.py`, `config_sync.py`, `main.py` — API/старт.
- [x] `deploy/*` — token/URL.
- [x] `ui/*` — страница, маршрут, ссылка, формы.
- [x] `spec-service/README.md`, `llm-proxy/README.md`, `deploy/README.md` — контракт.

**Acceptance Criteria:**
- Given любой авторизованный пользователь, when открывает `/settings/llm`, then видит провайдеров и модели без 403.
- Given изменение провайдера, when сохранение, then `llm-proxy /v1/models` отражает новый маршрут без рестарта.
- Given `api_key_env=DEEPSEEK_API_KEY`, when proxy делает запрос, then берёт ключ из env с этим именем.
- Given ключ передан в теле, when запрос, then 422 и ключ не сохраняется.
- Given `proxy_alias` дублируется, when сохранение, then 422.
- Given proxy недоступен, when сохранение, then конфиг сохранён и синхронизация повторится.
- Given рестарт proxy, when spec-service доступен, then конфиг подтянут по API.

## Implementation Notes

- Добавлены `llm_provider`, поля `llm_model.provider_name/upstream_model`, `llm_config_audit`.
- CRUD провайдеров/моделей доступен всем авторизованным; ключи не в БД/UI, только `api_key_env`.
- `spec-service` собирает runtime-конфиг и публикует в `llm-proxy` через `PUT /internal/config`; proxy при старте забирает `GET /internal/llm/routes`.
- `llm-proxy` получил runtime-конфиг с атомарной заменой и внутренний API; YAML не используется.
- UI: `/settings/llm`, ссылка в шапке, формы провайдеров/моделей и активация.
- Ключи провайдеров не привязаны к `DEEPSEEK_API_KEY`: compose подключает `deploy/llm-proxy.env` целиком, имя переменной задаётся в UI; после добавления новой переменной нужен recreate контейнера proxy.
- `deploy/llm-routes.yaml` удалён; `llm-proxy` не читает YAML.
- Проверки: spec-service 104 passed, llm-proxy 22, api-gateway 64, UI 121 + typecheck/build; миграция 0003 upgrade/downgrade на SQLite; compose config и docker build.

## Spec Change Log

## Review Triage Log

| Finding | Verdict | Evidence |
|---|---|---|
| В UI/БД мог попасть API-ключ | false | Схемы `extra=forbid` не содержат `api_key`; хранится только `api_key_env`; тест проверяет отсутствие ключа в теле. |
| Изменения не доходят до proxy | high | Исправлено: push `PUT /internal/config`, startup pull, retry каждые 30 с; тесты обеих сторон. |
| Proxy падает без spec-service | high | Исправлено: proxy стартует not_ready, pull-loop повторяет; YAML fallback удалён по решению владельца. |
| Дублирование `proxy_alias` ломает routes | medium | Исправлено: проверка в сервисе + уникальный индекс; тест. |
| Невалидный env/base_url попадает в конфиг | medium | Исправлено: валидация имени env и http(s) URL; тесты. |
| Ограничение доступа ролями противоречит решению | false | Проверки ролей удалены; UI и API доступны всем авторизованным. |
| Секреты в аудите | false | `changes` содержит только несекретные поля; значения ключей не читаются. |
| Сломанная миграция на существующей БД | high | Проверено `alembic upgrade head` и downgrade 0003 на SQLite с существующей `spec_job`; тест. |

## Design Notes

`spec_service` — единственный источник истины. `llm-proxy` хранит runtime config в памяти, стартует без маршрутов и получает конфиг по API; до получения `/ready` = 503. Push после изменений + фоновый pull. В БД только `api_key_env`, значение — в env контейнера proxy.

## Verification

**Commands:**
- `cd spec-service && uv run --extra dev pytest -q && uv run --extra dev ruff check .`
- `cd llm-proxy && uv run --extra dev pytest -q && uv run --extra dev ruff check .`
- `cd api-gateway && uv run --extra dev pytest -q`
- `cd ui && npm test && npm run typecheck && npm run build`
- `cd deploy && docker compose config -q`

**Manual checks:**
- Открыть `/settings/llm` под estimator, создать провайдера/модель, проверить `llm-proxy /v1/models`; значение ключа в env, не в UI/БД.
