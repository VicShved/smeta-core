---
title: 'Story 2.1 — Авторизация и привязка к компании'
type: 'feature'
created: '2026-10-19'
status: 'ready-for-dev'
route: 'dispatch'
review_loop_iteration: 0
story_key: '2-1-авторизация-и-привязка-к-компании'
epic_num: 2
story_num: 1
context:
  - epic-2-context.md
  - ../../planning-artifacts/epics-smeta1-2026-10-17/epics.md
  - ../../planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** В системе нет аутентификации и компаний: `auth-users` и `company-org` — заглушки с одним `/health`, в `deploy/docker-compose.yml` нет Postgres и этих сервисов (после Story 1.3 остался только `llm-gateway`). Сметчик не может войти и работать в изолированном пространстве своей организации.

**Approach:** Реализовать в `auth-users` регистрацию (создать пользователя → создать компанию через API `company-org` → сохранить `company_id` → вернуть JWT) и вход с выпуском JWT, а в `company-org` — идемпотентное создание компании; поднять один PostgreSQL (база на сервис) и оба сервиса в compose.

## Boundaries & Constraints

**Always:** AD-011 (слои `domain`/`application`/`infrastructure`/`transport`, зависимости только внутрь; FastAPI и SQLAlchemy не в `domain`/`application`). AD-012 (владелец `user` — auth-users, `company` — company-org; создание компании только через API, идемпотентно, без распределённой транзакции). AD-014 (один контейнер Postgres, отдельные БД `auth_users` и `company_org`, Alembic у каждого сервиса). UUID, ошибки `application/problem+json`, пути `/v1/...`, `snake_case`, конфигурация через env. Единая роль `estimator`. Существующие `tests/test_health.py` остаются зелёными.

**Never:** Договоры, этапы, подразделы (Stories 2.2–2.4). api-gateway и всё, что связано с проверкой JWT, identity-заголовками и публикацией портов (вынесено в deferred-work). MinIO и прочие сервисы в compose. Refresh-токены, роли >1, вступление в существующую компанию. Shared-lib. Прямое чтение БД другого сервиса. Коммит `.env`. Изменения кода `llm-gateway` и `ui`.

**Решения по контракту (утверждены человеком):** учётные данные — **email + пароль**; `company_name` — обязательное поле запроса регистрации; хэш пароля — **stdlib `hashlib.scrypt`** (параметры-константы + случайная соль, без внешних зависимостей); регистрация **сразу возвращает токен**; внутренний вызов auth-users → company-org аутентифицируется **доверием внутренней сети + `Idempotency-Key`** (пользователя ещё нет); при сбое создания компании **созданный `user` удаляется** и возвращается ошибка.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| REGISTER_HAPPY | `POST /v1/auth/register` {email, password, company_name} | 201, user создан, компания создана, `company_id` сохранён, возвращён JWT | N/A |
| LOGIN_HAPPY | `POST /v1/auth/login` {email, password} | 200, JWT HS256, claims `sub`/`company_id`/`role`/`iat`/`exp`, TTL 168ч | N/A |
| LOGIN_BAD_CREDENTIALS | неверный пароль или неизвестный email | 401, один и тот же ответ для обоих случаев | problem+json |
| REGISTER_DUPLICATE | email уже зарегистрирован | 409, вторая компания/пользователь не создаются | problem+json |
| REGISTER_VALIDATION | невалидный email, короткий пароль, пустой `company_name` | 422 | problem+json со списком полей |
| COMPANY_API_DOWN | `company-org` недоступен при регистрации | 502, созданный user удалён (компенсация), повтор регистрации возможен | problem+json |
| IDEMPOTENT_COMPANY | повтор `POST /v1/companies` с тем же `Idempotency-Key` | та же компания, дубль не создаётся | N/A |
| COMPANY_MISSING_KEY | `POST /v1/companies` без `Idempotency-Key` | 400, компания не создаётся | problem+json |
| REPO_SCOPE_REQUIRED | запрос к репозиторию без `company_id` | отказ, а не выборка всех строк | исключение/401 в слое выше |

</frozen-after-approval>

## Code Map

Каркас уже есть в обоих сервисах: `app/__init__.py`, `app/main.py` (FastAPI, `/health`, `SERVICE_NAME` из env), `tests/test_health.py`, `pyproject.toml` (`fastapi>=0.115`, `uvicorn[standard]>=0.30`; dev: `pytest`, `httpx`, `ruff`; line-length 100), `Dockerfile` (`python:3.12-slim`, HEALTHCHECK на `/health`, CMD uvicorn на `APP_PORT`), `.dockerignore`, `uv.lock`, CI (ruff + pytest). `app/main.py` не переписывать — только подключать роутер.

Новая структура (AD-011) в `auth-users/app/` и `company-org/app/`: `domain/` (сущности, порты), `application/` (use case'ы), `infrastructure/` (репозиторий SQLAlchemy, `db.py`, HTTP-клиент, хэш/JWT), `transport/` (роуты, Pydantic-схемы, problem+json). Миграции — `<service>/alembic.ini` + `migrations/` (`env.py` читает `POSTGRES_DSN`). В `pyproject.toml` обоих сервисов: `sqlalchemy>=2`, `alembic`, `psycopg[binary]`; в `auth-users` ещё `pyjwt`.

`deploy/docker-compose.yml`: добавить `postgres` (том, healthcheck), `deploy/initdb/01-create-databases.sql` (БД `auth_users`, `company_org`), сервисы `auth-users` и `company-org` (`expose: 8000`, без `ports`, `depends_on: postgres: service_healthy`). `deploy/.env.example`: `POSTGRES_USER/PASSWORD/DB`, `POSTGRES_DSN` (со своим именем БД у каждого сервиса), `JWT_SECRET`.

## Tasks & Acceptance

**Execution:**
- [ ] `company-org/app/domain/`, `company-org/app/application/` -- сущность `Company` (id UUID, name, idempotency_key, created_at), порт репозитория и use case создания с идемпотентностью -- AD-011/012/014
- [ ] `company-org/app/infrastructure/`, `company-org/app/transport/` -- SQLAlchemy-репозиторий (поиск по `idempotency_key`), `POST /v1/companies` -- AD-012
- [ ] `company-org/alembic.ini`, `company-org/migrations/`, `company-org/pyproject.toml` -- Alembic, миграция `company` с уникальным индексом по `idempotency_key`, зависимости -- AD-014
- [ ] `auth-users/app/domain/`, `auth-users/app/application/` -- `User` (id UUID, email уникальный, password_hash, company_id, role), порты; use case'ы регистрации (user → company-org → `company_id`, компенсация удалением) и входа -- AD-011/012
- [ ] `auth-users/app/infrastructure/` -- репозиторий (все выборки с `company_id`), scrypt-хэш (соль + константы N/r/p), выпуск JWT HS256, HTTP-клиент company-org с `Idempotency-Key` -- AD-010/013/014
- [ ] `auth-users/app/transport/` -- `POST /v1/auth/register`, `POST /v1/auth/login`, problem+json -- Intent
- [ ] `auth-users/alembic.ini`, `auth-users/migrations/`, `auth-users/pyproject.toml` -- Alembic, миграция `user`, зависимости (sqlalchemy, alembic, psycopg, pyjwt) -- AD-014
- [ ] `deploy/docker-compose.yml`, `deploy/initdb/01-create-databases.sql`, `deploy/.env.example` -- postgres + оба сервиса + env -- AD-014/NFR-005
- [ ] `company-org/tests/test_companies.py` -- создание, повтор с тем же ключом → та же компания, отсутствие ключа → 400, валидация -- I/O Matrix
- [ ] `auth-users/tests/test_auth.py` -- регистрация (мок company-org) → 201 + токен и сохранённый `company_id`; дубликат email → 409; неверный пароль → 401; сбой company-org → 502 и удалённый user; хэш пароля не равен plaintext -- I/O Matrix

**Acceptance Criteria:**
- Given незарегистрированный пользователь, when он регистрируется, then создаётся компания, пользователь привязан к ней, роль `estimator`, и возвращён JWT
- Given зарегистрированный пользователь, when он входит, then получает JWT с `company_id` и TTL 168ч
- Given сбой `company-org` при регистрации, when вызов company-org не удался, then пользователь не остаётся без компании (удалён), возвращена понятная ошибка
- Given две компании, when сервис выбирает пользователей, then возвращаются только строки запрошенной компании, а выборка без `company_id` отклоняется

## Implementation Notes


## Spec Change Log


## Review Triage Log


## Design Notes

- Регистрация: валидация → insert user → HTTP `POST /v1/companies` (`Idempotency-Key` = ключ, сгенерированный auth-users и сохранённый с компанией) → update `company_id` → выпуск токена. При сбое вызова — удаление user.
- scrypt: `hashlib.scrypt(password, salt=os.urandom(16), n=2**14, r=8, p=1)`, хранение `scrypt$n$r$p$salt_b64$hash_b64`; сравнение — `hmac.compare_digest`.
- JWT: `PyJWT`, HS256, `JWT_SECRET` из env; claims `sub` (user_id), `company_id`, `role`, `iat`, `exp` (now + 168ч).
- Оба сервиса слушают 8000 внутри сети; `auth-users` адресует `company-org` по `COMPANY_ORG_URL`.

## Verification

**Commands:**
- `uv run --extra dev --directory /home/shved/projects/smeta/company-org pytest -q` и то же для `auth-users` -- expected: зелёные (старые `test_health` + новые)
- `(cd /home/shved/projects/smeta/<svc> && uvx ruff check .)` для `company-org`, `auth-users` -- expected: `All checks passed`
- `uv run --directory /home/shved/projects/smeta/<svc> alembic upgrade head` (`company-org` и `auth-users`) -- expected: миграции применены
- `docker compose -f /home/shved/projects/smeta/deploy/docker-compose.yml config -q` -- expected: без ошибок
- `docker compose -f ... up -d --build` + `ps` -- expected: postgres, auth-users, company-org healthy
- `docker compose -f ... exec auth-users python -c "import httpx; r=httpx.post('http://127.0.0.1:8000/v1/auth/register', json={'email':'a@b.c','password':'secret123','company_name':'ООО Тест'}); print(r.status_code, r.json())"` -- expected: 201 и токен; повтор с тем же email -- expected: 409
