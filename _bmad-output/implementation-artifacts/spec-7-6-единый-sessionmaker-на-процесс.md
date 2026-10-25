---
title: 'Story 7.6 — один engine/sessionmaker на процесс во всех сервисах'
type: 'chore'
created: '2026-10-21'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-4-retro-10-20-2026.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Почти каждый сервис вызывает `make_session_factory()` без DSN в нескольких местах wiring, и каждый вызов создаёт новый `create_engine` со своим пулом. Количество пулов растёт с числом use case'ов (5 в `company-org`, 4 в `pdf-extract`, 2 в `rate-catalog` и `auth-users`), соединения не переиспользуются, а при reload контейнера пулы копятся.

**Approach:** Сделать фабрику сессий для дефолтного DSN единой на процесс: кэшировать её по разрешённому DSN, оставив явный DSN (тесты) без кэша.

## Boundaries & Constraints

**Always:** один engine/sessionmaker на процесс для одного `POSTGRES_DSN`; явный DSN в вызове (тесты) создаёт отдельный engine и не отравляет кэш; поведение соединений (`pool_pre_ping`, `check_same_thread` для SQLite, `expire_on_commit=False`) не меняется; сервисы остаются независимыми (общий модуль не вводится).

**Never:** shared-lib между сервисами; изменение схемы, миграций, контрактов API; изменение пула/таймаутов; правки в `deploy`; изменения в тестах, кроме новых кейсов на кэш.

**Решения (2026-10-20):** кэш по разрешённому DSN (`lru_cache`), явный DSN — без кэша.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Прод-вызов | `make_session_factory()` дважды | один и тот же объект фабрики | — |
| Смена env | другой `POSTGRES_DSN` | отдельная фабрика для нового DSN | — |
| Явный DSN | `make_session_factory(dsn)` дважды | разные фабрики (тесты) | — |
| SQLite | DSN `sqlite:///...` | прежний `check_same_thread=False` | — |

</frozen-after-approval>

## Code Map

- `company-org/app/infrastructure/db.py`, `pdf-extract/app/infrastructure/db.py`, `rate-catalog/app/infrastructure/db.py`, `estimate/app/infrastructure/db.py`, `auth-users/app/infrastructure/db.py` -- кэш фабрики по DSN.
- `company-org/tests/test_db.py`, `pdf-extract/tests/test_db.py`, `rate-catalog/tests/test_db.py`, `estimate/tests/test_db.py`, `auth-users/tests/test_db.py` -- новый общий по смыслу тест на кэш и явный DSN.
- Не трогать: wiring (вызовы остаются `make_session_factory()`), миграции, контракты, `deploy`.

## Tasks & Acceptance

**Execution:**
- [x] `*/app/infrastructure/db.py` -- кэш дефолтной фабрики -- один пул на процесс.
- [x] `*/tests/test_db.py` -- кэш/явный DSN/смена env -- регрессия.
- [x] `company-org`, `pdf-extract`, `rate-catalog`, `estimate`, `auth-users` -- ruff/pytest -- верификация.

**Acceptance Criteria:**
- Given сервис стартует, when wiring создаёт несколько use case'ов, then все используют один engine для дефолтного `POSTGRES_DSN`.
- Given тест передаёт явный DSN, when фабрика запрошена дважды, then возвращаются разные объекты и они не попадают в кэш прода.
- Given `POSTGRES_DSN` изменён между вызовами, when фабрика запрошена без DSN, then для нового DSN создаётся отдельная фабрика.

## Implementation Notes

## Spec Change Log

## Review Triage Log

## Design Notes

- `make_session_factory(dsn=None)`: явный DSN — `create_engine` на вызов; без DSN — `_default_session_factory(resolved)` с `lru_cache`. Ключ — разрешённый DSN, поэтому тесты с разными временными DSN не мешают друг другу.

## Verification

**Результат 2026-10-25:** `ruff check .` — чисто во всех пяти сервисах; `pytest -q tests/test_db.py` — 3 passed в каждом (auth-users, company-org, estimate, pdf-extract, rate-catalog).

**Commands:**
- `cd company-org && .venv/bin/ruff check . && .venv/bin/pytest -q` -- expected: зелёные.
- `cd pdf-extract && .venv/bin/ruff check . && .venv/bin/pytest -q` -- expected: зелёные.
- `cd rate-catalog && .venv/bin/ruff check . && .venv/bin/pytest -q` -- expected: зелёные.
- `cd estimate && .venv/bin/ruff check . && .venv/bin/pytest -q` -- expected: зелёные.
- `cd auth-users && .venv/bin/ruff check . && .venv/bin/pytest -q` -- expected: зелёные.
