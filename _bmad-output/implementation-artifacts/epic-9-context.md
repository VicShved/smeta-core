# Epic 9 Context: Пересчёт ЕИ и словарь правил

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

После привязки позиции к материалу сметчик получает количество в ЕИ справочника и видит, откуда взялся коэффициент. Эпик вводит детерминированный пересчёт ЕИ, глобальный редактируемый словарь правил и явную пометку неизвестных случаев, чтобы стоимость не считалась молчаливо неверно. Рассчитанные версии защищены снапшотами и не меняются. Скоуп целевой — на 2026-10-26 в коде отсутствует; статус эпика — backlog.

## Stories

- Story 9.1: Модель словаря, миграция и детерминированный движок
- Story 9.2: API словаря, валидация, аудит и отчёт о влиянии
- Story 9.3: Пересчёт при match-rates/correct, manual-lock и отчёт
- Story 9.4: Расчёт по пересчитанному количеству и причины uncovered
- Story 9.5: UI — две ЕИ, коэффициент, причины и словарь
- Story 9.6: Доказательство — детерминизм, e2e и защита от молчаливых ошибок

## Requirements & Constraints

- Пересчёт запускается после `match-rates` (включая авто-подтверждение), `correct`, ручного коэффициента и по действию «Пересчитать ЕИ» для активной нерассчитанной версии. Правка словаря сама позиции не пересчитывает.
- Приоритет: совпадение ЕИ → словарь/кратность → размеры из `name` → упаковка. Ручной коэффициент (`conversion_source=manual`) главнее словаря; `match-rates` его не сбрасывает; смена `material_id` или `unit_orig` снимает manual-lock.
- `unit_orig` — единица количества, не размер; размеры из `name`, `unit_orig` — fallback. Поддерживаются `x/х/×/*`, 2–3 измерения, `mm/cm/m`; без единицы — мм. Формулы: `шт/упак → м²/м³` — площадь/объём изделия × количество; `м → м²` — периметр сечения; `м → м³` — площадь сечения; `м²/м³` — identity; `кг/т` — словарь/упаковка; иначе `conversion_required`. Стенка требует толщины; конфликт источников размеров — `conversion_required`.
- Неизвестное правило: `conversion_required=true`, `needs_review=true`, позиция вне расчёта; причины `uncovered`: `no_rule`, `catalog_unit_changed`, `missing_qty_catalog`.
- Точность: `qty_catalog` — 4 знака `ROUND_HALF_UP`; `conversion_factor` — `Numeric(24,12)`; Decimal precision 28, float запрещён; деньги — 2 знака. `calculate` не читает словарь и считает `qty_catalog_snapshot × rate_amount_snapshot`; рассчитанная/неактивная версия — `409`; legacy не backfill-ится.
- Словарь глобальный, без `company_id`; правит любой авторизованный (без серверной проверки `admin`); каждое изменение — append-only `unit_conversion_audit`. Валидация: положительный коэффициент, известная `base_unit`, уникальный alias, без циклов и пользовательских формул — иначе `422`; используемое правило нельзя удалить (`409`).
- Ошибки — `application/problem+json`; чужая сущность — `404`; недоступный справочник — `502` без частичных записей.
- Критерий успеха: 0 молчаливо неверных стоимостей; повторный пересчёт даёт те же `qty_catalog`/`cost`. Тест-векторы: `шт + 200х300 мм + qty=2 → 0.1200 м²`; воздуховод `м + 200х300 мм + qty=5 → 5.0000 м²`; конфликт размеров → `conversion_required`.

## Technical Decisions

- `estimate` владеет словарём, движком, API, пересчётом и расчётом; `rate-catalog` — источник `material.unit` verbatim и цены; UI ничего не пересчитывает. `conversion_factor` — множитель «ЕИ сметы → ЕИ справочника»: `qty_catalog = qty × conversion_factor`; `qty`/`unit_orig` не мутируют. `unit_normalized` в пересчёте не участвует; `unit_catalog` — замороженная ЕИ справочника.
- Таблицы: `unit_conversion_rule` (`kind` `alias|multiple|dimensions|package`, `dimension` `length|area|volume|mass|count`, `base_unit` `m|m2|m3|kg|шт`, `factor_to_base?`, `multiple?`, `config?`, `priority`, `enabled`, автор, timestamps); `unit_conversion_alias` (уникальный нормализованный alias → `rule_id`, без alias→alias); `unit_conversion_audit` (`user_id?`, `action`, `rule_id?`, `before`, `after`, `reason?`, `created_at`).
- Эффективный коэффициент — `multiple × factor_to_base` (недостающее = 1; у `multiple` `factor_to_base=1`, у `alias`/`dimensions` `multiple=1`, у `package` коэффициент вычисляет парсер из `config`). `config` — только данные, без regex/формул. `priority`: меньше — раньше, при равенстве — по `id`.
- `estimate_item` получает nullable `qty_catalog`, `unit_catalog`, `conversion_factor Numeric(24,12)`, `conversion_source`, `conversion_required` (default false), `conversion_reviewed_at/by`, `conversion_note`, `qty_catalog_snapshot`, `conversion_factor_snapshot`. Миграция nullable, без backfill; существующие колонки не меняются.
- Снапшот расчёта фиксирует также `unit_snapshot` (ЕИ справочника), `rate_amount_snapshot`, `rate_snapshot_id`; diff сравнивает поля пересчёта. Новая калькуляция без `qty_catalog` даёт `uncovered(conversion_required)`. `conversion_required` не меняет статус (`confirmed`); `needs_review` — производный флаг (порог 0.70 или `conversion_required`).
- API: CRUD `/v1/unit-conversions/rules` через gateway с фильтрами `kind`/`dimension`/`enabled`; перед изменением — отчёт `affected_estimates`/`affected_items`; ручной коэффициент и `unit_catalog` — через `PATCH .../items/{id}`; явный пересчёт версии возвращает `converted`/`conversion_required`/`skipped_manual`.

## UX & Interaction Patterns

- Строка сметы: `исходное количество/ЕИ → количество/ЕИ справочника`, коэффициент и источник правила. `conversion_required` выделена с причиной; manual помечен «ручной коэффициент», при изменении правила — «правило изменилось».
- Ручной коэффициент — положительное число через PATCH; ошибка — текстом без внутренних деталей. «Пересчитать ЕИ» доступна только активной нерассчитанной версии.
- Экран `/units`: список правил, создание/изменение/отключение, отчёт о влиянии, ошибки валидации, история аудита. UI не пересчитывает, не округляет и не показывает `Decimal(24,12)` как есть; есть стабильные `data-testid` и доступные подписи.
- E2E: привязка `200х300` → пересчёт, неизвестное правило → `conversion_required`, ручной коэффициент, правка словаря → отчёт и кнопка пересчёта.

## Cross-Story Dependencies

- Нужно сопоставление Epic 8 (`material_id`, `rate_code`, `confidence`, `confirmed`) и справочник Epic 3 (`material.unit`, цена периода).
- Нужны версии, заморозка и расчёт Epic 6: ровно одна активная версия, `calculate`/`diff`, `uncovered`; Epic 9 меняет формулу на `qty_catalog_snapshot` и расширяет снапшоты.
- Нужен UI-каркас Epic 7; Story 9.5 добавляет колонки и `/units`, Story 9.6 расширяет Playwright.
- Внутри эпика: 9.1 — фундамент для 9.2/9.3; 9.4 опирается на 9.3; 9.5 использует API 9.2–9.4; 9.6 проверяет всё на реальном Postgres. Ручной коэффициент также пишется в `match_feedback` и JSONL-экспорт Epic 8.
