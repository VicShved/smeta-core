---
title: 'Story 1.3 — Политика облачного fallback и аудит'
type: 'feature'
created: '2026-10-19'
baseline_commit: 78bb41638237451f0acdd037938acae2eec53dde
status: 'done'
story_key: '1-3-политика-облачного-fallback-и-аудит'
epic_num: 1
story_num: 3
context:
  - epic-1-context.md
  - ../../planning-artifacts/epics-smeta1-2026-10-17/epics.md
  - ../../planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md
---

# Story 1.3 — Политика облачного fallback и аудит

## Overview

Реализовать политику безопасности llm-gateway: облачный fallback LLM по умолчанию запрещён (`llm.allow_cloud_fallback=false`). При попытке облачного инференса с дефолтной конфигурацией запрос блокируется с фиксацией события аудита `denied`. При разрешённом fallback с режимом `llm.redaction_mode=structure_only` во внешнюю LLM передаются только структурно-анонимизированные фрагменты (типы полей, схема), без наименований, количеств и ЕИ. Любое включение облачного fallback требует явного согласия и фиксируется в аудите. Все события аудита (`attempt`, `fallback`, `denied`, `allowed`) сохраняются в журнале с полями: timestamp, инициатор, причина, объём структурных блоков. Облачный провайдер MVP не реализуется — только политика, интерфейс и аудит.

## Requirements

- **Given** конфигурация по умолчанию
  **When** система пытается выполнить облачный инференс
  **Then** `llm.allow_cloud_fallback=false` блокирует запрос и фиксируется событие `denied`
- **And** при `llm.redaction_mode=structure_only` во внешнюю LLM передаются только структурно-анонимизированные фрагменты (типы полей, схема), без наименований, количеств и ЕИ
- **And** любое включение облачного fallback требует явного согласия и записывает событие аудита (timestamp, инициатор, причина, объём структурных блоков)
- **And** все события (`attempt`, `fallback`, `denied`, `allowed`) сохраняются в журнале аудита

## Tasks & Acceptance

**Execution:**
- [x] `/home/shved/projects/smeta/llm-gateway/app/config.py` — добавить переменные конфигурации: `LLM_ALLOW_CLOUD_FALLBACK` (bool, default `false`), `LLM_REDACTION_MODE` (default `structure_only`), настройки аудита (`LLM_AUDIT_LOG_PATH`, `LLM_AUDIT_FORMAT`); валидация значений, форс `allow_cloud_fallback=false` по умолчанию.
- [x] `/home/shved/projects/smeta/llm-gateway/app/audit.py` (новый) — реализовать аудит LLM-событий. Типы событий: `attempt`, `fallback`, `denied`, `allowed`. Поля каждой записи: `timestamp` (ISO8601), `event`, `initiator`, `reason`, `structured_blocks_count`, `metadata` (опц.). Backend аудита — логгер + файловый JSONL (по умолчанию `/tmp/llm-audit.jsonl` или через env). Метод `log_event(...)` потокобезопасный (минимально).
- [x] `/home/shved/projects/smeta/llm-gateway/app/policy.py` (новый) — политика облачного fallback: определение необходимости облачного вызова, проверка `allow_cloud_fallback`, фиксация событий `attempt`/`denied`/`allowed`/`fallback`, применение структурной редукции (`structure_only`) к payload перед отправкой в облако. При запрете — выбрасывать исключение/возвращать сигнал deny с событием `denied`. Любое включение fallback требует явного согласия (проверять конфиг + флаг явного согласия, фиксировать в аудите).
- [x] `/home/shved/projects/smeta/llm-gateway/app/main.py` — интегрировать политику аудита (минимально): при сценариях, связанных с попыткой облачного инференса, вызывать аудит; не ломать существующие эндпоинты local-first.
- [x] `/home/shved/projects/smeta/llm-gateway/.env.example` — добавить `LLM_ALLOW_CLOUD_FALLBACK=false`, `LLM_REDACTION_MODE=structure_only`, `LLM_AUDIT_LOG_PATH`, `LLM_AUDIT_FORMAT=jsonl`.
- [x] `/home/shved/projects/smeta/deploy/.env.example` — добавить те же переменные.
- [x] `/home/shved/projects/smeta/llm-gateway/README.md` — кратко описать политику fallback (default false, structure_only, аудит событий).
- [x] Тесты: `/home/shved/projects/smeta/llm-gateway/tests/test_fallback_policy.py` (новый) — покрыть сценарии: (1) дефолт — попытка облачного → deny + событие `denied` в аудите; (2) `redaction_mode=structure_only` — payload редуцирован (только схемы/типы без наименований/количеств/ЕИ); (3) явное согласие — включение фиксируется, разрешает путь с аудитом `allowed`/`fallback`; (4) все события `attempt/fallback/denied/allowed` присутствуют в журнале аудита. Сохранить существующие health-тесты без изменений.

**Acceptance Criteria:**
- Given конфигурация по умолчанию, When система пытается выполнить облачный инференс, Then `llm.allow_cloud_fallback=false` блокирует запрос и фиксируется событие `denied`
- And при `llm.redaction_mode=structure_only` во внешнюю LLM передаются только структурно-анонимизированные фрагменты (типы полей, схема), без наименований, количеств и ЕИ
- And любое включение облачного fallback требует явного согласия и записывает событие аудита (timestamp, инициатор, причина, объём структурных блоков)
- And все события (`attempt`, `fallback`, `denied`, `allowed`) сохраняются в журнале аудита

## Design Notes

- Политика реализуется в `llm-gateway` (AD-003). MVP не подключает реальный облачный провайдер — только интерфейс/политика + аудит.
- Аудит: backend логгер + JSONL (MVP). Файл аудита настраивается через `LLM_AUDIT_LOG_PATH`, формат `jsonl`. Поля записи: timestamp (ISO8601 UTC), event, initiator, reason, structured_blocks_count, metadata.
- Структурная редукция `structure_only`: при подготовке payload для облака удаляются/заменяются на типы поля, исключая имена, количества, единицы измерения (ЕИ). Детали редукции фиксируются в тестах (контракт).
- Явное согласие: трактуем как явное разрешение (например env `LLM_ALLOW_CLOUD_FALLBACK=true` + явный флаг согласия). Для MVP достаточно проверки конфигурации с фиксацией в аудите причины/инициатора.

## Test Plan

- Юнит-тесты политики с моками (без реального облачного провайдера).
- Проверка записи всех 4 типов событий в JSONL.
- Проверка deny по дефолту, структуры payload при structure_only, сценария с явным согласием.

## Dependencies

- Опирается на каркас 1.1 и локальный рантайм 1.2 (не ломает local-first).

## Risks

- Небольшое расширение конфигурации (низкий риск). Изменения изолированы в llm-gateway.

## Контракт (перенесено из канонического SPEC)

Перенесено 2026-10-19 из `_bmad-output/specs/spec-llm-cloud-fallback-policy/SPEC.md` (слой канонических спек отменён решением retro-epic1-3; канонический файл удалён).

**Capabilities:**
- **CAP-1** — блокировка облачного инференса по умолчанию (`allow_cloud_fallback=false`): при попытке с дефолтной конфигурацией запрос отклоняется, фиксируется `denied`, данные не покидают систему.
- **CAP-2** — редукция до структуры при fallback (`redaction_mode=structure_only`): наружу уходят только типы полей и схема — без наименований/количеств/ЕИ; проверяется тестом на состав исходящих данных.
- **CAP-3** — аудит всех LLM-событий (`attempt`, `fallback`, `denied`, `allowed`) с timestamp, инициатором, причиной и объёмом структурных блоков; проверяется чтением журнала после сценариев.
- **CAP-4** — явное согласие для включения fallback: перевод `allow_cloud_fallback=true` без явного согласия не активирует fallback; каждое включение порождает аудит-событие с инициатором и причиной.

**Non-goals:**
- Реализация самого облачного провайдера LLM — только политика и интерфейс, не интеграция с внешним API.
- UI для управления политикой fallback.
- Отдельная БД для аудита — MVP использует журнал (файловый JSONL/логгер).

**Success signal:** при конфигурации по умолчанию облачный запрос блокируется с событием `denied` в журнале; при `structure_only` наружу уходят только структурные схемы без значений. Демонстрация: тест подтверждает deny + `denied`-запись и отсутствие наименований/количеств/ЕИ в исходящем payload при включённом fallback.

**Assumptions:**
- llm-gateway — единственный компонент, реализующий политику fallback (AD-003 binds llm-gateway, pdf-extract, mapping-service; реализация политики — в llm-gateway).
- Облачный провайдер будет подключён позже; в этом спеке только интерфейс и политика.

**Open Questions:**
- Какой конкретный backend для журнала аудита в MVP — файл (JSONL) или логгер?
- Как передаётся явное согласие на включение fallback — env-флаг, API-эндпоинт, или оба?

## Review Triage Log

- [medium][accepted 2026-10-19] **adversarial** app/config.py:10-15 — жёсткое ограничение redaction_mode. Evidence: app/config.py:55-60. Fix: рассмотреть расширение через список разрешённых режимов при необходимости. **Принято (retro-epic1-4):** fail-closed — design intent: `structure_only` единственный режим, пока AD-003 не изменится (YAGNI); расширение — при появлении реального второго режима.
- [low] **adversarial** app/audit.py:28-36 — при ошибке создания директории только warning. Evidence: app/audit.py:28-36. Fix: при желании fallback на temp.
- [low] **adversarial** app/policy.py:47-90 — смешение проверок и редукции. Evidence: app/policy.py. Fix: косметическое разделение.
- [low] **adversarial** app/audit.py:40-76 — нет валидации полей события. Evidence: app/audit.py. Fix: валидация event/initiator.
- [low] **adversarial** app/policy.py:31-44 — рекурсивный count без глубины. Evidence: app/policy.py:25-44. Fix: добавить max_depth.
- [low] **edge-case-hunter** app/policy.py:84-89 — неизвестный redaction_mode не блокируется. Evidence: app/policy.py:80-89. Fix: жёстче валидировать.
- [low] **edge-case-hunter** tests/test_fallback_policy.py — порядок событий не проверяется. Evidence: tests/test_fallback_policy.py:70-91. Fix: добавить проверку порядка.
- [low] **verification-gap** app/main.py:65-70 — интеграция cloud_policy без интеграционных тестов. Evidence: app/main.py:65-70. Fix: опц.
- [low] **verification-gap** app/audit.py — глобальный audit_logger между тестами. Evidence: app/audit.py:76-78. Fix: при необходимости сброс.
