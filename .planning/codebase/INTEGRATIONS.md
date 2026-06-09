# External Integrations

**Analysis Date:** 2026-06-09

## APIs & External Services

**Cinatra LLM Bridge:**
- Cinatra internal LLM routing API — the sole HTTP integration; the `author` ApiNode POSTs to `{{CINATRA_BASE_URL}}/api/llm-bridge` to invoke the language model
  - SDK/Client: raw HTTP via Cinatra ApiNode (no SDK; platform handles request construction)
  - Auth: platform-injected (no explicit auth env var in repo; Cinatra runtime provides credentials)
  - Preferred model: `gpt-5.5` via `preferredProvider: "openai"` (declared in `cinatra/oas.json` under `$referenced_components.author.data.cinatra_llm`)
  - Fallback: model selection is delegated to the Cinatra LLM bridge; the agent does not pin a specific model key

**Cinatra Agent-Creation Pipeline (upstream caller):**
- This agent is dispatched by the Cinatra agent-creation pipeline; it is not a standalone HTTP server
- Inputs received: `agent_run_id` (string), `packageSlug` (string), `spec` (string, required)
- Output emitted: `draft` (string, JSON-encoded wrap-object `{"draft":{...}}`)
- Dispatch contract defined in `cinatra/oas.json`

## Data Storage

**Databases:**
- Not applicable — no database client or connection configured; the agent is stateless

**File Storage:**
- Not applicable — no file storage integration; draft artifacts are passed as strings in the data flow

**Caching:**
- Not detected

## Authentication & Identity

**Auth Provider:**
- Cinatra platform (implicit) — `agent_run_id` is threaded through the flow as a correlation/auth token; the LLM bridge uses it for session tracking
- No external auth provider SDK present

## Monitoring & Observability

**Error Tracking:**
- Not detected — no Sentry, Datadog, or similar SDK

**Logs:**
- Platform-managed — Cinatra runtime captures agent node execution logs; no application-level logging code in this repo

## CI/CD & Deployment

**Hosting:**
- Cinatra cloud platform — agent is published to the Cinatra marketplace and executed within the platform runtime

**CI Pipeline:**
- GitHub Actions (`.github/workflows/ci.yml`) — runs on push/PR to `main`
  - Node.js 24 + corepack/pnpm setup
  - First-party dependency shape validation (enforces no `@cinatra-ai/*` in direct deps)
  - `extension-kind-gate.mjs` — validates `cinatra/oas.json` structure and scans for retired CRM primitives in LLM-visible prompt strings
- GitHub Actions (`.github/workflows/release.yml`) — release automation (contents not fully read; managed by extraction pipeline)

## Environment Configuration

**Required env vars:**
- `CINATRA_BASE_URL` — injected by the Cinatra platform at runtime; interpolated into `cinatra/oas.json` `url` fields

**Secrets location:**
- No secrets stored in repo; all credentials are platform-injected at runtime

## Webhooks & Callbacks

**Incoming:**
- Not applicable — this agent does not expose an HTTP endpoint; it is invoked by the Cinatra platform orchestrator

**Outgoing:**
- `POST {{CINATRA_BASE_URL}}/api/llm-bridge` — the only outbound call, made by the `author` ApiNode in `cinatra/oas.json`

## Skill Integrations

**agent-authoring skill:**
- Runtime methodology delivered to the LLM via the Cinatra skill dispatch system
- Skill definition: `skills/agent-authoring/SKILL.md`
- Registered in Cinatra catalog; referenced by `agent_id: "author-agent"` match rule
- The skill instructs the LLM on OAS Flow 26.1.0 structure, node types, and draft artifact shape

**skill-creator (referenced, not vendored):**
- `@anthropics/skills:skill-creator` — referenced in `skills/agent-authoring/SKILL.md` as the canonical authority for authoring new SKILL.md files; not included in this repo

---

*Integration audit: 2026-06-09*
