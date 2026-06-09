<!-- refreshed: 2026-06-09 -->
# Architecture

**Analysis Date:** 2026-06-09

## System Overview

```text
┌─────────────────────────────────────────────────────────────┐
│              Agent-Creation Pipeline (caller)               │
│  dispatches with { agent_run_id, packageSlug, spec }        │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTP / agent_run dispatch
                           ▼
┌─────────────────────────────────────────────────────────────┐
│               OAS Flow 26.1.0 (cinatra/oas.json)            │
│                                                              │
│  StartNode "start"  →  ApiNode "author"  →  EndNode "end"   │
│  `cinatra/oas.json`                                          │
└──────────────────────────┬──────────────────────────────────┘
                           │ POST {{CINATRA_BASE_URL}}/api/llm-bridge
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              Cinatra LLM Bridge (external)                   │
│  preferredProvider: openai  preferredModel: gpt-5.5          │
│  System prompt + agent-authoring skill injected at runtime   │
└──────────────────────────┬──────────────────────────────────┘
                           │ returns { "draft": { … } }
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  Downstream pipeline: agent_source_* primitives              │
│  (validate + commit the draft artifact)                      │
└─────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| OAS Flow definition | Declares the agent's inputs, outputs, node graph, and data/control edges | `cinatra/oas.json` |
| StartNode "start" | Accepts `agent_run_id`, `packageSlug`, `spec`; surfaces only `spec` to the user | `cinatra/oas.json` (`$referenced_components.start`) |
| ApiNode "author" | POSTs to `/api/llm-bridge` with system + user prompt; emits `draft` wrap-object | `cinatra/oas.json` (`$referenced_components.author`) |
| EndNode "end" | Passes `draft` string output to the pipeline caller | `cinatra/oas.json` (`$referenced_components.end`) |
| agent-authoring skill | Runtime methodology: OAS shape rules, node-type selection, sub-agent decomposition, skill authoring guidance | `skills/agent-authoring/SKILL.md` |
| extension-kind-gate | Self-contained CI gate: validates `cinatra/oas.json` parses and contains no banned retired-CRM primitive tokens in LLM-visible fields | `extension-kind-gate.mjs` |
| CI pipeline | Classifies repo (source-mirror vs standalone), runs typecheck/test, runs agent OAS gate | `.github/workflows/ci.yml` |

## Pattern Overview

**Overall:** Content-only Cinatra agent extension (no `src/` TypeScript sources). The agent is defined entirely as a declarative OAS Flow 26.1.0 JSON specification with a companion runtime skill delivered via a SKILL.md file. Execution logic lives in the Cinatra platform's LLM bridge; this package only declares the agent's interface and prompt.

**Key Characteristics:**
- Single-node flow: `start → author (ApiNode) → end`
- The ApiNode issues one HTTP POST to `{{CINATRA_BASE_URL}}/api/llm-bridge` — all LLM reasoning happens there
- The agent emits a **wrap-object** `{"draft":{…}}` which a downstream `DataFlowEdge` extracts into the `draft` output
- Skills (methodology) are delivered at runtime by the Cinatra platform — never inlined into `oas.json`
- No TypeScript sources; `tsconfig.json` is present as a skeleton for if `src/` is added in future

## Layers

**OAS Flow Layer:**
- Purpose: Declares the agent's public interface (inputs/outputs), graph topology, data/control edges, and the ApiNode's prompt template
- Location: `cinatra/oas.json`
- Contains: StartNode, ApiNode, EndNode definitions; data flow and control flow edge lists; `$referenced_components`
- Depends on: Cinatra OAS Flow 26.1.0 schema (validated marketplace-side at publish)
- Used by: Cinatra agent-creation pipeline, Cinatra marketplace

**Skill / Methodology Layer:**
- Purpose: Provides the runtime methodology injected into the LLM context (how to draft OAS flows, node selection rules, output contract, sub-agent decomposition)
- Location: `skills/agent-authoring/SKILL.md`
- Contains: Frontmatter (`name`, `description`, `match_when`), methodology prose, `draft` JSON shape spec, node-type decision table, what NOT to do
- Depends on: Cinatra skill catalog (`match_when.agent_id: "@cinatra-ai/author-agent"`)
- Used by: Cinatra platform at runtime to populate the LLM system context

**CI Gate Layer:**
- Purpose: Pre-publish sanity checks run in standalone GitHub Actions CI without the @cinatra-ai monorepo
- Location: `extension-kind-gate.mjs`, `.github/workflows/ci.yml`, `.github/workflows/release.yml`
- Contains: `validateAgent`, `validateWorkflow`, `validateBpmnSanity`, `runGate` — pure functions, zero external deps, Node builtins only
- Depends on: Node.js ≥24 (no npm/pnpm install needed for the gate itself)
- Used by: `.github/workflows/ci.yml` job `kind-gates` → `node extension-kind-gate.mjs --package-root .`

## Data Flow

### Primary Request Path

1. Pipeline caller dispatches `{ agent_run_id, packageSlug, spec }` into the OAS Flow's `StartNode` (`cinatra/oas.json` `$referenced_components.start`)
2. `ControlFlowEdge` `start_to_author` advances control; three `DataFlowEdge`s thread all three inputs into the `ApiNode "author"` (`cinatra/oas.json` `$referenced_components.author`)
3. ApiNode POSTs to `{{CINATRA_BASE_URL}}/api/llm-bridge` with the system prompt (agent-authoring methodology injected by platform) and a user prompt interpolating `{{ packageSlug }}` and `{{ spec }}`
4. LLM bridge returns `{ "draft": { package, oas, skills } }` (wrap-object)
5. DataFlowEdge `author_to_end_draft` extracts `draft` and threads it into `EndNode "end"` (`cinatra/oas.json` `$referenced_components.end`)
6. Downstream `agent_source_*` primitives receive the `draft` artifact for validation and file commits

### CI Gate Path

1. GitHub Actions triggers on push/PR to `main` (`.github/workflows/ci.yml`)
2. `build` job classifies the repo: detects `@cinatra-ai/*` optional peer deps → marks as source mirror, skips standalone install/typecheck/test
3. `kind-gates` job (after `build`) detects `cinatra.kind: "agent"` → runs `node extension-kind-gate.mjs --package-root .`
4. Gate parses `cinatra/oas.json`, walks all LLM-visible string fields (`system`, `user`, `description`), fails on any banned retired-CRM primitive token

**State Management:**
- The agent is stateless. No persistent storage. `agent_run_id` is threaded through for tracing purposes only.

## Key Abstractions

**OAS Flow 26.1.0:**
- Purpose: The declarative agent specification format; defines all nodes, edges, inputs, and outputs
- Examples: `cinatra/oas.json`
- Pattern: JSON document with `$referenced_components` map, `control_flow_connections[]`, `data_flow_connections[]`

**Wrap-object contract (`{"draft":{…}}`):**
- Purpose: The required output envelope; a `DataFlowEdge` in the caller's flow extracts the named key (`draft`)
- Examples: Defined in `skills/agent-authoring/SKILL.md` (output schema section)
- Pattern: LLM MUST emit exactly `{"draft":{…}}` — no other keys at the top level

**SKILL.md runtime methodology:**
- Purpose: Provides skill-indexed methodology delivered to the LLM at runtime via the Cinatra skill catalog
- Examples: `skills/agent-authoring/SKILL.md`
- Pattern: YAML frontmatter (`name`, `description`, `match_when`) + Markdown methodology body

**extension-kind-gate (exported functions):**
- Purpose: Reusable, pure validation functions for agent/workflow extension CI
- Examples: `extension-kind-gate.mjs` exports `validateAgent`, `validateWorkflow`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate`, `parseArgs`
- Pattern: Pure functions (string[] errors), no side effects, no external deps

## Entry Points

**OAS Flow (runtime):**
- Location: `cinatra/oas.json`
- Triggers: Cinatra platform dispatches an `agent_run` with `{ agent_run_id, packageSlug, spec }`
- Responsibilities: Routes inputs through StartNode → ApiNode → EndNode, calls the LLM bridge, surfaces `draft` output

**CI Gate (CI/CD):**
- Location: `extension-kind-gate.mjs` (exported `main()` invoked when file is run directly)
- Triggers: `node extension-kind-gate.mjs --package-root .` in `.github/workflows/ci.yml`
- Responsibilities: Validates `cinatra/oas.json` parses and contains no banned primitive tokens; exits 0 (pass) or 1 (violations)

## Architectural Constraints

- **Threading:** Not applicable — no runtime process; execution occurs inside the Cinatra platform
- **Global state:** None. `extension-kind-gate.mjs` is entirely pure (no module-level mutable state)
- **Circular imports:** Not applicable — no TypeScript sources; `extension-kind-gate.mjs` imports only Node builtins
- **Source-mirror constraint:** This repo declares NO `@cinatra-ai/*` packages in `dependencies`/`devDependencies`. Any host-internal first-party dep must be an optional `peerDependency` — CI enforces this and skips standalone install/typecheck/test when host-internal peers are present
- **Skill inlining forbidden:** Skill IDs and methodology prose must never appear inline in `oas.json` (enforced by the `oas-skill-free` gate referenced in `SKILL.md`)
- **No direct filesystem writes:** The agent emits only a `draft` artifact; `agent_source_*` primitives in the pipeline own all file writes

## Anti-Patterns

### Inlining skill methodology in oas.json

**What happens:** Embedding methodology text or skill IDs directly in the `system` or `user` fields of `cinatra/oas.json`
**Why it's wrong:** The Cinatra platform injects skills at runtime via the skill catalog; inlining creates drift, duplication, and violates the `oas-skill-free` gate
**Do this instead:** Place methodology in `skills/<slug>/SKILL.md` with proper frontmatter; the platform delivers it at dispatch time

### Emitting output without the wrap-object envelope

**What happens:** LLM returns `{…}` (flat draft fields) instead of `{"draft":{…}}`
**Why it's wrong:** The downstream `DataFlowEdge` extracts the named key `draft`; a missing envelope causes extraction failure
**Do this instead:** Always emit `{"draft":{…}}` as specified in `skills/agent-authoring/SKILL.md`

## Error Handling

**Strategy:** CI gate uses pure-function error accumulation (returns `string[]`); `main()` prints violations and exits 1.

**Patterns:**
- `validateAgent` / `validateWorkflow` accumulate error strings, return early on fatal parse failures
- OAS runtime errors are handled by the Cinatra platform (not this package)

## Cross-Cutting Concerns

**Logging:** CI gate logs to `console.log` (pass) / `console.error` (violations) only
**Validation:** Two levels — (1) lightweight CI gate (`extension-kind-gate.mjs`) for banned primitives; (2) authoritative marketplace-side OAS runtime-invariant validation at publish/install
**Authentication:** Not applicable at this layer — `{{CINATRA_BASE_URL}}` and LLM bridge auth are resolved by the Cinatra platform at runtime

---

*Architecture analysis: 2026-06-09*
