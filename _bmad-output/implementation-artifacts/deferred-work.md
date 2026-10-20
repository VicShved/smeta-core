- source_spec: `_bmad-output/implementation-artifacts/spec-2-1-авторизация-и-привязка-к-компании.md`
  summary: Edge-слой аутентификации — api-gateway проверяет JWT, затирает/устанавливает identity-заголовки и проксирует запросы, наружу публикуется только он
  evidence: Вынесено из Story 2.1 при сужении объёма (спека ~3560 токенов > 1600). Ядро истории (регистрация, компания, вход/токен) закрывается без gateway; edge-контракт AD-010 — отдельный shippable deliverable, зависит только от формата claims из этой истории.
  status: resolved
  resolved_by: `_bmad-output/implementation-artifacts/spec-2-gateway-jwt-i-identity-заголовки.md` (2026-10-19, внеплановая build-спека)
- source_spec: `_bmad-output/implementation-artifacts/spec-2-2-договоры-произвольной-глубины-adjacency-list.md`
  summary: `epic-2-context.md` потерял упоминание «один контейнер PostgreSQL в compose» в AD-014 и сохраняет противоречие: цель эпика говорит, что edge-слой уже реализован Story 2.0, а Cross-Story Dependencies — что «Epic 2 вводит фактический edge-слой»
  evidence: Правки `epic-2-context.md` сделаны до baseline Story 2.2 (docs-работа прошлой сессии), не этой историей; при ревью диффа видно, что AD-014 сокращён до «Отдельная база на сервис», а строка про edge-слой осталась от прежней редакции.

## Deferred from: code review of spec-2-1-авторизация-и-привязка-к-компании.md (2026-10-20)

- Миграции и запросы проверяются только на SQLite, а деплой идёт на PostgreSQL: нет ни одного CI-прогона `alembic upgrade head`/запросов на `postgresql+psycopg`. Требуется postgres service-контейнер в CI или compose-smoke (закрывается patch-находкой о проверке контура контейнера).

## Deferred from: code review of spec-2-2-договоры-произвольной-глубины-adjacency-list.md (2026-10-20)

- `deploy/README.md` рассинхронизирован с compose (заявляет «Postgres 16» при `postgres:18.6`; «database-per-service — open question» при реализованном AD-014). Правка относится к репозиторию `deploy` и к Story 2.1 — сделана в рамках её Review Findings.

## Deferred from: code review of spec-2-3-этапы-договора.md (2026-10-20)

- `make_session_factory` не кэширует engine: каждый `make_*_runner` строит свою фабрику, на один импорт `main.py` создаётся 5 engine с отдельными пулами. Pre-existing паттерн Story 2.1/2.2; фикс — `@lru_cache`/единый sessionmaker в wiring.

