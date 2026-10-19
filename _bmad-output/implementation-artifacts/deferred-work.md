- source_spec: `_bmad-output/implementation-artifacts/spec-2-1-авторизация-и-привязка-к-компании.md`
  summary: Edge-слой аутентификации — api-gateway проверяет JWT, затирает/устанавливает identity-заголовки и проксирует запросы, наружу публикуется только он
  evidence: Вынесено из Story 2.1 при сужении объёма (спека ~3560 токенов > 1600). Ядро истории (регистрация, компания, вход/токен) закрывается без gateway; edge-контракт AD-010 — отдельный shippable deliverable, зависит только от формата claims из этой истории.
  status: resolved
  resolved_by: `_bmad-output/implementation-artifacts/spec-2-gateway-jwt-i-identity-заголовки.md` (2026-10-19, внеплановая build-спека)
- source_spec: `_bmad-output/implementation-artifacts/spec-2-2-договоры-произвольной-глубины-adjacency-list.md`
  summary: `epic-2-context.md` потерял упоминание «один контейнер PostgreSQL в compose» в AD-014 и сохраняет противоречие: цель эпика говорит, что edge-слой уже реализован Story 2.0, а Cross-Story Dependencies — что «Epic 2 вводит фактический edge-слой»
  evidence: Правки `epic-2-context.md` сделаны до baseline Story 2.2 (docs-работа прошлой сессии), не этой историей; при ревью диффа видно, что AD-014 сокращён до «Отдельная база на сервис», а строка про edge-слой осталась от прежней редакции.
