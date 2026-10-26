# Review: adversarial two-units attempt — пересчёт ЕИ (FR-018/FR-019)

- Verdict: **changes requested** — 6 high, 8 medium, 2 low.
- Scope: `ARCHITECTURE-SPINE.md` (ред. 2026-10-26, AD-002/AD-016/AD-017/AD-018 + Data Model) против контекста PRD FR-007/FR-009/FR-017/FR-018/FR-019.
- Reviewer: adversarial lens, run inline.

Метод: для каждого участка строятся две единицы на уровень ниже. Каждая буквально соблюдает **все** AD, но реализует другой контракт. Если две легальные единицы дают разные `qty_catalog`/`cost`, разное состояние позиции или разные аудит-записи — это дыра, которую нужно закрыть новым/уточнённым AD. Spine не изменялся; ниже — только предложения.

Контекст: в шапке spine `sources: []`, PRD не указан. Часть дыр — правила, которые есть в PRD, но отсутствуют в субстрате; билдер, читающий только spine, вправе реализовать иначе.

---

## A-1 (high) — ручной коэффициент: «последний приоритет» vs «переопределяет словарь»

**Локации:** AD-017 (`ARCHITECTURE-SPINE.md:121`, «Приоритет: … → ручной коэффициент»); PRD FR-007 (`prd.md:262`, «переопределяет словарь»); PRD FR-018 (`prd.md:299`); pipeline шаг 6 (`ARCHITECTURE-SPINE.md:252-255`); Data Model `estimate_item.conversion_source` (`ARCHITECTURE-SPINE.md:214`).

**Пара:**
- Unit A — движок пересчёта `estimate`: реализует порядок буквально, ручной коэффициент применяется только если не сработали identity/словарь/размеры/упаковка.
- Unit B — API/application `estimate`: трактует сохранённый `conversion_source=manual` как безусловный override (как требует FR-007).

**Почему обе легальны:** AD-017 ставит manual последним; FR-007 говорит, что manual переопределяет словарь. Оба текста — часть контракта, каждый билдер может выбрать свой.

**Несовместимость:** при наличии ручного коэффициента и совпадающего правила словаря A даёт автоматический `qty_catalog`, B — ручной. Дополнительно повторный `match-rates`: A (manual — fallback) перезапишет ручной коэффициент, B сохранит («повторный `match-rates` его не сбрасывает»). Одна и та же позиция — разная стоимость и разный аудит.

**Предлагаемое закрытие:** уточнить AD-017: список приоритетов — только порядок **автоматического** вывода; presence валидного `conversion_source=manual` — отдельный override-уровень выше всех автоматических. Явно зафиксировать, что `match-rates` не перезаписывает manual.

---

## A-2 (high) — `conversion_factor` не имеет направления; `qty` мутируется → двойной пересчёт

**Локации:** AD-017 (`ARCHITECTURE-SPINE.md:121`, «пересчитывает `qty` из ЕИ сметы в ЕИ справочника», `conversion_factor`, cost); AD-001 (`ARCHITECTURE-SPINE.md:39`); Data Model `qty`, `qty_catalog?`, `conversion_factor?` (`ARCHITECTURE-SPINE.md:214`); PRD-глоссарий (`prd.md:92`).

**Пара:**
- Unit A: `conversion_factor` — эффективный множитель `orig → catalog`, `qty_catalog = qty × factor`; пересчёт всегда стартует от неизменного `qty`.
- Unit B: `conversion_factor` — это `factor_to_base` исходной ЕИ (как в `unit_conversion_rule`), `qty_catalog = qty × base(orig) / base(catalog)`; результат пишется в `qty` (или повторный пересчёт читает `qty_catalog`).

**Почему обе легальны:** AD-017 не задаёт формулу и не говорит, что `qty`/`unit_orig` immutable. Data Model хранит и `qty`, и `qty_catalog`, но не запрещает перезапись `qty`. PRD-формула `qty_catalog = qty × коэффициент` (`prd.md:92`) в spine отсутствует.

**Несовместимость:** `unit_orig=м`, `unit_catalog=мм`: A получает factor `1000`, B — `factor_to_base(м)=1` и умножает на 1 → ошибка в 1000×. После повторного «Пересчитать ЕИ» B конвертирует уже пересчитанное → двойной пересчёт. `conversion_factor_snapshot` и `cost` расходятся.

**Предлагаемое закрытие:** AD-017 — точная формула: `conversion_factor = base(unit_orig) / base(unit_catalog)`, `qty_catalog = qty × conversion_factor`. `qty` и `unit_orig` immutable; источник пересчёта всегда — исходные поля, никогда `qty_catalog`; `identity ⇒ factor=1`. `conversion_factor` в позиции — не `factor_to_base`.

---

## A-3 (high) — два владельца интерпретации ЕИ: `material.unit` в rate-catalog vs словарь в estimate

**Локации:** AD-017 Binds (`ARCHITECTURE-SPINE.md:119`); AD-008 (`ARCHITECTURE-SPINE.md:74`, импорт обновляет строки по натуральному ключу); AD-002 (`ARCHITECTURE-SPINE.md:44`); AD-016 (`ARCHITECTURE-SPINE.md:116`, контракт `matches`); Data Model `material.unit` (`ARCHITECTURE-SPINE.md:216`) и `estimate_item.unit_catalog?` (`ARCHITECTURE-SPINE.md:214`); PRD R-13 (`prd.md:515`).

**Пара:**
- Unit A — импорт `rate-catalog`: нормализует ЕИ при импорте (маппер колонок AD-008 «ЕИ»), хранит каноническое `м2`, исходную строку теряет.
- Unit B — движок `estimate`: ожидает сырой `material.unit` и нормализует его через `unit_conversion_rule`; при `match-rates` кэширует целевую ЕИ в `unit_catalog` и больше не читает справочник.

**Почему обе легальны:** AD-017 говорит, что rate-catalog — «источник `material.unit`», но не запрещает нормализацию при импорте и не говорит, кто единственный интерпретатор. AD-008 разрешает обновлять материал по натуральному ключу (ЕИ может измениться). `unit_catalog` в позиции не описан как «замороженная копия» или «ссылка».

**Несовместимость:** после повторного импорта A отдаёт `м2`, B ждёт `м²`; если `unit_catalog` кэширован, третья легальная реализация перечитывает `material.unit` при каждом пересчёте — та же позиция даёт разные `conversion_factor`/`qty_catalog`. PRD R-13 («пересчёт при следующем действии») не говорит, какое действие и перечитывается ли ЕИ.

**Предлагаемое закрытие:** новый AD (или расширение AD-017): `rate-catalog` хранит `material.unit` **verbatim**, не нормализует и не конвертирует; единственный интерпретатор — `estimate`. Контракт `POST /v1/rate-catalog/matches` обязан возвращать `material.unit`. `unit_catalog` — замороженная целевая ЕИ, зафиксированная при `match-rates`/`correct`; смена `material.unit` не меняет её автоматически, только явное действие с аудитом. `unit_normalized` — производное поле `estimate`, не вход.

---

## A-4 (high) — `unit_snapshot` не определён: исходная или каталожная ЕИ; когда снимается снапшот

**Локации:** AD-002 (`ARCHITECTURE-SPINE.md:44`); AD-017 (`ARCHITECTURE-SPINE.md:121`); Data Model (`ARCHITECTURE-SPINE.md:214`: `unit_orig`, `unit_catalog?`, `unit_snapshot`); pipeline шаг 7 (`ARCHITECTURE-SPINE.md:256-258`); PRD FR-009 (`prd.md:338`).

**Пара:**
- Unit A: `unit_snapshot = unit_orig` (сохраняет исходный замысел, UI показывает «orig → catalog»).
- Unit B: `unit_snapshot = unit_catalog` (расценка задана за каталожную ЕИ, значит снапшот обязан быть каталожным).

**Почему обе легальны:** AD-002 перечисляет `unit_snapshot` без семантики; Data Model держит оба поля. AD-017 замораживает версию, но не говорит, в какой момент и из каких полей пишутся снапшоты.

**Несовместимость:** валидатор стоимости, diff и экспорт читают `unit_snapshot` по-разному: A отвергает корректный `cost` (единица не та), B принимает; `GET /diff` показывает разные ЕИ для одной версии. Вторая ось: A копирует live-значения в снапшот на `match-rates` (позже `calculate` использует устаревшее), B — на `calculate` и заново пересчитывает ЕИ (нарушает «расчёт не пересчитывает ЕИ заново», `prd.md:338`). Обе версии буквальны.

**Предлагаемое закрытие:** AD-002: `unit_snapshot = unit_catalog` (целевая ЕИ расчёта). Снапшоты пишет **только** `calculate`, копируя уже вычисленные live-поля; `calculate` не читает rate-catalog/словарь и не запускает пересчёт. Явно назвать `conversion_source_snapshot` (см. A-9).

---

## A-5 (high) — округление: что именно хранит `qty_catalog_snapshot` и на какой точности считается factor

**Локации:** AD-017 (`ARCHITECTURE-SPINE.md:121`: `qty_catalog` — 4 знака `ROUND_HALF_UP`; cost = `qty_catalog_snapshot × rate_amount_snapshot`); AD-001 (`ARCHITECTURE-SPINE.md:39`); Data Model (`ARCHITECTURE-SPINE.md:214`); PRD (`prd.md:92-94, 306, 336`).

**Пара:**
- Unit A: считает factor в неограниченном Decimal-контексте, `qty_catalog` квантует до 4 знаков, но в `qty_catalog_snapshot` кладёт **неокруглённое** значение; cost = неокруглённое × rate, затем 2 знака.
- Unit B: в снапшот кладёт уже 4-знаковое значение; cost = 4-знаковое × rate.

**Почему обе легальны:** AD-017 требует квантовать `qty_catalog`, но не говорит, что снапшот хранит именно квантованное значение. Промежуточная точность деления (`1/0.06`, `1/7`) и Decimal-контекст не заданы; PRD говорит «`conversion_factor` не округляется до результата» (`prd.md:306`), что допускает разные толкования.

**Несовместимость:** `qty=3`, factor `1/7`, rate `1000`: A — `qty_snapshot=0.428571…`, cost `428.57`; B — `qty_snapshot=0.4286`, cost `428.60`. Итог версии расходится; SM-11 (детерминизм) не выполняется между двумя легальными реализациями.

**Предлагаемое закрытие:** AD-017 — фиксированный конвейер: factor вычисляется и хранится `Numeric(24,12)` с `ROUND_HALF_UP`; `qty_catalog = quantize(qty × factor, 4, ROUND_HALF_UP)`; `qty_catalog_snapshot` хранит **то же** 4-знаковое значение; `cost = quantize(snapshot × rate, 2, ROUND_HALF_UP)`; промежуточные деления — фиксированный Decimal context, float запрещён.

---

## A-6 (high) — «ручная» блокировка: `reviewed_at` vs `conversion_reviewed_at`; инвалидация при смене целевой ЕИ

**Локации:** Data Model `estimate_item` (`ARCHITECTURE-SPINE.md:214`: `conversion_reviewed_at/by`, `conversion_note?`, `reviewed_at/by`, `conversion_factor?`); AD-017 (`ARCHITECTURE-SPINE.md:121`); AD-018 (`ARCHITECTURE-SPINE.md:126`); PRD FR-007 (`prd.md:262, 274`); `match_feedback` (`ARCHITECTURE-SPINE.md:227`).

**Пара:**
- Unit A: признак «ручного» — `conversion_reviewed_at` / `conversion_source=manual`; сохраняет коэффициент при любом `correct`, даже если новый материал/`unit_catalog` имеет другое измерение.
- Unit B: признак «ручного» — `reviewed_at` (решение сметчика); PATCH только `conversion_factor` не ставит `reviewed_at`, поэтому следующий `match-rates` перезаписывает «ручной» коэффициент.

**Почему обе легальны:** spine вводит и `reviewed_at`, и `conversion_reviewed_at`, не говоря, какой из них означает manual-lock. AD-017 требует «автор/время ручного коэффициента», но не определяет предикат сохранения. FR-007 запрещает сброс при `match-rates`, но не говорит, что делает `correct` при смене целевой ЕИ.

**Несовместимость:** A применяет старый множитель к новой ЕИ (размерно неверно, молчаливо неверный cost); B теряет ручной коэффициент при повторном сопоставлении (нарушение FR-007). Поля `conversion_note` и `match_feedback.comment` дублируют причину.

**Предлагаемое закрытие:** AD-017/AD-018 — один предикат manual-lock и один набор полей. Ручной коэффициент привязан к `(material_id, unit_orig, unit_catalog)`; при смене любого элемента он инвалидируется (позиция → `conversion_required` или требует повторного подтверждения) с записью в аудит. Явно: PATCH только `conversion_factor` ставит manual-lock; `match-rates` его не сбрасывает.

---

## A-7 (medium) — `conversion_required` — статус или производный флаг?

**Локации:** AD-017 (`ARCHITECTURE-SPINE.md:121`); AD-016 (`ARCHITECTURE-SPINE.md:116`); Data Model `status` (`ARCHITECTURE-SPINE.md:214`); PRD (`prd.md:96, 99, 238`: статус остаётся `confirmed`).

**Пара:**
- Unit A: добавляет `conversion_required` в enum `status` (буквально «позиция получает `conversion_required`»).
- Unit B: оставляет `status=confirmed`, выставляет `uncovered_reason=conversion_required`/флаг.

**Почему обе легальны:** AD-017 не говорит, что это причина, а не статус; Data Model содержит `status` и отдельного поля причины нет. AD-016 говорит только про `needs_review`.

**Несовместимость:** фильтрация `needs_review`, состав `uncovered`, исключение из расчёта, приоритет с `rejected`/`no_match`, повторное открытие после ручного коэффициента — всё расходится. Экспорт получает разные `status`.

**Предлагаемое закрытие:** AD-017/Data Model — `conversion_required` только `uncovered_reason`/производный флаг, никогда `status`; задать приоритет причин (`rejected > no_match > conversion_required > …`) и условие снятия (валидный manual/правило + явный пересчёт).

---

## A-8 (medium) — enum `conversion_source` не совпадает с `unit_conversion_rule.kind` (`multiple`)

**Локации:** AD-017 (`ARCHITECTURE-SPINE.md:121`: `identity|dictionary|dimensions|package|manual`); AD-018 и Data Model (`ARCHITECTURE-SPINE.md:228`: `identity|dictionary|multiple|dimensions|package`); Data Model `conversion_source` (`ARCHITECTURE-SPINE.md:214`); PRD (`prd.md:305`).

**Пара:** Unit A пишет для кратной единицы `conversion_source=dictionary`; Unit B расширяет enum источника значением `multiple` (или кладёт туда `kind`).

**Почему обе легальны:** списки в AD-017 и AD-018 действительно разные; маппинг не задан.

**Несовместимость:** `match_feedback.conversion_source` и экспорт FR-017 содержат разные значения для одного правила; аналитика/UI не могут сравнить. Валидация одного сервиса отвергает данные другого.

**Предлагаемое закрытие:** один словарь значений + явная таблица `kind → conversion_source`; решить, сворачивается ли `multiple` в `dictionary` или становится отдельным источником.

---

## A-9 (medium) — аудит: какое событие в какую таблицу; не хватает полей для FR-017; снапшот теряет провенанс

**Локации:** AD-018 (`ARCHITECTURE-SPINE.md:126`); Data Model `match_feedback` (`ARCHITECTURE-SPINE.md:227` — нет `unit_orig`/`unit_catalog`/`qty_catalog`) и `unit_conversion_audit` (`ARCHITECTURE-SPINE.md:229`); AD-002 (`ARCHITECTURE-SPINE.md:44` — нет `conversion_source_snapshot`); PRD FR-017 (`prd.md:285`) и FR-007 (`prd.md:264`).

**Пара:**
- Unit A: правку `unit_catalog` позиции пишет в `unit_conversion_audit` с `rule_id=null` (это не правило словаря) и не пишет в `match_feedback`; экспортировать исходную/пересчитанную ЕИ и `qty_catalog` не может — полей нет.
- Unit B: пишет это в `match_feedback` (ручное решение) и не пишет в `unit_conversion_audit`; словарный аудит правку теряет.

**Почему обе легальны:** AD-018 требует «каждое изменение словаря → `unit_conversion_audit`» и «ручной коэффициент **дополнительно** → `match_feedback`», но не задаёт матрицу для правки ЕИ позиции и для события «Пересчитать ЕИ». Data Model `match_feedback` не содержит полей, которые PRD FR-017 требует в экспорте.

**Несовместимость:** одно действие даёт ноль, одну или две записи в зависимости от реализации; экспорт FR-017 либо неполон, либо дублирует события. Для рассчитанной версии по снапшоту нельзя восстановить, каким правилом получен `conversion_factor` (нет `conversion_source_snapshot`/rule id) — «источник правила» есть только в live-позиции.

**Предлагаемое закрытие:** матрица «событие → таблица(ы) → ровно N записей»; расширить `match_feedback` полями unit/qty или ввести item-scoped запись в `unit_conversion_audit`; добавить `conversion_source_snapshot` (и `conversion_rule_id_snapshot`) в AD-002.

---

## A-10 (medium) — триггеры пересчёта: весь версия или затронутые позиции; какие `correct` и какие импорты

**Локации:** AD-017 (`ARCHITECTURE-SPINE.md:121`); pipeline шаг 6 (`ARCHITECTURE-SPINE.md:252-255`); решения (`ARCHITECTURE-SPINE.md:315-317`); PRD (`prd.md:261-262, 321`).

**Пара:**
- Unit A: после любого `match-rates` пересчитывает **все** позиции активной версии.
- Unit B: пересчитывает только позиции, которых коснулся вызов (новые matched/`corrected`).

**Почему обе легальны:** AD-017 говорит «после успешного `match-rates` … пересчитывает», не задавая множество. PRD принимает `correct` с `unit_catalog` и/или `conversion_factor` (`prd.md:259`) — неясно, входит ли `correct` без материала в триггер. Импорт справочника меняет `material.unit` (AD-008), но пересчёт не запускает; «явное действие» не описано по объёму (вся версия / только `conversion_required` / только непривязанные).

**Несовместимость:** одинаковая история вызовов оставляет в версии разные наборы live `qty_catalog`; при последующем `calculate` снапшоты и итог различаются. При manual-lock (A-6) A может перезаписать больше ручных значений, чем B.

**Предлагаемое закрытие:** таблица триггеров: событие → множество позиций → идемпотентный источник. Явно: правка словаря, импорт справочника и активация версии **не** пересчитывают; ручное «Пересчитать ЕИ» — вся активная нерассчитанная версия, из исходных `qty`; матрица входов `correct`.

---

## A-11 (medium) — детерминизм приоритета: направление `priority`, tie-break, канонизация алиасов, владелец `unit_normalized`

**Локации:** AD-017 (`ARCHITECTURE-SPINE.md:121`: «точное/нормализованное совпадение», порядок); AD-018 (`ARCHITECTURE-SPINE.md:126`); Data Model (`ARCHITECTURE-SPINE.md:228`: `priority` без типа/направления; `ARCHITECTURE-SPINE.md:214`: `unit_normalized?`).

**Пара:**
- Unit A: `priority` — ascending (меньше = раньше), нормализация через алиасы словаря (casefold, кириллица/латиница, `м2`≡`м²`).
- Unit B: `priority` — descending (больше = важнее), нормализация жёсткая (`strip`+lower), `unit_normalized` заполняет `spec-service` при импорте.

**Почему обе легальны:** AD-018 не задаёт направление и tie-break; «нормализованное совпадение» не определено. Два включённых правила могут матчить один алиас/измерение; `unit_normalized` может писать и `spec-service`, и `estimate`.

**Несовместимость:** один вход → разные правила → разные `conversion_factor` и `qty_catalog`; `м2` у A совпадает, у B — нет; `unit_normalized`, записанный при импорте, расходится с пересчитанным. SM-11 нарушается.

**Предлагаемое закрытие:** AD-017/AD-018 — алгоритм канонизации (Unicode NFKC, casefold, таблица синонимов, кириллица/латиница) и владелец (`estimate`, на пересчёте; `unit_normalized` — производное, не авторитет). `priority`: тип/диапазон/направление; tie-break строго `(priority, id)`; определить tier для `multiple`.

---

## A-12 (medium) — CRUD словаря: cross-dimension упаковка, уникальность алиасов, конкурентные правки

**Локации:** AD-018 (`ARCHITECTURE-SPINE.md:126`: «положительный коэффициент, отсутствие противоречивых алиасов и циклов, поддерживаемое измерение; иначе `422`»); Data Model (`ARCHITECTURE-SPINE.md:228`); PRD FR-019 (`prd.md:318-322`).

**Пара:**
- Unit A: трактует «поддерживаемое измерение» как один dimension на правило и отклоняет `упак → кг`/`м²` как cross-dimension (`422`).
- Unit B: принимает `упак → кг`, потому что упаковочное правило по определению кросс-размерное (вес/площадь изделия).

**Почему обе легальны:** AD-017/FR-018 требуют пересчёт `упак ↔ кг` и `упак ↔ м²/м³`, а AD-018 требует «поддерживаемое измерение» — прямой конфликт не разрешён. Не задано также, уникален ли алиас глобально или в пределах dimension; конкурентные правки глобального словаря (любой авторизованный) не имеют precondition → lost update, аудит фиксирует оба «before/after» без победителя.

**Несовместимость:** словарь, валидный в A, невозможно создать в B; противоречивые алиасы валидируются по-разному; две одновременные правки дают разное итоговое правило и разный аудит. Нет контракта API (payload, alias collection, `multiple`, `priority`).

**Предлагаемое закрытие:** AD-018 — `package` — отдельный cross-dimension kind с явным списком допустимых целевых измерений; уникальность алиаса задать (глобально либо `(alias, dimension)`); добавить optimistic concurrency (`updated_at`/`If-Match`) или append-only события; сослаться на контракт API в истории.

---

## A-13 (medium) — грамматика размеров/упаковки: зависимость от словаря и составные случаи

**Локации:** AD-017 (`ARCHITECTURE-SPINE.md:121`: размеры из `unit_orig`, затем `name`; разделители; 2–3 измерения; mm/cm/m; без единицы — мм; вес упаковки); AD-018 (`ARCHITECTURE-SPINE.md:126`: разбор — детерминированный код); PRD R-11 (`prd.md:513`), допущения (`prd.md:418`).

**Пара:**
- Unit A: парсер размеров использует **жёсткую** таблицу символов (`mm|cm|m|мм|см|м`), не зависящую от словаря.
- Unit B: парсер берёт символы измерений из `unit_conversion_rule` (переиспользует алиасы), поэтому правка/отключение правила меняет разбор.

**Почему обе легальны:** AD-017 не говорит, откуда парсер берёт допустимые символы; AD-018 говорит «детерминированный код», но не «независимый от словаря». Не определены: приоритет `unit_orig` vs `name`, если размеры есть в обоих; `100 шт` как количество в упаковке vs множитель; комбинация N штук в упаковке с размерами изделия; несколько размеров в одном наименовании.

**Несовместимость:** один и тот же `unit_orig`/`name` даёт разные factor; правка словаря (алиас `см`) молча меняет будущий разбор, хотя «правка словаря не запускает массовый пересчёт» (`prd.md:309`) — при явном пересчёте результат другой. Неизвестная нотация у A → `conversion_required`, у B → случайное правило.

**Предлагаемое закрытие:** AD-017 — фиксированная грамматика и закрытая таблица символов, **независимая** от словаря; явный приоритет источника размеров; семантика «N шт в упак» и объединения с размерами; всё вне грамматики → `conversion_required`.

---

## A-14 (medium) — identity: материализовать `qty_catalog`/factor или оставить NULL

**Локации:** Data Model (`ARCHITECTURE-SPINE.md:214`: `qty_catalog?`, `conversion_factor?`, `conversion_source?` nullable); AD-017 (`ARCHITECTURE-SPINE.md:121`); PRD (`prd.md:237`: при совпадении ЕИ коэффициент равен 1).

**Пара:**
- Unit A: для каждой привязанной позиции материализует identity (`qty_catalog=qty`, `factor=1`, `source=identity`).
- Unit B: при совпадении ЕИ оставляет поля NULL и заполняет только при реальном пересчёте.

**Почему обе легальны:** Data Model помечает поля nullable; AD-017 не говорит, что identity обязан быть записан. PRD требует factor=1, но spine этого не повторяет.

**Несовместимость:** проверка «пересчёт определён?», состав `uncovered`/`needs_review`, diff, экспорт и наличие `conversion_factor_snapshot` различаются: NULL у B может трактоваться как «не пересчитано» и исключать позицию, либо как identity и включать. Одна и та же версия даёт разный итог.

**Предлагаемое закрытие:** AD-017 — identity всегда материализуется (`factor=1`, `qty_catalog=qty`, `source=identity`); NULL означает только «пересчёт не выполнялся» и недопустим для привязанной позиции; `conversion_required` — только при отсутствии правила, не при NULL.

---

## A-15 (low) — схема `unit_conversion_audit`: `before/after` и обязательность `reason`

**Локации:** AD-018 (`ARCHITECTURE-SPINE.md:126`); Data Model (`ARCHITECTURE-SPINE.md:229`: `before`, `after`, `reason?`); PRD FR-019 (`prd.md:318-322`: изменение пишется с причиной).

**Пара:** Unit A пишет `before/after` как полный JSON правила; Unit B — как diff только изменённых полей. `reason` у A опционален (как в Data Model), у B обязателен (как в PRD).

**Почему обе легальны:** тип `before/after` не задан; `reason?` nullable, а PRD требует причину. Экспорт/UI, читающие аудит, несовместимы; часть изменений без причины.

**Предлагаемое закрытие:** задать схему `before/after` (полный снимок или JSON-patch — один вариант) и сделать `reason` обязательным для update/disable/delete.

---

## A-16 (low) — `kind=identity` как хранимое правило и тип `priority`

**Локации:** Data Model (`ARCHITECTURE-SPINE.md:228`: `kind … identity …`, `priority` без типа); AD-017 (`ARCHITECTURE-SPINE.md:121`: identity — результат совпадения).

**Пара:** Unit A хранит `identity`-правила в словаре и матчит их как обычные; Unit B считает identity производным результатом (совпадение ЕИ), хранимых identity-правил не создаёт.

**Почему обе легальны:** `kind` включает `identity`, но AD-017 описывает identity как совпадение, а не правило; `priority` не типизирован.

**Несовместимость:** лишние/конфликтующие правила в словаре; разный порядок (identity-правило с `priority` может перебить точное совпадение). `priority` как строка/float/int меняет сортировку.

**Предлагаемое закрытие:** убрать `identity` из хранимых `kind` (или явно объявить производным, не участвующим в приоритете); задать `priority smallint` с диапазоном.

---

## Сводка

- **High (6):** A-1, A-2, A-3, A-4, A-5, A-6 — все ведут к молчаливо неверному `cost` или нарушают FR-007/SM-10/SM-11.
- **Medium (8):** A-7…A-14 — расходятся состояния/аудит/контракты, но без обязательной ошибки суммы в каждом сценарии.
- **Low (2):** A-15, A-16 — форматы аудита и избыточные правила.

Минимально достаточный набор правок перед реализацией FR-018/FR-019: закрыть A-1…A-5 (формула, направление, владелец целевой ЕИ, семантика снапшота, точность) и A-6 (manual-lock); остальное — уточнения AD-017/AD-018/Data Model.
