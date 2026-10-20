---
title: 'Story 5.1 — LLM-извлечение спецификации (local-first)'
type: 'feature'
created: '2026-10-20'
status: 'done'
route: 'dispatch'
baseline_commit: 'ee72c9644557e2e5359364665aee32e68bee1f83'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-5-context.md'
  - '_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** pdf-extract уже отдаёт текст сметы, но структурированной спецификации нет: mapping-service и расчёт не получают список позиций «материал/работа, ЕИ, количество, раздел».

**Approach:** Расширить llm-gateway внутренним синхронным `POST /v1/spec/extract`: он принимает текст артефакта, режет его на смысловые чанки, вызывает локальную CPU-only модель через существующий `LlamaClient`, валидирует JSON и возвращает позиции в исходном порядке. Результат не сохраняется — его потребит Story 5.2.

## Boundaries & Constraints

**Always:** только локальный CPU-only рантайм (AD-004); в этом пути нет облачного fallback и вызовов `cloud_policy` (AD-003); `X-Company-Id` обязателен (AD-013); на вход — текст из pdf-extract, не PDF; поля позиции: `name`, `kind` (`material|work`), `unit`, `quantity`, `section`; порядок позиций соответствует тексту; модель вызывает один `LlamaClient` под общим lock; ошибки — `application/problem+json`; лимиты входа и числа страниц/чанков не дают неограниченному тексту уйти в модель; ответ детерминирован по структуре, но не по значениям (LLM).

**Never:** OCR/разбор PDF, mapping/нормализация ЕИ и confidence (5.2), ручная проверка (5.3), сохранение позиций в БД, очередь/брокер, UI, публичный маршрут через api-gateway, облачная LLM, арифметика/расчёт, новый сервис `llm-spec`.

**Решения (2026-10-20):** llm-spec живёт в llm-gateway (Epic 5 context: «llm-gateway (расширение)»), endpoint внутренний — gateway/deploy не меняются; запрос `POST /v1/spec/extract` с `{"text": "..."}` и `X-Company-Id`; ответ `200 {"positions": [...], "chunk_count": N}`; `quantity` — decimal-строка; текст режется по страницам `\f` и строкам на чанки ≤ 3000 символов без разрыва строки; на чанк — один вызов с `temperature=0`, `LLM_SPEC_MAX_TOKENS` (default 2048), промпт на русском требует только JSON; ответ чистится от code fences, парсится `json.loads` и валидируется Pydantic; при невалидном JSON — один repair-вызов, затем `502 llm-invalid-output`; отсутствие модели/ошибка инференса — `503 llm-unavailable`; пустой текст — `422`, текст > `LLM_SPEC_MAX_TEXT_CHARS` (default 200000) — `413`; `kind` вне enum или отсутствие поля — ошибка чанка, без молчаливого отбрасывания; дубликаты не схлопываются. Реальная оценка покрытия ≥95% отложена: в репозитории нет GGUF-модели и PDF ГРАНД/АВК; сейчас проверяются чанкинг, схема, порядок и ошибки на fake-LLM, реальный замер — после появления модели и корпуса.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Извлечение | текст сметы, `X-Company-Id` задан | `200`, позиции в порядке текста | — |
| Многостраничный текст | страницы через `\f`, строки таблиц | чанки по страницам/строкам, позиции склеены по порядку | — |
| Модель вернула fenced JSON | ```json ... ``` | JSON извлечён, позиции валидированы | — |
| Невалидный JSON дважды | модель отвечает не-JSON | — | `502` `llm-invalid-output` |
| Модель недоступна | нет GGUF/ошибка `LlamaClient` | — | `503` `llm-unavailable` |
| Нет identity | без `X-Company-Id` | — | `401` |
| Пустой текст | `text=""` | — | `422` |
| Слишком длинный текст | > `LLM_SPEC_MAX_TEXT_CHARS` | — | `413` |
| Облачный fallback разрешён | `LLM_ALLOW_CLOUD_FALLBACK=true` | вызов всё равно только локальный | — |

</frozen-after-approval>

## Code Map

- `llm-gateway/app/spec/models.py` -- новый: Pydantic `Position` (`name`, `kind`, `unit`, `quantity: Decimal`, `section`), `ExtractionResult`, `ExtractionResponse`.
- `llm-gateway/app/spec/chunking.py` -- новый: разбиение по `\f` и строкам, бюджет 3000 символов, без разрыва строки; пустые чанки пропускаются.
- `llm-gateway/app/spec/prompt.py` -- новый: системный/пользовательский промпт на русском, схема JSON, правила «материал/работа», пропуск заголовков/итогов, запрет пояснений.
- `llm-gateway/app/spec/parser.py` -- новый: снятие code fences, поиск JSON-массива, `json.loads`, Pydantic-валидация, repair-промпт при ошибке.
- `llm-gateway/app/spec/service.py` -- новый: оркестрация чанков, последовательные вызовы `LlamaClient.chat`, склейка позиций, `asyncio.Lock`; не импортирует FastAPI.
- `llm-gateway/app/spec/errors.py` -- новый: `SpecExtractionError` (+ `InvalidModelOutput`, `ModelUnavailable`, `TextTooLong`, `EmptyText`).
- `llm-gateway/app/problem.py` -- новый: минимальный problem+json (`ProblemException`, `install_problem_handlers`, 401/413/422/502/503).
- `llm-gateway/app/config.py` -- `spec_max_text_chars` (200000), `spec_max_tokens` (2048), `spec_chunk_chars` (3000).
- `llm-gateway/app/main.py` -- `POST /v1/spec/extract`, `X-Company-Id` guard, сборка `SpecExtractionService` из существующего `_client`; проблемные обработчики.
- `llm-gateway/tests/test_spec_extraction.py` -- новый: матрица I/O на fake `LlamaClient`; чанки/порядок/fenced JSON/repair/422/413/401/502/503; cloud policy не вызывается.
- `llm-gateway/tests/test_spec_chunking.py` -- новый: границы 3000, страницы `\f`, длинные строки, пустые страницы.
- `llm-gateway/README.md` -- контракт, лимиты, env, локальный характер.
- `deploy/*`, `api-gateway/*`, `pdf-extract/*`, `mapping-service/*` -- не трогать.

## Tasks & Acceptance

**Execution:**
- [x] `llm-gateway/app/spec/{models,errors}.py` -- типы позиций и ошибок -- валидация контракта до ответа.
- [x] `llm-gateway/app/spec/chunking.py` -- чанкер -- не разрывать строки и сохранять порядок.
- [x] `llm-gateway/app/spec/prompt.py` -- промпт и JSON-схема -- модель возвращает только позиции.
- [x] `llm-gateway/app/spec/parser.py` -- парсер + repair -- невалидный вывод не проходит молча.
- [x] `llm-gateway/app/spec/service.py` -- `SpecExtractionService` -- последовательные локальные вызовы, lock, склейка.
- [x] `llm-gateway/app/problem.py`, `main.py` -- маршрут и problem+json -- 401/413/422/502/503.
- [x] `llm-gateway/app/config.py` -- лимиты/токены -- управляемые значения.
- [x] `llm-gateway/tests/test_spec_{extraction,chunking}.py` -- матрица и чанкер -- регрессия.
- [x] `llm-gateway/README.md` -- контракт и ограничения -- документация.
- [x] `llm-gateway` -- ruff/pytest -- верификация.

**Acceptance Criteria:**
- Given текст сметы с позициями, when `POST /v1/spec/extract` с `X-Company-Id`, then `200` и список позиций с `name`, `kind`, `unit`, `quantity`, `section` в порядке текста.
- Given текст на нескольких страницах, when извлечение, then страницы обработаны чанками и позиции склеены без потери порядка.
- Given модель вернула fenced JSON, when парсинг, then позиции извлечены; given невалидный JSON дважды, then `502 llm-invalid-output`.
- Given модель не загружена, when запрос, then `503 llm-unavailable`, без облачного fallback даже при `LLM_ALLOW_CLOUD_FALLBACK=true`.
- Given нет `X-Company-Id`, when запрос, then `401`; пустой текст → `422`; текст сверх лимита → `413`.

## Implementation Notes

## Spec Change Log

## Review Triage Log

- **2026-10-20 · blind-hunter + edge-case-hunter · нет лимита чанков/вызовов и общего timeout** — verdict: medium. evidence: 200 000 символов при 3000-бюджете дают ~67 чанков и до 134 вызовов под lock без дедлайна; запрос может висеть неограниченно. route: patch → `LLM_SPEC_MAX_CHUNKS`, `LLM_SPEC_TIMEOUT_SECONDS` и 504.
- **2026-10-20 · blind-hunter + edge-case-hunter · lock только в SpecExtractionService, `/inference` идёт мимо** — verdict: medium. evidence: `main.py` вызывает `_client.chat` напрямую; одновременный инференс с extract не сериализуется, хотя спека требует общий lock. route: patch → общий `asyncio.Lock` для всех путей.
- **2026-10-20 · blind-hunter + edge-case-hunter · repair отправляет весь предыдущий ответ, контекст может переполниться** — verdict: medium. evidence: repair-промпт содержит чанк + полный raw output при `n_ctx=4096`; без реальной модели переполнение не измерено, но риск явный. route: patch → усечь raw output в repair.
- **2026-10-20 · blind-hunter + edge-case-hunter · дефолты 3000/2048 и длинная строка могут превысить n_ctx** — verdict: maybe-false. evidence: точный token-count без GGUF не проверен; строка >3000 по решению спеки остаётся целой. route: defer → нужен прогон на реальной модели/токенайзере.
- **2026-10-20 · blind-hunter · `finish_reason=length` теряется, обрезка выглядит как 502** — verdict: low. evidence: `LlamaClient.chat` возвращает только текст; без реальной модели факт обрезки не установлен. route: defer.
- **2026-10-20 · blind-hunter · `except Exception` превращает программные ошибки в 503** — verdict: medium. evidence: `_chat` ловит всё, включая TypeError/AttributeError; `/inference` различает 503 и 500. route: patch → сузить до RuntimeError/OSError/ValueError.
- **2026-10-20 · blind-hunter · нет логов/метрик по чанкам и ошибкам** — verdict: low. evidence: логирования в сервисе нет; наблюдаемость не входит в AC. route: reject.
- **2026-10-20 · blind-hunter · `X-Company-Id` не валидируется как UUID и не логируется** — verdict: low. evidence: identity нужна только как guard, запись/аудит не выполняются; gateway уже присылает UUID. route: reject.
- **2026-10-20 · blind-hunter · запятая/NBSP в quantity и экспонента в ответе** — verdict: medium. evidence: `Decimal("12,5")` падает → repair/502; `str(Decimal("1E+2"))` даёт экспоненту, README обещает десятичную строку. route: patch → нормализация и `format(..., "f")`.
- **2026-10-20 · blind-hunter · парсер берёт первый JSON; случайный `[]` даёт 200 и пустые позиции** — verdict: medium. evidence: `_decode_first_json` возвращает первый декодируемый кандидат, `_positions_payload` принимает bare-list; тихая потеря данных. route: patch → принимать только объект с `positions`, `RecursionError` → ParseError.
- **2026-10-20 · blind-hunter + edge-case-hunter · снятие code fences портит тройные бэктики внутри значений** — verdict: medium. evidence: regex удаляет ` ``` ` в любом месте; имя/ЕИ с бэктиками изменится молча. route: patch → снимать только обрамляющий фенс.
- **2026-10-20 · blind-hunter + edge-case-hunter · чанкер теряет пустые строки/границы страниц и не переносит раздел** — verdict: medium. evidence: section может стать пустым/неверным, если заголовок в предыдущем чанке; качество без реальной модели не измерено. route: defer → нужен реальный прогон/эталон.
- **2026-10-20 · blind-hunter + edge-case-hunter · prompt injection через `>>>` и директивы в тексте** — verdict: medium. evidence: текст вставляется между `<<<`/`>>>` без экранирования; текст с `>>>` ломает границу. route: patch → JSON-кодирование текста и правило «данные, не инструкции».
- **2026-10-20 · blind-hunter · нет версии промпта и few-shot примеров** — verdict: low. evidence: качество извлечения без модели/корпуса не измеряется; версия — улучшение воспроизводимости. route: defer.
- **2026-10-20 · blind-hunter · OpenAPI без responses/Field-описаний** — verdict: low. evidence: контракт описан в README; потребитель 5.2 может читать OpenAPI, но это не дефект поведения. route: reject.
- **2026-10-20 · blind-hunter · пример README содержит section, которого нет во входном тексте** — verdict: low. evidence: пример противоречит правилу промпта. route: patch → добавить заголовок в пример.
- **2026-10-20 · blind-hunter · `LLM_SPEC_*` не передаются в compose** — verdict: low. evidence: дефолты работают; frozen-решение запрещает правки deploy в этой истории. route: reject.
- **2026-10-20 · blind-hunter · тело парсится до проверки 200k** — verdict: low. evidence: FastAPI парсит JSON до endpoint; лимит 200k срабатывает после; body-лимит — общая edge-задача. route: reject.
- **2026-10-20 · blind-hunter + edge-case-hunter · int-env падает на опечатке при импорте** — verdict: low. evidence: так устроены все настройки llm-gateway (`LLM_N_CTX` и др.), pre-existing. route: reject.
- **2026-10-20 · blind-hunter · unit пустой, длины не ограничены, quantity допускает отрицательные/JSON-float, extra=forbid не документирован** — verdict: medium. evidence: `unit=""` проходит контракт; ограничения длины отсутствуют; extra-поля дают 502. route: patch → min/max длины, нормализация quantity, тесты extra.
- **2026-10-20 · blind-hunter + verification-gap · тесты не проверяют prompt, unit/section, chunk-env, пробельный identity, cloud-мок** — verdict: medium. evidence: мутации prompt/response/настройки/заголовка/мока не ломают 29 тестов. route: patch → добавить ассерты и кейсы.
- **2026-10-20 · blind-hunter + edge-case-hunter + verification-gap · глобальные problem-обработчики меняют ошибки `/inference` без теста** — verdict: medium. evidence: `/inference` 503/422 теперь problem+json, регрессии не закреплены. route: patch → тесты и README-заметка.
- **2026-10-20 · edge-case-hunter · пробельный текст длиннее лимита даёт 422, а не 413** — verdict: low. evidence: пустота проверяется раньше размера; приоритет 422 для пустого текста защитим и задокументируем. route: reject.
- **2026-10-20 · edge-case-hunter · identity проверяется после валидации тела** — verdict: low. evidence: без заголовка и с битым телом FastAPI даст 422 раньше 401; для валидного тела 401 соблюдён; фикс требует middleware. route: reject.
- **2026-10-20 · edge-case-hunter · `RecursionError` из глубокого JSON не превращается в ParseError** — verdict: medium. evidence: `_decode_first_json` может бросить RecursionError, он уйдёт в 500 вместо repair/502. route: patch → ловить и маппить.
- **2026-10-20 · edge-case-hunter · 401 без `WWW-Authenticate`** — verdict: low. evidence: статус и problem+json корректны; заголовок — SHOULD, остальные сервисы его не ставят. route: reject.


## Verification

**Commands (выполнено 2026-10-20):**
- `cd llm-gateway && uv run --extra dev ruff check .` → `All checks passed!`
- `cd llm-gateway && uv run --extra dev pytest -q` → `42 passed` (чанкинг, порядок, fenced JSON, repair, лимиты чанков/текста, timeout, 401/413/422/502/503/504, отсутствие cloud-вызова, нормализация quantity, problem+json `/inference`)
- `cd deploy && docker compose config -q` → exit 0
- `cd llm-gateway && docker build -q -t llm-gateway:ci .` → образ собран

**Не покрыто автоматически:** реальная модель GGUF и корпус ГРАНД/АВК; AC ≥95% на реальных PDF не измеряется без них (см. Open Questions).
