---
title: 'Story 7.5 — pdf-extract: целостность объекта, лимиты разбора и тестовая обвязка'
type: 'bugfix'
created: '2026-10-21'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-4-retro-10-20-2026.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** `pdf-extract` принимает PDF только с `%PDF-` в самом начале файла, не ограничивает число страниц и символов, при извлечении не сверяет объект из хранилища с записью файла (подменённый объект молча извлекается), а ошибки записи в хранилище дают 500 вместо единого контракта. Тестовая обвязка продублирована в четырёх файлах.

**Approach:** Закрыть эти пункты Epic 4: искать заголовок PDF в первых 1024 байтах, ввести настраиваемые лимиты страниц и символов, сверять размер (и число страниц) объекта при извлечении, перевести ошибки хранилища на общий `502`, вынести общую тестовую обвязку в `tests/conftest.py`.

## Boundaries & Constraints

**Always:** цифровые PDF без OCR (AD-005); лимит файла 50 МиБ сохраняется; лимиты страниц/символов — из env с безопасными дефолтами; превышение лимита — `422` с понятным текстом; несовпадение объекта — `502`, данные не извлекаются; ошибки — `application/problem+json`; при отказе не остаётся ни объекта в MinIO, ни строки в БД.

**Never:** OCR, очередь, новые эндпоинты, изменение контрактов `GET /v1/files/{id}` и `/v1/files/{id}/text`; изменения в других сервисах; миграции БД; реальный корпус ГРАНД/АВК (item-8 — данные от владельца).

**Решения (2026-10-20):** лимиты по умолчанию — 1000 страниц и 2 000 000 символов (`PDF_EXTRACT_MAX_PAGES`, `PDF_EXTRACT_MAX_CHARS`); проверка целостности — по размеру и числу страниц из записи файла; item-18 (контрактный тест на расхождение парсеров) откладывается до появления реального PDF, который pypdfium2 разбирает, а pdfplumber — нет.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Заголовок не в начале | `%PDF-` в первых 1024 байтах | файл принят | — |
| Заголовка нет вовсе | нет `%PDF-` | — | `415` |
| Слишком много страниц | страниц > лимита | — | `422` |
| Слишком много символов | символов > лимита | — | `422` |
| Подменённый объект | размер объекта ≠ записи | — | `502` |
| Извлечение при недоступном хранилище | ошибка чтения/записи MinIO | — | `502` |
| Запись в хранилище упала | ошибка `put_object` | строка файла не создаётся | `502` |

</frozen-after-approval>

## Code Map

- `pdf-extract/app/application/store_pdf.py` -- поиск `%PDF-` в первых 1024 байтах; проверка лимитов страниц/символов.
- `pdf-extract/app/domain/errors.py` -- `PdfTooLarge` (422), `StoredObjectMismatch` (502).
- `pdf-extract/app/domain/file.py` -- `PDF_HEADER_WINDOW`, лимиты по умолчанию.
- `pdf-extract/app/application/extract_text.py` -- сверка размера/страниц объекта с записью до извлечения; лимиты.
- `pdf-extract/app/infrastructure/storage.py` -- `put` переводит любые ошибки в `StorageUnavailable` (502).
- `pdf-extract/app/transport/problem.py` -- обработчики `PdfTooLarge` и `StoredObjectMismatch`.
- `pdf-extract/app/wiring.py` -- лимиты из env.
- `pdf-extract/tests/conftest.py` -- общая обвязка (`FakeStorage`, миграции, `make_pdf`, `harness`).
- `pdf-extract/tests/*` -- новые кейсы и переход на общий conftest.
- Не трогать: остальные сервисы, миграции, контракты ответов.

## Tasks & Acceptance

**Execution:**
- [x] `pdf-extract/app/domain/*`, `application/store_pdf.py` -- окно заголовка и лимиты -- приём и отказ.
- [x] `pdf-extract/app/domain/errors.py`, `transport/problem.py` -- новые ошибки -- 422/502.
- [x] `pdf-extract/app/application/extract_text.py` -- сверка объекта -- подмена не извлекается.
- [x] `pdf-extract/app/infrastructure/storage.py` -- единый контракт записи -- 502.
- [x] `pdf-extract/app/wiring.py` -- лимиты из env -- настраиваемость.
- [x] `pdf-extract/tests/conftest.py`, `tests/*` -- общая обвязка и новые кейсы -- регрессия.
- [x] `pdf-extract` -- ruff/pytest -- верификация.

**Acceptance Criteria:**
- Given PDF с `%PDF-` в первых 1024 байтах, when загрузка, then файл принят; без заголовка — `415`.
- Given PDF больше лимита страниц или символов, when загрузка или извлечение, then `422` без записи объекта.
- Given объект в хранилище не совпадает с записью по размеру или страницам, when извлечение, then `502`, текст не извлекается.
- Given ошибка записи в MinIO, when загрузка, then `502` и строка файла не создаётся.
- Given лимиты заданы в env, when загрузка, then используются эти значения.

## Implementation Notes

- Окно заголовка: позиция `%PDF-` ищется через `content.find(PDF_MAGIC)`, файл принимается при `0 <= index < PDF_HEADER_WINDOW` (1024); сигнатура может начинаться на смещении 1020 и заканчиваться за окном. Окончательное решение о валидности остаётся за `PdfiumInspector`.
- Лимиты на извлечении проверяются дважды: по результату инспектора и по результату экстрактора (`extracted.page_count`, непробельные символы `extracted.text`) до `upsert` артефакта.
- Восстановлен тест повторного успешного извлечения: stateful-экстрактор возвращает разный текст, оба ответа `200`, в `file_text` ровно одна строка с содержимым второго прогона.
- Тесты фиксируют дефолты `1000`/`2_000_000`, прохождение ровно на лимите и обе стороны расхождения размера/числа страниц.

## Spec Change Log

## Review Triage Log

## Design Notes

- Окно заголовка: `PDF_HEADER_WINDOW = 1024`, принимается позиция начала `%PDF-`, для которой `0 <= index < PDF_HEADER_WINDOW` (сигнатура может выходить за окно); дальше PDF разбирает инспектор.
- Лимиты: `PDF_EXTRACT_MAX_PAGES` (1000), `PDF_EXTRACT_MAX_CHARS` (2_000_000); нечисловое/неположительное значение — дефолт. Проверяются и по инспектору, и по результату экстрактора до записи артефакта; ровно лимит допустим.
- Целостность: сравнение `len(content)` с `file.size` и `inspection.page_count` с `meta["pages"]` (если ключ есть) в обе стороны. Расхождение — `StoredObjectMismatch` (502).
- Ошибка записи: любой сбой `put_object` (включая `S3Error` и сетевые) → `StorageUnavailable`; строка файла не добавляется, потому что `repository.add` идёт после `storage.put`.
- Общая тестовая обвязка переносится в `tests/conftest.py` без изменения поведения тестов; тест повторного успешного извлечения сохраняется (идемпотентный upsert одной строки `file_text`).

## Verification

**Commands (выполнено 2026-10-21):**
- `cd pdf-extract && .venv/bin/ruff check app/application/store_pdf.py app/application/extract_text.py tests/test_upload.py tests/test_extract.py` -- `All checks passed!`
- `cd pdf-extract && .venv/bin/pytest -q tests/test_upload.py tests/test_extract.py` -- `68 passed` (1 warning о deprecated `httpx` в Starlette TestClient); полный прогон — на стороне ревьюера.
- `cd deploy && docker compose config -q` -- exit 0.
