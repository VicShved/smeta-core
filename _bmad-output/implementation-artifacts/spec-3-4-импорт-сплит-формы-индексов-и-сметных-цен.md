---
title: 'Story 3.4 — Импорт сплит-формы индексов и сметных цен'
type: 'feature'
created: '2026-10-24'
status: 'done'
route: 'dispatch'
context:
  - '_bmad-output/implementation-artifacts/spec-3-1-импорт-справочника-из-excel-с-маппером-колонок.md'
  - 'docs/architecture-spine.md'
---

> **2026-10-24:** модель уточнена Story 3.5 — материалы региональные, база 2022 вынесена в `material_base_price`, текущие цены зоновые. Глобальная схема из этой истории заменена.

## Intent

**Problem:** справочник цен принимал только формат КСР из 4 колонок с готовой ценой. Реальные файлы ФСБЦ приходят сплит-формой из 9 колонок: шапка с названиями и нумерацией, над таблицей — заголовок, реквизиты и пояснительный текст; цена в текущем уровне цен может быть `-`; актуальная цена получается умножением сметной цены на 01.01.2022 на индекс изменения сметной стоимости.

**Approach:** научить `rate-catalog` потоково читать все строки листа, находить шапку таблицы, распознавать оба формата, хранить все 9 колонок и вычислять `rate_amount = base_estimate_price × index_value`. Снять ограничение на число строк данных.

## Boundaries & Constraints

**Always:** только `.xlsx`; файл ≤ 20 МБ; лимита строк нет; невалидные строки не прерывают импорт; регион/год/квартал задаются только query-параметрами; ценовые зоны пока не учитываются; `rate_amount` — единственная цена, используемая сметами.

**Never:** не менять контракт `estimate`/gateway; не добавлять выборку по ценовой зоне; не парсить субъект/период из шапки файла; не терять поддержку старого формата КСР.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Сплит-форма | 9 колонок, шапка ниже заголовка/текста | 200: все валидные строки импортированы, `rate_amount = base × index` | — |
| Строка нумерации `1…9` | сразу после шапки | пропускается, в данные не попадает | — |
| `-` в текущей цене | колонка 8 | `current_estimate_price = null` | — |
| Нет индекса | есть цена текущего уровня | `rate_amount = current_estimate_price` | — |
| Нет источника цены | нет колонок 4/5/8/9 | строка в `rejected` | HTTP 200, причина `price is not available` |
| Битый индекс/цена | нечисловое значение | строка в `rejected` | HTTP 200 |
| > 5000 строк | большой файл | импортируются все строки | — |
| Нестандартная шапка | явный `mapping` | импорт по маппингу | 422, если поля не покрыты |
| Файл > 20 МБ | — | — | 413 |

## Code Map

- `rate-catalog/app/domain/rate.py` — поля групп/цен в `Material`, `MaterialPrice`, `Rate`, `RateInput`.
- `rate-catalog/app/domain/ports.py` — потоковый `WorkbookParser.iter_rows`.
- `rate-catalog/app/infrastructure/excel_parser.py` — ленивое чтение первого листа.
- `rate-catalog/app/application/mapping.py` — алиасы КСР/ФСБЦ, поиск шапки, обязательные поля и источник цены.
- `rate-catalog/app/application/import_rates.py` — потоковый импорт без лимита строк, формула, fallback, батчевые эмбеддинги.
- `rate-catalog/app/infrastructure/models.py`, `repository.py`, `migrations/versions/0003_split_form_price_columns.py` — хранение новых полей.
- `rate-catalog/app/transport/schemas.py`, `routes.py` — новые nullable-поля в ответе.
- `ui/src/api/client.ts`, `ui/src/views/PriceCatalogView.vue` — показ всех колонок.

## Tasks & Acceptance

- [x] Потоковый парсер `.xlsx` и поиск шапки в первых 1000 строках — шапки КСР и ФСБЦ распознаются, текст сверху пропускается.
- [x] Пропуск строки нумерации колонок и forward-fill групп — merged-ячейки групп не теряются.
- [x] Формула `base_estimate_price × index_value` с округлением до 4 знаков; fallback на текущую/старую цену — цена для смет корректна.
- [x] Снять `MAX_DATA_ROWS`; импортировать все непустые строки — тест на 5001 строку проходит.
- [x] Хранить и отдавать все 9 колонок — API и UI показывают группы, базовые цены, индекс и вычисленную цену.
- [x] Миграция 0003 и `alembic check` — схема без дрейфа.

**Acceptance Criteria:**
- Given сплит-форма из примера, when POST импорта, then 3 строки импортированы, `rate_amount` = `74273.7303`, `30790.1517`, `1272.5255`, группы сохранены, `current_estimate_price = null`.
- Given файл с 5001 строкой, when импорт, then `imported = 5001`, без 422 по количеству строк.
- Given старый КСР-файл, when импорт, then поведение и ответ не изменились.
- Given строка без источника цены, when импорт, then она в `rejected`, остальные строки импортированы.

## Implementation Notes

- Шапка ищется по набору заголовков: `code`, `name`, `unit` + источник цены. Если за 1000 строк шапка не найдена, используется первая непустая строка (совместимость с явным маппингом нестандартных файлов).
- `ParsedRow.number` — реальный номер строки Excel; номера в `rejected` соответствуют файлу.
- Ценовые зоны игнорируются: файлы разных зон одного региона/периода перезаписывают совпавшие коды. Это ограничение зафиксировано осознанно.
- Эмбеддинги считаются батчами по 500; при сбое материалы/цены сохраняются, векторы добираются `POST /embeddings/backfill`.
- `group_code`/`group_name` обновляются только при непустом значении, чтобы импорт старого формата не стирал классификацию.

## Verification

- `cd rate-catalog && .venv/bin/ruff check .` → OK.
- `POSTGRES_TEST_DSN=... .venv/bin/pytest -q` → 129 passed, 1 skipped.
- `POSTGRES_DSN=... .venv/bin/alembic check` → `No new upgrade operations detected`.
- `cd ui && npm test` → 150 passed; `npm run build` → OK.
- Регрессия: `estimate` → 347 passed, 2 skipped; `api-gateway` → 70 passed.

## Review Triage Log

- Ожидает ревью.
