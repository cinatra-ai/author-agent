# Technology Stack

**Analysis Date:** 2026-06-09

## Languages

**Primary:**
- JavaScript (ESM) — runtime CI gate script `extension-kind-gate.mjs`
- JSON — agent flow definition `cinatra/oas.json`, package manifest `package.json`

**Secondary:**
- TypeScript (ES2023 target) — configured via `tsconfig.json` for a `src/` directory; no `src/` files are committed in this extracted repo (the monorepo owns compilation)
- Markdown — skill methodology `skills/agent-authoring/SKILL.md`

## Runtime

**Environment:**
- Node.js 24 (pinned in `.github/workflows/ci.yml` via `actions/setup-node@v4` with `node-version: "24"`)

**Package Manager:**
- pnpm (corepack-managed; `corepack enable` step in CI)
- `auto-install-peers=false` set in `.npmrc`
- No lockfile committed (extracted source-mirror repo; monorepo workspace provides lockfile)

## Frameworks

**Core:**
- Cinatra OAS Flow 26.1.0 — the agent flow DSL; the agent is defined entirely in `cinatra/oas.json` as a three-node Flow (StartNode → ApiNode → EndNode)

**Testing:**
- Not applicable — no test files present in this extracted repo; testing runs in the monorepo workspace

**Build/Dev:**
- `tsconfig.json` — TypeScript compiler config targeting `src/` → `dist/`; `noEmit: false`, outputs declaration maps and source maps
- `extension-kind-gate.mjs` — zero-dependency Node.js CI sanity gate (no build tool required; runs as plain `node`)

## Key Dependencies

**Critical:**
- No runtime `dependencies` declared in `package.json`; this is an agent extension package with no bundled code
- All Cinatra platform packages (`@cinatra-ai/*`, `@cinatra/*`) are declared as optional `peerDependencies` only (enforced by CI gate)

**Infrastructure:**
- GitHub Actions — CI and release automation (`.github/workflows/ci.yml`, `.github/workflows/release.yml`)
- Cinatra LLM Bridge — the agent's `author` node calls `{{CINATRA_BASE_URL}}/api/llm-bridge` at runtime; preferred model is `gpt-5.5` via `preferredProvider: "openai"` (declared in `cinatra/oas.json`)

## Configuration

**Environment:**
- `CINATRA_BASE_URL` — runtime env var interpolated by the Cinatra platform into `cinatra/oas.json` `url` fields; never stored in repo
- No `.env` file present

**Build:**
- `tsconfig.json` — strict TypeScript; `moduleResolution: "bundler"`, `verbatimModuleSyntax: true`, `jsx: "react-jsx"`, outputs to `dist/`
- `.npmrc` — `auto-install-peers=false`

## Platform Requirements

**Development:**
- Node.js 24+
- pnpm (via corepack)
- Cinatra monorepo workspace for full install/typecheck/test (this extracted repo has no standalone installable deps)

**Production:**
- Cinatra platform runtime (cloud-hosted); the agent runs as a Cinatra Flow node, not as a standalone Node.js server
- `riskLevel: "low"`, `hasApprovalGates: true`, `toolAccess: []` per `package.json` cinatra metadata

---

*Stack analysis: 2026-06-09*
