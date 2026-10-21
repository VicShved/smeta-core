---
title: 'Story 7.1b — UI: загрузка PDF и статусы обработки'
type: 'feature'
created: '2026-10-21'
status: 'done'
route: 'dispatch'
baseline_commit: '339ad78'
review_loop_iteration: 0
context:
  - '_bmad-output/implementation-artifacts/epic-7-context.md'
  - '_bmad-output/planning-artifacts/architecture-smeta1-2026-10-17/ARCHITECTURE-SPINE.md'
  - '_bmad-output/implementation-artifacts/spec-7-1a-ui-вход-и-дерево-смет.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Сметчик может загрузить PDF-смету только прямым вызовом `POST /v1/files`, а из интерфейса не видно, на каком шаге обработка: связь «файл ↔ смета» не существует, состояние файла никто не спрашивает, а четыре стадии конвейера не собраны в один ответ.

**Approach:** Дать UI загрузку PDF в контексте этапа (смета создаётся и связывается с файлом), кнопку запуска извлечения текста и сводный статус обработки «загрузка → извлечение → маппинг → расчёт», который собирает `estimate` (файл спрашивается у `pdf-extract` по API).

## Boundaries & Constraints

**Always:** принимается только цифровой PDF с текстовым слоем размером ≤ 5 МБ — валидацию выполняет `pdf-extract`, UI показывает понятный текст; загрузка идёт через `api-gateway`; связь «смета ↔ файл» хранится в `estimate` (`source_file_id`), файл проверяется запросом в `pdf-extract` (AD-014); сводный статус собирает `estimate` (файловый статус — из `pdf-extract`, позиции и расчёт — из своих данных); UI — проекция: не пересчитывает стоимость, не применяет маппинг, не ходит в LLM; стадии конвейера, недоступные через gateway (спецификация LLM), UI не запускает; ошибки — текстом без внутренних деталей; `401` — возврат к экрану входа.

**Never:** хранение файлов и бизнес-данных на клиенте, кроме access-токена; отправка PDF в обход `api-gateway`; запуск LLM-спецификации из UI; правка позиций, смет и справочника; UI diff и валидация маппинга (1.1); показ стоимости и таблицы позиций (7.2).

**Решения (2026-10-20):** сводный статус собирает `estimate` (`GET /v1/estimates/{id}/processing-status`), UI не склеивает стадии; UI запускает только извлечение текста (`POST /v1/files/{id}/extract`) — остальные стадии отображаются; связь с файлом — отдельная операция `POST /v1/estimates/{id}/source` (не меняет контракт создания сметы); стадия «маппинг» считается выполненной, когда в активной версии есть импортированные позиции (Story 5.3) — по наличию файла она не определяется.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Загрузка PDF | цифровой PDF ≤ 5 МБ | `201` от `pdf-extract`, затем смета создана и связана с файлом | — |
| Скан без текстового слоя | PDF без текста | файл не принят, понятный текст пользователю | `422` от `pdf-extract` |
| Не-PDF или большой файл | `.docx`, > 5 МБ | файл не принят | `415` / `413` |
| Привязка чужого/неизвестного файла | `file_id` вне компании | — | `404` |
| Повторная привязка | у сметы уже есть файл | — | `409` |
| Статус без файла | смета без `source_file_id` | `200`: «загрузка» не выполнена, остальные стадии — по данным | — |
| Статус после загрузки | файл принят, текст не извлечён | «загрузка» — выполнена, «извлечение» — нет | — |
| Статус после извлечения | есть текстовый артефакт | «извлечение» — выполнена | — |
| Статус после импорта позиций | у активной версии есть позиции | «маппинг» — выполнена | — |
| Статус после расчёта | `calculated_at` заполнен | «расчёт» — выполнена | — |
| Запуск извлечения | нажата кнопка | `POST /v1/files/{id}/extract`, статус обновляется | — |
| pdf-extract недоступен | сеть/5xx при сборе статуса | — | `502` |
| Нет identity | без `X-Company-Id` | — | `401` |

</frozen-after-approval>

## Open Questions

## Code Map

- `pdf-extract/app/domain/file.py`, `app/application/get_file_status.py` -- новый use case: файл компании + признак наличия текстового артефакта.
- `pdf-extract/app/transport/{schemas,routes,problem}.py` -- `GET /v1/files/{file_id}`: `{id, mime_type, size, created_at, extracted_at?, page_count?}`; `404` для чужого/неизвестного.
- `pdf-extract/app/wiring.py`, `tests/` -- сборка use case и тесты (200/404/файл без артефакта).
- `estimate/app/domain/estimate.py`, `app/domain/ports.py` -- `source_file_id` у сметы; порт чтения состояния файла (`PdfExtractPort`).
- `estimate/app/infrastructure/pdf_extract_client.py` -- новый: `GET /v1/files/{id}` с `X-Company-Id`; `404` → `None`, сеть/5xx → `PdfExtractUnavailable`.
- `estimate/app/infrastructure/repository.py`, `migrations/versions/0003_add_source_file.py` -- колонка `estimate.source_file_id` (nullable) и её запись.
- `estimate/app/application/{attach_source,processing_status}.py` -- привязка файла и сбор стадий (загрузка → извлечение → маппинг → расчёт) по активной версии.
- `estimate/app/transport/{schemas,routes,problem}.py` -- `POST /v1/estimates/{id}/source` и `GET /v1/estimates/{id}/processing-status`.
- `estimate/app/wiring.py` -- сборка клиента `pdf-extract` (`PDF_EXTRACT_URL`) и операций.
- `estimate/tests/` -- привязка (201/404/409), статус во всех сочетаниях, `502` при недоступности `pdf-extract`.
- `ui/src/api/client.ts` -- поддержка `FormData` (multipart без ручного `Content-Type`), таймаут на запрос, тексты для `413`/`415`.
- `ui/src/components/EstimateUpload.vue`, `ProcessingStatus.vue` -- выбор и загрузка PDF в контексте этапа, панель четырёх стадий.
- `ui/src/views/HierarchyView.ts` -- действие «Загрузить PDF-смету» в этапе и опрос статуса.
- `ui/tests/` -- vitest: multipart-загрузка, разбор ошибок 413/415/422, панель стадий, опрос.
- `deploy/docker-compose.yml`, `.env.example` -- `PDF_EXTRACT_URL` для `estimate`.
- Не трогать: `api-gateway/*` (префиксы `/v1/files` и `/v1/estimates` уже есть), `mapping-service/*`, `llm-gateway/*`, `rate-catalog/*`, `auth-users/*`, `company-org/*`.

## Tasks & Acceptance

**Execution:**
- [x] `pdf-extract/app/domain/*`, `application/get_file_status.py`, `transport/*`, `wiring.py` -- чтение состояния файла -- признак «извлечено».
- [x] `pdf-extract/tests/` -- 200/404/файл без артефакта -- контракт статуса.
- [x] `estimate/migrations/versions/0003_add_source_file.py` -- колонка `source_file_id` -- хранение связи.
- [x] `estimate/app/domain/*`, `app/infrastructure/pdf_extract_client.py` -- порт и клиент чтения файла -- AD-014.
- [x] `estimate/app/application/{attach_source,processing_status}.py` -- привязка и сбор стадий -- сводный статус.
- [x] `estimate/app/transport/*`, `app/wiring.py` -- HTTP-контракт -- 200/201/404/409/502.
- [x] `estimate/tests/` -- матрица, tenancy, недоступность `pdf-extract` -- регрессия.
- [x] `ui/src/api/client.ts` -- multipart, таймаут, тексты 413/415 -- загрузка без ручных заголовков.
- [x] `ui/src/components/{EstimateUpload,ProcessingStatus}.vue`, `views/HierarchyView.ts` -- загрузка и панель стадий -- пользовательский сценарий.
- [x] `ui/tests/` -- vitest: загрузка, ошибки, статусы -- регрессия UI.
- [x] `deploy/*` -- `PDF_EXTRACT_URL` для `estimate` -- запуск в стенде.
- [x] `pdf-extract` + `estimate` -- ruff/pytest; `ui` -- typecheck/build/test -- верификация.

**Acceptance Criteria:**
- Given авторизованный пользователь, when в контексте этапа загружен PDF ≤ 5 МБ с текстовым слоем, then файл принят, создана смета и она связана с файлом.
- Given скан, не-PDF или файл > 5 МБ, when загрузка, then понятное сообщение пользователю без внутренних деталей.
- Given смета с файлом, when запрошен `GET /v1/estimates/{id}/processing-status`, then `200` с четырьмя стадиями и их состояниями, соответствующими фактическим данным.
- Given смета без файла, when запрошен статус, then стадия «загрузка» не выполнена, остальные — по данным.
- Given активная версия с импортированными позициями и рассчитанная версия, when запрошен статус, then стадии «маппинг» и «расчёт» выполнены.
- Given pdf-extract недоступен, when запрошен статус, then `502`; без `X-Company-Id` — `401`.
- Given нажата кнопка извлечения, when выполняется запрос, then UI вызывает `POST /v1/files/{id}/extract` и показывает обновлённый статус.

## Implementation Notes

- **pdf-extract**: `GET /v1/files/{file_id}` -> `200 {id, mime_type, size, created_at, extracted_at?, page_count?}`. Необязательные ключи **отсутствуют**, а не равны `null` (маршрут объявлен с `response_model_exclude_none=True`): `extracted_at` — время текстового артефакта, `page_count` — число страниц из метаданных загрузки (`meta.pages`, заполняется инспектором при приёме файла; доступно сразу после загрузки). `page_count_from_meta` проверяет тип метаданных: не-dict, не-int и неположительное значение дают `None`. `storage_key` наружу не отдаётся. Чужой и неизвестный файл — неотличимые `404`; невалидный UUID в пути — `422`; без `X-Company-Id` — `401` до use case. Use case `GetFileStatus` (`app/application/get_file_status.py`) читает файл и артефакт в одной сессии; commit делает composition root.
- **estimate (домен)**: `Estimate.source_file_id: UUID | None` (nullable, без внешнего ключа — файл принадлежит pdf-extract, AD-014). Новые типы: `PdfFileState` (состояние файла во внешнем сервисе), `ProcessingStage`, `ProcessingStatus`, `SourceAttachment` (результат привязки с признаком «новая привязка» — он различает `201` и `200`).
- **estimate (порт и клиент)**: `PdfExtractPort.get_file(company_id, file_id) -> PdfFileState | None`; `HttpPdfExtractClient` (`app/infrastructure/pdf_extract_client.py`) читает `GET /v1/files/{id}` с `X-Company-Id`: `404` -> `None`, сеть/5xx/некорректный ответ -> `PdfExtractUnavailable` (`502`). Ответ с чужим `id` отвергается: в статус сметы не должен попасть чужой файл.
- **estimate (привязка)**: `AttachSource` (`app/application/attach_source.py`) — порядок проверок: (1) сметы нет/чужой -> `404`; (2) привязан тот же файл -> идемпотентный `200` без обращения к pdf-extract; (3) привязан другой файл -> `409` (проверка локального состояния, без вызова вверх); (4) файл не найден в pdf-extract -> `404`; (5) иначе связь записывается -> `201`. Порядок (3) перед (4) выбран сознательно: конфликт определяется состоянием самой сметы и не требует сетевого вызова; когда выполняются оба условия (чужой `file_id` **и** уже есть файл), отдаётся `409` -- это не задано I/O-матрицей, оба кода корректны. Запись в БД условная (`source_file_id IS NULL`): параллельные привязки разных файлов не перезаписывают друг друга, второй получает `409`.
- **estimate (HTTP)**: `POST /v1/estimates/{id}/source` (тело `{file_id}`) -> `201`/`200` `{estimate_id, source_file_id}`; `GET /v1/estimates/{id}/processing-status` -> `200 {estimate_id, source_file_id, stages:[{code, done, at}]}`. Ошибки — `application/problem+json`: `404` (смета/файл вне скоупа), `409` (у сметы уже другой файл), `502` (pdf-extract недоступен), `401` без identity, `422` для невалидного тела. Необработанные исключения тоже отдаются как `application/problem+json` (`500` без деталей) — обработчики `StarletteHTTPException` и `Exception` в `app/transport/problem.py`.
- **estimate (статус)**: `GetProcessingStatus` (`app/application/processing_status.py`) собирает четыре стадии по активной версии: `uploaded` — файл есть в pdf-extract (время — `created_at` файла); `extracted` — есть текстовый артефакт (время — `extracted_at`); `mapped` — у активной версии есть хотя бы одна позиция (время — создание самой ранней позиции); `calculated` — у активной версии заполнен `calculated_at`. Коды стадий — перечисление `ProcessingStageCode` в домене, в схеме ответа он же (в OpenAPI виден как enum). Поле `at` присутствует **всегда**, `null` — стадия не выполнена; это отличается от `pdf-extract`, где необязательные поля ответа опускаются, — два соглашения описаны в README обоих сервисов. Отсутствие файла делает файловые стадии невыполненными, но не ошибкой; удалённый из pdf-extract файл — тоже не ошибка. Сметы без активной версии: файловые стадии — по данным, `mapped`/`calculated` — не выполнены. pdf-extract недоступен -> `502` (`PdfExtractUnavailable`).
- **estimate (репозиторий)**: добавлены `set_source_file` (условная запись), `get_active_version`, `first_item_created_at`; выборки ограничены `company_id` (AD-013). Миграция `0003_add_source_file.py`: колонка `estimate.source_file_id` (nullable) + индекс `ix_estimate_source_file_id`.
- **ui (клиент)**: `api_request` принимает `FormData`: тело уходит как есть, `Content-Type` **не** выставляется (границу multipart-запроса задаёт браузер), JSON-запросы работают как раньше. Добавлены `UPLOAD_STATUS_TEXTS` (`413` — «Файл больше 5 МБ.», `415` — «Загрузите PDF-файл.», `422` — «Не удалось принять PDF-файл.»), `UPLOAD_PROBLEM_TEXTS` (по `type` из problem+json: `pdf-without-text-layer` — «В PDF нет текстового слоя — это скан.», `invalid-pdf` — «PDF повреждён или защищён паролем.»), поля `texts`/`problem_texts` в `RequestOptions` и индивидуальный `timeout_ms` (для загрузки файла — 120 с, общий таймаут 15 с).
- **ui (сценарий)**: действие «Загрузить PDF-смету» — в каждом этапе дерева (`components/TreeNode.vue`); панель этапа с виджетом загрузки и панелью стадий рендерит `views/HierarchyView.ts`. Поток загрузки (`components/EstimateUpload.vue`): проверка размера (≤ 5 МиБ) и типа/расширения PDF на клиенте → `POST /v1/files` (multipart) → `POST /v1/estimates` (смета по имени файла, `contract_id` из узла дерева, `stage_id` из этапа) → `POST /v1/estimates/{id}/source`. Частичный сбой компенсируется: загруженный файл и созданная смета запоминаются, поэтому повтор продолжает с места сбоя (а не создаёт вторую смету и второй файл). Панель стадий (`components/ProcessingStatus.vue`) — проекция ответа `processing-status`; UI стадии не склеивает и ничего не пересчитывает. После загрузки перезапрашивается список смет этапа (новая смета видна в дереве сразу).
- **ui (опрос)**: статус обновляется по кнопке «Обновить» и автоматически раз в 3 с (`STATUS_POLL_INTERVAL_MS`), пока есть незавершённые стадии; число попыток ограничено 30 (`STATUS_POLL_MAX_ATTEMPTS`), после исчерпания показывается подсказка обновить вручную. Таймер опроса снимается при смене этапа/сметы и при размонтировании вида; после размонтирования не планируются следующие запросы (флаг `unmounted`). Поздний ответ по статусу не переписывает панель, если смета уже сменилась. Кнопка «Извлечь текст» вызывает `POST /v1/files/{id}/extract` (через `api-gateway`; операция идемпотентна) и сразу перезапрашивает статус; кнопка показывается, пока у привязанного файла нет извлечённого текста.
- **ui (прочее)**: строка сметы в списке этапа стала кнопкой выбора (панель стадий по этой смете); `EstimateList` и `TreeNode` пробрасывают события `select-stage`/`select-estimate` вверх по рекурсивному дереву. В `localStorage` по-прежнему только access-токен: файл, смета и стадии живут в состоянии вкладки.
- **ui/nginx (лимит тела)**: `client_max_body_size` поднят с `5m` до `6m`. Лимит файла (5 МиБ) проверяет pdf-extract, а multipart-обрамка (граница и заголовки части) добавляет к телу несколько сотен байт: при `5m` файл ровно в 5 МиБ отбрасывался nginx'ом, хотя pdf-extract такой файл принимает. Проверку лимита перенесли в сервис; собственный `413` nginx отдаётся тем же контрактом `application/problem+json` (`error_page 413`), а не HTML-страницей.

## Spec Change Log

- 2026-10-21: реализация без изменения frozen-блока; уточнения контракта на уровне реализации:
  - `GET /v1/files/{id}` в pdf-extract: необязательные ключи `extracted_at`/`page_count` **отсутствуют**, а не равны `null`; `page_count` берётся из метаданных загрузки (`meta.pages`).
  - `POST /v1/estimates/{id}/source`: ответ `{estimate_id, source_file_id}`; `201` — новая привязка, `200` — повтор того же файла.
  - `GET /v1/estimates/{id}/processing-status`: стадии всегда в порядке `uploaded`, `extracted`, `mapped`, `calculated`; `at` — время операции (для `mapped` — создание самой ранней позиции активной версии).
  - Порядок проверок при привязке: конфликт «у сметы уже другой файл» (`409`) проверяется до обращения в pdf-extract; сочетание «чужой `file_id` + уже есть файл» отдаёт `409` (в I/O-матрице не задано).
  - `mapped` определяется по наличию импортированных позиций активной версии (Story 5.3), `calculated` — по `calculated_at` активной версии (Story 6.3); обе стадии не зависят от наличия файла.
  - UI: панель этапа и опрос статуса живут в `views/HierarchyView.ts`, виджет загрузки и панель четырёх стадий — в `components/{EstimateUpload,ProcessingStatus}.vue`; дерево (`components/TreeNode.vue`) только передаёт действия наверх.
  - `api-gateway` не менялся: префиксы `/v1/files` и `/v1/estimates` уже покрывают новые маршруты.
  - `ui/nginx.conf`: `client_max_body_size 5m -> 6m` (multipart-обрамка не должна выталкивать файл ровно в 5 МиБ за лимит nginx) и `413` nginx отдаётся как `application/problem+json`.
- 2026-10-21 (живой стенд, compose): подтверждён полный сценарий через `ui` (nginx) -> `api-gateway` -> `estimate`/`pdf-extract`: загрузка PDF (multipart, `201`), чтение состояния файла, привязка файла (`201`, повтор — `200`), сводный статус до и после извлечения, стадия `mapped` после импорта позиций, `401` без токена.
- 2026-10-21 (по итогам ревью реализации): frozen-блок не менялся; исправлены найденные проблемы:
  - **Гонка привязок**: запись связи условная (`UPDATE ... WHERE source_file_id IS NULL`); при занятой связи — `409`, а не перезапись. Повтор того же файла остаётся идемпотентным `200` и не обращается к pdf-extract (в том числе при его недоступности).
  - **`404` от pdf-extract**: отсутствием файла считается только ответ `application/problem+json`; иначе — `502` (`PdfExtractUnavailable`), чтобы неверный `PDF_EXTRACT_URL` не выглядел как «файлов нет».
  - **Необработанные ошибки `estimate`**: добавлены обработчики `StarletteHTTPException` и `Exception` — ответ всегда `application/problem+json`, `500` без деталей.
  - **Коды стадий** — перечисление `ProcessingStageCode` (домен) и enum в схеме ответа: опечатка в коде больше не проходит незамеченной, в OpenAPI видны допустимые значения.
  - **`source_file_id`** добавлен в `EstimateResponse` и `EstimateSummaryResponse` (nullable).
  - **OpenAPI** `POST /v1/estimates/{id}/source` описывает и `201`, и `200` (повтор того же файла).
  - **Поле `at`** в `processing-status` присутствует всегда (`null` — стадия не выполнена); тип в UI приведён к тому же соглашению (`at: string | null`).
  - **UI**: поздний ответ статуса не переписывает панель после перехода на другую смету; опрос прекращается при размонтировании вида; после загрузки перезапрашивается список смет этапа (новая смета видна сразу); частичный сбой загрузки компенсируется (повтор продолжает с места сбоя, а не создаёт вторую смету и второй файл); размер (≤ 5 МиБ) и тип файла проверяются на клиенте; для загрузки файла — увеличенный таймаут; `422` различается по `type` из problem+json (`pdf-without-text-layer` — скан, `invalid-pdf` — повреждён/зашифрован).
  - **pdf-extract**: `page_count_from_meta` не падает на не-dict метаданных и не принимает неположительное число страниц.

## Review Triage Log

- **2026-10-21 · blind-hunter + edge-case-hunter · привязка файла без защиты от гонки** — verdict: medium. evidence: чтение `source_file_id` и запись шли без блокировки, два параллельных `POST /source` оба получали `201`, связь перезаписывалась. route: patch → условный `UPDATE ... WHERE source_file_id IS NULL` + `rowcount` → 409.
- **2026-10-21 · edge-case-hunter · `404` от неверного `PDF_EXTRACT_URL` трактуется как «файл не найден»** — verdict: medium. evidence: при неверном адресе все файлы выглядели бы отсутствующими, привязка невозможна. route: patch → `404` без problem+json → `PdfExtractUnavailable` (502).
- **2026-10-21 · edge-case-hunter · необработанные исключения возвращают не problem+json** — verdict: medium. evidence: в `estimate` не было catch-all, клиент получал стандартный 500. route: patch → обработчики `StarletteHTTPException` и `Exception` по образцу `pdf-extract`.
- **2026-10-21 · edge-case-hunter · поздний ответ по статусу смета A перезаписывает панель смета B** — verdict: medium. evidence: гонка в UI при переключении смет. route: patch → сверка выбранной сметы перед записью в состояние.
- **2026-10-21 · blind-hunter · опрос статуса продолжается после размонтирования** — verdict: medium. evidence: таймер снимался, но начавшийся запрос снова ставил `setTimeout`. route: patch → флаг размонтирования.
- **2026-10-21 · blind-hunter + edge-case-hunter · после загрузки список смет этапа не перезапрашивается** — verdict: medium. evidence: новая смета появлялась только после перезагрузки страницы. route: patch → сброс кэша списка после загрузки.
- **2026-10-21 · edge-case-hunter + blind-hunter + verification-gap · нет компенсации частичного сбоя загрузки** — verdict: medium. evidence: при падении `POST /source` файл оставался в pdf-extract, а повтор создавал вторую смету и второй файл. route: patch → сохранение id файла и сметы, повтор только упавшего шага.
- **2026-10-21 · blind-hunter · нет клиентской проверки типа и размера файла** — verdict: medium. evidence: 20 МиБ уходили на сервер целиком. route: patch → проверка размера и PDF до загрузки.
- **2026-10-21 · blind-hunter + verification-gap · общий таймаут 15 с применяется к загрузке до 5 МиБ** — verdict: medium. evidence: на медленном канале загрузка обрывалась «сервер не ответил вовремя», хотя файл мог сохраниться. route: patch → отдельный таймаут для загрузки.
- **2026-10-21 · edge-case-hunter · `422` от pdf-extract всегда показывается как «это скан»** — verdict: medium. evidence: pdf-extract различает `pdf-without-text-layer` и `invalid-pdf` по `type`. route: patch → разные тексты по `type`.
- **2026-10-21 · blind-hunter · код стадии — свободная строка** — verdict: medium. evidence: опечатка прошла бы незамеченной и в API, и в UI. route: patch → перечисление кодов в домене и схеме.
- **2026-10-21 · blind-hunter · `source_file_id` не видна в ответах смет** — verdict: low. evidence: по списку смет нельзя было понять, у какой есть PDF. route: patch → поле в `EstimateResponse` и `EstimateSummaryResponse`.
- **2026-10-21 · blind-hunter · OpenAPI у `POST /source` объявляет только `201`** — verdict: low. evidence: повтор того же файла возвращает `200`. route: patch → описаны оба статуса.
- **2026-10-21 · blind-hunter + edge-case-hunter + verification-gap · соглашение по полю `at` различается в `estimate` и `pdf-extract`** — verdict: low. evidence: в `estimate` `at` всегда присутствует, в `pdf-extract` optional-поля опускаются. route: patch → зафиксировано в README, спеке и типе UI.
- **2026-10-21 · blind-hunter + edge-case-hunter · `page_count_from_meta` не проверяет тип и допускает 0** — verdict: medium. evidence: не-dict в `meta` дал бы исключение, `pages == 0` ушёл бы в API. route: patch → строгая проверка + юнит-тесты.
- **2026-10-21 · verification-gap · лимит опроса и прекращение опроса не закреплены тестами** — verdict: medium. evidence: удаление проверки лимита не ломало suite — опрос шёл бы бесконечно. route: patch → тесты на лимит, подсказку и остановку.
- **2026-10-21 · verification-gap · идемпотентный повтор привязки не закреплён как «без обращения к pdf-extract»** — verdict: medium. evidence: при сбое pdf-extract повтор отдавал бы `502` вместо `200`. route: patch → тест с `calls == []` и недоступным pdf-extract.
- **2026-10-21 · verification-gap · времена стадий проверяются только на истинность** — verdict: medium. evidence: подмена `file_state.created_at` на `now()` не ломала тесты. route: patch → точные проверки времени для `uploaded`/`extracted`/`mapped`.
- **2026-10-21 · verification-gap · парсер клиента estimate не проверялся на реальном теле pdf-extract** — verdict: medium. evidence: смена формата времени на стороне producer'а тихо ломала бы статус в рантайме. route: patch → кейс с фактическим телом и фиксация формата у producer'а.
- **2026-10-21 · blind-hunter · README pdf-extract утверждал, что `page_count` появляется только после извлечения** — verdict: low. evidence: `page_count` берётся из `meta.pages` и доступен сразу. route: patch → описание исправлено, добавлен `422` в таблицу ошибок.
- **2026-10-21 · blind-hunter · в таблицах ошибок README не хватало `422`** — verdict: low. evidence: невалидный UUID в пути и невалидное тело `{file_id}` дают `422`. route: patch → строки добавлены.
- **2026-10-21 · blind-hunter · перекомпиляция `epic-7-context.md` потеряла требования для 7.2** — verdict: medium. evidence: пропали порог `confidence < 0.70`, признак `needs_review` и правило «внутренние сервисы — `expose`, без `ports`». route: patch → требования возвращены.
- **2026-10-21 · edge-case-hunter · `413` вне `/v1/` отдаётся HTML** — verdict: low. evidence: `error_page 413` задан только в `location /v1/`. route: reject (тело вне API не бывает: UI-маршруты — GET без тела).
- **2026-10-21 · edge-case-hunter · сессия БД удерживается во время вызова pdf-extract** — verdict: medium. evidence: HTTP-вызов внутри открытой транзакции. route: defer → вынос вызова за пределы сессии — отдельная задача при росте нагрузки.
- **2026-10-21 · blind-hunter · `page_count` отбрасывается клиентом estimate и не доходит до статуса** — verdict: low. evidence: контракт стадий не предусматривает числа страниц. route: reject.
- **2026-10-21 · edge-case-hunter · `at` заявлен необязательным, но приходит всегда** — verdict: low. evidence: расхождение только в формулировке. route: patch (вместе с правкой соглашения по `at`).
- **2026-10-21 · edge-case-hunter · привязка чужого файла к смете с уже привязанным файлом даёт `409`, а не `404`** — verdict: low. evidence: локальное состояние проверяется раньше обращения к pdf-extract; порядок зафиксирован в Implementation Notes. route: reject (зафиксированное решение).
- **2026-10-21 · blind-hunter · diff смешивает репозитории с одинаковыми путями** — verdict: false. evidence: артефакт подготовки ревью, а не изменение продукта; в каждом репозитории пути корректны. route: reject.
- **2026-10-21 · blind-hunter · Review Triage Log пуст при статусе `review`** — verdict: medium. evidence: результат прохода не зафиксирован в артефакте. route: patch → лог заполнен (этот раздел).

- _Пусто: ревью отдельным проходом (bmad-review) не выполнялось в рамках этой реализации._

## Design Notes

- `POST /v1/estimates/{id}/source` с телом `{file_id}` → `201` (или `200` при повторе того же файла); файл проверяется вызовом `pdf-extract`: чужой/неизвестный → `404`, у сметы уже другой файл → `409`.
- `GET /v1/estimates/{id}/processing-status` → `200 {estimate_id, source_file_id, stages:[{code, done, at}]}`; коды стадий: `uploaded`, `extracted`, `mapped`, `calculated` (в домене — перечисление `ProcessingStageCode`, в OpenAPI — enum). Поле `at` присутствует **всегда**: `null` — стадия не выполнена (в отличие от `pdf-extract`, где необязательные поля ответа опускаются).
- Стадии и их признаки: `uploaded` — файл существует в `pdf-extract` (время — `created_at` файла); `extracted` — есть текстовый артефакт (`extracted_at`); `mapped` — у активной версии есть хотя бы одна позиция; `calculated` — у активной версии заполнен `calculated_at`. Отсутствие файла делает стадии файловыми невыполненными, но не ошибкой.
- Извлечение запускается из UI: `POST /v1/files/{id}/extract` через `api-gateway`; операция идемпотентна, поэтому таймаут запроса не опасен — UI показывает статус и даёт повторить.
- Загрузка файла: UI отправляет `multipart/form-data` в `POST /v1/files` через `api-gateway` (проксирование multipart уже проверено в Story 4.1); `Content-Type` с границей выставляет браузер, клиент не задаёт его вручную.
- Ошибки загрузки: `415` (не PDF) → «Загрузите PDF-файл», `413` (> 5 МБ) → «Файл больше 5 МБ», `422` (нет текстового слоя) → «В PDF нет текстового слоя — это скан».
- Опрос статуса: UI обновляет панель по кнопке и автоматически раз в несколько секунд, пока есть незавершённые стадии; число попыток ограничено, после исчерпания показывается подсказка обновить вручную.
- Смета создаётся из UI по имени файла; `contract_id` берётся из узла дерева, `stage_id` — из выбранного этапа.

## Verification

**Commands (выполнено 2026-10-21):**
- `cd pdf-extract && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `81 passed`
- `cd estimate && .venv/bin/ruff check . && .venv/bin/pytest -q` → `All checks passed!`, `225 passed`
- `cd ui && npm ci && npm run typecheck && npm run build && npm run test` → без ошибок, `68 passed` (vitest), сборка `dist/` собрана
- `cd deploy && docker compose config -q` → exit 0

**Живой стенд (compose; UI на 8080 -> nginx -> api-gateway -> estimate/pdf-extract):**
- `POST /v1/auth/register` → токен; `POST /v1/contracts` → `POST /v1/contracts/{id}/stages` → `POST /v1/estimates` — отработали через nginx UI.
- `POST /v1/files` (multipart, PDF с текстовым слоем) → `201 {id, mime_type, size, meta, created_at}`.
- `GET /v1/files/{id}` → `200 {id, mime_type, size, created_at, page_count}`; после извлечения появляется `extracted_at`.
- `POST /v1/estimates/{id}/source` → `201 {estimate_id, source_file_id}`; повтор того же файла → `200` с тем же телом; файл вне компании → `404`; привязка второго файла → `409`.
- `GET /v1/estimates/{id}/processing-status` → `200 {estimate_id, source_file_id, stages:[...]}`: до извлечения `uploaded=true`, `extracted=false`; после `POST /v1/files/{id}/extract` → `extracted=true`; после импорта позиций → `mapped=true`; `calculated` — после расчёта версии.
- Без токена → `401`; без `X-Company-Id` внутри сервиса → `401` (проверено тестами).
- Границы размера файла через nginx: ровно 5 МиБ → `201`; 5 МиБ + 1 байт → `413 application/problem+json` («file exceeds 5 MiB», ответ pdf-extract); 7 МиБ → `413 application/problem+json` (лимит nginx).

**Не покрыто автоматически:** UI не проверялся в реальном браузере (нет e2e-стенда — только vitest/jsdom); миграция `0003` проверена на SQLite в CI-тестах и на PostgreSQL живого стенда (upgrade `0002 -> 0003` в логах контейнера), но отдельного автотеста на Postgres в репозитории нет.
