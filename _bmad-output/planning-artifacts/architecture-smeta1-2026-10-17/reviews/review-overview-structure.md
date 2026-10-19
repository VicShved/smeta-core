# Review: ARCHITECTURE-OVERVIEW.md — structure lens

- Verdict: **pass with findings** — 3 medium, 2 low.
- Reviewer: `doc_standards` lens `structure` (bmad-review), run inline (subagents unavailable).

## Findings
- **S-1 (medium)** Render note for the Mermaid C4 diagrams sat *after* the container diagram, though it applies to both. → moved above the first diagram.
- **S-2 (medium)** Section 6 (запуск локально) implied the whole compose stack runs today; only `llm-gateway` exists. → reworded to say so plainly.
- **S-3 (medium)** Section 5 bullets referenced AD IDs inconsistently (some did, some didn't). → every bullet now cites its AD.
- **S-4 (low)** Glossary missing the terms the doc leans on (spine, AD). → added.
- **S-5 (low)** No pointer back to the spine at the point where a reader would want rulings. → section 5 opens with the pointer.

## Checked and clean
- Section order follows the reader's path: what → context → containers → ownership → decisions → run → roadmap → glossary.
- Every diagram has a purpose; no empty or decorative diagrams.
- The ownership table matches the Services section of the spine.
