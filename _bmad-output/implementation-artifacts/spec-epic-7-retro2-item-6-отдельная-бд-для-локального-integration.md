---
title: 'Отдельная БД для локального integration-прогона pdf-extract'
type: 'chore'
created: '2026-10-25'
status: 'done'
route: 'oneshot'
review_loop_iteration: 0
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Локальный integration-прогон `pdf-extract` (`PDF_EXTRACT_INTEGRATION=1`) вызывает `migrate(POSTGRES_DSN)` и пишет строки в живую dev-БД `pdf_extract`. Прогон может применить миграции к dev-схеме, помешать запущенному сервису и оставить данные при падении.

**Approach:** По умолчанию использовать отдельную БД `pdf_extract_test`, создавая её при отсутствии через admin-подключение к серверу из `POSTGRES_DSN`. Разрешить явный `PDF_EXTRACT_TEST_DSN`; dev-БД не трогать. Тестовый bucket остаётся отдельным (`pdf-extract-test`).

</frozen-after-approval>

## Tasks & Acceptance

**Execution:**
- [x] `pdf-extract/tests/test_real_services.py` -- тестовая БД `pdf_extract_test`, auto-create, guard, truncate, restore env -- изоляция от dev-БД.
- [x] `pdf-extract/README.md` -- `PDF_EXTRACT_TEST_DSN`, `PDF_EXTRACT_TEST_BUCKET`, CREATEDB и ограничения -- документация.
- [x] `pdf-extract` -- ruff/pytest/integration -- верификация.

**Acceptance Criteria:**
- Given `PDF_EXTRACT_INTEGRATION=1` и dev `POSTGRES_DSN`, when integration-прогон, then тест работает в `pdf_extract_test`, а dev-БД не меняет строки и `alembic_version`.
- Given `PDF_EXTRACT_TEST_DSN` указывает на ту же БД, что `POSTGRES_DSN`, when прогон, then ошибка до любых миграций и записей.
- Given тестовой БД нет, when прогон, then она создаётся, мигрирует и очищается; повторный прогон не накапливает строки.

## Review Triage Log

- Guard same DSN -- **fixed**: `_test_dsn` сравнивает имя БД; негативный прогон вернул `RuntimeError` до записи.
- Duplicate create race -- **fixed**: `CREATE DATABASE` ловит SQLSTATE `42P04`.
- Maintenance DB `postgres` -- **fixed**: существующая тестовая БД проверяется прямым подключением; создание требует `postgres` и задокументировано.
- README CREATEDB -- **fixed**: добавлены права, maintenance-база и fallback на `PDF_EXTRACT_TEST_DSN`.
- Test DB persists/stale rows -- **fixed**: `TRUNCATE file, file_text` перед прогоном; сохранение БД задокументировано.
- Env not restored -- **fixed**: `_schema` сохраняет и восстанавливает `POSTGRES_DSN`.
- `DEFAULT_DSN` ignored -- **fixed**: fallback на `DEFAULT_DSN` при отсутствии env.
- Stale docstrings -- **fixed**: модуль и `created_files` описывают отдельную БД.
- README env/local command -- **fixed**: добавлены `PDF_EXTRACT_TEST_BUCKET`, ограничения и команда запуска.
- No unit tests for helpers -- **defer**, low: integration-прогон и негативная проверка покрывают ветки; моки БД не добавлялись. Запись в `deferred-work.md`.
- Spec format -- **fixed**: добавлены Tasks & Acceptance и Review Triage Log; статус `done`. `story_key` не применим: это retro action item, а не sprint-story.
- Spec operational facts/evidence -- **fixed**: отражены CREATEDB, maintenance DB, persistence, truncation и проверка dev-БД (53 строки/`0003` до и после, тестовая — 0).

## Implementation Notes

- Решение: имя по умолчанию — `pdf_extract_test`; при наличии `PDF_EXTRACT_TEST_DSN` используется он. БД создаётся автоматически, если её нет; схема поднимается Alembic на тестовой БД.
- `POSTGRES_DSN` остаётся источником admin-подключения и может указывать на dev-БД; внутри integration-теста env переключается на тестовую БД.
- Изменены `pdf-extract/tests/test_real_services.py` (выбор/создание тестовой БД, `_schema`) и `pdf-extract/README.md` (env и описание изоляции).
- Проверка: `ruff check .` — чисто; `pytest -q` — 133 passed, 2 skipped; integration на реальных Postgres/MinIO — 2 passed. Создана `pdf_extract_test`; dev-БД `pdf_extract` не мигрировалась тестом.

## Verification

**Commands:**
- `cd pdf-extract && .venv/bin/ruff check .` -- expected: clean; actual: `All checks passed!`.
- `cd pdf-extract && .venv/bin/pytest -q` -- expected: green; actual: `133 passed, 2 skipped`.
- `POSTGRES_DSN=.../pdf_extract PDF_EXTRACT_INTEGRATION=1 ... pytest -q tests/test_real_services.py` -- expected: `2 passed`; actual: `2 passed`.
- Негативная проверка: `PDF_EXTRACT_TEST_DSN` = `POSTGRES_DSN` -- expected: ошибка до записи; actual: `RuntimeError`.
- Изоляция: dev `pdf_extract` — 53 строки, `alembic_version=0003` до и после; `pdf_extract_test` — 0 строк после прогона.
