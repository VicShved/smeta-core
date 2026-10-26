---
epic: 8
date: 2026-10-26
verdict: accepted
criteria: profiled
headless: false
supersedes_for_closeout: '_bmad-output/implementation-artifacts/epic-8-retro-10-25-2026.md'
---

# Ретроспектива-закрытие Epic 8: сопоставление и ручная проверка

## Epic summary

Epic 8 закрыт в объёме Stories 8.1–8.8. Исходное ретро 2026-10-25 покрывало только Story 8.1; это ретро фиксирует фактическое состояние всего эпика.

- **8.1** — matching по эмбеддингам, период сметы, UI-кнопка сопоставления.
- **8.2** — append-only `match_feedback`, `rejected`, `correction_link`, lookup, JSONL-экспорт.
- **8.3** — авто-подтверждение, `reviewed_at`, чекбокс «Верно», sticky-колонки.
- **8.4** — модальный выбор материала, поиск `q`, проверка периода.
- **8.5** — `material_id` вместо `rate_catalog_id`, backfill, material-API.
- **8.6** — защита тестов от рабочей БД.
- **8.7** — `unmatch`/`no_match`, `reviewed_by`, пустой confidence, метка автора.
- **8.8** — пагинация справочника в модалке.

## Evidence

- `estimate`: **364 passed, 2 skipped**; миграции `0009`–`0015`, живая БД на `0015 (head)`.
- `rate-catalog`: **148 passed, 2 skipped**; guard тестовой БД.
- `auth-users`: **19 passed**; claim `email`.
- `api-gateway`: **70 passed**; `X-User-Email` и защита от подмены.
- `ui`: **162 passed**, typecheck; production build.
- E2E: **5 passed**, включая ручную проверку — выбор материала, `исправлено: <логин>`, пустой confidence, `no_match` без чекбокса, пагинация модалки.
- Backfill: **792/792** позиции, **5/5** feedback, unresolved `0`.
- Живой smoke: `material_id` в items, модалка с пагинацией, метка автора, `no_match` без чекбокса.

## Findings

### Resolved

- **F1 — UI-путь проверки.** Stories 8.2–8.8 дают полный путь: авто-подтверждение, снятие чекбокса, выбор материала, «нет в справочнике», автор исправления.
- **F2 — Ссылка на материал.** `estimate_item.material_id` — ссылка на `material`, цена берётся за период; `rate_snapshot_id` сохраняет историческую цену.
- **F3 — Безопасность тестов.** Тесты `rate-catalog` и integration не могут писать в рабочую БД; остатки удалены.
- **F4 — E2E-покрытие ручной проверки.** Добавлен браузерный сценарий на `correct`/`unmatch`/`reviewed_by`/пагинацию; попутно исправлен locator заголовка таблицы.

### Accepted / deferred

- Калибровка `needs_review=0.70` на реальном корпусе — принятый риск (`deferred-work.md`).
- Attributes/rerank/abstention matching — вне текущего объёма.
- UI diff — 1.1.
- Серверная проверка роли `admin` для импорта/настроек — deferred.

## Open questions

Открытых вопросов, блокирующих закрытие Epic 8, нет.

## Action items

1. **Shved:** пересмотреть калибровку порога при появлении реального корпуса PDF/справочника — accepted risk, не блокер.
2. **dev:** UI diff — отдельная история 1.1.
3. **dev:** attributes/rerank/abstention — отдельное улучшение matching, если понадобится по метрикам.

## Verdict

**accepted.** Epic 8 завершён: реализация, миграции, backfill, документация, тесты и браузерный e2e согласованы. Остаточные пункты приняты как риск или вынесены в 1.1.
