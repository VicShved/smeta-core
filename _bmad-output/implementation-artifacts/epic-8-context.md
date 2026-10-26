# Epic 8 Context: Семантическое сопоставление и ручная проверка (as-built)

<!-- Обновлено 2026-10-26 по Stories 8.1–8.8. Исходная версия описывала только 8.1–8.2. -->

## Goal

Дать сметчику автоматическое сопоставление позиций сметы со справочником цен и удобную ручную проверку результата. `rate-catalog` считает E5-эмбеддинг наименования и возвращает ближайшую по cosine цену за точный регион/год/квартал/зону. `estimate` хранит ссылку на материал, цену периода и снапшоты, а UI позволяет подтвердить, исправить или пометить отсутствие материала.

## Stories

- **8.1** — сопоставление по эмбеддингам, период сметы, кнопка «Сопоставить со справочником» — done.
- **8.2** — `match_feedback`, `rejected`, `correction_link`, lookup по коду, экспорт JSONL — done.
- **8.3** — авто-подтверждение найденных позиций, `reviewed_at`, чекбокс «Верно», sticky-колонки — done.
- **8.4** — модальный выбор материала, поиск `q`, проверка периода, sticky-таблица — done.
- **8.5** — `material_id` вместо `rate_catalog_id`, backfill, `material_ids` в rate-catalog — done.
- **8.6** — защита тестов от рабочей БД (`*_test`, отдельные integration DSN) — done.
- **8.7** — `unmatch`/`no_match`, `reviewed_by`, пустой confidence у ручных решений, метка «исправлено: логин» — done.
- **8.8** — пагинация справочника в модалке по 50 строк — done.

## Requirements & Constraints

- Период справочника (`catalog_region_code`, `catalog_year`, `catalog_quarter`, `catalog_zone`) хранится в смете; цена ищется только за этот период и зону.
- В позиции хранится `material_id` — логическая ссылка на `rate_catalog.material.id`; `rate_snapshot_id` фиксирует строку цены после расчёта.
- Найденная при `match-rates` позиция становится `confirmed` с `reviewed_at=null`. Повторный matching пересопоставляет `proposed` и авто-подтверждённые, но не позиции с `reviewed_at`.
- Ручное решение заполняет `reviewed_at` и `reviewed_by` (email из claim `email` → `X-User-Email`); `correct` и `unmatch` очищают `confidence`.
- Статусы: `proposed`, `confirmed`, `corrected`, `rejected`, `no_match`. `rejected`/`no_match` не входят в расчёт; причина `no_match` сохраняется в uncovered.
- `correct` выбирает материал через модалку; `unmatch` означает «нет материала в справочнике» и очищает `material_id`/`rate_code`/`correction_link`.
- UI: подписи «Верно/Неверно/Не сопоставлена» под чекбоксом; для `no_match` чекбокса нет; `corrected` показывает `исправлено: <логин>`; справочник листается по 50 строк.
- Рассчитанная/неактивная версия — `409`; чужая — `404`; недоступный справочник — `502` без частичных записей; ошибки — `application/problem+json`.

## Technical Decisions

- `rate-catalog`: `POST /matches`, `GET /prices` с `q`, `GET /rates?material_ids=...`; `RateResponse` содержит `id` цены и `material_id`.
- `estimate`: `material_id`, период, `reviewed_at/reviewed_by`, миграции `0007`–`0015`; matching и review пишут `match_feedback` в той же транзакции.
- `auth-users`/`api-gateway`: claim `email` и заголовок `X-User-Email`; шлюз затирает клиентские identity-заголовки.
- `ui`: чекбокс, picker с поиском и пагинацией, empty choice, метка автора; расчёт по-прежнему запускается через API.
- Тесты: rate-catalog unit/integration не могут стартовать на DSN без `test` и на `POSTGRES_DSN`; estimate integration использует `ESTIMATE_TEST_DSN`.

## UX & Interaction Patterns

- Экран сметы: период и «Сопоставить со справочником»; после matching найденные строки отмечены «Верно».
- «Исправить» → модалка справочника периода: поиск по коду/наименованию, 50 строк на страницу, выбор строки или «Нет подходящего материала».
- При выборе материала confidence становится `—`; при `no_match` в ячейке проверки только «Нет в справочнике».
- Рассчитанная версия заморожена: период, matching и проверка недоступны.
- Недоступность справочника — текст с повтором, позиции и итоги остаются.

## Accepted Risks / Deferred

- Калибровка порога `needs_review=0.70` на реальном корпусе — принятый риск (`deferred-work.md`).
- Attributes/rerank и abstention matching — вне текущего объёма.
- UI diff — 1.1.
- Серверная проверка роли `admin` для импорта/настроек — deferred.
