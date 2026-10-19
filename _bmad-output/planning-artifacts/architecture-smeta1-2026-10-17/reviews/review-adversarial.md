# Review: adversarial two-units attempt

- Verdict: **pass with findings** — 1 high, 2 medium, 2 low.
- Reviewer: configured finalize lens #2, run inline (subagents unavailable).

Method: construct pairs of units one level down that each obey **every** AD literally, then see whether they still build incompatibly.

## A-1 (high) — "where does company_id live?" (pair: company-org vs estimate)
Unit A reads AD-013 literally ("company_id required on all domain rows") and denormalizes the key onto `stage` and `stage_subdivision`. Unit B reads the Data Model, sees no `company_id` on those rows, derives scope through `contract` and cannot filter "in the repository" without a join. Both are defensible; their repositories and APIs disagree on where tenancy is enforced.
**Fix applied:** AD-013 states where the key physically lives and how positions derive scope; Data Model lists `company_id` on `contract`, `estimate`, `stage`, `stage_subdivision`, `rate_catalog`.

## A-2 (medium) — forged identity from inside the network (pair: estimate vs calc-service)
AD-010 stops forgery at the edge only. Unit A trusts identity headers blindly (compliant); unit B adds an internal signature check (also compliant, but now incompatible with A's callers). A compromised or buggy in-network process can read another company's data under A.
**Fix applied:** the trust boundary is now an explicit convention (internal network, mTLS Deferred), so implementations agree on the guard.

## A-3 (medium) — live catalog reference vs snapshot-only (pair: estimate vs rate-catalog)
`estimate_item.rate_catalog_id?` is nullable and unqualified. Unit A keeps it as a live reference and joins to the current catalog; unit B stores only snapshots. Both satisfy AD-002's letter if the field's role is unstated.
**Fix applied:** AD-002 Rule now says `rate_catalog_id` is provenance only and never participates in calculation.

## A-4 (low) — retry duplicates companies (pair: auth-users vs company-org)
AD-012 requires compensation but not idempotency; a retried create-company call yields two companies.
**Fix applied:** AD-012 Rule requires an idempotency key.

## A-5 (low) — stage ordering and name uniqueness undecided
Two units could order stages by name or by an explicit position, and disagree on duplicate names.
**Fix applied:** added to Deferred with a revisit condition (story 2.3).

## Considered and dismissed
- `role` has a single MVP value; no incompatible pair found.
- Date/id/error conventions are fixed, so no drift there.
