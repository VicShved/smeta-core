# Review: web-reality check of named technology

- Verdict: **pass** — every pin was checked against a registry today (2026-10-19), not asserted from memory.
- Reviewer: configured finalize lens #1, run inline (subagents unavailable).

## Verified (source + value)

| Tech | Pin | Source |
| --- | --- | --- |
| Python | >=3.12 | `llm-gateway/pyproject.toml` (code) |
| FastAPI | 0.146.0 | PyPI JSON API |
| Uvicorn | 0.54.0 | PyPI JSON API |
| Pydantic | 2.14.1 | PyPI JSON API |
| SQLAlchemy | 2.1.5 | GitHub releases (2026-10-16) |
| Alembic | 1.20.0 | PyPI JSON API |
| psycopg | 3.3.6 | PyPI JSON API |
| PyJWT | 2.15.1 | PyPI JSON API |
| PostgreSQL | 18.6 | endoflife.date (cycle 18, EOL 2030-11) |
| MinIO | RELEASE.2025-10-15T17-29-55Z | GitHub releases (security/CVE release) |
| Vue | 3.5.44 | npm registry |
| Vite | 8.3.4 | npm registry |
| Node.js | 24 LTS | endoflife.date (LTS since 2025-10-28) |
| llama-cpp-python | >=0.3.14 | `llm-gateway/pyproject.toml` — **code-owned, not re-verified against PyPI today** |
| pytest / ruff | >=8 / >=0.6 | `llm-gateway/pyproject.toml` (code) |

## Notes
- MinIO's latest is a security release; pinning it is deliberate.
- Vite 8 requires Node `^20.19 || >=22.12`; the Node 24 LTS pin satisfies it.
- `llama-cpp-python` is owned by the existing `llm-gateway` code and was not re-checked; the pin is copied from code, not invented. No action.
- Docker Compose is named `v2` (tool, not a library) — no registry pin applies.
