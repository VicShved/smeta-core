---
title: Architecture Spine: smeta1
status: draft
created: 2026-10-17
updated: 2026-10-17
altitude: system
purpose: Build substrate — invariants preventing drift between services
---

# Architecture Spine: smeta1

## Paradigm
- **Domain-driven boundaries + service-per-capability**. Микросервисы с чёткими границами по домену (org, estimate, catalog, extraction/mapping, calc). 
- **Determinism first**: расчёт стоимости полностью детерминирован. LLM — только извлечение/маппинг, **не** арифметика.

## Invariants (Architecture Decisions)

### AD-001: Детерминированный расчёт
- **Binds:** calc-service, estimate_item (snapshot-значения), валидация сумм
- **Prevents:** утечка LLM-нестабильности в финальную стоимость
- **Rule:** Calc-service использует только числовые snapshot-поля (`rate_amount_snapshot`, количества) и фиксированные формулы. LLM никогда не выполняет вычисления.

### AD-002: Snapshot расценок
- **Binds:** rate-catalog, estimate, estimate_version, estimate_item
- **Prevents:** пересчёт ранее рассчитанных активных версий смет при изменении справочника
- **Rule:** При расчёте версии сметы сохраняются `rate_snapshot_id`, `unit_snapshot`, `rate_amount_snapshot`. Изменение текущей версии rate_catalog не влияет на уже существующие версии смет.

### AD-003: Гибрид LLM с редукцией данных
- **Binds:** llm-gateway, pdf-extract, mapping-service
- **Prevents:** утечка наименований/количеств/ЕИ во внешнюю LLM
- **Rule:** `llm.allow_cloud_fallback = false` (по умолчанию). `llm.redaction_mode = structure_only`. Во внешнюю LLM передаются только структурно-анонимизированные фрагменты (типы полей, схема, без значений). Включение только с явным согласием + аудит событий (timestamp, инициатор, причина, объём структурных блоков).

### AD-004: CPU-only локальная LLM
- **Binds:** llm-gateway (local runtime)
- **Prevents:** привязка к GPU, выход за 8 ГБ ОЗУ
- **Rule:** Локальный runtime — CPU-only, 8 ГБ ОЗУ. Предпочтение GGUF малых квантов **q2–q3**. Без GPU.

### AD-005: Цифровые PDF без OCR
- **Binds:** pdf-extract
- **Prevents:** избыточная сложность MVP
- **Rule:** Поддерживаются только цифровые PDF с текстовым слоем (источники: **ГРАНД-Смета**, **АВК-Смета**, совместимые). OCR не реализуется в MVP.

### AD-006: Произвольная иерархия договоров
- **Binds:** company-org
- **Prevents:** искусственное ограничение вложенности
- **Rule:** Договоры хранятся в виде **adjacency list** (`parent_id`). Поддерживается **произвольная глубина**. Навигация/отображение — на уровне API/UI без жёсткого фиксированного уровня.

### AD-007: Версионность смет + готовность к diff
- **Binds:** estimate, estimate_version, estimate_item
- **Prevents:** потеря истории правок
- **Rule:** У сметы может быть несколько версий, **ровно одна активная** (`is_active=true`). Структура данных (версия + позиции со снапшотами) готова к вычислению diff. UI diff — **1.1**.

### AD-008: Импорт справочника Excel с маппером колонок
- **Binds:** rate-catalog
- **Prevents:** жёсткая привязка к единому шаблону Excel
- **Rule:** Импорт только из **.xlsx**. Реализуется **маппер колонок** (наименование, ЕИ, цена/расценка и др.). Каждая успешная загрузка создаёт новую версию справочника.

### AD-009: Поли-репозиторная топология
- **Binds:** все сервисы (api-gateway, auth-users, company-org, estimate, pdf-extract, llm-gateway, mapping-service, rate-catalog, calc-service, ui) + deploy
- **Prevents:** спонтанный монорепо-код, скрытое переиспользование кода между сервисами без явного контракта, рассинхрон версий shared-lib
- **Rule:** Каждый сервис — **самодостаточная папка/репозиторий** в дереве `smeta/<service>/`, общий инфраструктурный деплой — отдельной папкой `smeta/deploy/` (docker-compose + k8s). Shared-lib на старте **не вводится**; общие контракты между сервисами — только через API-схемы и сообщения (версионируемые, единый владелец). Git-инициализация и remote — отдельным шагом (локальные папки созданы заранее).

## Services (микросервисные границы)
- **api-gateway** — маршрутизация, аутентификация проксирование, rate-limits при необходимости
- **auth-users** — авторизация, привязка пользователя к компании, единая роль (MVP)
- **company-org** — компании, договоры (adjacency list, произвольная глубина), этапы, подразделы этапа
- **estimate** — сметы, версии (активная ровно одна), позиции (со снапшотами)
- **pdf-extract** — PDF→текст (pdfplumber/pypdfium2), валидация цифрового PDF
- **llm-gateway** — local (CPU-only, GGUF q2–q3) + cloud (только structure_only + аудит + allow_cloud_fallback)
- **mapping-service** — сопоставление позиций со справочником, нормализация ЕИ, **confidence** (0–1), пометка <0.70
- **rate-catalog** — импорт Excel (.xlsx), маппер колонок, версии, хранение
- **calc-service** — **детерминированный** расчёт детальной/итоговой стоимости
- **s3/minio** — хранение загруженных PDF и импортируемых Excel
- **ui (Vue)** — простой UI MVP (валидация маппинга/diff — 1.1)

## Конвейер обработки
1. Загрузка PDF → s3/minio (валидация цифрового)
2. pdf-extract → текст
3. llm-spec (через llm-gateway local-first) → черновик спецификации
4. mapping-service → предложения маппинга с confidence
5. calc-service (детерминированный) → расчёт при валидированных/подтверждённых сопоставлениях

**Инвариант конвейера:** шаги 3–4 используют llm-gateway только в режиме local-first; cloud-fallback — исключительно при включённом флаге + structure_only + аудит.

## Data Model (ключевые сущности)
- **company** (id, name, ...)
- **contract** (id, company_id, parent_id, name, ...) — adjacency list, произвольная глубина
- **stage** (id, contract_id, name, ...)
- **stage_subdivision** (id, stage_id, name, ...) — «подраздел этапа»
- **estimate** (id, stage_subdivision_id, name, ...)
- **estimate_version** (id, estimate_id, version_no, is_active, calculated_at, ...) — ровно одна is_active
- **estimate_item** (id, estimate_version_id, row_ref, material_work, unit_orig, qty, section, rate_catalog_id?, **rate_snapshot_id**, **unit_snapshot**, **rate_amount_snapshot**, confidence, status, ...)
- **rate_catalog** (id, company_id, current_version_id, ...)
- **rate_catalog_version** (id, rate_catalog_id, version, imported_at, source_file_hash, ...)
- **file** (id, storage_key, mime_type, size, meta)

## NFR-проверяемые цели
- Spec coverage ≥ 95% (цифровые PDF ГРАНД/АВК)
- Price mapping ≥ 90–95%
- Доля позиций с confidence < 0.70 ≤ 15%
- Время обработки 30–60с при локальной CPU-only модели
- Расчёт 100% детерминированный
- Объёмы: 20–50 смет/мес, 100–500 позиций/смета, PDF ≤ 5 МБ
- Локальная LLM CPU-only 16ГБ RAM, GGUF q2–q3, без GPU

## Deferred (не фиксируем в spine)
- **Queue broker**: RabbitMQ/Redis Streams/NATS/Kafka — выбор отложить до архитектурного детализационного шага (зависит от деплоя K8s/local)
- **Точное имя локальной GGUF-модели**: будет подобрано под CPU-only 16ГБ (q2–q3) при прототипировании
- **UI деталей**: валидация маппинга UI, UI diff — **1.1**

## Open Questions (минимум)
- Конкретный брокер очереди? (Deferred)
- Точная локальная модель под 16ГБ CPU-only? (Deferred)
