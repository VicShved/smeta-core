---
id: SPEC-llm-cloud-fallback-policy
companions: []
sources:
  - ../../planning-artifacts/epics-smeta1-2026-10-17/epics.md
  - ../implementation-artifacts/epic-1-context.md
  - ../../planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md
---

> **Canonical contract.** This SPEC and the files in `companions:` are the complete, preservation-validated contract for what to build, test, and validate. Source documents listed in frontmatter are for traceability — consult them only if you need narrative rationale or prose color this contract intentionally omits.

# Story 1.3 — Политика облачного fallback и аудит (llm-gateway)

## Why

Владелец данных требует гарантии, что данные смет не утекают во внешнюю LLM без явного согласия — конфиденциальность сметных данных. Это mandate (политика безопасности) + pain (риск утечки наименований/количеств/ЕИ при неправильно настроенном fallback). Локальный CPU-only рантайм (Story 1.2) работает без обращения к облаку, но политика должна защитить и любой потенциальный облачный путь, чтобы данные смет оставались внутри периметра без явного разрешения.

## Capabilities

- **CAP-1**
  - **intent:** Система блокирует облачный инференс по умолчанию, когда `llm.allow_cloud_fallback=false`.
  - **success:** При попытке облачного запроса с дефолтной конфигурацией запрос отклоняется и фиксируется событие `denied` в журнале аудита; данные не покидают систему.
- **CAP-2**
  - **intent:** Система редуцирует данные до структуры при облачном fallback (`llm.redaction_mode=structure_only`), передавая наружу только структурно-анонимизированные фрагменты.
  - **success:** Исходящий во внешнюю LLM payload содержит только типы полей и схему — без наименований, количеств и ЕИ; проверяется тестом на состав исходящих данных.
- **CAP-3**
  - **intent:** Система фиксирует аудит всех LLM-событий (`attempt`, `fallback`, `denied`, `allowed`) с полными метаданными.
  - **success:** Журнал аудита содержит записи всех четырёх типов событий, каждая с timestamp, инициатором, причиной и объёмом структурных блоков; проверяется чтением журнала после сценариев.
- **CAP-4**
  - **intent:** Система требует явного согласия для включения облачного fallback и фиксирует это включение в аудите.
  - **success:** Перевод `allow_cloud_fallback` в `true` без явного согласия не активирует fallback; каждое успешное включение порождает аудит-событие с инициатором и причиной.

## Constraints

- `llm.allow_cloud_fallback=false` по умолчанию — любая попытка облачного инференса блокируется (AD-003, NFR-002).
- `llm.redaction_mode=structure_only` — во внешнюю LLM передаются только структурно-анонимизированные фрагменты (типы полей, схема), без наименований/количеств/ЕИ (AD-003).
- Включение облачного fallback требует явного согласия и записывает аудит-событие (timestamp, инициатор, причина, объём структурных блоков) (AD-003).
- Все события `attempt`/`fallback`/`denied`/`allowed` сохраняются в журнале аудита (Story 1.3 AC).

## Non-goals

- Реализация самого облачного провайдера LLM — только политика и интерфейс, не интеграция с внешним API.
- UI для управления политикой fallback.
- Отдельная БД для аудита — MVP использует журнал (файловый/логгер).

## Success signal

При конфигурации по умолчанию облачный запрос блокируется с событием `denied` в журнале аудита; при `redaction_mode=structure_only` наружу уходят только структурные схемы без значений. Демонстрация: тест подтверждает deny + `denied`-запись при дефолте и отсутствие наименований/количеств/ЕИ в исходящем payload при включённом fallback.

## Assumptions

- llm-gateway — единственный компонент, реализующий политику fallback (AD-003 binds llm-gateway, pdf-extract, mapping-service; но реализация политики — в llm-gateway).
- Облачный провайдер будет подключён позже; в этом спеке только интерфейс и политика.

## Open Questions

- Какой конкретный backend для журнала аудита в MVP — файл (JSONL) или логгер?
- Как передаётся явное согласие на включение fallback — env-флаг, API-эндпоинт, или оба?
