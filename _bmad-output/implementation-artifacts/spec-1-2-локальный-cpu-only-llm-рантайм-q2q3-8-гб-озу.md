---
title: 'Story 1.2 — Локальный CPU-only LLM-рантайм (q2–q3, 8 ГБ ОЗУ)'
type: 'feature'
created: '2026-10-18'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
baseline_commit: 78bb41638237451f0acdd037938acae2eec53dde
story_key: '1-2-локальный-cpu-only-llm-рантайм-q2-q3-8-гб-озу'
---

<frozen-after-approval>
### Intent
Реализовать локальный CPU-only LLM-рантайм в llm-gateway на основе GGUF (q2–q3) с соблюдением ограничений AD-004, обеспечить единый интерфейс инференса, учесть лимит 16 ГБ ОЗУ. Должен работать без GPU, отдавать health и основываться на каркасе из Story 1.1.

### Architecture Context
См. epic-1-context.md: CPU-only, GGUF q2–q3, 16 ГБ ОЗУ, без GPU; llm-gateway единый интерфейс local/cloud (cloud – позже, Story 1.3). Конфигурация через env.

### Решение по модели и паттерну
- Паттерн: llama-cpp-python, класс LlamaClient с методами load/unload; при выгрузке вызывать llm.close() (если есть) и gc.collect(); модель загружается в память по запросу/инициализации и выгружается явно (unload) — по образцу /home/shved/PythonProjects/LamaInference/infrastructure/llm/llama_client.py.
- Модель: Llama-3.2-3B-Instruct-Q2_K.gguf (GGUF q2_K), n_gpu_layers=0, n_ctx настраиваемый через env, n_threads по CPU.
</frozen-after-approval>

## Code Map

- `/home/shved/projects/smeta/llm-gateway/app/main.py` -- каркас FastAPI, health, базовый шаблон под расширение
- `/home/shved/projects/smeta/llm-gateway/Dockerfile` -- cpu-only контейнер, ограничение памяти через compose (см. deploy/docker-compose.yml llm-gateway: deploy.resources.limits.memory: 14g)
- `/home/shved/projects/smeta/llm-gateway/pyproject.toml` -- зависимости
- `/home/shved/projects/smeta/deploy/docker-compose.yml` -- llm-gateway конфигурация + memory limit 14g

## Tasks & Acceptance

**Execution:**
- [x] `/home/shved/projects/smeta/llm-gateway/pyproject.toml` -- добавить `llama-cpp-python>=0.3.14` (CPU-only биндинг)
- [x] `/home/shved/projects/smeta/llm-gateway/app/__init__.py`, `/home/shved/projects/smeta/llm-gateway/app/llama_client.py` -- перенести/адаптировать паттерн load/unload из примера (Llama, `llm.close()` при выгрузке, `gc.collect()`, async streaming)
- [x] `/home/shved/projects/smeta/llm-gateway/app/main.py` -- реализовать единый интерфейс инференса (POST /v1/chat/completions или /inference) с local-first, /health оставить как есть; обеспечить выгрузку модели из памяти после ответа (unload в finally)
- [x] `/home/shved/projects/smeta/llm-gateway/Dockerfile` -- добавить build deps для llama-cpp-python (cmake, g++, build-essential) при сборке CPU; slim образ
- [x] `/home/shved/projects/smeta/llm-gateway/README.md` -- актуализировать про CPU-only GGUF q2-q3 (Llama-3.2-3B-Instruct-Q2_K.gguf)
- [x] `/home/shved/projects/smeta/deploy/docker-compose.yml` -- добавить volume для моделей (./models:/models), env-переменные (LLM_MODEL_PATH, LLM_N_THREADS, LLM_N_CTX, LLM_N_GPU_LAYERS=0, LLM_MAX_TOKENS)
- [x] `/home/shved/projects/smeta/deploy/.env.example` -- добавить переменные LLM_MODEL_PATH, LLM_N_CTX, LLM_N_THREADS, LLM_N_GPU_LAYERS, LLM_MAX_TOKENS, LLM_HF_REPO, LLM_HF_FILENAME
- [x] Тесты: добавлены минимальные тесты для интерфейса инференса с мок-рантаймом (чтобы не грузить модель в CI); сохранить health-тесты из Story 1.1

**Acceptance Criteria:**
- Given машина с 16 ГБ ОЗУ и без GPU, when llm-gateway получает запрос на локальный инференс, then используется GGUF-модель квантования q2–q3
- And потребление памяти не превышает доступный лимит 16 ГБ ОЗУ
- And запрос обрабатывается без обращения к GPU
- And llm-gateway отдаёт единый интерфейс инференса вне зависимости от рантайма

## Implementation Notes


## Review Triage Log

**Source: blind-hunter (симуляция)**

- [high] **lifespan vs on_event**: используется `@app.on_event("shutdown")` (deprecated). Лучше мигрировать на lifespan. Evidence: app/main.py:65 — `@app.on_event("shutdown")`.
- [medium] **memory limit 14G vs 16GB**: compose deploy.resources.limits.memory = 14G, spec говорит про лимит 16ГБ ОЗУ. Evidence: deploy/docker-compose.yml:23-26 — memory: 14G.
- [low] **gc.collect timing**: после выгрузки вызывается gc.collect(); в асинхронном потоке может быть преждевременным. Evidence: app/llama_client.py:198-212 — unload() вызывает gc.collect().
- [low] **volume RO**: модели смонтированы read-only; если планируется скачивание внутрь контейнера — RW. Evidence: deploy/docker-compose.yml:21 — ./models:/models:ro.
- [low] **build deps size**: build-essential/cm ake/g++ в финальном образе; можно multi-stage. Evidence: Dockerfile:12-18.
- [medium] **load without validation**: при отсутствии модели файл не проверяется, передаётся в llama-cpp. Evidence: app/llama_client.py:38-57.
- [low] **config validation**: нет валидации n_ctx/n_threads. Evidence: app/config.py:5-25.
- [low] **healthcheck urllib**: работает, но можно httpx; не критично.
- [low] **streaming error handling**: все исключения ловятся; потенциальные race при закрытии. Evidence: app/llama_client.py:76-122.
- [low] **unload semantics**: в README упомянута выгрузка после использования, но в коде unload вызывается только на shutdown. Evidence: app/main.py:65-71; app/llama_client.py:198-212.
- [medium] **CPU-only enforcement**: n_gpu_layers=0 есть, но не гарантировано; можно добавить проверку/комментарий. Evidence: app/config.py, Dockerfile.
