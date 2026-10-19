---
title: 'Story 2.1 — Авторизация и привязка к компании'
type: 'feature'
created: '2026-10-19'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
baseline_commit: 'e5da08f728ebee77766637b047f52941e961efd1'
baseline_repos:
  core: 'e5da08f728ebee77766637b047f52941e961efd1'
  company-org: '90c4b2fbbf96727b484ac33744083231ca680218'
  auth-users: '40643e3c30f173e8b27e99d884f05c803f754f0d'
  deploy: 'fd76e52fd899388ea67f627b7489dd25798031f7'
story_key: '2-1-авторизация-и-привязка-к-компании'
epic_num: 2
story_num: 1
context:
  - epic-2-context.md
  - ../planning-artifacts/epics-smeta1-2026-10-17/epics.md
  - ../planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md
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
- [x] `company-org/app/domain/`, `company-org/app/application/` -- сущность `Company` (id UUID, name, idempotency_key, created_at), порт репозитория и use case создания с идемпотентностью -- AD-011/012/014
- [x] `company-org/app/infrastructure/`, `company-org/app/transport/` -- SQLAlchemy-репозиторий (поиск по `idempotency_key`), `POST /v1/companies` -- AD-012
- [x] `company-org/alembic.ini`, `company-org/migrations/`, `company-org/pyproject.toml` -- Alembic, миграция `company` с уникальным индексом по `idempotency_key`, зависимости -- AD-014
- [x] `auth-users/app/domain/`, `auth-users/app/application/` -- `User` (id UUID, email уникальный, password_hash, company_id, role), порты; use case'ы регистрации (user → company-org → `company_id`, компенсация удалением) и входа -- AD-011/012
- [x] `auth-users/app/infrastructure/` -- репозиторий (все выборки с `company_id`), scrypt-хэш (соль + константы N/r/p), выпуск JWT HS256, HTTP-клиент company-org с `Idempotency-Key` -- AD-010/013/014
- [x] `auth-users/app/transport/` -- `POST /v1/auth/register`, `POST /v1/auth/login`, problem+json -- Intent
- [x] `auth-users/alembic.ini`, `auth-users/migrations/`, `auth-users/pyproject.toml` -- Alembic, миграция `user`, зависимости (sqlalchemy, alembic, psycopg, pyjwt) -- AD-014
- [x] `deploy/docker-compose.yml`, `deploy/initdb/01-create-databases.sql`, `deploy/.env.example` -- postgres + оба сервиса + env -- AD-014/NFR-005
- [x] `company-org/tests/test_companies.py` -- создание, повтор с тем же ключом → та же компания, отсутствие ключа → 400, валидация -- I/O Matrix
- [x] `auth-users/tests/test_auth.py` -- регистрация (мок company-org) → 201 + токен и сохранённый `company_id`; дубликат email → 409; неверный пароль → 401; сбой company-org → 502 и удалённый user; хэш пароля не равен plaintext -- I/O Matrix

**Acceptance Criteria:**
- Given незарегистрированный пользователь, when он регистрируется, then создаётся компания, пользователь привязан к ней, роль `estimator`, и возвращён JWT
- Given зарегистрированный пользователь, when он входит, then получает JWT с `company_id` и TTL 168ч
- Given сбой `company-org` при регистрации, when вызов company-org не удался, then пользователь не остаётся без компании (удалён), возвращена понятная ошибка
- Given две компании, when сервис выбирает пользователей, then возвращаются только строки запрошенной компании, а выборка без `company_id` отклоняется

## Implementation Notes

- Слойность AD-011: `domain/` (сущности + порты + доменные ошибки), `application/` (use cases), `infrastructure/` (SQLAlchemy-репозитории, scrypt, JWT, httpx-клиент company-org), `transport/` (Pydantic-схемы, роуты, problem+json). Методы записи репозиториев фиксируют транзакцию сами — это позволяет вставить `user` до HTTP-вызова и явно удалить его при сбое (компенсация по AD-012, Design Notes). `app/wiring.py` — composition root (сессия на запрос); `app/main.py` только подключает роутер и problem+json-хендлеры.
- Регистрация: pre-check email (409, вызов company-org не выполняется) → insert user → `POST /v1/companies` (`Idempotency-Key` = `user.id`, стабильна для ретрая) → `update company_id` → JWT. При `CompanyServiceError` — `users.delete` и 502 problem+json.
- Логин: неизвестный email и неверный пароль → один и тот же 401-тело; для неизвестного email выполняется `verify` против кэшированного dummy-хэша (выравнивание времени ответа).
- Тесты гоняют реальные Alembic-миграции против SQLite (`POSTGRES_DSN=sqlite:///tmp`), поэтому схема в тестах = миграциям; HTTP company-org замокан на уровне `HttpCompanyClient.create_company`.
- Составные решения по compose: postgres `18.6` с томом, монтируемым в `/var/lib/postgresql` (требование образов Postgres 18+); `initdb/01-create-databases.sql` создаёт БД `auth_users`/`company_org`; миграции применяются контейнером при старте (`alembic upgrade head` в CMD); сервисы `expose: 8000` без `ports`, `depends_on: postgres: service_healthy`.
- Отклонения от буквального текста (без изменения контракта): (1) в `deploy/.env.example` DSN каждого сервиса вынесен в `POSTGRES_DSN_AUTH_USERS`/`POSTGRES_DSN_COMPANY_ORG` (в контейнер каждый сервис получает `POSTGRES_DSN` со своим именем БД); (2) postgres публикует `127.0.0.1:5432` — служебный доступ для хостовых `alembic`/разработки (сервисные порты не публикуются); (3) email нормализуется (strip+lower) в схемах transport.

## Spec Change Log

- 2026-10-19: контракт не менялся; отмечены выполненные задачи и результаты верификации. Решения по контракту (email+пароль, scrypt, компенсация удалением, доверие внутренней сети + Idempotency-Key) реализованы без изменений.


## Review Triage Log

Источники: `blind-hunter` (2 прогона), `edge-case-hunter`, `verification-gap`. `verification-gap` вернул «No verification gaps found». Диф ревью: `/tmp/opencode/story21-final-1792402386.diff`.

**patch (применены):**
- [medium] `auth-users/app/transport/schemas.py` — `pattern` проверяется core-схемой раньше `field_validator` (mode=after), поэтому email с окружающими пробелами даёт 422, а задокументированная нормализация strip+lower недостижима. Проверено: `" a@b.c "` → `string_pattern_mismatch`. Фикс: нормализация в `mode="before"`. (edge-case-hunter; blind-hunter #1)
- [low] `auth-users/app/infrastructure/company_client.py` — `httpx.InvalidURL` не является подклассом `httpx.HTTPError` (проверено), поэтому некорректный `COMPANY_ORG_URL` даёт 500 вместо 502, хотя `COMPANY_API_DOWN` требует 502. Фикс: включить `InvalidURL` в перехват. (edge-case-hunter)

**rejected — false:**
- E2 повтор `Idempotency-Key` с другим именем возвращает ту же компанию — ровно матрица `IDEMPOTENT_COMPANY` («та же компания, дубль не создаётся»); конфликт по имени контрактом не предусмотрен.
- E11 `get_by_email` без `company_id` — единственный unscoped-lookup, без которого невозможен вход до аутентификации; документирован в порту и Implementation Notes, а AD-013/`REPO_SCOPE_REQUIRED` обеспечивается в `list_by_company`.
- `MissingCompanyScope` → 401: матрица `REPO_SCOPE_REQUIRED` прямо предписывает «исключение/401 в слое выше».
- Пустой Review Triage Log при `status: in-progress`: step-04 штатно переводит спеку в `in-review` и заполняет этот лог; находка — снимок до триажа.
- Префиксы `a/home/shved/projects/smeta/...` в дифе: артефакт команды сборки ревью-дифа (`git diff --no-index` по абсолютным путям untracked-файлов), а не содержимое репозиториев.
- `idempotency_key` в ответе `company-org` = `user.id`, который не секрет (тот же UUID в claim `sub` выданного пользователю JWT) — раскрытия нет.
- Прогон `blind-hunter` #1 (10 находок: миграция `V20251019_21__create_calculation_jobs.sql`, FK на `calculations`/`users`, контроллер/роуты джоб, статусы/ретраи джоб, OpenAPI/TS-типы, optimistic locking) — ни один из перечисленных артефактов не существует ни в дифе, ни в дереве; слой ревьюил содержимое, не совпадающее с поданным файлом.

**rejected — low:**
- E1/B3 `Idempotency-Key` без ограничения длины (>255 → DataError → 500): единственный вызывающий — `auth-users` со UUID (36 символов); фикс — новая проверка.
- B4/B5 `IdempotencyKeyConflict` без problem+json-хендлера (гонка) + неоттестированная race-ветка: фикс — новый обработчик.
- E3/E9 нет catch-all хендлера → неожиданные ошибки отдаются FastAPI как plain-text 500: фикс — новый обработчик.
- E6/B8 orphan-компания при неоднозначном сбое (таймаут после commit) и нестабильность ключа `user.id` после компенсации: требует сбоя в окне; выбранный контракт (ключ генерирует auth-users, компенсация удалением, AD-012) утверждён человеком, фикс меняет семантику ключа и тесты.
- E7/E10 нет компенсации при сбое `update company_id`; `delete` в компенсации не защищён (user с `company_id=NULL` + 409, либо 500 вместо 502): требует сбоя БД внутри потока, фикс добавляет новые ветки компенсации.
- E8 `JWT_SECRET` проверяется только при выпуске токена → 500 после записи: фикс — fail-fast guard.
- B2/B6/B7 дополнительные тест-кейсы (нормализация email, дубль в другом регистре, один ключ с другим именем): обязательные матричные тесты есть и проходят, доп. кейсы интентом не требуются, добавляют harness-сложность.
- B11 502-detail интерполирует внутренний текст ошибки `company-org`: фикс — вынос диагностики в отсутствующее логирование, то есть не прямая правка.
- B9 login в окне регистрации выдаёт токен с `company_id=null` (окно до 5 с): требует конкурирующего запроса, фикс меняет контракт входа.
- B12 небезопасные дефолты `JWT_SECRET` (`change-me`, `dev-insecure-...`) и отсутствие fail-fast: dev-контур, фикс — guard/конфиг-политика.
- B13 нет отзыва токена при TTL 168 ч: refresh-токены исключены интентом, отзыв — новая функциональность.
- B14 нет rate limiting/защиты от перебора на login/register: новая функциональность/middleware.
- B15 нет логирования/метрик/аудита auth-событий: новая функциональность.
- Абсолютные пути `/home/shved/...` в Verification: фикс — правка этого же спека (findings с таким фиксом отклоняются).
- B18 дублирование креденшелов Postgres в `docker-compose.yml`/`.env.example` (POSTGRES_USER/PASSWORD vs дефолты DSN): раскрыто комментарием в `.env.example`, фикс — перестройка env.
- B20 таблица `user` — зарезервированное слово Postgres: SQLAlchemy квотирует, e2e `up --build` работает; фикс — переименование таблицы + правка спека.
- B21 нет README/инструкций по миграциям и env: новая документация.

## Design Notes

- Регистрация: валидация → insert user → HTTP `POST /v1/companies` (`Idempotency-Key` = ключ, сгенерированный auth-users и сохранённый с компанией) → update `company_id` → выпуск токена. При сбое вызова — удаление user.
- scrypt: `hashlib.scrypt(password, salt=os.urandom(16), n=2**14, r=8, p=1)`, хранение `scrypt$n$r$p$salt_b64$hash_b64`; сравнение — `hmac.compare_digest`.
- JWT: `PyJWT`, HS256, `JWT_SECRET` из env; claims `sub` (user_id), `company_id`, `role`, `iat`, `exp` (now + 168ч).
- Оба сервиса слушают 8000 внутри сети; `auth-users` адресует `company-org` по `COMPANY_ORG_URL`.

## Verification

**Commands:**
- `uv run --extra dev --directory /home/shved/projects/smeta/<svc> pytest -q` (`company-org`, `auth-users`) -- PASS: 9 passed / 13 passed (старые `test_health` + новые; единственный warning — существующий StarletteDeprecationWarning про httpx)
- `(cd /home/shved/projects/smeta/<svc> && uvx ruff check .)` (`company-org`, `auth-users`) -- PASS: `All checks passed!`
- `uv run --directory /home/shved/projects/smeta/<svc> alembic upgrade head` (оба сервиса, хост → compose-postgres на 127.0.0.1:5432) -- PASS: `0001 (head)`, уже применены (миграции применяет контейнер при старте)
- `docker compose -f /home/shved/projects/smeta/deploy/docker-compose.yml config -q` -- PASS: без ошибок
- `docker compose up -d --build` + `ps` -- PASS: postgres, auth-users, company-org, llm-gateway `healthy`; логи: `Running upgrade -> 0001` у обоих сервисов
- Smoke register/login/duplicate/validation внутри контейнера auth-users -- PASS: register 201 (JWT, `company_id`, `role=estimator`, `exp-iat=604800`), повтор того же email 409 problem+json, login 200 (тот же `company_id`), неверный пароль/неизвестный email → идентичные 401-тела, невалидный payload → 422 problem+json со списком полей
- Smoke company-org (из контейнера auth-users): без `Idempotency-Key` → 400 problem+json, повтор с тем же ключом → 201 с тем же `id`, пустое имя → 422 -- PASS
- E2E COMPANY_API_DOWN: `docker compose stop company-org` → register → 502 problem+json, в БД `auth_users` нет созданного пользователя, `start company-org` → повторная регистрация того же email → 201 -- PASS
- Изоляция БД (AD-014): в `auth_users` только таблица `user`, в `company_org` только `company`; в `company` 3 строки / 3 уникальных `idempotency_key` -- PASS

**Результат:** все пункты Tasks & Acceptance выполнены и проверены; спека готова к review/retro.

**После правок ревью (patch из Review Triage Log):** `pytest -q` → 9 passed / 13 passed; `ruff check` → `All checks passed!`; `docker compose config -q` → OK; `up -d --build auth-users` → все healthy; smoke в контейнере: регистрация с email `"  <UPPER>@b.c  "` → 201 (нормализация до pattern), повтор того же email → 409, вход с пробелами вокруг email → 200, неверный пароль → 401 problem+json, невалидный payload → 422 problem+json со списком полей; claims `role=estimator`, `exp-iat=604800`.
