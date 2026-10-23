---
title: 'LLM-настройки в БД и JSON-спецификация через DeepSeek'
type: 'feature'
created: '2026-10-22'
status: 'done'
route: 'dispatch'
baseline_commit: 'polyrepo: spec-service=a1c14e1232b2c76b197b48cd5b74b9060d171699 llm-proxy=b715d4013231708b7aa07068d549be0b2cffb853 deploy=622738234aa6672ba63d961fd9f2253205d81db1 api-gateway=4ec0f1d91ab62809d17c5f54964ce6fb072fc9f9 ui=ccab340e8c28a269cc9191a71179534df99a99e1 core=e16ac1ea10ae37521ea28362b341d0423aca5fe8'
review_loop_iteration: 0
context:
  - '{project-root}/docs/architecture-spine.md'
  - '{project-root}/docs/prd.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** `spec-service` жёстко берёт одну LLM из env, поэтому нельзя выбрать DeepSeek для большого PDF и нельзя хранить настройки моделей/активную модель в БД. Для документа 1,3 МБ нужен предсказуемый JSON-результат без mapping.

**Approach:** Добавить в `spec_service` каталог LLM-моделей и активную модель компании; при постановке задачи принимать необязательный alias. Профиль конкретной модели определяет chunk/output/JSON/thinking, а worker передаёт его в извлечение. Для `deepseek-flash` использовать чанк 20 000 символов, `json_object`, thinking disabled, output 8192; mapping можно отключить.

## Boundaries & Constraints

**Always:** Компания видит и активирует только свои настройки. Секреты и URL провайдеров остаются в env/YAML `llm-proxy`; в БД только `proxy_alias` и несекретные параметры. Cloud-модель требует `LLM_ALLOW_CLOUD=true` и заголовка `X-Allow-Cloud`, proxy пишет аудит. `spec_job` хранит снимок выбранной модели, чтобы retry не зависел от смены активной. Каждый чанк сохраняется в MinIO, финал — `specification.json`.

**Never:** Не хранить API-ключи в БД. Не менять контракт `file_id` как единственного ключа файла. Не добавлять OCR. Не трогать `llm-gateway` и mapping-логику за пределами флага пропуска. Не логировать текст, промпты, позиции и ключи.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Модель из запроса | `model=deepseek-flash`, cloud разрешён | job с `llm_alias=deepseek-flash` и снимком | 202 |
| Активная модель | `model` не задан | берётся активная модель компании | 202 |
| Нет настройки компании | строки `company_llm_setting` нет | fallback `LLM_MODEL_ID` или первая enabled | лог fallback |
| Cloud без согласия | `LLM_ALLOW_CLOUD=false` | запрос отклонён | 403 `cloud_requires_consent` |
| Неизвестная/disabled модель | `model=missing` | запрос отклонён | 404/422 |
| Другой tenant | чужая `company_id` | настройки/задача недоступны | 404/401 |
| Mapping выключен | `mapping_enabled=false` | `specification.json` сохранён, mapping не вызван | job succeeded |
| DeepSeek output limit | `finish_reason=length` | adaptive resplit | повторный чанк |

</frozen-after-approval>

## Code Map

- `spec-service/app/config.py` — глобальные fallback-настройки, `allow_cloud`, `mapping_enabled`, лимиты.
- `spec-service/app/domain/job.py` — `JobRecord`; добавить `llm_alias`, `mapping_enabled`.
- `spec-service/app/domain/llm.py` — новый `LlmProfile` и snapshot.
- `spec-service/app/infrastructure/models.py` — `LlmModelRow`, `CompanyLlmSettingRow`, поля `SpecJobRow`.
- `spec-service/app/infrastructure/llm_model_repository.py` — каталог, активная модель, upsert.
- `spec-service/app/infrastructure/job_repository.py` — enqueue/claim/record с моделью.
- `spec-service/app/infrastructure/llm_gateway_client.py` — model/thinking/response_format/allow_cloud на запрос.
- `spec-service/app/application/extract.py` — `prepare`/`extract_chunk` по профилю, `json_object`.
- `spec-service/app/application/process.py` — профиль и пропуск mapping.
- `spec-service/app/application/jobs.py` — резолвинг модели и активация.
- `spec-service/app/transport/{schemas,routes}.py` — `model`, `mapping_enabled`, LLM API.
- `spec-service/migrations/versions/0002_llm_settings.py` — таблицы, колонки, сид.
- `spec-service/app/application/prompt.py` — запрет ведущих пробелов/текста вне JSON.
- `deploy/llm-routes.yaml` — backend/route `deepseek-flash`.
- `deploy/docker-compose.yml`, `deploy/.env.example` — ключи/согласие/флаги.
- `ui/src/api/client.ts` — типы и вызовы моделей.
- `ui/src/views/HierarchyView.ts`, `ui/src/components/ProcessingStatus.vue` — выбор и активация.
- `api-gateway` — код не менять; `/v1/spec` уже проксируется.

## Tasks & Acceptance

**Execution:**
- [x] `spec-service/migrations/versions/0002_llm_settings.py` — создать `llm_model`, `company_llm_setting`, поля job и сид local/DeepSeek.
- [x] `spec-service/app/infrastructure/models.py` — ORM-модели и колонки.
- [x] `spec-service/app/infrastructure/llm_model_repository.py` — список, выбор, upsert, активация, tenant-фильтр.
- [x] `spec-service/app/domain/llm.py` — профиль и snapshot.
- [x] `spec-service/app/infrastructure/job_repository.py` — сохранять/читать модель и mapping-флаг.
- [x] `spec-service/app/application/jobs.py` — резолвинг request/active/fallback, проверка cloud.
- [x] `spec-service/app/application/extract.py` — бюджет и output из профиля, JSON-режим, thinking.
- [x] `spec-service/app/infrastructure/llm_gateway_client.py` — per-request model, `X-Allow-Cloud`.
- [x] `spec-service/app/application/process.py` — профиль и пропуск mapping.
- [x] `spec-service/app/transport/{schemas,routes}.py` — новые поля и `/v1/spec/llm/models`.
- [x] `spec-service/tests/*` — матрица моделей, cloud, snapshot, mapping, профиль.
- [x] `deploy/llm-routes.yaml`, `deploy/docker-compose.yml`, `deploy/.env.example` — DeepSeek и флаги.
- [x] `ui/src/api/client.ts`, `HierarchyView.ts`, `ProcessingStatus.vue` — dropdown, активация, `model` в POST.
- [x] `ui/tests/*` — выбор, активная, отправка, ошибка.
- [x] `spec-service/README.md`, `llm-proxy/README.md`, `deploy/README.md` — контракт.

**Acceptance Criteria:**
- Given активная DeepSeek у компании, when `POST /v1/spec/extract` без `model`, then job использует `deepseek-flash`.
- Given `model=deepseek-flash`, when cloud разрешён, then LLM-запрос содержит `response_format=json_object`, thinking disabled и `X-Allow-Cloud`.
- Given `mapping_enabled=false`, when чанки обработаны, then `specification.json` сохранён, mapping не вызван, job succeeded.
- Given 1,3 МБ текста, when профиль 20k, then создаётся не более 65 чанков для ~1,3 млн символов и каждый сохраняется.
- Given cloud запрещён, when выбран DeepSeek, then 403 без вызова облака.
- Given вторая компания, when читает/активирует модель, then видит только свои настройки.

## Implementation Notes

- Добавлены `llm_model` и `company_llm_setting`; активная модель компании — в `company_llm_setting`.
- `spec_job` хранит `llm_alias`, JSON-снимок `llm_config` и `mapping_enabled`.
- `ProcessSpecification`/`ExtractSpecification` принимают `LlmProfile`; профиль задаёт chunk/output/JSON/thinking.
- `HttpLlmGatewayClient` передаёт выбранную модель, `thinking` и `X-Allow-Cloud`.
- UI загружает модели, выбирает активную по умолчанию и отправляет `model` в `POST /v1/spec/extract`.
- DeepSeek route и env добавлены в `deploy`; mapping для задачи отключается `LLM_SPEC_MAPPING_ENABLED=false`.
- Проверки: spec-service 95 passed, llm-proxy 14, api-gateway 63, UI 115 + typecheck/build; alembic upgrade head проверен на SQLite.

## Spec Change Log

## Review Triage Log

| Finding | Verdict | Evidence / route |
|---|---|---|
| Роль по умолчанию `estimator` авторизует запрос без `X-Role` | high | Исправлено: `role in ALLOWED_*` без fallback; тесты на `viewer` и отсутствие роли. |
| `PUT /llm/models` позволял тенанту менять глобальный каталог | high | Исправлено: каталог только `admin/owner`; README обновлён. |
| Активация cloud-модели без `LLM_ALLOW_CLOUD` | high | Исправлено: `CloudConsentRequired` в `activate`; тест. |
| Неявный fallback выбирал cloud-модель из-за сортировки alias | high | Исправлено: сначала non-cloud; тест `test_implicit_fallback_prefers_non_cloud_model`. |
| Невалидные параметры модели могли отравить каталог | medium | Исправлено: `LlmProfile` проверяет context/chunk, `update_model` валидирует merged snapshot; тест. |
| `PUT` возвращал `active=false` для активной модели | low | Исправлено: маршрут вычисляет active через `list_models`; тест ответа. |
| Невалидный `X-User-Id` молча терял аудит | medium | Исправлено: 400 `X-User-Id must be a UUID`. |
| Worker/снимок профиля не проверялись | medium | Добавлен `tests/test_worker.py` и round-trip `snapshot/from_snapshot`. |
| Role-forbidden путь не проверялся | medium | Добавлены тесты `LlmSettingsForbidden` для `activate`/`update_model`. |
| Модель/mapping из тела запроса не проверялись на границе HTTP | medium | Добавлен `test_endpoint_passes_model_and_mapping_flag`. |
| `LLM_SPEC_MAPPING_ENABLED` default не проверялся | medium | Добавлен `test_mapping_default_comes_from_settings`. |
| Профиль не проверялся на repair/resplit | medium | Добавлен тест с DeepSeek-профилем на всех вызовах. |
| Миграция 0002 не проверялась | medium | Добавлен `tests/test_migrations.py` с существующей `spec_job` и сидом. |
| Retry не проверял обновление снимка | medium | Добавлен `test_retry_updates_model_snapshot`. |
| Бюджет чанка профиля не проверялся | medium | Добавлен `test_profile_chunk_budget_overrides_settings`. |
| Дублирование `Settings.resolved_chunk_chars` | low | Удалено; бюджет считается только в `LlmProfile`. |
| Остаточный `OPENAI_API_KEY` при отсутствии openai route | low | Удалён из compose/.env.example. |
| UI не проверял ошибку активации | low | Добавлен тест 403 с пользовательским текстом. |
| UI “По умолчанию” фактически всегда отправляет alias | low | Отклонено: явный alias фиксирует выбор на момент запроса; вреда нет. |
| UI не показывает `model`/`mapping_enabled` из job | low | Отклонено: модель видна в dropdown; mapping управляется env. |
| `enqueue` при running игнорирует новый model | low | Отклонено: идемпотентная семантика существующей задачи; ответ содержит фактическую модель. |
| Отключение последней enabled-модели | low | Отклонено: каталогом управляет admin/owner, ошибка явная при следующем resolve. |
| `proxy_alias`/cloud не сверяются с YAML llm-proxy | medium | Defer: требует runtime-валидации через `/v1/models`; сейчас seed и YAML согласованы, ошибка явная на вызове. |
| Per-company consent вместо глобального `LLM_ALLOW_CLOUD` | medium | Defer: текущая модель согласия — env владельца + аудит; per-company consent требует отдельного решения. |
| K8s-манифесты не обновлены | low | Defer: `deploy/k8s` остаётся пустым каркасом, MVP использует compose. |
| `mapping={}` не отличает skip от пустого mapping | low | Отклонено: `SpecProcessResponse` внутренний, job содержит `mapping_enabled=false`. |
| Оценка «≤65 чанков» для 1,3 млн символов | low | Отклонено: README даёт диапазон; фактическое число зависит от страниц. |

## Design Notes

`llm_model` — глобальный каталог; `company_llm_setting.active_alias` — выбор компании. `spec_job.llm_config` — JSON-снимок профиля. `proxy_alias` отделяет внутренний alias от маршрута proxy. `response_format` и `thinking` передаются из профиля, поэтому proxy остаётся pass-through.

## Verification

**Commands:**
- `cd spec-service && uv run --extra dev pytest -q && uv run --extra dev ruff check .` — тесты/линт зелёные.
- `cd ui && npm test && npm run typecheck && npm run build` — тесты/типы/сборка зелёные.
- `cd llm-proxy && uv run --extra dev pytest -q` — конфиг DeepSeek/аудит зелёные.
- `cd api-gateway && uv run --extra dev pytest -q` — маршрут `/v1/spec` не сломан.

**Manual checks:**
- `POST /v1/spec/extract` с `model=deepseek-flash`, `mapping_enabled=false`; в MinIO есть `chunks/*.json` и `specification.json`, нет `mapping.json`; в аудите proxy есть `allowed`, нет текста.
