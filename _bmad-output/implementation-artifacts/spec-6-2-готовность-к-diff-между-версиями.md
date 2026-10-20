---
title: 'Story 6.2 — Готовность к diff между версиями'
type: 'feature'
created: '2026-10-20'
status: 'done'
route: 'dispatch'
baseline_commit: '122aa03'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-6-context.md'
  - '_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md'
  - '_bmad-output/implementation-artifacts/spec-6-1-создание-сметы-и-версий-ровно-одна-активная.md'
  - '_bmad-output/implementation-artifacts/spec-5-3-api-ручной-проверки-маппинга.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** У сметы уже есть несколько версий, но сравнить их нельзя: нет API, который покажет, какие позиции добавились, удалились или изменились между редакциями.

**Approach:** Добавить read-only `GET /v1/estimates/{id}/diff`: по двум версиям одной сметы сопоставить сохранённые позиции по устойчивому ключу и вернуть `added`/`removed`/`changed` с изменёнными полями. Сравнение идёт только по сохранённым данным версии, без пересчёта и обращений к rate-catalog.

## Boundaries & Constraints

**Always:** `company_id` из `X-Company-Id`; обе версии обязаны принадлежать смете компании (иначе `404`); сравнение — только сохранённые поля позиций, без повторного расчёта; при наличии snapshot-полей (`unit_snapshot`, `rate_snapshot_id`) сравниваются они, иначе выбранные значения (`unit_normalized`, `rate_catalog_id`); количество, ЕИ и расценка; детерминированный порядок ответа; Decimal — строкой; ошибки — `application/problem+json`; слои AD-011.

**Never:** UI diff (1.1); создание/редактирование/импорт позиций (Story 5.3); расчёт стоимости и заполнение snapshot-полей (6.3); LLM; обращение к rate-catalog за актуальной расценкой; изменение версий или их активация; миграции БД; правки api-gateway/deploy.

**Решения (2026-10-20):** ключ сопоставления — `row_ref`; `null`/дубликаты паруются по порядку вхождения. Story 5.3 реализована: импорт принимает `row_ref` (если не передан — подставляет индекс), поэтому для содержательного diff клиент обязан передавать стабильный `row_ref` между версиями. 6.2 поставляет только read-only API diff.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Добавление | позиция есть только в `to` | `200`, позиция в `added` | — |
| Удаление | позиция есть только в `from` | `200`, позиция в `removed` | — |
| Изменение количества | тот же ключ, другой `qty` | `200`, `changed` с `changes=["quantity"]` | — |
| Изменение ЕИ | другой `unit_snapshot`/`unit_normalized` | `200`, `changes=["unit"]` | — |
| Изменение расценки | другой `rate_snapshot_id`/`rate_catalog_id` | `200`, `changes=["rate"]` | — |
| Без изменений | одинаковые версии | `200`, пустые списки | — |
| Пустые версии | нет позиций | `200`, пустые списки | — |
| Обратный порядок | `from` новее `to` | `200`, `added`/`removed` меняются местами | — |
| Чужая/неизвестная смета | `company_id` не совпадает | — | `404` |
| Чужая/неизвестная версия | версия другой сметы/компании | — | `404` |
| Нет identity | без `X-Company-Id` | — | `401` |
| Невалидные параметры | нет/не-UUID `from_version_id`/`to_version_id` | — | `422` |
| Одинаковые версии с keyless-позициями | `from_version_id == to_version_id` | пустые списки (short-circuit) | — |
| Дубликаты `row_ref` | два и более одинаковых ключа | паруются по порядку вхождения | — |
| Позиции без `row_ref` | `row_ref=null` | попадают в `added`/`removed` | — |
| Snapshot только с одной стороны | есть в `from` или `to` | позиция в `changed` | — |

</frozen-after-approval>

## Code Map

- `estimate/app/domain/diff.py` -- новый: `ChangedItem`, `VersionDiff`, `DiffView`, keyless-правило, сравнение `quantity`/`unit`/`rate`, детерминированная сортировка.
- `estimate/app/application/diff_versions.py` -- новый: `DiffVersions`: проверка сметы и обеих версий по `company_id`, short-circuit для одинаковых версий, загрузка позиций, построение diff.
- `estimate/app/domain/ports.py` -- без изменений (`get_estimate`, `get_version`, `list_items` уже scope-ятся компанией).
- `estimate/app/transport/schemas.py`, `routes.py` -- `GET /v1/estimates/{estimate_id}/diff?from_version_id=&to_version_id=`, ответ `added`/`removed`/`changed` + summary.
- `estimate/app/wiring.py` -- метод `diff_versions` в `EstimateOperations`.
- `estimate/tests/test_diff.py` -- новый: матрица I/O, ключи, snapshot/fallback, tenancy, 401/404/422.
- `estimate/tests/test_diff.py` -- локальный `_seed` для позиций со snapshot-полями; интеграционный кейс import 5.3 → diff.
- `estimate/README.md` -- контракт diff, правило ключа и полей сравнения.
- Не трогать: `api-gateway/*`, `deploy/*`, `mapping-service/*`, `rate-catalog/*`, `calc-service/*`, `ui/*`, миграции.

## Tasks & Acceptance

**Execution:**
- [x] `estimate/app/domain/diff.py` -- модель и алгоритм diff -- единый контракт сравнения.
- [x] `estimate/app/application/diff_versions.py`, `wiring.py` -- use case и композиция -- tenancy и read-only.
- [x] `estimate/app/transport/{schemas,routes}.py` -- HTTP-контракт -- 200/401/404/422.
- [x] `estimate/tests/{test_diff,conftest}.py` -- матрица, ключи, snapshot/fallback, tenancy -- регрессия.
- [x] `estimate/README.md` -- документация контракта -- потребители 1.1/6.3.
- [x] `estimate` -- ruff/pytest -- верификация.

**Acceptance Criteria:**
- Given две версии одной сметы с позициями, when `GET /v1/estimates/{id}/diff`, then `200` и списки `added`, `removed`, `changed` с summary.
- Given изменённая позиция, when diff, then в `changes` перечислены только изменившиеся поля из `quantity`/`unit`/`rate`.
- Given одинаковые версии или пустые версии, when diff, then все списки пусты.
- Given сохранённые snapshot-поля, when diff, then сравнение идёт по ним; без них — по `unit_normalized` и `rate_catalog_id`.
- Given чужая/неизвестная смета или версия, when diff, then `404`; без `X-Company-Id` → `401`; невалидные параметры → `422`.

## Implementation Notes

## Spec Change Log

## Review Triage Log

- **2026-10-20 · blind-hunter + edge-case-hunter · same-version diff с keyless-позициями возвращает added+removed** — verdict: medium. evidence: `build_version_diff` отправляет `row_ref=null` в оба списка, а AC требует пустой diff для одинаковых версий. route: patch → short-circuit при `from_version_id == to_version_id` и тест.
- **2026-10-20 · blind-hunter · keyless-ветка недостижима через 5.3, интеграция import → diff не проверяется** — verdict: medium. evidence: 5.3 всегда подставляет `row_ref`, тесты сидят keyless напрямую; реальный путь не покрыт. route: patch → интеграционный тест import 5.3 → diff; keyless остаётся защитной веткой.
- **2026-10-20 · blind-hunter · позиционный fallback `row_ref` даёт шумный diff, README не предупреждает** — verdict: medium. evidence: вставка строки сдвигает индексы; клиент должен передавать стабильный `row_ref`. route: patch → предупреждение в README.
- **2026-10-20 · blind-hunter · порядок дубликатов `row_ref` определяется `created_at,id` и может быть случайным** — verdict: low. evidence: для одинаковых ключей порядок детерминирован для данного набора; дубликаты с одинаковыми name/unit неразличимы, явный ordinal — отдельная задача. route: reject.
- **2026-10-20 · blind-hunter · `changed` с одинаковым key не различает вхождения** — verdict: low. evidence: контракт задаёт `key=row_ref`; `before.id`/`after.id` различают вхождения. route: reject.
- **2026-10-20 · blind-hunter · `rate_amount_snapshot` не участвует в сравнении** — verdict: low. evidence: изменение расценки определяется по `rate_snapshot_id`/`rate_catalog_id`; amount — информационное поле. route: patch → зафиксировать в README/Design Notes.
- **2026-10-20 · blind-hunter · переименование при том же `row_ref` не видно в diff** — verdict: low. evidence: AC сравнивает только quantity/unit/rate; `row_ref` — идентичность. route: patch → зафиксировать в README.
- **2026-10-20 · blind-hunter · `changes: list[str]` не ограничен допустимыми полями** — verdict: medium. evidence: опечатка в домене прошла бы в OpenAPI. route: patch → `Literal["quantity","unit","rate"]`.
- **2026-10-20 · blind-hunter · нет проверки, что diff не вызывает rate-catalog** — verdict: medium. evidence: ограничение заявлено, но `rate_catalog.calls` в diff-тестах не проверялся. route: patch → assert в основном тесте.
- **2026-10-20 · blind-hunter · tenancy не покрывает версию другой сметы той же компании** — verdict: medium. evidence: 404-тест использовал чужую компанию/случайный UUID. route: patch → кейс с версией другой сметы.
- **2026-10-20 · blind-hunter + verification-gap · тест «prefers snapshots» не изолирует приоритет** — verdict: medium. evidence: менялись и snapshot, и catalog; реализация без snapshot прошла бы. route: patch → изолирующие кейсы.
- **2026-10-20 · blind-hunter + verification-gap · детерминированная сортировка не проверяется** — verdict: medium. evidence: в тестах по одному added/removed; удаление sort не ломало suite. route: patch → несколько added/removed + keyless.
- **2026-10-20 · blind-hunter · Code Map называет `DiffItem`/`ItemChange` и helper в conftest** — verdict: low. evidence: в коде `ChangedItem`/`VersionDiff`/`DiffView`, helper — локальный `_seed`. route: patch → синхронизировать спеку.
- **2026-10-20 · blind-hunter · I/O Matrix не содержит дубликатов/keyless/snapshot одной стороны/missing to_version** — verdict: low. evidence: эти случаи влияют на контракт. route: patch → добавить строки матрицы.
- **2026-10-20 · blind-hunter · README не описывает порядок, key-семантику, keyless и игнор полей** — verdict: low. evidence: контракта недостаточно потребителям 1.1/6.3. route: patch → дополнить README.
- **2026-10-20 · blind-hunter · `DiffItemResponse` не содержит `needs_review`/таймстемпов** — verdict: low. evidence: diff — отдельный ресурс; спека задаёт его поля, клиенту не нужны review-поля. route: reject.
- **2026-10-20 · blind-hunter · два `list_items` в READ COMMITTED могут дать смешанный снимок** — verdict: low. evidence: read-only MVP, один сметчик; параллельная правка между чтениями маловероятна. route: reject → допущение зафиксировано в Design Notes.
- **2026-10-20 · verification-gap · несколько изменённых полей и порядок `changes` не проверены** — verdict: medium. evidence: все тесты меняли одно поле. route: patch → кейс quantity+unit+rate.
- **2026-10-20 · verification-gap · snapshot только с одной стороны не проверен** — verdict: medium. evidence: тесты заполняли снапшоты с обеих сторон или ни с одной. route: patch → кейс one-side snapshot.
- **2026-10-20 · verification-gap · игнор `name`/`section`/`status`/`confidence` не проверен** — verdict: medium. evidence: во всех парах эти поля совпадали. route: patch → тест с разными не сравниваемыми полями.
- **2026-10-20 · verification-gap · `summary.added`/`removed` неразличимы при равных счётчиках** — verdict: medium. evidence: единственный summary 1/1. route: patch → кейс 3/2.
- **2026-10-20 · verification-gap · непустой `rate_amount_snapshot` не сериализуется тестом** — verdict: medium. evidence: везде None; ошибка формата дала бы 500. route: patch → кейс `"123.4500"`.
- **2026-10-20 · verification-gap · ветка `from_version is None` не исполняется** — verdict: medium. evidence: 404-тест подавал неизвестный `to_version`. route: patch → неизвестный `from_version_id`.

## Design Notes

- `GET /v1/estimates/{estimate_id}/diff?from_version_id=...&to_version_id=...` → `200 {from_version:{id,version_no}, to_version:{id,version_no}, added:[item], removed:[item], changed:[{key,before,after,changes}], summary:{added,removed,changed}}`.
- `DiffItem`: `{id, row_ref, name, material_work, unit_orig, unit_normalized, qty, section, rate_catalog_id, unit_snapshot, rate_snapshot_id, rate_amount_snapshot, confidence, status}`; Decimal — строкой.
- Ключ — `row_ref`; позиции без него не сопоставляются и попадают в `added`/`removed`; одинаковые ключи паруются по порядку вхождения (n-й с n-м). `row_ref` не сравнивается как поле, он идентифицирует позицию.
- `unit` effective = `unit_snapshot` если заполнен, иначе `unit_normalized`; `rate` effective = `rate_snapshot_id` если заполнен, иначе `rate_catalog_id`. Если snapshot есть только с одной стороны, effective-значения различаются и позиция попадает в `changed` (это ожидаемое следствие расчёта 6.3).
- Сравниваются только `quantity`, `unit`, `rate`; `name`/`section`/`status`/`confidence` в `changes` не входят. `rate_amount_snapshot` передаётся информационно и в сравнении не участвует (изменение расценки определяется по `rate_snapshot_id`/`rate_catalog_id`).
- `changes` — фиксированный порядок `quantity`, `unit`, `rate`. Сортировка ответа: `added`/`removed` по `(row_ref is None, row_ref, id)`; `changed` по `(key, before.id)`. Позиции не пересчитываются и не обогащаются из rate-catalog.
- Смета и обе версии читаются через существующие scope-методы; чужая и несуществующая сущности неотличимы (`404`). Одинаковые `from_version_id`/`to_version_id` дают пустой diff без чтения позиций. Изменение БД не требуется. Diff read-only: параллельная правка между чтениями версий может дать смешанный снимок (допущение MVP, один сметчик).

## Verification

**Commands (выполнено 2026-10-20):**
- `cd estimate && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `110 passed`
