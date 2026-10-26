---
title: 'Story 9.1 — модель словаря ЕИ, миграция и детерминированный движок'
type: 'feature'
created: '2026-10-26'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
context: []
baseline_commit: 'a7668f19166588779588860fbe2f310ae96a4f29'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** В `estimate` нет словаря ЕИ и полей пересчёта, поэтому количество в ЕИ справочника не вычисляется, а расчёт может молча использовать чужую единицу.

**Approach:** Добавить глобальный словарь, nullable-поля в `estimate_item` (без backfill) и чистый Decimal-движок. Движок возвращает `qty_catalog`, `unit_catalog`, `conversion_factor`, `conversion_source`, `conversion_required`. Триггеры, API, UI и расчёт — в следующих историях.

## Boundaries & Constraints

**Always:** словарь без `company_id`; `qty`/`unit_orig` неизменны; Decimal 28 (без float), 4/12 знаков, `HALF_UP`; приоритет: manual → identity → dictionary/multiple → dimensions → package; кандидаты — только `enabled=true`, порядок `priority` asc, затем `id` asc; размеры из `name`; `unit_orig` — fallback, только если в `name` размеров нет; конфликт источников → required; неизвестное, неположительные размеры или qty<0 → required независимо от manual. `unit_catalog` = ЕИ материала verbatim (и при required); при required `qty_catalog`/`conversion_factor` = null.

**Never:** не менять колонки/статусы/`cost`; не делать backfill, seed, API, UI, триггеры `match-rates`/`correct`/`calculate`; не использовать LLM, сеть, пользовательские regex/формулы; не добавлять `company_id`; не считать `conversion_required` статусом; в 9.1 не менять update-пути репозитория.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected |
|----------|--------------|----------|
| identity | `м²` → `м2`, qty 2.5 | factor 1, `identity`, 2.5000 |
| manual | factor 2.5 | `manual`, qty ×2.5; factor ≤0 → required |
| dictionary/multiple | `см` → `м`; `100 шт` → `шт` | 0.01 `dictionary`; 100 `multiple` |
| package | `упак`, `мешок 25 кг` → `кг` | factor 25, `package`; нет веса → required |
| dims count→area | `шт`, `Плитка 200х300 мм`, qty 2 → `м2` | factor 0.06, 0.1200, `dimensions` |
| dims length→area | `м`, `200х300 мм`, qty 5 → `м2` | factor 1 (2×(0.2+0.3)), 5.0000 |
| dims length→volume | `м`, `200х300х4000 мм`, qty 5 → `м3` | factor 0.06 (0.2×0.3), 0.3000; 3-е игнорируется |
| dims count→volume | `шт`, `200х300х4000 мм`, qty 2 → `м3` | factor 0.24, 0.4800; 2D → required |
| reverse | `м2`, qty 1.2, изделие 0.06 м2 → `шт` | factor 16.666666666667, 20.0000 |
| conflict | `unit_orig=200х400`, name `200х300 мм` | required, null/null |

</frozen-after-approval>

## Code Map

- Корень кода и тестов: `/home/shved/projects/smeta/estimate`; пути ниже относительны ему.
- `app/domain/unit_conversion.py` — `convert_units(*, qty, unit_orig, unit_target, name, manual_factor, rules, aliases) -> ConversionResult`; DTO `ConversionRule(id, kind, dimension, base_unit, factor_to_base, multiple, config, priority, enabled)`, `ConversionResult(qty_catalog, unit_catalog, conversion_factor, conversion_source, conversion_required)`; нормализация, парсеры.
- `app/domain/item.py` — все новые поля с default `None`, кроме `conversion_required=False`; `qty_catalog`/snapshot `Numeric(18,4)`, factor/snapshot `Numeric(24,12)`, `conversion_reviewed_at` tz, `conversion_reviewed_by` String(255), `conversion_note` Text; `create()` не расширять.
- `app/infrastructure/models.py` — `unit_conversion_rule(id, kind, dimension, base_unit, factor_to_base, multiple, config JSON, priority int, enabled, updated_by, created_at, updated_at)` + checks kind `alias|multiple|dimensions|package`, dimension `length|area|volume|mass|count`, `base_unit in (m,m2,m3,kg,шт)`, `factor_to_base>0`, `multiple>0`, `priority>=0`; `unit_conversion_alias(alias PK normalized, rule_id FK ON DELETE CASCADE, created_at)`; `unit_conversion_audit(id, user_id, action, rule_id, before JSON, after JSON, reason, created_at)`; indexes `enabled`, `(kind,dimension)`, `priority`.
- `app/infrastructure/repository.py` — маппинг полей в `_item_to_domain`/`_item_to_row`; add/list round-trip; update-методы не трогать.
- `migrations/versions/0016_unit_conversion.py` — `down_revision="0015"`; таблицы + nullable-колонки; `conversion_required NOT NULL server_default false`; downgrade.
- `tests/test_unit_conversion.py`, `tests/test_migrations.py` — матрица/приоритет/округление; ожидаемые наборы схемы, апгрейд `0015` с позицией и downgrade.

## Tasks & Acceptance

**Execution:**
- [x] `app/domain/unit_conversion.py` + `app/domain/item.py` — движок, DTO и поля.
- [x] `app/infrastructure/models.py` + `migrations/versions/0016_unit_conversion.py` — схема и миграция.
- [x] `app/infrastructure/repository.py` — маппинг add/list.
- [x] `tests/test_unit_conversion.py` — матрица, приоритет, округление, идемпотентность, без float.
- [x] `tests/test_migrations.py` — наборы схемы, апгрейд, чтение старых полей, downgrade.

**Acceptance Criteria:**
- Given пустая БД или `0015`, when применяется `0016`, then три таблицы, nullable-поля и `conversion_required=false`; старые позиции читаются с пустыми полями пересчёта.
- Given позиция/правила, when вызывается движок, then результат по матрице, `qty`/`unit_orig` неизменны; повтор даёт тот же результат.
- Given неизвестная ЕИ, конфликт, неположительные размеры или qty<0, then `conversion_required=true`, `qty_catalog`/`conversion_factor=null`.
- Given `unit_orig=м` и 3D-размеры, when цель `м2`/`м3`, then третье измерение не участвует; стенка/полость — ручной коэффициент.

## Implementation Notes

- Реализовано в `estimate`; `conversion_source=None` при `conversion_required`.
- Площади/объёмы разных единиц (`см2→м2`) намеренно дают required: Design Notes допускает только identity; канонические множители сохранены для будущих правил.
- `package` с весом в `config` имеет приоритет над весом из `name`; без обоих — required.
- Alias нормализуется движком; запись нормализованного alias — 9.2.
- Добавлен `tests/test_item_conversion_fields.py`: round-trip add/get/list новых полей.
- `update_item*` репозитория не менялись (по границам 9.1).

## Spec Change Log

## Review Triage Log

- high / patch — blind+edge: размерные формулы не учитывают масштаб cm/mm; подтверждено `5 см → м2` даёт factor 1/qty 5 вместо 0.05; применять `_FACTOR_TO_BASE` источника и цели.
- high / patch — blind+edge: большие конечные входы дают `InvalidOperation`, а результат может не влезть в Numeric; ловить ошибку квантования и проверять factor < 1e12, qty_catalog < 1e14.
- high / patch — edge: цель `0 шт` даёт `DivisionByZero`; отклонять неположительную кратность до выбора кандидатов.
- medium / patch — blind: `0 шт` парсится как multiple 0 и уходит в identity; `_parse_multiple` должен отклонять ≤0.
- medium / patch — blind: `100 упак`/`1 упак` не пересчитываются; добавить package-кратность и снимать числовой префикс в package identity.
- medium / patch — blind: `м→м2` с обоими размерами и площадью в name возвращает required; для length→area игнорировать direct area и считать периметр по первым двум числам.
- medium / patch — blind: `ConversionRule.dimension` не читается, правило с несовместимыми dimension/base_unit пересчитывает; отбрасывать такие правила.
- medium / patch — blind: вес пакета не берётся из `unit_target`; добавить target.raw в fallback парсера веса.
- medium / patch — blind+verification: нет теста успешного fallback размеров из `unit_orig`; добавить `200х300 → 0.06`.
- medium / patch — verification: нет теста tie-break по `id` при равном `priority`; добавить два alias-правила.
- medium / patch — verification: round-trip `conversion_required=True` не проверен; добавить в repository-тест.
- medium / patch — verification: round-trip `conversion_reviewed_at/by` не проверен; добавить.
- medium / patch — verification: alias в production-форме (`UUID`) не проверен; добавить тест.
- medium / patch — verification: правило `kind="multiple"` не исполняется тестами; добавить тест.
- low / patch — verification: required-результат не проверяет `conversion_source=None`; добавлен assert.
- low / patch — verification: round-trip не проверяет неизменность `qty`/`unit_orig`; добавлены проверки в оба repository-теста.
- low / patch — blind: крошечный factor (`1e-13`) квантуется в 0, но возвращает успех; после квантования требовать factor > 0.
- low / patch — blind: package-вес из target-правила может переопределить source; связывать вес с package-стороной и при конфликте возвращать required.
- false — blind: идентификаторы `3x2.5` (кабель) трактуются как размеры; спека фиксирует грамматику NxM/NxMxK и «без единицы — мм», поэтому это следствие выбранного intent, а не дефект кода.
- false — blind: `1.2 м2 → упак` даёт 20 упак; спека явно разрешает обратный пересчёт в `упак`, а число штук в упаковке контрактом не задано.
- false — blind: правило без factor/multiple даёт 1; AD-018 фиксирует «недостающий множитель = 1».
- false — blind: нет reason code в `ConversionResult`; причины `uncovered` и `conversion_note` относятся к 9.3/9.4.
- false — blind: alias не нормализуется БД; в 9.1 движок нормализует lookup, запись нормализованного alias — 9.2.
- false — blind: audit не append-only и без индексов; аудит пишется в 9.2.
- false — blind: update-методы не сохраняют новые поля; границы 9.1 явно исключают их изменение, это работа 9.3.
- false — blind: `_is_package` substring ловит `мешалка`; результат — required, а не неверный пересчёт; безопасно.
- false — edge: `_as_decimal` принимает float; преобразование идёт через `str`, бинарных артефактов нет, сигнатура API требует Decimal.
- low / patch — edge: package identity `1 упак` отличается от `упак`; тот же дефект package-кратности, исправлен вместе с `100 упак`.

## Design Notes

- Канон: mm/cm/m = 1e-3/1e-2/1; площади/объёмы — квадрат/куб; g/kg/t = 1e-3/1/1e3; шт=1; кириллица/латиница эквивалентны (`м=m`, `кг=kg`, `т=t`); `упак` без веса не переводится. Нормализация: lower/trim, `²→2`, `³→3`, `×/х→x`, без точек/пробелов; alias хранится нормализованным и ссылается только на правило.
- Формулы: `м→м2` = `2*(a+b)`; `м→м3` = `a*b`; `шт/упак→м2` = `a*b` (2 числа); `шт/упак→м3` = `a*b*c` (3 числа); обратно — `1/площадь` или `1/объём`. `м2/м3→м2/м3` — только identity, иначе required. `100 шт` → factor `100`; обратное `шт→100 шт` — `1/100`. Вес пакета: `25 кг` → кг `25`, т `0.025`; обратно — деление.
- `config` — данные правила; `package` использует `weight` только если он есть в `config`, дефолта нет. `kind→source`: alias→dictionary, multiple→multiple, dimensions→dimensions, package→package. Audit пишется в 9.2, снапшоты — в 9.4.

## Verification

**Commands:**
- `cd /home/shved/projects/smeta/estimate && .venv/bin/ruff check .` — без ошибок.
- `cd /home/shved/projects/smeta/estimate && .venv/bin/pytest -q` — ok.
