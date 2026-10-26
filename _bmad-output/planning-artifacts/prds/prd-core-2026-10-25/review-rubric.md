# PRD Quality Review — smeta1

## Overall verdict

This is a substantive PRD with a real thesis — «LLM извлекает спецификацию и никогда не считает стоимость» — and unusually concrete, testable consequences for most FRs. It is at risk in two places: the MVP has no UI path to the `confirmed`/`corrected` state that the total calculation requires, and the PRD/addendum disagree on the matching endpoint contract. Both would bite story creation and acceptance testing immediately; the unmeasured primary metrics are a known, honestly declared risk rather than a hidden one.

## Decision-readiness — thin

> Историческая оценка PRD до реализации Stories 8.2–8.8. Актуальный статус находок — в разделе «Resolution (2026-10-26)».

The PRD usually states decisions rather than considerations, and it names what was given up: addendum §6 rejects cloud-only LLM, LLM-based mapping, a separate queue broker, per-company catalogs, and versioned catalogs with reasons. The access decision is unusually candid — §8 «Доступ» says «серверной проверки `admin` нет (решение 2026-10-25)» and R-6 records the residual risk instead of smoothing it. That is the right register.

What breaks decision-readiness is the confirmation loop. §4.6 says «Полный экран валидации — 1.1, но API и базовый просмотр есть в MVP», while §4.8 says «UI не пересчитывает суммы и не подтверждает позиции», and FR-009 says «В итог входят только `confirmed`/`corrected` позиции». UJ-2's resolution nonetheless has «Анна подтверждает или исправляет спорные позиции». A decision-maker cannot tell whether the MVP can produce a non-empty total through the product or only through hand-issued API calls. The second problem is open-question hygiene: Q3 is answered by the body, and Q2 collides with a commitment already made in §8.

### Findings

- **[critical]** No MVP path to a calculable estimate (§4.6, §4.8 FR-011, FR-009, UJ-2 Resolution) — The UI «не подтверждает позиции», the validation screen is deferred to 1.1, and the total includes only `confirmed`/`corrected`; yet UJ-2 says Anna confirms positions and UJ-3 assumes «активная версия с подтверждёнными позициями». As written, the shipped UI can match and display but not produce the product's headline number. *Fix:* decide and state one of two contracts — either add a minimal per-row confirm/correct control to the MVP estimate screen (moving it out of the 1.1 non-goal), or make `proposed` matched positions eligible for calculation with a visible "unconfirmed" marker, and update FR-009/UJ-2/UJ-3 accordingly.
- **[medium]** Q3 is not actually open (§11 Q3, FR-006) — «Нужен ли fallback на зону по умолчанию, если для периода нет цен?» is already answered by FR-006 «Ненайденная позиция получает `material_id=null`, `confidence=0`» and by §6.1 «за точный период/зону». Keeping it open invites a downstream reader to build a fallback the PRD forbids. *Fix:* close Q3 with the FR-006 rule and move the remaining question (whether a default-zone policy is desirable in 1.1) to a deferred item.
- **[medium]** Q2 contradicts an already-made commitment (§11 Q2 vs §8, FR-012/FR-013) — The PRD asks «Нужен ли per-company флаг согласия на облако при глобальном `LLM_ALLOW_CLOUD`?» while §8 already fixes the model as a global flag plus per-call `X-Allow-Cloud`, and FR-013 has no per-company consent field. An open question that the body has effectively answered in the opposite direction is a decision disguised as a question. *Fix:* either resolve Q2 in the body (global flag + per-call header is the MVP answer) or add the per-company consent field to FR-012/FR-013 and §6.1 so the question has real scope.

## Substance over theater — adequate

This is largely earned content. The Vision is product-specific and cannot be transplanted: the trust boundary in §1 («LLM извлекает спецификацию и никогда не считает стоимость»), the 20–50 estimates/month scale, and the local Docker Compose constraint are all this product's. The two personas are not furniture by count — Anna carries UJ-1–UJ-3 and Igor carries UJ-4 — and the JTBD list is operational rather than aspirational. Constraints in §8 carry thresholds (8 GB, CPU-only, q2–q3, 0.70) instead of adjectives.

One persona is weaker than its framing suggests: Igor is introduced as «старший сметчик, отвечает за актуальность цен и модель извлечения», but UJ-4's own entry state says the capability is available to every authenticated user and §4.1 repeats «доступны всем авторизованным». His accountability does not translate into a permission, a handoff, or a decision anywhere in the PRD.

### Findings

- **[medium]** Igor's role is narrative only (UJ-4, §4.1, §2.2) — UJ-4 is titled around a responsible senior estimator, but the entry state immediately removes the distinction: «доступ к справочнику и настройкам LLM есть у всех авторизованных», and §5 Non-Goals explicitly rejects role switching. The persona implies ownership the system does not model, so downstream UX cannot tell whether to surface catalog/LLM settings to Anna too. *Fix:* either drop the ownership framing and make UJ-4 a plain operator journey, or state the operational convention (Igor is the team's designated operator; the system does not enforce it) and use that convention in the UI copy/onboarding.

## Strategic coherence — adequate

The PRD has a thesis, and the features follow it: extraction by LLM, mapping by embeddings with `confidence`, arithmetic by frozen snapshots. The arc is coherent, the MVP scope kind is problem-solving for a known internal workflow, and counter-metrics exist and are pointed at the right temptations — SM-C1 «не должна расти за счёт отказа от ручной проверки», SM-C2 «не за счёт игнорирования периода/зоны», SM-C3 «не ценой скрытия деградации». That is stronger than most PRDs at this stage.

The metrics, however, do not yet validate the thesis. SM-1 is explicitly scoped «без учёта ручной проверки» — which is precisely the human-in-the-loop half of the value claim «сократить часы рутины до минут». SM-2/SM-3 are declared but unmeasured, and there is no instrumentation or review trigger beyond «при появлении данных».

### Findings

- **[high]** The end-to-end value claim is unmeasured (§7 SM-1, §1 Vision) — SM-1 times «от загрузки PDF до рассчитанной сметы» at 30–60 s but excludes manual review, so the metric can pass while the estimator's actual workday does not improve. The product's differentiating claim is that review is bounded and visible (`needs_review` ≤ 15%), yet no metric covers time-to-trusted-total including review. *Fix:* add an SM for end-to-end time-to-approved-estimate (including review) or, at minimum, for reviewer minutes per 100 positions; keep SM-1 as the pipeline sub-metric.
- **[medium]** Unmeasured primary metrics have no collection plan or review trigger (§7 SM-2, SM-3; R-2) — «реальный корпус отсутствует, метрика не измерена» is honest, but nothing says who captures the first corpus, what "coverage" is computed against, or what event forces recalibration of 0.70. Downstream planning will treat 95% and 15% as commitments. *Fix:* name the first-corpus source and the triggering event (e.g., first 10 real estimates), and label SM-2/SM-3 as provisional targets until then.
- **[low]** SM-7's «без деградации» is not operationalized (§7 SM-7) — The 20–50/month and 100–500/estimate volumes are concrete, but «без деградации» has no threshold (latency, error rate, timeout rate), so the metric cannot fail. *Fix:* attach the SM-1 latency bound and an error/timeout ceiling to SM-7.

## Done-ness clarity — adequate

Most FRs carry genuinely testable consequences: HTTP status codes, exact rounding (`ROUND_HALF_UP`), snapshot fields, idempotency semantics, `409`/`404`/`502` distinctions. This is the PRD's strongest mechanical trait and story creation can work from it. The gaps cluster in the async extraction contract and in the UI criteria.

The most serious is an internal contradiction about mutability: FR-005 lets «`POST /v1/estimates/{id}/specification` заменяет позиции активной версии и сбрасывает расчёт», while FR-009/FR-010 freeze a calculated version and require a new version for changes («Повторный расчёт — `409`; новая цена требует новой версии сметы»). The PRD does not say whether specification replacement on a calculated active version is allowed, `409`, or silently creates a version.

### Findings

- **[high]** Extraction job state machine is undefined (§4.3 FR-005, FR-014) — The status endpoint is named but its states, terminal failure, timeout, and retry behavior are not. FR-014's stage list is `uploaded`, `extracted`, `mapped`, `calculated` — no `queued`, `running`, or `failed` — while UJ-4's edge case assumes model errors are surfaced as text. A UI and an API client cannot be built against an unspecified enum, and acceptance tests cannot assert failure behavior. *Fix:* enumerate job states and transitions, the failure/timeout contract, and how a failed job appears in `processing-status`.
- **[high]** Specification replacement vs. frozen calculation is unresolved (§4.3 FR-005 vs FR-009/FR-010) — «заменяет позиции активной версии и сбрасывает расчёт» collides with «рассчитанная версия заморожена». Whether re-importing a specification on a calculated active version resets it (and thus unfreezes history) is the exact behavior that determines version immutability. *Fix:* state the rule explicitly — e.g., replacement allowed only on a non-calculated active version, `409` otherwise, new version required after calculation — and add the consequence to FR-005.
- **[high]** SM-2's "coverage" has no measurement basis (§7 SM-2, FR-004) — «Покрытие извлечения спецификации ≥ 95%» does not say whether coverage is measured in characters, line items, or fields, nor against what ground truth; FR-004 adds «низкое покрытие не блокирует шаг», so the number is neither computed nor enforced. The metric cannot be evaluated as written. *Fix:* define the numerator/denominator and the ground-truth source, and say which artifact records the value.
- **[medium]** UI acceptance criteria are adjectives (§4.8 FR-011) — «Низкоуверенные позиции визуально выделены», «пустые состояния различаются», «Суммы и `confidence` отображаются как пришли от API» — only the last is checkable. "Visually highlighted" and "distinct empty states" admit arbitrary implementations and unverifiable e2e assertions (SM-6 depends on this screen). *Fix:* name the highlight affordance and the empty-state cases (no positions, no matches, unavailable service), and make each an e2e assertion.
- **[medium]** Active-model activation has no in-flight rule (§4.9 FR-013, UJ-4) — «Изменения публикуются в `llm-proxy`» and UJ-4's climax «модель применяется к новым задачам извлечения» do not define what happens to queued or running `spec_job`s, nor what the caller observes on failed publication. *Fix:* state whether activation applies only to jobs created after the change, and add a consequence for publication failure (error surfaced, previous model retained).
- **[low]** «понятным типом ошибки» is untestable (§4.2 FR-003) — The status code `422` is concrete, but the error type is not named. *Fix:* name the `application/problem+json` `type`/`code` value for "no text layer".
- **[low]** «длинные таймауты» has no bound (§10 R-7, FR-014) — R-7 mitigates degradation with «длинные таймауты» without a number, and FR-014's degradation flags have no latency budget. *Fix:* state the timeout values and the maximum time before `rates_unavailable`/`source_unavailable` is shown.

## Scope honesty — adequate

Scope is more honest than average: §5 has eight real non-goals, §2.2 names non-users, §6.2 records deferred work, and §12 indexes assumptions. De-scoping is stated rather than silent, and accepted risks are admitted in the metrics and risk list.

The weakness is that the omissions are aggregated at section level instead of attached to the claims they qualify. There are nine Assumptions Index entries and zero inline `[ASSUMPTION: …]` tags and zero `[NOTE FOR PM]` callouts, so a downstream reader pulling FR-009 alone cannot see that "only confirmed positions count" is an inference, not a confirmed decision. Several operational questions are also simply absent, which reads as "not needed" rather than "not decided".

### Findings

- **[medium]** Material omissions are not declared as non-goals (§5, §6) — Account recovery/password reset, currency and VAT/tax treatment, concurrent editing of one estimate, and catalog record deletion are absent from both scope lists. For an internal tool some may be genuinely out of scope, but a reader will assume auth recovery exists because FR-001 describes full registration/login. *Fix:* add explicit non-goals or MVP decisions for account recovery, currency/VAT, concurrent edits, and catalog deletion.
- **[medium]** No `[NOTE FOR PM]` at the PRD's real tensions (§4.1/§8 access, §6.2 threshold calibration, §11 Q2) — The rubric's tension markers are unused even where the PRD itself records a contested decision (open catalog/LLM settings, uncalibrated 0.70, cloud consent model). The tensions live in §10 R-6 and §11, detached from the features they affect. *Fix:* place a note at §4.1 access, FR-006/FR-007 threshold, and FR-012/FR-013 consent model pointing to the risk/open question.
- **[medium]** The "one user type" assumption is contradicted by the PRD (§12 §2.1, §4.1, UJ-4) — The index says «один тип пользователя, сметчик», but §4.1 defines roles `estimator` and `admin` and UJ-4 gives Igor a distinct responsibility. The assumption is not a harmless summary; it erases the role model that story creation must implement. *Fix:* replace it with the actual assumption (a single operational role for MVP; `admin` exists only for bootstrap) and align §2.1/§4.1 wording.

## Downstream usability — adequate

The PRD is a chain-top artifact (addendum §7 points to epics/stories and §3 to the architecture spine), so extractability matters. IDs are all present — FR-001…FR-016, UJ-1…UJ-4, SM-1…SM-7 plus three counter-metrics, R-1…R-7 — and every FR names the UJs it serves. The Glossary covers the core nouns, and each FR is written to be readable alone.

Three things will trip UX/architecture extraction: a contract that differs between PRD and addendum, a hierarchy noun that does not exist, and an unresolved materials-vs-works distinction.

### Findings

- **[high]** Matching endpoint contract differs between PRD and addendum (§4.5 FR-006 vs addendum §3) — The PRD defines `POST /v1/estimates/{id}/versions/{vid}/match-rates` with estimate-scoped period and ownership checks; the addendum defines «`POST /v1/rate-catalog/matches` — ближайшая цена за точный регион/год/квартал/зону». These imply different owners of the period, different tenancy enforcement, and different clients. Architecture and stories cannot be sourced from both. *Fix:* choose the canonical endpoint, put it in the PRD, and make the addendum reference it rather than restate a different path.
- **[medium]** «Проект» appears as an entity with no definition (UJ-1, §3 Glossary) — UJ-1's button is «Загрузить PDF проекта», but the hierarchy is «Компания → Договор → Этап → Сметы» and the Glossary has no Project. A UX designer will either invent a Project screen or mislabel the estimate. *Fix:* replace with «Загрузить PDF сметы/этапа» and state which entity owns the uploaded file, or add Project to the hierarchy and glossary.
- **[medium]** Materials vs. works is unresolved in the catalog model (Glossary «Справочник», FR-011) — The Glossary defines the catalog as «набор материалов и цен», while FR-011 displays «материал/работу» and the matching sections speak only of materials and `material_id`/`material_price` (addendum §3). Whether work items are matched against the same catalog, or whether "работа" is only a display label, determines the data model and FR-006 consequences. *Fix:* state the catalog's item types explicitly and use one term for a catalog row across Glossary, FR-006, FR-008, FR-011, and addendum §3.

## Shape fit — strong

The shape matches the product. It is an internal tool with a small operator group, so capability-spec treatment is right, and the PRD avoids the trap of over-formalized UJs while still using four journeys where they carry real sequence and failure behavior (UJ-1 edge cases, UJ-2 unavailable catalog, UJ-3 uncovered positions, UJ-4 partial import). The two named protagonists have inline context, and UJ-2/UJ-3 correctly reuse Anna with a changed entry state rather than inventing new personas. Because the PRD feeds UX → architecture → stories, the addition of a Glossary, explicit non-goals, and counter-metrics is proportionate rather than decorative. The heavy technical material that would have distorted the shape (topology, DB-per-service, HS256 headers, HNSW) is correctly pushed to the addendum.

## Mechanical notes

- **Assumption Index roundtrip is broken** — All nine §12 entries are section references, and the body contains no inline `[ASSUMPTION: …]` tags at all. The index is therefore a second, unlinked summary; a reader of FR-009 or §4.1 cannot see what was inferred. *Fix:* add inline tags at the nine points (or reduce the index to a statement that these are section-level inferences and link each entry by heading anchor).
- **FR IDs are complete but out of sequence** — FR-001…FR-016 all exist, yet §4.4 opens with FR-008 then FR-016, §4.5 uses FR-006/FR-007, §4.9 lists FR-013 before FR-012. Story creation usually mirrors document order, so the narrative of the build will not match the ID order. *Fix:* renumber by section order (or reorder sections to the FR sequence) before epic breakdown.
- **Glossary drift and gaps** — «расценка» and «цена» alternate without a rule (Glossary «Снапшот расценки — значения цены»; FR-016 «управление ценами»; FR-009 `rate_amount_snapshot`), and §4.4's domain terms «КСР», «сплит-форма ФСБЦ», «ГРАНД/АВК» are used but never defined, although they determine SM-2's corpus and the importer's variants. *Fix:* define one canonical term for the catalog price and add КСР/ФСБЦ/ГРАНД/АВК to the Glossary.

## Resolution (2026-10-26)

- **Critical — no UI path to calculable estimate:** resolved Stories 8.2–8.8. UI has per-row чекбокс «Верно» (`confirm`/`reject`), «Исправить» с модальным выбором материала, «Нет в справочнике», логин автора исправления и пагинацию; corrected positions входят в расчёт.
- **High — end-to-end value metric:** добавлен SM-9 (время до проверенного итога, цель после пилота).
- **High — extraction state machine:** FR-005 описывает `queued/running/succeeded/failed` и фазы.
- **High — spec replacement vs frozen calculation:** FR-005 разрешает замену только для нерассчитанной активной версии, иначе `409`.
- **High — coverage measurement basis:** SM-2 определяет coverage по позициям против эталонной разметки; остаётся accepted risk без корпуса.
- **High — matching endpoint divergence:** PRD фиксирует внешний endpoint `estimate .../match-rates`; addendum — внутренний `rate-catalog /matches`.
- **Medium — Q2/Q3:** закрыты в PRD (глобальный cloud-флаг + per-call consent; fallback на зону не вводится).
- **Medium — Igor ownership:** зафиксировано как операционная роль, не enforced.
- **Medium — omissions:** добавлены non-goals для восстановления аккаунта, валют/НДС, concurrent editing, удаления справочника.
- **Medium — UI adjectives:** FR-011 называет конкретные действия/статусы, включая пустые состояния и ручную проверку.
- **Medium — model activation in-flight:** FR-013 фиксирует, что активация действует на новые задачи, ошибка публикации сохраняет прежнюю модель.
- **Medium — materials vs works:** PRD больше не заявляет тип позиции/работы, которого нет в API; справочник — материалы/цены.
- **Low — error type/timeout:** FR-003 называет problem `type`; R-7 — таймауты 120/600/900 с.
- **Deferred:** UI diff, top-K альтернативы, улучшение matching (атрибуты/rerank/abstention) — вне MVP.
