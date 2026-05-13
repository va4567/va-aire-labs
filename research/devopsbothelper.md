# DevOps Helper Bot — Findings and Priorities

> PoC review: Slack-native RAG assistant. Go binary, Anthropic Claude (tool-use), Gemini embeddings, Qdrant. Single-pod, Socket Mode.

Severity: **Critical** / **High** / **Medium**  
Dimensions: Business Impact · Security · Impl. Effort

---
 
## What's Already Working Well
 
The bot is delivering real value today. These are solid engineering decisions worth preserving.
 
**Graceful degradation chain** — multi-layer fallback (`embedder nil → Claude-only`, both searches empty → Claude-only) means users always get a response. No hard failures exposed to Slack. This is the right default for an internal tool.
 
**RAG dual-search** — parallel query against `solutions` (vector) and `docs` (DocSearcher) with threshold filtering (`DOCS_RAG_THRESHOLD 0.70`) gives meaningful signal separation between KB types. Running both in parallel is a good latency choice.
 
**Replace semantics via `import_id`** — the docs flow does a `DeleteByPayload(import_id)` before writing, so every repo push results in a clean, deduplicated chunk set. No stale docs accumulate from CI-driven imports.
 
**Section-aware chunking** — the Chunker splits on `##` / `###` / `####` without size-merging or splitting, keeping prose and fenced code blocks as coherent units. This directly improves retrieval precision compared to naive character-based chunking.
 
**Compressor producing structured JSON** — Claude compresses Slack threads into `title / problem / solution / tags` before embedding. Structured payloads make filtering, search, and future tooling significantly easier than raw thread text.
 
**Knowledge Capture UX** — the "Save solution" button appears only on Claude-only answers (no RAG hit), which is exactly when a new solution is worth capturing. The `/save` command as a manual fallback covers edge cases cleanly.
 
**Tool loop with local handlers** — custom tools (`get_gitlab_job_error`, `get_argocd_app`, `search_knowledge`, etc.) give Claude real infrastructure context without exposing credentials to the model directly. The tool boundary is the right abstraction here.
 
**Multi-turn context via thread history** — using Slack thread replies as conversation history enables coherent follow-up questions inside a thread. This is non-trivial to implement correctly in Socket Mode and it works.
 
**Single binary deployment** — one Go binary with embedded system prompt (`//go:embed`) keeps the operational surface minimal. Easy to version, easy to roll back, no runtime config drift.
 
**`EnsureCollection` idempotent on startup** — Qdrant collections are created idempotently at boot. Safe to restart without manual provisioning steps, which matters for a pod-based deployment.
 
---

## Security

### `DOCS_IMPORT_TOKEN` exposure via GitLab CI env
- **Business Impact:** Critical | **Security:** Critical | **Impl. Effort:** Low
- Token is passed as an HTTP Bearer header from the CI pipeline. Requires a rotation policy, short TTL, and injection via Vault or Google Secret Manager. No mention of token scoping or revocation path.

### `DOCS_ALLOWED_USERS` — no authz model shown
- **Business Impact:** High | **Security:** Critical | **Impl. Effort:** Medium
- `/doc add` is guarded only by a flat user list. No RBAC, no audit log of who added what. A single allowlist in env is a bypass risk if the env leaks.

### `GEMINI_API_KEY` / SA key via base64 in env
- **Business Impact:** High | **Security:** Critical | **Impl. Effort:** Medium
- Passing an SA key as a base64 env var is a known secret sprawl pattern. In GKE this should be WIF (Workload Identity Federation) only. SA key = hard rotation, no audit trail.

### No mention of Slack token scoping or signing secret validation
- **Business Impact:** High | **Security:** High | **Impl. Effort:** Low
- Socket Mode bypasses HTTP, but the bot token still needs minimal scopes documented. No mention of request signing validation for HTTP endpoints (`/api/docs/import`).

---

## Reliability / Production Readiness

### Single pod = SPOF
- **Business Impact:** Critical | **Security:** Low | **Impl. Effort:** Medium
- One Go binary, one pod, one persistent WebSocket. Pod restart = full downtime. No mention of HPA, PDB, or reconnect/backoff strategy for Socket Mode.

### Qdrant — no HA, no persistence config visible
- **Business Impact:** Critical | **Security:** Low | **Impl. Effort:** Medium
- Qdrant runs as a single node with no documented snapshots, backup schedule, or recovery RTO. Knowledge base loss on pod eviction is a critical business risk.

### `CLAUDE_MAX_TOOL_ITERATIONS` cap without timeout
- **Business Impact:** High | **Security:** Low | **Impl. Effort:** Low
- Default of 5 iterations is reasonable, but there is no per-request wall-clock timeout. A hanging tool call (GitLab, ArgoCD) results in a hung Slack thread with no fallback message timer.

### No rate limiting on `/api/docs/import`
- **Business Impact:** High | **Security:** High | **Impl. Effort:** Low
- The HTTP endpoint accepts bulk imports from any CI pipeline holding the token. No per-`import_id` rate limit, no payload size cap. One bad pipeline push can flood Qdrant.

---

## Observability Gap

### No metrics / tracing layer mentioned
- **Business Impact:** Critical | **Security:** Low | **Impl. Effort:** Medium
- No mention of Prometheus metrics, OTEL spans, or Grafana dashboards. Cannot answer: what is the p95 RAG latency? How often does the Claude-only fallback trigger? What is the embed error rate?

### No alerting on Graceful Degradation triggers
- **Business Impact:** High | **Security:** Low | **Impl. Effort:** Low
- The fallback chain (`embedder nil → Claude-only`) is clean but silent. Ops needs an alert when the Claude-only rate spikes — it signals a broken RAG pipeline that may go unnoticed for hours.

### Knowledge base quality — no feedback loop
- **Business Impact:** High | **Security:** Low | **Impl. Effort:** Medium
- No mechanism to mark a saved solution as incorrect, outdated, or duplicate. RAG scores are not exposed to users. Stale solutions will silently degrade answer quality over time.

---

## Architecture / Implementation

### `/doc add` creates unbounded separate records
- **Business Impact:** High | **Security:** Low | **Impl. Effort:** Medium
- Without `import_id`, each `/doc add` call adds a new independent point. No deduplication, no replace semantics. The knowledge base will drift with orphaned stale records over time.

### Compressor (Claude) as a synchronous bottleneck on Save
- **Business Impact:** High | **Security:** Low | **Impl. Effort:** Medium
- Knowledge Capture calls Claude for compression before upsert. If the Claude API is slow or unavailable, the save flow blocks inside the goroutine. Requires an async queue or circuit breaker.

### `ALLOWED_CHANNELS` as sole input gate
- **Business Impact:** Medium | **Security:** Medium | **Impl. Effort:** Low
- Channel allowlist is a good start but insufficient at scale. No per-user throttle, no abuse detection (bulk mentions, spam). One noisy user can exhaust the Claude API quota and impact all teams.

### 3072-dim embeddings — no migration path documented
- **Business Impact:** Medium | **Security:** Low | **Impl. Effort:** High
- `gemini-embedding-001` at 3072 dims is locked in at collection creation. Switching models requires a full re-embed and a new collection. No versioning strategy or blue/green collection swap is described.

---

## Summary: blockers before production

| Category | Blockers |
|---|---|
| Security | Token rotation, WIF instead of SA key, audit log for `/doc add` |
| Reliability | Single-pod SPOF, Qdrant backup / HA |
| Observability | Minimum: fallback-rate counter + alert on RAG degradation |
| Architecture | `/doc add` deduplication, async Save flow |