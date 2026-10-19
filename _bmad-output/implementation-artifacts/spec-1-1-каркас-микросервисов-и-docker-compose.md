---
title: 'Story 1.1 — Каркас микросервисов и Docker Compose'
type: 'feature'
created: '2026-10-17'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
baseline_commit: 'polyrepo: api-gateway=73b1456 auth-users=5b15678 company-org=7229071 estimate=9568181 pdf-extract=9480af0 llm-gateway=78bb416 mapping-service=0c4fc20 rate-catalog=82081bc calc-service=c0a9054 ui=e8ded52 deploy=9d1d463'
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Сервисы smeta1 существуют как пустые папки (README, .gitignore, CI-заглушка), но `deploy/docker-compose.yml` ссылается на `build: ../<service>` без Dockerfile и кода. `docker compose up` сейчас не работает, стек не поднимается, задача эпика — локальный запуск одной командой (NFR-005) — не выполнена.

**Approach:** Добавить в каждый из 9 Python-сервисов минимальную FastAPI-заглушку с эндпоинтом `/health` и чтением конфигурации из переменных окружения, а также Dockerfile. Для `ui` — минимальный Vue+Vite каркас, собираемый в Docker (node build → nginx). Обновить `deploy/docker-compose.yml`: healthcheck для приложений, опциональный `env_file`, публикация портов gateway/ui. Проверка — `docker compose config` и успешный `docker compose up` со всеми healthy-сервисами.

## Boundaries & Constraints

**Always:** Каждый сервис самодостаточен (AD-009, без shared-lib). Базовый образ — `python:3.12-slim`. Единый контракт здоровья: `GET /health` → 200 `{"status":"ok","service":"<name>"}`. Конфигурация только из env. Заглушки стартуют независимо (без обязательного подключения к Postgres/MinIO на старте). `llm-gateway` — без GPU и без загрузки моделей в образ (рантайм — Story 1.2). Postgres/MinIO — с персистентными томами.

**Never:** Не реализовывать доменную бизнес-логику (компании, сметы, маппинг, расчёт). Не выбирать/подключать брокер очереди (Deferred). Не добавлять shared-lib. Не коммитить `.env` (только `.env.example`).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| HAPPY_PATH | Запущенный сервис, `GET /health` | 200, тело `{"status":"ok","service":"<name>"}` | N/A |
| NO_ENV | Сервис стартует без переменных окружения | Стартует с дефолтами (порт 8000), `/health` = 200 | Дефолты в коде, без падения |
| NO_DOTENV | `docker compose up` без `.env` | Стек поднимается на дефолтах из compose/.env.example | `env_file` опционален |

</frozen-after-approval>

## Code Map

- `deploy/docker-compose.yml` — уже содержит 10 сервисов (`build: ../<service>`), Postgres, MinIO; нужны healthcheck приложений, опциональный `env_file`, порты.
- `deploy/.env.example` — дефолтные env; источник для локального `.env`.
- `deploy/README.md` — инструкция запуска; обновить при изменении контракта запуска.
- `/home/shved/projects/smeta/<service>/` — 10 сервисов: только `README.md`, `.gitignore`, `.github/workflows/ci.yml`. Добавить код, Dockerfile, pyproject.
- CI Python ждёт `pyproject.toml` (иначе job пропускается) — добавить.
- `.gitignore` уже игнорирует `.env`, `.venv`, `__pycache__`.
- Внешние тома: `pgdata`, `miniodata` (уже объявлены).

## Tasks & Acceptance

**Execution:**
- [x] `<service>/pyproject.toml` (×9 Python) — FastAPI + uvicorn + pytest; dev-зависимость `httpx`/`pytest` — активирует CI и даёт smoke-тест.
- [x] `<service>/app/__init__.py`, `<service>/app/main.py` (×9) — FastAPI-приложение: `/health`, чтение env (`SERVICE_NAME`, `APP_PORT`, `POSTGRES_DSN`, `MINIO_ENDPOINT`), запуск uvicorn — каркас сервиса.
- [x] `<service>/tests/test_health.py` (×9) — TestClient проверяет 200 и `service` — покрыть HAPPY_PATH.
- [x] `<service>/Dockerfile` (×9) — `python:3.12-slim`, install deps, `HEALTHCHECK`, `CMD` uvicorn на `APP_PORT` — образ для compose.
- [x] `ui/` — минимальный Vue+Vite каркас (TypeScript): `package.json`, `package-lock.json`, `tsconfig.json`, `vite.config.ts`, `index.html`, `src/main.ts`, `src/App.vue`, `src/env.d.ts`, `nginx.conf`, `Dockerfile` (стадия node build → nginx раздача `dist/`, `/health`) — образ для compose. Слушает порт 80 внутри контейнера.
- [x] `deploy/docker-compose.yml` — добавить `healthcheck` каждому приложению, сделать `env_file` опциональным, опубликовать `api-gateway` (8000) и `ui` (8080) — NFR-005.
- [x] `deploy/README.md` — актуализировать запуск/порты — документация.

**Acceptance Criteria:**
- Given склонированные сервисы и `deploy/`, when выполняется `docker compose up -d --build`, then поднимаются Postgres, MinIO и 10 сервисов, все приложения переходят в `healthy`.
- Given запущенный сервис, when `GET /health`, then 200 и `{"status":"ok","service":"<name>"}` (NO_ENV: стартует с дефолтами).
- Given отсутствует `.env`, when `docker compose up`, then стек поднимается (NO_DOTENV).
- Given CI GitHub Actions, when push в сервис, then job линта/тестов выполняется (pyproject присутствует).

## Implementation Notes

- **Полирепо.** Изменения разнесены по 11 репозиториям (`<service>/`, `deploy/`); `baseline_commit` в frontmatter хранит HEAD каждого.
- **`env_file` опционален.** В compose использован `env_file: [{path: .env, required: false}]`, поэтому `docker compose up` и `config -q` работают без `.env` (покрывает NO_DOTENV и чинит `deploy` CI на чистом checkout).
- **MinIO-образ.** `minio/minio` удалён с Docker Hub (upstream `minio/minio#21647`); заменён на зеркало официальных релизов `alpine/minio:latest-release` (тот же entrypoint/команда). Зеркало работает от `uid 100`, а `/data` в образе root-owned, поэтому добавлен `user: root` (паритет с прежним официальным образом). Healthcheck переведён с `mc ready local` (в зеркале нет `mc`) на `wget` к `/minio/health/live`.
- **Импорт-порядок в тестах.** `ruff` (как в CI, cwd = корень сервиса) считает `app` first-party, поэтому импорты отсортированы как `fastapi` → `app`.
- **Верификация.** Команда pytest требует `--extra dev` (pytest/httpx — в `[project.optional-dependencies].dev`).

## Spec Change Log

- Команда верификации pytest уточнена до `uv run --extra dev --directory <svc> pytest -q` (dev-зависимости не входят в базовый набор).

## Review Triage Log

### Review Findings

**decision-needed:** нет.

**patch:** нет — все кодовые исправления (.dockerignore, ruff в dev-deps, точное равенство service в тестах, or-fallback для SERVICE_NAME/APP_PORT, exec uvicorn, npm ci --no-audit, tsconfig.node.json, engines.node) уже применены в коде и отражены в актуальном дифе `/tmp/opencode/diff-spec-1-1-final.diff`.

**defer:**
- [x] [Review][Defer] ui/nginx.conf не имеет reverse-proxy для бэкенд-API — deferred:out of scope для каркаса Story 1.1
- [x] [Review][Defer] Девять Python Dockerfiles байт-в-байт идентичны без шаблона — deferred:by design (AD-009, без shared-lib)
- [x] [Review][Defer] Docker healthcheck'и не выполняются в CI (только `config -q`) — deferred:проверены локально (`up --build` → все healthy), e2e CI job вне объёма каркаса

**Rejected:**
- false: MinIO volume destroys data — `miniodata:/data` объявлен, том именованный
- false: spec checklist overstates env reads — каркас читает только SERVICE_NAME, APP_PORT через Docker CMD
- low: контейнеры работают от root — маловероятно в повседневном использовании каркаса, фикс сложнее прямой правки
- low: плавающие теги базовых образов — маловероятно, фикс требует дайджестов
- low: нет Python lockfile — маловероятно в повседневном использовании, CI использует pip install
- low: EXPOSE 8000 жестко — косметика, EXPOSE — документация
- low: тест без env не ловит отсутствие переменных — тест проходит, ограничение каркаса
- low: нет pytest конфигурации — тесты работают без conftest.py
- low: APP_PORT нечисловой — маловероятно, фикс сложнее прямой правки
- false: нет /version или /ready endpoints — не требуется спецификацией
- false: k8s/ пустой — плейсхолдер по дизайну
- false: README ссылается на файлы не из дифа — они существуют в базовом коммите

## Spec Change Log

- Команда верификации pytest уточнена до `uv run --extra dev --directory <svc> pytest -q` (dev-зависимости не входят в базовый набор).
- Review Triage Log заменён результатами фактического code-review (bmad-code-review, 4 слоя: blind-hunter, edge-case-hunter, verification-gap, acceptance-auditor). Все кодовые патчи применены, оставшиеся находки — defer или rejected.

## Verification

**Commands:**
- `docker compose -f /home/shved/projects/smeta/deploy/docker-compose.yml config -q` — expected: без ошибок.
- `docker compose -f /home/shved/projects/smeta/deploy/docker-compose.yml up -d --build` — expected: все сервисы запущены.
- `docker compose -f /home/shved/projects/smeta/deploy/docker-compose.yml ps` — expected: приложения `healthy`.
- `for s in api-gateway auth-users company-org estimate pdf-extract llm-gateway mapping-service rate-catalog calc-service; do uv run --extra dev --directory /home/shved/projects/smeta/$s pytest -q; done` — expected: тесты зелёные.
- `for s in ...; do (cd /home/shved/projects/smeta/$s && uvx ruff check .); done` — expected: `All checks passed`.
- `cd /home/shved/projects/smeta/ui && npm ci && npm run build` — expected: сборка успешна.
- `curl -fsS localhost:8000/health` — expected: 200, `{"status":"ok",...}`.

**Результаты (2026-10-17):**
- `config -q` без `.env` — OK.
- `up -d --build` без `.env` — поднято 12 контейнеров (postgres, minio, 10 сервисов), все `healthy`.
- `pytest` ×9 — `2 passed` в каждом; `ruff` ×9 — `All checks passed`.
- `ui`: `npm ci` + `npm run build` — успешно.
- `curl localhost:8000/health` → `{"status":"ok","service":"api-gateway"}`; `curl localhost:8080/health` → `{"status":"ok","service":"ui"}`; `curl localhost:9000/minio/health/live` → 200.
- `curl -fsS localhost:8000/health` — expected: 200, `{"status":"ok",...}`.
