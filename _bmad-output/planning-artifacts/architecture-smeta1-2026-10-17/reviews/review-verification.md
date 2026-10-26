# Verification: закрыты ли high-findings трёх ревью в обновлённом spine

- **Дата:** 2026-10-26
- **Spine:** `ARCHITECTURE-SPINE.md` (318 строк, `updated: 2026-10-26`)
- **Docs copy:** `docs/architecture-spine.md` (316 строк)
- **Ревью:** `review-rubric.md` (5 high), `review-web-reality.md` (2 high), `review-adversarial.md` (6 high)
- **Метод:** построчная сверка high-findings с текущим spine; фактический FastAPI — по `smeta/*/uv.lock`; spine не изменялся.
- **Нумерация строк** ниже — по `ARCHITECTURE-SPINE.md`, если не указано иное.

## Итог

**13 high-findings:** 5 RESOLVED, 6 PARTIAL, 2 OPEN. Остаются 8 критичных/high-вопросов (6 PARTIAL с существенным остатком + 2 OPEN).

**Проверки 1–7:** 2 RESOLVED, 4 PARTIAL, 1 OPEN.

| № | Проверка | Статус |
|---|---|---|
| 1 | As-built banner | RESOLVED (в spine), docs copy не синхронизирован |
| 2 | AD-013 tenancy exception для словаря ЕИ | RESOLVED |
| 3 | AD-017 (manual override, формула, qty, consistency, snapshot, conversion_required) | PARTIAL |
| 4 | AD-018 (base_unit, aliases, acyclic, config, data model) | PARTIAL |
| 5 | Data model под AD-017/018 | PARTIAL |
| 6 | FastAPI pin = as-built lock | PARTIAL |
| 7 | Новые противоречия от правок | OPEN |

## Статусы high-findings по ревью

### Rubric

| ID | Статус | Где закрыто / что осталось |
|---|---|---|
| RH-1 | **PARTIAL** | Banner исправлен (`:16`): AD-017/018 названы целевым скоупом. Но migration/backfill-контракта нет; `:298` по-прежнему отдаёт «схему БД детально … миграции» историям, что ревью считало слишком широким. `AD-001 :39` всё ещё говорит «используются только числовые snapshot-поля», хотя первый расчёт и проверка ЕИ читают live `material.unit`/цену. |
| RH-2 | **RESOLVED** | `:101` явно добавляет `unit_conversion_rule`, `unit_conversion_alias`, `unit_conversion_audit` в исключения AD-013 (глобальные, без `company_id`, справочные данные). Дублируется в AD-018 `:126`. |
| RH-3 | **PARTIAL** | `:121`: «Ручной `conversion_factor` (если задан) всегда переопределяет словарь и правила» — приоритет исправлен. Но `:316` («Решённые вопросы») сохраняет старый порядок, где ручной коэффициент последний. Явного «`match-rates` не перезаписывает manual» нет. |
| RH-4 | **RESOLVED** | `:121`: «При расчёте проверяется `unit_catalog == material.unit`; если ЕИ материала изменилась после привязки, пересчёт повторяется, а при отсутствии правила позиция получает `conversion_required`». |
| RH-5 | **PARTIAL** | Добавлены `base_unit` (`:126`, `:228`), отдельная `unit_conversion_alias` (`:229`), структурированный `config` (`:126`, `:228`), утверждение об ацикличности (`:126`). Но канонический набор базовых единиц по измерениям не назван; циклы по-прежнему невыразимы в схеме (`unit_conversion_alias(alias PK, rule_id FK)` — рёбер alias→alias/base нет); композиция `factor_to_base` vs `multiple` не задана; `dimension` включает `package`. |

### Web/reality-check

| ID | Статус | Где закрыто / что осталось |
|---|---|---|
| H-1 | **RESOLVED (spine)** | `:16` больше не утверждает as-built для AD-017/018. Остаточный дрейф: в `docs/architecture-spine.md` banner отсутствует (diff со spine — только эти 2 строки), а `docs/architecture-overview.md:100` описывает пересчёт как реализованный, тогда как `:119` называет его «новый скоуп». |
| H-2 | **RESOLVED** | `:101` + `:126`: глобальный словарь ЕИ в `estimate` — явное исключение AD-013, право записи любого аутентифицированного пользователя и append-only аудит зафиксированы. |

### Adversarial

| ID | Статус | Где закрыто / что осталось |
|---|---|---|
| A-1 | **PARTIAL** | `:121` — manual override исправлен. Остаток тот же, что RH-3: противоречие с `:316`; нет явной защиты manual от повторного `match-rates`. |
| A-2 | **RESOLVED** | `:121`: направление «ЕИ сметы → ЕИ справочника», формула `qty_catalog = qty × conversion_factor`, `qty`/`unit_orig` не мутируют — всё явно. |
| A-3 | **OPEN** | Нет правила «`rate-catalog` хранит `material.unit` verbatim и не нормализует»; контракт `POST /v1/rate-catalog/matches` не обязан возвращать `material.unit`; не зафиксировано, что `unit_catalog` — замороженная копия, а не перечитываемая; владелец `unit_normalized` не определён. `:121` лишь добавляет проверку при расчёте и повторный пересчёт. |
| A-4 | **PARTIAL** | `:121` фиксирует `unit_snapshot` = ЕИ справочника и «снапшот расчёта» (момент — расчёт); `:44` — сохранение при расчёте. Но `:121` разрешает расчёту перечитывать `material.unit` и повторять пересчёт, что противоречит `AD-001 :39` («только snapshot-поля») и рекомендации ревью «`calculate` не читает rate-catalog/словарь». |
| A-5 | **PARTIAL** | `:121`: `qty_catalog` квантуется до 4 знаков `ROUND_HALF_UP`; снапшот хранит уже квантованное; `conversion_factor` — `Numeric(24,12)`. Не заданы: режим округления самого factor, квантование `cost` до 2 знаков, фиксированный Decimal-context и явный запрет float в конвейере. |
| A-6 | **OPEN** | Нет предиката manual-lock: в Data Model одновременно `reviewed_at/by` и `conversion_reviewed_at/by` (`:214`), какой из них означает ручной коэффициент — не сказано. Нет инвалидации при смене `(material_id, unit_orig, unit_catalog)`; нет явного «PATCH только `conversion_factor` ставит manual-lock, `match-rates` его не сбрасывает». |

## Детальная проверка пунктов 1–7

### 1. As-built banner — RESOLVED (spine), с doc-дрейфом

- `:16`: «AD-017/AD-018 и модель пересчёта ЕИ — целевой скоуп, ещё не реализованы; as-built статус — в `sprint-status.yaml` и build-спеках». Ложного «as-built» больше нет.
- `docs/architecture-spine.md` не содержит этого banner (sole diff). При этом spine `:16` утверждает «Файл синхронизирован с `docs/architecture-spine.md`» — фактически не синхронизирован.
- `docs/architecture-overview.md:100` описывает пересчёт ЕИ в настоящем времени (как реализованный), `:119` — как «новый скоуп». Противоречие H-1 в overview сохранилось.

### 2. AD-013 tenancy exception — RESOLVED

- `:101`: «`estimate` хранит глобальный словарь ЕИ (`unit_conversion_rule`, `unit_conversion_alias`, `unit_conversion_audit`) без `company_id` — это справочные данные, а не доменные строки компании (AD-018)».
- `:126`: то же исключение повторено в AD-018.
- Запись/чтение: «любой авторизованный пользователь», append-only аудит — зафиксировано. Роль `admin` не требуется (осознанное решение, согласуется с `:101`).

### 3. AD-017

| Подпункт | Статус | Локация / комментарий |
|---|---|---|
| manual `conversion_factor` переопределяет словарь | **PARTIAL** | `:121` — да; `:316` — старый порядок, manual последний; явной защиты от `match-rates` нет. |
| направление и формула | **RESOLVED** | `:121`: «ЕИ сметы → ЕИ справочника», `qty_catalog = qty × conversion_factor`. |
| `qty` не мутирует | **RESOLVED** | `:121`: «исходные `qty`/`unit_orig` не мутируют». |
| `unit_catalog` vs `material.unit` при расчёте | **RESOLVED** | `:121`: проверка равенства, повторный пересчёт при смене ЕИ, иначе `conversion_required`. |
| точность/момент снапшота | **PARTIAL** | `:121`: qty 4 знака `ROUND_HALF_UP`, снапшот — уже квантованные значения, factor `Numeric(24,12)`; `:44`: сохранение при расчёте. Нет округления factor, квантования `cost` до 2, фиксированного Decimal-context; расчёт перечитывает `material.unit` (см. A-4/AD-001). |
| представление `conversion_required` | **RESOLVED** | `:121` + Data Model `:214`: отдельное поле `conversion_required` (`=true` исключает из расчёта), не `status`; AD-016 `:116` — производный `needs_review`. Приоритет причин/условие снятия (A-7) не заданы, но это medium. |

### 4. AD-018

| Подпункт | Статус | Локация / комментарий |
|---|---|---|
| `base_unit` | **PARTIAL** | `:126`, `:228` — поле есть, «базовая единица из канонического набора». Сам канонический набор по измерениям (m/mm, kg/t, m²/m³ и т.д.) не перечислен. |
| уникальные нормализованные алиасы | **RESOLVED** | `:126`: «`unit_conversion_alias.alias` уникален после нормализации»; `:229`: `alias PK`, «нормализованный уникальный алиас». |
| ациклический граф алиасов | **PARTIAL** | `:126` декларирует ацикличность, но `:229` (`alias PK, rule_id FK`) не содержит рёбер alias→alias/base, по которым цикл можно проверить. Требование остаётся неисполнимым на уровне схемы. |
| структурированный `config`, без regex/формул | **RESOLVED** | `:126`: `config` — только данные; исполняемый код — детерминированный парсер `estimate`; пользовательские regex/формулы запрещены. `:228` повторяет. |
| соответствие data model | **PARTIAL** | Таблицы `:228–230` есть, `config/base_unit` есть. Не хватает выразимости циклов/целевой единицы, не задана композиция `factor_to_base` + `multiple`, `dimension` включает `package`. |

### 5. Data model под AD-017/018 — PARTIAL

Поддерживает ядро:
- `estimate_item` `:214`: `qty_catalog?`, `unit_catalog?`, `conversion_factor?`, `conversion_source?`, `conversion_required`, `conversion_reviewed_at/by?`, `conversion_note?`, `qty_catalog_snapshot?`, `conversion_factor_snapshot?`, `unit_snapshot`, `rate_amount_snapshot`.
- `match_feedback` `:227`: `conversion_factor?`, `conversion_source?`.
- `unit_conversion_rule` `:228`, `unit_conversion_alias` `:229`, `unit_conversion_audit` `:230`.

Чего не хватает для полного покрытия high-требований:
- Нет `conversion_source_snapshot` / `conversion_rule_id_snapshot` — по рассчитанной версии нельзя восстановить, каким правилом получен factor (A-9, medium, но связано с A-4/A-5).
- `match_feedback` не содержит `unit_orig`/`unit_catalog`/`qty_catalog`, хотя FR-017 требует их в экспорте (A-9, medium).
- Дублирование `reviewed_at/by` и `conversion_reviewed_at/by` без определения manual-lock (A-6, high).
- `unit_conversion_alias` не даёт рёбер для проверки циклов (RH-5/A-12, high/medium).
- `unit_conversion_rule.priority` без типа/направления/tie-break (A-11/A-16, medium/low).

### 6. FastAPI pin — PARTIAL

- `:154`: `FastAPI | 0.143.3 (as-built lock)`.
- Фактические `uv.lock` на 2026-10-26:
  - `0.143.3`: api-gateway, auth-users, company-org, estimate, llm-gateway, pdf-extract (6 сервисов).
  - `spec-service`: **0.159.1**.
  - `llm-proxy`: **0.160.0**.
- Пин верен для большинства, но не является общим as-built для полирепозитория. Нужна матрица или диапазон (как и просило ревью M-1). SQLAlchemy/Alembic drift `spec-service` (2.1.6/1.20.1) также не отражён.

### 7. Новые/остаточные противоречия — OPEN

1. **`:121` vs `:316`** — manual override vs «ручной коэффициент» последним в приоритете. Прямое противоречие внутри AD-017 и «Решённых вопросов»; влияет на `qty_catalog`/`cost`.
2. **`AD-001 :39` vs `AD-017 :121`** — «используются только числовые snapshot-поля» против проверки `unit_catalog == material.unit` и повторного пересчёта при расчёте. Для первой калькуляции снапшотов ещё нет; формулировка AD-001 неверна.
3. **`spine :16` vs `docs/architecture-spine.md`** — заявлена синхронизация, но docs copy не содержит target-banner.
4. **`docs/architecture-overview.md:100` vs `:119`** — пересчёт одновременно «реализован» и «новый скоуп»; `:100` также повторяет старый порядок с ручным коэффициентом последним и не упоминает исключение словаря ЕИ в AD-013.
5. **`:121` vs `:228`** — `conversion_source` (`identity|dictionary|dimensions|package|manual`) не содержит `multiple`/`alias`, а `kind` содержит (`alias|multiple|dimensions|package`); маппинг `kind → source` не задан (было medium/low, не исправлено).
6. **`AD-002 :44`** — «цену за период сметы захватывает `rate_snapshot_id`» по-прежнему вводит в заблуждение: `material_price` обновляется in place, цену защищает только `rate_amount_snapshot` (RM-1, medium, не исправлено).

## Остающиеся critical/high

1. **RH-1 (high, PARTIAL):** нет migration/backfill-контракта для legacy-рассчитанных версий; `:298` всё ещё отдаёт миграции историям; `AD-001 :39` неверен для первого расчёта.
2. **RH-3 / A-1 (high, PARTIAL):** `:316` противоречит `:121` по приоритету ручного коэффициента; нет явного «`match-rates` не сбрасывает manual».
3. **A-3 (high, OPEN):** не зафиксирован единственный интерпретатор ЕИ; нет `material.unit` verbatim, контракта `matches → material.unit`, «замороженности» `unit_catalog`; владелец `unit_normalized` не определён.
4. **A-4 (high, PARTIAL):** `unit_snapshot`/момент снапшота уточнены, но расчёт перечитывает `material.unit` и может пересчитывать ЕИ, что противоречит `AD-001`.
5. **A-5 (high, PARTIAL):** не зафиксирован полный precision-конвейер: округление factor, квантование cost до 2, Decimal-context, запрет float.
6. **A-6 (high, OPEN):** нет manual-lock-предиката, выбора между `reviewed_at`/`conversion_reviewed_at`, инвалидации при смене материала/ЕИ.
7. **RH-5 (high, PARTIAL):** не названы канонические базовые единицы; ацикличность не выразима схемой; не задана композиция `factor_to_base` + `multiple`.
8. **Doc-дрейф (high по влиянию H-1):** docs copy без target-banner; overview `:100/:119` противоречит себе и повторяет старый приоритет manual.

## Что не входит в high, но стоит закрыть (medium/low)

- RH-2/H-2 закрыты, но authz на запись глобального словаря — «любой авторизованный»: кросс-тенантный риск принят осознанно, без лимитов/ролей.
- A-7: `conversion_required` как поле, но нет приоритета причин и условия снятия.
- A-8/RL-2: enum `kind` и `conversion_source` не согласованы.
- A-9: снапшот теряет провенанс правила; `match_feedback` неполон для FR-017.
- A-10/A-11/A-12/A-13/A-14: триггеры, приоритет/tie-break, CRUD/конкурентность, грамматика, identity-материализация.
- A-15/A-16: схема аудита, `priority` без типа, `identity` убран из `kind` (частично).
- RM-2: `needs_review` — неполный предикат (`reviewed_at`, исключения статусов).
- RM-3/M-1/L-1/L-3: FastAPI/SQLAlchemy/Alembic/MinIO drift.
- RM-4: Capability Map не покрывает feedback/audit/деградацию/удаление/diff; таймауты/лимиты не зафиксированы.
- RM-5: нормализация/грамматика размеров (Q7) всё ещё не закрыта; `:121` даёт только частичную грамматику.
- RL-1: AD-011 «flat-модуль, 417 строк» устарел (факт — 1426 строк, 9–10 модулей).

---

## Пост-верификационные правки (2026-10-26)

После этой проверки внесены правки в spine/docs:

- **RH-1:** AD-001 уточнён: первый расчёт захватывает снапшоты из rate-catalog, далее только snapshot-поля; `calculate` не читает словарь; legacy рассчитанные версии не backfill-ятся. Добавлен контракт миграции в «Решённые вопросы».
- **RH-3/A-1:** «Решённые вопросы» приведены к AD-017: ручной коэффициент переопределяет словарь, `match-rates` его не сбрасывает, смена `material_id`/`unit_orig` снимает manual-lock.
- **A-3:** AD-016/AD-017 фиксируют `material.unit` verbatim, возврат `material.unit` из `/matches`, единственный интерпретатор — `estimate`, `unit_catalog` — замороженная копия, `unit_normalized` не участвует в пересчёте.
- **A-4:** `calculate` не пересчитывает ЕИ и не читает словарь; сверяет `unit_catalog` с `rate.unit` и уводит в `uncovered(conversion_required)`.
- **A-5:** зафиксированы precision 28, фактор 12 знаков ROUND_HALF_UP, qty 4 знака, cost 2 знака, запрет float.
- **A-6:** manual-lock — `conversion_source=manual`/`conversion_reviewed_at`; инвалидация при смене `material_id`/`unit_orig`.
- **RH-5:** канонические base units (`m`, `m2`, `m3`, `kg`, `шт`), alias-таблица без alias→alias рёбер, композиция `multiple × factor_to_base`, `priority` integer + tie-break по id.
- **FastAPI/ORM:** stack pins заменены на диапазоны per-service `uv.lock`.
- **Doc drift:** target-banner добавлен в `docs/architecture-spine.md` и `docs/architecture-overview.md`; overview приведён к manual-override и AD-013-исключению; docs и run-копии синхронизированы.
- **Остаток:** low-находки lint (placeholders API-путей) и средние вопросы провенанса правила/аудита; не блокируют AD-017/018.
