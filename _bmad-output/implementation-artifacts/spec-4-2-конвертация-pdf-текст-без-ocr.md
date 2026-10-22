---
title: 'Story 4.2 — Конвертация PDF → текст без OCR'
type: 'feature'
created: '2026-10-20'
status: 'done'
route: 'dispatch'
baseline_commit: 'ebb448af8ab61396d80be990d9d129f0a4da7395'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-4-context.md'
  - '_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** После Story 4.1 pdf-extract принимает и хранит цифровой PDF, но не отдаёт текст: следующему шагу конвейера (LLM-извлечение спецификации) нечего читать.

**Approach:** Добавить идемпотентный шаг извлечения текста из уже загруженного PDF: `POST /v1/files/{file_id}/extract` читает объект из MinIO, извлекает layout-текст через pdfplumber, сохраняет текст в S3 рядом с PDF (`{company_id}/{file_id}/source.txt`), а метаданные — в `file_text` БД pdf-extract; `GET /v1/files/{file_id}/text` собирает артефакт из метаданных и объекта. Контракт артефакта транспортно-нейтрален: он доступен по HTTP, без очереди.

## Boundaries & Constraints

**Always:** только цифровые PDF с текстовым слоем (AD-005), OCR нет; `company_id` обязателен, файл и артефакт скоупятся компанией (AD-013); текст хранится в MinIO в папке файла (`source.txt`), метаданные — в БД pdf-extract; в чужой БД не ходим (AD-014); текст сохраняет структуру строк/таблиц; ошибки — `application/problem+json`; извлечение идемпотентно (повторный POST перезаписывает артефакт); `POST`/`GET` возвращают `200` с `file_id`, `content`, `page_count`, `char_count`, `coverage`, `extracted_at`; `storage_key` наружу не отдаётся.

**Never:** OCR, привязка файла к смете/этапу, автоматическое извлечение при загрузке, очередь/брокер, UI, удаление артефактов, изменение контракта `POST /v1/files`, правки gateway/deploy (префикс `/v1/files` уже есть); текст не хранится в MinIO.

**Решения (2026-10-20):** отдельные `POST /v1/files/{file_id}/extract` (200) и `GET /v1/files/{file_id}/text` (200); чужая/несуществующая запись и отсутствующий артефакт → одинаковый 404; пропавший объект/сбой MinIO → 502; битый PDF → 422 `invalid-pdf`; `coverage = min(1, непробельные символы pdfplumber / text_length pypdfium2)` — диагностическая метрика, не повод для отказа; `pdfplumber.extract_text(layout=True)`, строки с обрезанными хвостами, страницы через `\f`; `char_count` — непробельные символы; артефакт — текст в S3 `{company_id}/{file_id}/source.txt` и метаданные `file_text(file_id PK/FK, company_id, page_count, char_count, coverage, extracted_at)`, upsert метаданных по `file_id`. Покрытие ≥95% подтверждается прокси-метрикой на сгенерированных фикстурах; реальный корпус ГРАНД/АВК в репозитории отсутствует и остаётся непокрытым.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Извлечение | загруженный PDF, `X-Company-Id` задан | `200`, `source.txt` в S3, метаданные в БД, `content` со строками/колонками | — |
| Многостраничный PDF | текст и таблица на нескольких страницах | страницы разделены `\f`, строки и колонки сохранены | — |
| Повторное извлечение | артефакт уже есть | `200`, `source.txt` перезаписан, одна строка метаданных обновлена | — |
| Чтение до извлечения | артефакта нет | — | `404` problem+json |
| Чужая компания | `file_id` другой компании | — | `404`, неотличимо от отсутствия |
| Нет identity | без `X-Company-Id` | — | `401` |
| Объект пропал из MinIO | запись `file` есть, объекта нет | — | `502` problem+json |
| Битый PDF в хранилище | объект не парсится | — | `422` `invalid-pdf` |
| Низкое покрытие | извлечено < 95% символов | артефакт создаётся, `coverage` < 0.95 виден в ответе | — |

</frozen-after-approval>

- **CID→Unicode:** если pdfminer отдаёт `(cid:N)` (нет `ToUnicode`), маппинг строится по встроенному шрифту: для Type0 — `CIDToGIDMap` + обратный `cmap`/glyph names, для simple font — `/Differences` + AGL; остаток закрывает внешняя JSON-карта по SHA-256 шрифта или имени. Замена выполняется в `page.chars` до `chars_to_textmap`, layout не меняется; неизвестные CID остаются как есть и попадают в warning.

## Code Map

- `pdf-extract/app/domain/text.py` -- `ExtractedText`, `TextArtifact` (с контентом для API), `TextArtifactMetadata` (без контента), `text_storage_key` и `TextArtifact.create`.
- `pdf-extract/app/domain/errors.py` -- добавить `FileNotFound`, `TextArtifactNotFound` (404), `StorageUnavailable` (502).
- `pdf-extract/app/domain/ports.py` -- `FileStorage.put/get` для PDF и текста; `TextArtifactRepository.upsert_metadata/get_metadata`; `PdfTextExtractor.extract`.
- `pdf-extract/app/application/extract_text.py` -- новый: `ExtractText` (файл → MinIO → pdfplumber → coverage → upsert) и `GetTextArtifact` (чтение с 404).
- `pdf-extract/app/infrastructure/models.py` -- `FileTextRow` (`file_text`: file_id PK/FK cascade, company_id индекс, page_count, char_count, coverage Float, extracted_at); текст — в S3.
- `pdf-extract/app/infrastructure/repository.py` -- `SqlAlchemyFileRepository.get` со скоупом компании; `SqlAlchemyTextArtifactRepository` с upsert и get.
- `pdf-extract/app/infrastructure/pdf.py` -- `PdfplumberTextExtractor`: `pdfplumber.open(BytesIO(...))`, `extract_text(layout=True)`, `rstrip` строк, `\f`; pdfminer/pdfplumber-исключения → `InvalidPdf`.
- `pdf-extract/app/infrastructure/storage.py` -- `MinioFileStorage.get`: `get_object`, `read`, `close/release_conn`; любые ошибки чтения → `StorageUnavailable`.
- `pdf-extract/app/transport/schemas.py` -- `TextArtifactResponse`.
- `pdf-extract/app/transport/routes.py` -- `create_router(store_pdf, extract_text, get_text)`; `POST /v1/files/{file_id}/extract`, `GET /v1/files/{file_id}/text`; identity до use case.
- `pdf-extract/app/transport/problem.py` -- 404 для `FileNotFound`/`TextArtifactNotFound`, 502 для `StorageUnavailable`.
- `pdf-extract/app/wiring.py` -- `make_extract_text`/`make_get_text`: одна сессия и commit/rollback на вызов.
- `pdf-extract/migrations/versions/0002_create_file_text.py` -- новая миграция `file_text` с FK на `file`.
- `pdf-extract/pyproject.toml` + `uv.lock` -- runtime-зависимость `pdfplumber>=0.11`.
- `pdf-extract/tests/test_extract.py` -- новый: матрица I/O, структура таблиц, идемпотентность, 404/401/502/422, coverage, production composition.
- `pdf-extract/tests/test_storage.py` -- тесты `get`: успех, ошибка → `StorageUnavailable`.
- `pdf-extract/tests/test_migrations.py` -- `file_text`, FK, индексы, модель против схемы.
- `pdf-extract/tests/test_upload.py` -- `FakeStorage.get`, чтобы сквозной сценарий upload → extract работал.
- `api-gateway/tests/test_gateway.py` -- проксирование вложенного `/v1/files/{id}/extract` и `/text`; код gateway не меняется.
- `pdf-extract/README.md` -- описать извлечение, артефакт и coverage.
- `deploy/*`, `rate-catalog/*`, `estimate/*`, `calc-service/*` -- не трогать.

## Tasks & Acceptance

**Execution:**
- [x] `pdf-extract/app/domain/{text,errors,ports}.py` -- типы, ошибки и порты артефакта -- application не знает про FastAPI/SQLAlchemy/pdfplumber.
- [x] `pdf-extract/app/application/extract_text.py` -- `ExtractText`/`GetTextArtifact` -- порядок чтения, coverage, upsert и 404.
- [x] `pdf-extract/app/infrastructure/{models,repository}.py` -- метаданные `file_text` без контента -- изоляция компании и идемпотентный upsert.
- [x] `pdf-extract/app/infrastructure/{pdf,storage}.py` -- pdfplumber-экстрактор и MinIO.get -- реальный текст и 502 при сбое чтения.
- [x] `pdf-extract/migrations/versions/0002_create_file_text.py` -- миграция -- схема совпадает с моделью, FK на `file`.
- [x] `pdf-extract/app/transport/{schemas,routes,problem}.py`, `wiring.py` -- маршруты extract/text -- 200/401/404/422/502 и problem+json.
- [x] `pdf-extract/pyproject.toml`, `uv.lock` -- pdfplumber -- runtime-зависимость и lock.
- [x] `pdf-extract/tests/{test_extract,test_storage,test_migrations,test_upload}.py` -- матрица и регрессия -- все сценарии и coverage.
- [x] `api-gateway/tests/test_gateway.py` -- вложенный путь -- маршрут доступен только через edge.
- [x] `pdf-extract/README.md` -- контракт и примеры -- документация шага.
- [x] `pdf-extract` + `api-gateway` -- ruff/pytest; `deploy` -- compose config -- верификация.

**Acceptance Criteria:**
- Given загруженный цифровой PDF, when `POST /v1/files/{id}/extract`, then `200`, артефакт сохранён, `content` содержит строки и колонки таблиц, `page_count`/`char_count` заполнены.
- Given сохранённый артефакт, when `GET /v1/files/{id}/text`, then тот же текст и метаданные, доступные только своей компании.
- Given повторный `POST /extract`, then артефакт обновляется, дубля нет.
- Given чужая компания или неизвестный `file_id`, when extract/get, then `404` problem+json без раскрытия существования.
- Given объект пропал или повреждён, when extract, then `502` или `422` problem+json соответственно.
- Given набор сгенерированных PDF, when измеряется coverage, then прокси-покрытие ≥ 0.95; реальный корпус ГРАНД/АВК не покрыт (зафиксировано в Verification).

## Implementation Notes

- **Слои (AD-011):** `domain/text.py` содержит `ExtractedText`, `TextArtifact.create` и подсчёт непробельных символов; `application/extract_text.py` знает только порты; pdfplumber/pypdfium2 изолированы в `infrastructure/pdf.py`, MinIO — в `infrastructure/storage.py`, SQLAlchemy — в `infrastructure/{models,repository}.py`; FastAPI — только в `transport`.
- **Порядок extract:** проверка файла в репозитории с фильтром `company_id` (иначе 404) → чтение объекта из MinIO (сбой → 502) → `pdfplumber.extract_text(layout=True)` (битый/зашифрованный PDF → 422 `invalid-pdf`) → `coverage = min(1, pdfplumber_chars / pypdfium2_chars)` → `put source.txt` в S3 → upsert метаданных `file_text` по `file_id`.
- **Текст:** строки обрезаны справа, страницы соединены `\f`; `char_count` и числитель coverage считают только непробельные символы, поэтому layout-отступы не искажают метрику.
- **Идемпотентность и tenancy:** `file_text` имеет PK/FK `file_id` с `ON DELETE CASCADE`, `company_id` с индексом; `upsert_metadata` обновляет метаданные, `put source.txt` перезаписывает объект; `get_metadata` фильтрует `file_id` и `company_id`, поэтому чужая компания и отсутствие артефакта дают одинаковый 404. Чтение текста: метаданные из БД → объект `source.txt` из S3; нет объекта → 502.
- **Контракт:** `POST /v1/files/{file_id}/extract` и `GET /v1/files/{file_id}/text` → `200` с `{file_id, content, page_count, char_count, coverage, extracted_at}`; `X-Company-Id` проверяется до use case (401); `storage_key` наружу не отдаётся.
- **UTC:** SQLite теряет `tzinfo`, поэтому репозиторий нормализует `extracted_at`/`created_at` к UTC, сохраняя контракт ISO-8601.
- **Тесты:** sqlite + реальные Alembic-миграции; настоящие pdfplumber/pypdfium2; reportlab-фикстуры (ASCII); fake-storage с `get`; покрыты структура таблиц/страниц, идемпотентность, coverage < 0.95, 401/404/422/502 и production composition. Gateway-тесты подтверждают проксирование вложенных путей без изменения кода gateway.

## Spec Change Log

- 2026-10-21: слой 2.5 усилен: основным сигналом стало число букв в словах из словаря/частотного списка (wordfreq), добавлены identity-маппинг печатного ASCII (цифры/коды), контекстная пунктуация по соседям и лимит 1 млн символов; на реальном PDF (167 страниц) число CID снижено с 10 907 до 101.

- 2026-10-21: добавлен слой 2.5 — автоинференс offset по языковой модели документа (`PDF_EXTRACT_CID_AUTO`, по умолчанию включён): кандидаты ограничены алфавитом корректного текста, принимаются только при заметном отрыве по score; при малом объёме текста пропускается.

- 2026-10-21: добавлено восстановление CID→Unicode: `ToUnicode` → встроенный шрифт (`CIDToGIDMap` + `cmap`/glyph names) → внешняя карта по fingerprint (`PDF_EXTRACT_CID_MAP_PATH`) → fallback на страницу из pypdfium2 (`PDF_EXTRACT_PDFIUM_FALLBACK`, по умолчанию выключен) с проверкой правдоподобия и алфавита; неразрешённые CID остаются `(cid:N)` и логируются.

- 2026-10-21: контент перенесён из `file_text.content` в S3 (`{company_id}/{file_id}/source.txt`), в БД остались метаданные; миграция `0003` удаляет колонку, entrypoint запускает `app.tools.migrate_storage_layout`.

## Review Triage Log

- **2026-10-20 · blind-hunter · coverage не гейтит извлечение, реального корпуса нет** — verdict: false. evidence: спека прямо фиксирует coverage как диагностическую метрику и отсутствие корпуса ГРАНД/АВК; поведение соответствует решению, вред не наступает. route: reject.
- **2026-10-20 · blind-hunter · фикстуры только ASCII, кириллица/CID-шрифты не покрыты** — verdict: low. evidence: pdfplumber возвращает Unicode; тест с кириллицей требует встроенного шрифта и не проверяет контракт. route: reject.
- **2026-10-20 · blind-hunter · PDF без текстового слоя в хранилище даёт 200 с пустым артефактом** — verdict: medium. evidence: `ExtractText` не отказывает при `text_length=0`, coverage становится 0.0; объект можно заменить после валидации. route: patch → `PdfWithoutTextLayer` при нулевом эталоне.
- **2026-10-20 · blind-hunter + edge-case-hunter + verification-gap · TypeError/Malformed не покрыты, битый объект может дать 500** — verdict: medium. evidence: проверено на PDF без `/MediaBox`: `PdfiumInspector` видит 13 символов, pdfplumber бросает `TypeError`, обработчик не ловит. route: patch → расширить набор исключений и тесты.
- **2026-10-20 · blind-hunter + edge-case-hunter · конкурентный upsert: два POST видят отсутствие строки и вставляют** — verdict: low. evidence: гонка требует одновременного extract одного файла; последовательный повтор идемпотентен; атомарный upsert — диалект-зависимая правка. route: reject.
- **2026-10-20 · blind-hunter · двойной parse и отсутствие повторной проверки размера/лимита страниц при extract** — verdict: low. evidence: объект уже прошёл upload-валидацию, лимит страниц отложен ещё в 4.1; повторный parse — принятая цена coverage. route: reject.
- **2026-10-20 · blind-hunter · coverage не логируется и не версионируется** — verdict: low. evidence: логирования в сервисе нет; поле отдаётся в ответе, этого достаточно для контракта. route: reject.
- **2026-10-20 · blind-hunter · `file_text.company_id` денормализован без составного FK** — verdict: low. evidence: оба поля пишутся из `StoredFile`, чтение фильтрует file_id и company_id; продемонстрированного обхода нет. route: reject.
- **2026-10-20 · blind-hunter · `MinioFileStorage.get` ловит голый Exception** — verdict: low. evidence: решение спеки — любые ошибки чтения → 502; сужение catch не меняет наблюдаемое поведение. route: reject.
- **2026-10-20 · blind-hunter · `create_router` заранее строит extract/get при переданном store_pdf** — verdict: low. evidence: engine и Minio-клиент ленивые, сетевых вызовов нет; тесты зелёные. route: reject.
- **2026-10-20 · blind-hunter · wiring теряет протоколы в Callable и коммитит read-only get** — verdict: low. evidence: сигнатуры совпадают, commit на чтении безвреден; типовой дефект без пользовательского вреда. route: reject.
- **2026-10-20 · blind-hunter · README не описывает 500, форму validation-ошибок и `\f`-неоднозначность** — verdict: low. evidence: контракт матрицы описан; дополнительные детали не влияют на использование. route: reject.
- **2026-10-20 · blind-hunter · gateway не проверяет проброс 404/422/502 для вложенных путей** — verdict: low. evidence: прокси не переписывает статус/тело, но регрессия не закреплена. route: patch → тест 422 problem+json на `/extract`.
- **2026-10-20 · blind-hunter · `__repr__` артефакта содержит полный текст** — verdict: low. evidence: логирования тел нет; repr всплывает только при отладке. route: reject.
- **2026-10-20 · blind-hunter + verification-gap · `char_count`/числитель coverage не сверяются с содержимым** — verdict: medium. evidence: проверено мутацией `non_whitespace_count → len`: 52 passed; контракт метрики не защищён. route: patch → точные ассерты.
- **2026-10-20 · blind-hunter + verification-gap · нижняя граница pdfplumber 0.11 не содержит `utils.exceptions`** — verdict: medium. evidence: модуль появляется в 0.11.6; резолвер с 0.11.0 установит несовместимую версию и приложение не импортируется. route: patch → `pdfplumber>=0.11.6` и lock.
- **2026-10-20 · blind-hunter · нет failure-injection теста rollback и дублируется FakeStorage** — verdict: low. evidence: rollback в wiring есть; тестовая гигиена не влияет на контракт. route: reject.
- **2026-10-20 · edge-case-hunter · страница без MediaBox → TypeError → 500** — verdict: medium. evidence: воспроизведено: pdfium принимает PDF, pdfplumber падает с TypeError. route: patch (тот же, что выше).
- **2026-10-20 · edge-case-hunter · конкурентный upsert → 500 второму вызову** — verdict: low. evidence: та же гонка; не воспроизводится в обычном последовательном сценарии. route: reject.
- **2026-10-20 · edge-case-hunter · claim: битый объект отдаёт 500 вместо 422** — verdict: medium. evidence: подтверждено для TypeError; Malformed уже ловится (проверено). route: patch.
- **2026-10-20 · verification-gap · `char_count` не равен числу непробельных символов при мутации** — verdict: medium. evidence: мутация проходит 52 теста. route: patch → ассерт по содержимому.
- **2026-10-20 · verification-gap · rstrip и удаление хвостовых пустых строк не проверены** — verdict: medium. evidence: удаление `rstrip` раздувает content с 237 до 10 623 символов, набор зелёный. route: patch → ассерты строк.
- **2026-10-20 · verification-gap · ветка `MalformedPDFException` не исполняется тестами** — verdict: medium. evidence: замена класса в except не роняет 52 теста. route: patch → параметризованный тест маппинга.
- **2026-10-20 · verification-gap · `min(1.0, ...)` не исполняется ни одним тестом** — verdict: low. evidence: на фикстурах отношение ровно 1.0; удаление min не ломает тесты. route: patch → unit-тест зажима.
- **2026-10-20 · verification-gap · обновление `extracted_at` при повторном extract не проверено** — verdict: low. evidence: AC требует обновления содержимого и отсутствия дубля; управление часами в репозитории отсутствует. route: defer.


## Verification

**Commands (выполнено 2026-10-20):**
- `cd pdf-extract && uv lock --check` → lock согласован (`Resolved 53 packages`)
- `cd pdf-extract && .venv/bin/ruff check .` → `All checks passed!`
- `cd pdf-extract && .venv/bin/pytest -q` → `58 passed` (extract/get, coverage ≥ 0.95 и clamp, coverage < 0.95, пустой текст, идемпотентность, tenancy, 401/404/422/502, storage, миграции, production composition)
- `cd api-gateway && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `41 passed`
- `cd deploy && docker compose config -q` → exit 0
- `cd pdf-extract && docker build -q -t pdf-extract:ci .` → образ собран

**Не покрыто автоматически:** реальные PDF ГРАНД-Смета/АВК-Смета (в репозитории нет корпуса); AC ≥95% подтверждается только прокси-метрикой pdfplumber/pypdfium2 на сгенерированных фикстурах.
