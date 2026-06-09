# Codebase Structure

**Analysis Date:** 2026-06-09

## Directory Layout

```
author-agent/
├── cinatra/                  # Cinatra platform sidecar directory
│   └── oas.json              # OAS Flow 26.1.0 agent specification (the agent definition)
├── skills/                   # Runtime methodology skills
│   └── agent-authoring/      # Skill bundle for this agent
│       └── SKILL.md          # Authoring methodology delivered to LLM at runtime
├── .github/
│   └── workflows/
│       ├── ci.yml            # Standalone CI: classify, typecheck, test, agent OAS gate
│       └── release.yml       # Release workflow
├── .planning/
│   └── codebase/             # GSD codebase map documents (this directory)
├── extension-kind-gate.mjs   # Self-contained CI gate for agent/workflow kinds (Node builtins only)
├── package.json              # Extension manifest (cinatra metadata, no runtime deps)
├── tsconfig.json             # TypeScript config skeleton (no src/ currently)
├── .npmrc                    # npm/pnpm registry config (do not read — may contain tokens)
├── LICENSE                   # Apache-2.0
└── README.md                 # User-facing description of the author-agent
```

## Directory Purposes

**`cinatra/`:**
- Purpose: Platform sidecar directory — contains the agent's declarative OAS specification
- Contains: `oas.json` (OAS Flow 26.1.0 document defining nodes, edges, inputs, outputs, prompts)
- Key files: `cinatra/oas.json`

**`skills/agent-authoring/`:**
- Purpose: Runtime skill bundle — methodology injected by the Cinatra platform into the LLM context at dispatch
- Contains: `SKILL.md` with YAML frontmatter and Markdown methodology body
- Key files: `skills/agent-authoring/SKILL.md`

**`.github/workflows/`:**
- Purpose: GitHub Actions CI/CD pipelines for this standalone extracted repo
- Contains: `ci.yml` (baseline gate + agent OAS validation), `release.yml` (release automation)
- Key files: `.github/workflows/ci.yml`, `.github/workflows/release.yml`

**`extension-kind-gate.mjs`:**
- Purpose: Zero-dependency, self-contained CI gate; validates `cinatra/oas.json` for banned retired-CRM primitives in LLM-visible fields; also handles workflow kind validation (BPMN sanity check)
- Contains: Exported pure functions: `validateAgent`, `validateWorkflow`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate`, `parseArgs`

## Key File Locations

**Entry Points:**
- `cinatra/oas.json`: The agent definition — all runtime behavior is declared here

**Configuration:**
- `package.json`: Extension manifest; `cinatra` block declares `kind`, `apiVersion`, `packageType`, `riskLevel`, `toolAccess`, `hasApprovalGates`
- `tsconfig.json`: TypeScript compiler config (skeleton; targets `src/` which does not currently exist)
- `.npmrc`: Registry config (existence noted; contents not read)

**Core Logic:**
- `cinatra/oas.json`: Node graph, prompts, data/control flow edges
- `skills/agent-authoring/SKILL.md`: Methodology rules, output contract, node-type decision table
- `extension-kind-gate.mjs`: CI validation logic

**Testing:**
- No test files present. CI runs `pnpm test --if-present` but this repo has no test script. Gate validation in `extension-kind-gate.mjs` serves as the functional correctness check for the OAS artifact.

## Naming Conventions

**Files:**
- `oas.json` — agent OAS specification (fixed name, required by platform)
- `SKILL.md` — skill methodology file (uppercase, fixed name, required by skill catalog)
- `extension-kind-gate.mjs` — lowercase kebab-case, `.mjs` ES module extension
- `package.json`, `tsconfig.json` — standard lowercase JSON config names

**Directories:**
- `cinatra/` — platform-reserved sidecar directory name (fixed by Cinatra convention)
- `skills/<methodology-slug>/` — lowercase kebab-case slug matching the skill's `name` frontmatter field

**Package Naming:**
- Agent packages: `@cinatra-ai/<slug>-agent` (kind suffix at end, enforced by gate)
- Workflow packages: `@cinatra-ai/<slug>-workflow`
- Skill packages: `@cinatra-ai/<slug>-skill`

## Where to Add New Code

**New agent behavior / prompt changes:**
- Edit the `data.system` and `data.user` fields in `cinatra/oas.json` → `$referenced_components.author`
- Do NOT inline methodology prose — put it in `skills/agent-authoring/SKILL.md`

**New or updated methodology rules:**
- Edit `skills/agent-authoring/SKILL.md`
- Keep frontmatter `match_when.agent_id` set to `"@cinatra-ai/author-agent"`

**Additional nodes in the flow:**
- Add entries to `cinatra/oas.json` → `nodes[]`, `$referenced_components`, `control_flow_connections[]`, `data_flow_connections[]`

**TypeScript source code (if added in future):**
- Place under `src/` (configured in `tsconfig.json` as `rootDir`)
- Output will go to `dist/` (`outDir` in `tsconfig.json`)

**Additional skills:**
- Create `skills/<new-slug>/SKILL.md` with correct frontmatter
- Register in the Cinatra skill catalog (external step)

**CI gate updates:**
- Edit `extension-kind-gate.mjs` — the gate is self-contained (Node builtins only)
- The `BANNED_PRIMITIVES` and `BANNED_TYPEHINTS` arrays in `extension-kind-gate.mjs` are the authoritative list of retired primitives to scan for

## Special Directories

**`cinatra/`:**
- Purpose: Platform-reserved sidecar; contains `oas.json` for agents or `workflow.bpmn` for workflows
- Generated: No (hand-authored / pipeline-generated via `agent_source_*` primitives)
- Committed: Yes

**`.planning/codebase/`:**
- Purpose: GSD codebase analysis documents (ARCHITECTURE.md, STRUCTURE.md, etc.)
- Generated: Yes (by GSD map-codebase tooling)
- Committed: Yes (tracked as planning artifacts)

**`dist/`:**
- Purpose: TypeScript compiler output (if `src/` is ever added)
- Generated: Yes
- Committed: No (would be in `.gitignore`)

---

*Structure analysis: 2026-06-09*
