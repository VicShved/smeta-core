---
title: 'Story 8.6 — Защита тестов rate-catalog от рабочей БД'
type: 'chore'
created: '2026-10-26'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/spec-8-5-ссылка-на-материал-справочника.md'
---

## Intent

**Problem:** В рабочей БД `rate_catalog` остались материалы `C-1` / `C-2` («Бетон первый/второй») из unit-теста `test_match_tie_break_is_deterministic_by_code`. Тестовый DSN можно было случайно направить на рабочую БД; `truncate()` перед тестом мог уничтожить справочник, а integration-тест использовал `POSTGRES_DSN` напрямую.

**Approach:** Запретить тестовые DSN без сегмента `test` и совпадающие с `POSTGRES_DSN`; integration-тест перевести на `RATE_CATALOG_TEST_DSN`; удалить тестовые остатки.

## Boundaries & Constraints

**Always:** guard до `create_database/migrate/truncate`; integration пишет только в отдельную БД и bucket; рабочая БД после тестов не меняется.

**Never:** не запускать truncate на рабочей БД; не использовать `POSTGRES_DSN` как тестовый; не удалять реальные материалы.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Unit tests | `POSTGRES_TEST_DSN=.../rate_catalog_test` | тестовая БД | — |
| Wrong DB | `.../rate_catalog` | fail before any DB change | RuntimeError |
| Same as working | test DSN = `POSTGRES_DSN` | fail before truncate | RuntimeError |
| Integration | `RATE_CATALOG_TEST_DSN` | отдельная БД, cleanup codes | — |

## Code Map

- `rate-catalog/tests/support.py` — `assert_safe_test_dsn`, `test_dsn`.
- `rate-catalog/tests/conftest.py` — guard перед созданием/truncate.
- `rate-catalog/tests/test_real_services.py` — `RATE_CATALOG_TEST_DSN`.
- `rate-catalog/tests/test_test_db_guard.py` — регрессия guard.
- `rate-catalog/.github/workflows/ci.yml` — integration DSN.
- `rate-catalog/README.md` — правила запуска.
- `estimate/tests/test_real_services.py`, `estimate/tests/test_integration_guard.py` — аналогичный guard `ESTIMATE_TEST_DSN` для integration.

## Tasks & Acceptance

- [x] guard тестового DSN до любых изменений БД.
- [x] integration на отдельный DSN.
- [x] регрессионные тесты guard.
- [x] удалить остатки `C-1`/`C-2` из рабочей БД.
- [x] проверить неизменность рабочей БД после прогона.

**Acceptance Criteria:**
- Given `POSTGRES_TEST_DSN=.../rate_catalog`, when pytest, then fail до truncate.
- Given рабочий DSN и тестовый DSN с `test`, when прогон, then рабочая БД не меняется.
- Given integration, when запуск, then используется `RATE_CATALOG_TEST_DSN`.

## Verification

- `cd rate-catalog && .venv/bin/ruff check . && .venv/bin/pytest -q` — **148 passed, 2 skipped**.
- Рабочая БД до/после прогона: `280955` материалов, `max(created_at)=2026-10-26 03:30:18` — без изменений; после удаления остатков — `280953`.
- Запуск с `POSTGRES_TEST_DSN=.../rate_catalog` — exit 1 до truncate, рабочая БД не изменилась.
- Удалены `3bf2eba4-942c-4fbc-948f-ca165682013a` и `8030bf27-8ebd-4e28-ad27-a61b4368dd37`; `material_price`/`material_embedding` — 0.
- `estimate`: **360 passed, 2 skipped**; integration на `ESTIMATE_TEST_DSN` — **2 passed**, рабочая `estimate` не изменилась.
