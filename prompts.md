**Prompt P1-V — Validation Gate**

```
Run a static analysis and test simulation mentally:
- Ensure imports resolve
- Ensure middleware order is correct (size limiter before app logic)
- Ensure AsyncClient is not duplicated per request when injected as a dependency

OUTPUT
- If issues: emit diffs to fix
- Finally, a "PASS report" with: { "lint": "ok|fix", "types": "ok|fix", "tests_collected": N, "key_assertions": [...] }
(No chain-of-thought)
```

---

# PR-2 — MCP Server Safety & Robustness

**Prompt P2-A — Tool Registry, Schemas, Timeouts, Rate-limit**

```
OBJECTIVE
Harden apps/mcp:
- Move tools to apps/mcp/tools/*.py with explicit allowlist & registry
- Strict Pydantic v2 input/output schemas
- asyncio.timeout() wrapper per tool
- Per-tool + global rate limiting (simple in-memory or Redis if available)
- Log scrubbing for secrets
- Tracing spans per tool

WORK ITEMS
1) New: apps/mcp/tools/registry.py, schemas.py, middleware.py
2) Update: apps/mcp/server.py to use registry & middleware
3) Add health endpoints: tools/list, tools/health

OUTPUT
- Diffs/new files
- JSON checklist with "tools_registered":[...]
```

**Prompt P2-B — Tests for MCP**

```
OBJECTIVE
Add safety & policy tests.

TESTS
- tests/mcp/test_tools_schema_validation.py (fuzz invalid shapes; expect 400)
- tests/mcp/test_tool_timeout.py
- tests/mcp/test_capabilities_allowlist.py
- tests/mcp/test_rate_limit.py

CONSTRAINTS
- Use pytest-asyncio; any external I/O must be faked

OUTPUT
- NEW FILES with full contents
- JSON checklist
```

**Prompt P2-V — Validation Gate**

```
Verify:
- Unknown fields rejected (schema forbid)
- Timeout surfaces structured ToolTimeout
- Rate limit 429 with retry-after
- Secrets never in logs

OUTPUT
- Diffs if fixes needed, then PASS report JSON
```

> Positioning aligns to “put instructions first,” structured outputs, and safety emphasis. ([OpenAI Platform][1])

---

# PR-3 — `packages/r2r` Typed SDK & Resilience

**Prompt P3-A — Client, Models, Errors, Telemetry**

```
OBJECTIVE
Create a typed async client for R2R with retries/backoff, idempotency, and telemetry.

FILES
- packages/r2r/client.py
- packages/r2r/models.py
- packages/r2r/errors.py
- packages/r2r/config.py (env + optional infra/r2r.toml loader)
- packages/r2r/__init__.py

API (examples)
- R2RClient.search(query: str, top_k: int=10) -> SearchResultV1
- R2RClient.index(doc: DocV1, idempotency_key: str|None=None) -> IndexAckV1

CONSTRAINTS
- httpx AsyncClient with shared retry/backoff policy (3 attempts, exp+jitter)
- Error taxonomy: Auth, RateLimited, Unavailable, Timeout, BadRequest
- OTel spans "r2r.request" with attrs: path, status_code, attempt, duration_ms
- 100% mypy-clean in package scope

OUTPUT
- NEW FILES
- JSON checklist
```

**Prompt P3-B — Tests for `packages/r2r`**

```
OBJECTIVE
Add hermetic tests with respx and hypothesis.

TESTS
- tests/packages/test_r2r_client_errors.py  (status→typed errors)
- tests/packages/test_r2r_timeouts_retries.py
- tests/packages/test_r2r_idempotency.py   (server mock dedupes)
- tests/packages/test_r2r_models_compat.py (golden JSON roundtrip)

OUTPUT
- NEW FILES
- JSON checklist
```

**Prompt P3-V — Validation Gate**

```
Check:
- Missing env yields helpful init error
- Idempotency key threads through header
- Retries occur only on retryable errors; no replay on POST without idempotency key

OUTPUT
- Diffs if needed; PASS report JSON
```

---

# PR-4 — Cross-Store Consistency (Transactional Outbox)

**Prompt P4-A — Outbox Core**

```
OBJECTIVE
Implement a lightweight outbox for cross-store side-effects (Qdrant/Neo4j/R2R).

WORK ITEMS
1) DB models: apps/api/app/outbox/models.py  (id, topic, payload JSON, attempt_count, next_attempt_at, inserted_at)
2) Repo: apps/api/app/outbox/repo.py  (enqueue inside same DB tx)
3) Worker: scripts/outbox_worker.py  (at-least-once; exponential backoff; stop after N→DLQ)
4) DLQ: apps/api/app/outbox/dlq.py + table

CONSTRAINTS
- Idempotent handlers; allow idempotency key per payload
- Metrics: backlog size, DLQ count, handler latency (log or OTel)
- Config via env; safe defaults

OUTPUT
- NEW/DIFF files
- JSON checklist
```

**Prompt P4-B — Handlers & Wiring**

```
OBJECTIVE
Add example handlers for:
- Memory persisted → emit Qdrant upsert
- KB link created → emit Neo4j edge
- Document added → emit R2R index

WORK ITEMS
- apps/api/app/outbox/handlers/{qdrant.py,neo4j.py,r2r.py}
- Wire service write paths to enqueue outbox messages in the same tx

OUTPUT
- Diffs/new files + JSON checklist
```

**Prompt P4-C — Tests for Outbox**

```
OBJECTIVE
Hermetic tests.

TESTS
- tests/outbox/test_outbox_happy_path.py
- tests/outbox/test_outbox_dlq.py
- tests/outbox/test_outbox_idempotent_handlers.py

CONSTRAINTS
- Use in-memory DB or test schema; side-effects mocked

OUTPUT
- NEW FILES + JSON checklist
```

**Prompt P4-V — Validation Gate**

```
Verify tx correctness & idempotency; simulate transient vs permanent failures and DLQ paths.
Emit diffs if fixes needed; then PASS report JSON.
```

---

# PR-5 — Test & CI Upgrades

**Prompt P5-A — Test Harness & Coverage Gates**

```
OBJECTIVE
- Install/declare: pytest-asyncio, respx, hypothesis, faker, coverage
- Coverage gates: 85% packages/r2r; 75% API; 70% MCP

WORK ITEMS
- Update pyproject.toml / requirements*
- Add .coveragerc tuned for src dirs
- Add minimal perf smoke: tests/perf/test_parallel_agent_turns.py (50 parallel; asserts p95 budget as constant)

OUTPUT
- Diffs + NEW FILES
- JSON checklist
```

**Prompt P5-B — GitHub Actions**

```
OBJECTIVE
Add CI workflow:
- Lint, type-check, unit tests
- Cache deps
- Save artifacts on fail (logs, coverage.xml)
- Matrix if needed (py versions)

WORK ITEMS
- .github/workflows/ci.yml
- badges in README updated

OUTPUT
- NEW FILE ci.yml + README diff
- JSON checklist
```

**Prompt P5-V — Validation Gate**

```
Mentally simulate CI:
- Steps order, caching keys, timeout budget
- Ensure coverage thresholds enforced

OUTPUT
- Diffs if corrections needed; PASS report JSON
```

---

# 🧪 Final System Validation & Docs

**Prompt FIN-A — Integrity & Reference**

```
OBJECTIVE
- Generate CHANGELOG.md (Keep a Changelog format)
- MIGRATION.md for ops (env changes, new services, how to run worker)
- Update README badges/sections
- Produce a PR summary per PR with risk notes and rollback plan

OUTPUT
- NEW/DIFF files
- JSON checklist
```

**Prompt FIN-B — Self-Review & Patch Hygiene**

```
OBJECTIVE
Run a final self-review:
- No secret leakage
- Consistent naming & imports
- All new modules referenced from __init__ where appropriate
- Test names, marks, and fixtures consistent

OUTPUT
- If issues: diffs to fix
- Finally emit a compact report JSON:
  { "ready_for_review": true, "prs": ["PR-1 ...", "PR-2 ...", ...], "followups_suggested":[...] }
```

---
