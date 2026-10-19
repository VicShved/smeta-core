# Review: rubric walker (good-spine checklist)

- Verdict: **pass with findings** — 1 high, 2 medium, 1 low.
- Reviewer: rubric walker (good-spine checklist), run inline (subagents unavailable in this environment).

## Findings

### R-1 (high) — company scoping of child rows is under-specified
AD-013 declares `company_id` required on *all* domain rows, yet the Data Model omits it for `stage`, `stage_subdivision`, `estimate` (they reach a company only through `contract`). Two units obeying the AD to the letter build incompatibly: one denormalizes `company_id` onto children, the other derives it through the parent and cannot filter "in the repository" as the Rule demands.
**Fix applied:** AD-013 now names where the key physically lives (aggregate roots + directly-addressable child rows) and how positions are scoped; the Data Model lists `company_id` on `contract`, `estimate`, `stage`, `stage_subdivision`, `rate_catalog`.

### R-2 (medium) — trust boundary of the internal network was implicit
AD-010 proves identity only at the edge. The checklist's operational/environmental envelope wants the trust assumption stated, not assumed: any process inside the compose network can mint `X-User-Id`. Two units could disagree on whether to re-validate headers.
**Fix applied:** added the convention "internal network is the trust boundary" and a Deferred entry for mTLS.

### R-3 (medium) — API contract versioning absent from conventions
AD-009 promises "versioned contracts" but no convention says how a service versions its API, so two services pick different schemes (`/v1` vs header vs none).
**Fix applied:** convention row — contracts versioned in the path (`/v1/...`).

### R-4 (low) — AD-012 retry semantics unspecified
`auth-users → company-org` create-company call with a retry could duplicate companies; compensation is mentioned, idempotency is not.
**Fix applied:** AD-012 Rule now requires an idempotency key.

## Checked and clean
- AD IDs ascending, unique, none renumbered (AD-001…AD-015).
- Every AD has Binds/Prevents/Rule; each Rule is enforceable.
- Deferred items cannot let two units diverge (schema details are per-service; broker choice carries a revisit condition).
- Brownfield ratification: `llm-gateway` matches its description; the AD-011 exception is explicit.
- No spec drove this run; the Capability → Architecture Map covers Epic 1–2.
- Dimensions: paradigm, boundaries, data, auth, tenancy, storage, deployment (MVP compose; k8s Deferred), operations (logs + deferred metrics/backups) — all decided or deferred.
