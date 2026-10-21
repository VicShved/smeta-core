---
title: 'Story 7.7 — CI pdf-extract на реальных Postgres и MinIO'
type: 'chore'
created: '2026-10-21'
status: 'in-progress'
route: 'dispatch'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-4-retro-10-20-2026.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** CI pdf-extract гоняет только sqlite и `FakeStorage`: миграции, SQL-диалект Postgres и адаптер MinIO не проверяются. Регрессия в `MinioFileStorage` или в миграциях проходит зелёной (epic-4-retro-item-9, epic-2-retro-item-5).

**Approach:** Добавить в CI отдельную job `integration` с сервис-контейнерами Postgres и MinIO, накатить реальные Alembic-миграции и прогнать сквозной кейс upload → extract → get → чтение объекта из MinIO.

## Boundaries & Constraints

**Always:** юнит-джоба остаётся на sqlite/FakeStorage; интеграционный кейс включается только при `PDF_EXTRACT_INTEGRATION=1` и по умолчанию скипается; используется штатный wiring и адаптеры (без тестовых подмен Postgres/MinIO); проверяется, что байты объекта в MinIO совпадают с загруженным PDF.

**Never:** сервисные контейнеры не подменяют юнит-тесты; не менять контракты и миграции; не добавлять секреты в репозиторий; не трогать другие сервисы.

**Решения (2026-10-20):** Postgres — service-контейнер с healthcheck; MinIO — service-контейнер с ожиданием TCP-портом (в образе нет гарантированного `curl`/`mc`).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Локальный прогон | без `PDF_EXTRACT_INTEGRATION` | кейс скипнут | — |
| CI integration | Postgres+MinIO подняты | миграции, upload 201, extract 200, text, объект в MinIO | падение job |
| MinIO не готов | порт закрыт | ожидание до 60 с | `SystemExit` с сообщением |

</frozen-after-approval>

## Code Map

- `pdf-extract/.github/workflows/ci.yml` -- job `integration`, сервис-контейнеры, миграции, запуск кейса.
- `pdf-extract/tests/test_real_services.py` -- сквозной кейс на реальных адаптерах (skip без env).
- Не трогать: `tests/conftest.py`, wiring, миграции, deploy.

## Tasks & Acceptance

**Execution:**
- [ ] `pdf-extract/tests/test_real_services.py` -- upload → extract → get → MinIO -- реальные адаптеры.
- [ ] `pdf-extract/.github/workflows/ci.yml` -- job `integration` -- Postgres/MinIO.
- [ ] `pdf-extract` -- ruff/pytest и локальный прогон кейса против Postgres/MinIO -- верификация.

**Acceptance Criteria:**
- Given CI integration, when job выполняется, then миграции накатываются на Postgres, а кейс проходит на реальном MinIO.
- Given локальный запуск без env, when pytest, then интеграционный кейс скипается и юнит-тесты остаются зелёными.
- Given загруженный PDF, when кейс завершён, then объект в MinIO побайтно равен загруженному.

## Implementation Notes

## Spec Change Log

## Review Triage Log

## Design Notes

- Кейс лежит в `tests/` рядом с `conftest.py`, чтобы переиспользовать `make_pdf`/`migrate`; включение — по env, чтобы обычный `pytest -q` не требовал сервисов.
- `MinioFileStorage` сам создаёт bucket при первой записи — отдельный init не нужен.
- Ожидание MinIO — TCP-connect из Python: образ не гарантирует наличие `curl`/`mc`.

## Verification

**Commands:**
- `cd pdf-extract && .venv/bin/ruff check . && .venv/bin/pytest -q` -- expected: `110 passed, 1 skipped`.
- Локально с Postgres+MinIO: `POSTGRES_DSN=... MINIO_ENDPOINT=... PDF_EXTRACT_INTEGRATION=1 pytest -q tests/test_real_services.py` -- expected: `1 passed`.
- `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))"` -- expected: YAML валиден.
