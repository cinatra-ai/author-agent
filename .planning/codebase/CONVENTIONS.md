# Coding Conventions

**Analysis Date:** 2026-06-09

## Overview

This is a content-only Cinatra extension repo. It ships no TypeScript source files — its primary artifacts are `cinatra/oas.json` (the agent flow definition), `skills/agent-authoring/SKILL.md` (runtime methodology), and `extension-kind-gate.mjs` (a self-contained CI validation script). Conventions below reflect what is enforced and observable across these files.

## Naming Patterns

**Files:**
- OAS flow definition: `cinatra/oas.json` (fixed path, enforced by CI gate)
- Skill files: `skills/<methodology-slug>/SKILL.md` (kebab-case slug, uppercase filename)
- Gate script: `extension-kind-gate.mjs` (kebab-case, `.mjs` ES module extension)
- Config: `tsconfig.json`, `package.json`, `.npmrc` (standard lowercase)

**Package names:**
- Must follow `@<scope>/<slug>-<kind>` convention, e.g. `@cinatra-ai/author-agent`
- Kind is always at the end: `-agent`, `-skill`, `-connector`, `-artifact`, `-workflow`
- This is enforced in `extension-kind-gate.mjs` via `WORKFLOW_PACKAGE_NAME_RE`

**OAS node IDs:**
- Lowercase, short identifiers: `"start"`, `"author"`, `"end"`
- DataFlowEdge names: `<source>_to_<destination>_<field>`, e.g. `"start_to_author_packageSlug"`
- ControlFlowEdge names: `<source>_to_<destination>`, e.g. `"start_to_author"`

**Functions (in `extension-kind-gate.mjs`):**
- camelCase: `parseArgs`, `walkLlmStrings`, `scanOasString`, `validateAgent`, `validateWorkflow`, `runGate`, `findWorkflowSidecars`, `validateBpmnSanity`, `validateWorkflowPackageShape`
- Predicates and helpers use descriptive verb prefixes: `validate*`, `find*`, `scan*`, `walk*`

**Variables:**
- camelCase throughout; constants use UPPER_SNAKE_CASE for sets/arrays: `LLM_VISIBLE_FIELDS`, `BANNED_PRIMITIVES`, `BANNED_TYPEHINTS`, `PRIMITIVE_PATTERNS`, `BPMN_MODEL_NS`, `WORKFLOW_PACKAGE_NAME_RE`

## Code Style

**Formatting:**
- No formatter config file detected (no `.prettierrc`, no `biome.json`, no `eslint.config.*`)
- Code in `extension-kind-gate.mjs` uses 2-space indentation consistently
- Double quotes for strings in JS; template literals for interpolation

**Linting:**
- Not detected — no `.eslintrc*` or equivalent

**Module system:**
- ES modules only: `"type": "module"` in `package.json`
- `extension-kind-gate.mjs` uses named ESM exports and Node built-ins only (`node:fs`, `node:path`)
- `verbatimModuleSyntax: true` in `tsconfig.json` enforces explicit `import type` for type-only imports

**TypeScript config (for any future TS sources):**
- `strict: true`, `noImplicitAny: false` (allows implicit `any` as a relaxation)
- `target: ES2023`, `module: ESNext`, `moduleResolution: bundler`
- `isolatedModules: true` — each file must be independently compilable
- Output: `dist/`, source: `src/`

## Import Organization

**`extension-kind-gate.mjs` pattern:**
- Node built-ins listed together at top: `node:fs`, `node:path`
- No third-party imports (zero-dependency by design — required for standalone CI)
- No path aliases (not applicable to a `.mjs` standalone script)

## Error Handling

**Pattern in `extension-kind-gate.mjs`:**
- Functions return `string[]` errors arrays (pure, no throws for expected validation failures)
- `try/catch` around I/O operations (file reads), with error message extracted via `err instanceof Error ? err.message : String(err)`
- Early returns from validation functions when a prerequisite check fails (avoids cascading false errors)
- `process.exit(1)` for gate violations; `process.exit(0)` for success
- `process.exit(2)` used distinctly in CI shell scripts to signal dependency-shape regressions (separate from gate failures)

**Example pattern:**
```js
try {
  parsed = JSON.parse(readFileSync(oasPath, "utf8"));
} catch (err) {
  errors.push(`cinatra/oas.json failed to parse: ${err instanceof Error ? err.message : String(err)}`);
  return errors;
}
```

## Logging

**Framework:** `console.log` / `console.error` (no logging library)

**Patterns:**
- Success messages to `console.log` with a `✓` prefix
- Error/violation messages to `console.error` with a `✗` prefix and bullet `•` per item
- CI steps use `echo` shell commands for status; `::error::` GitHub Actions annotation for failures

## Comments

**When to Comment:**
- File-level block comments explain scope, design rationale, and constraints (e.g. why zero-dependency, what the gate does and does not check)
- Section dividers with `// ---` separators group logical sections in `.mjs` files
- Inline comments explain non-obvious logic (regex construction, namespace resolution approach)
- Shell scripts in CI YAML include extensive inline comments on branching logic

**JSDoc/TSDoc:**
- JSDoc used on exported functions in `extension-kind-gate.mjs` with `/** ... */` block style
- Documents purpose, purity, and return type in prose rather than `@param`/`@returns` tags

## Module Design

**Exports (`extension-kind-gate.mjs`):**
- Named exports for all validation functions: `parseArgs`, `validateAgent`, `validateWorkflow`, `validateBpmnSanity`, `validateWorkflowPackageShape`, `findWorkflowSidecars`, `runGate`
- `main()` is NOT exported — it's invoked only when the file is run directly (checked via `process.argv[1]` / `import.meta.url` comparison)

**Content files:**
- `cinatra/oas.json` — not a module; declarative JSON consumed by the Cinatra runtime
- `skills/agent-authoring/SKILL.md` — YAML frontmatter + Markdown prose; loaded by the agent-authoring lane at dispatch

## OAS Authoring Conventions

**Required fields (agent kind):**
- `agentspec_version: "26.1.0"`, `component_type: "Flow"`, `id`, `name`, `description`
- `metadata.cinatra: { type, packageName, packageVersion }`
- `inputs[]`, `outputs[]`, `start_node`, `nodes[]`, `control_flow_connections[]`, `data_flow_connections[]`, `$referenced_components`

**Forbidden in OAS LLM-visible fields** (`system`, `user`, `description`):
- Retired CRM primitives listed in `BANNED_PRIMITIVES` in `extension-kind-gate.mjs`
- Legacy entity typeHints (`@cinatra-ai/entity-accounts:account`, `@cinatra-ai/entity-contacts:contact`)
- Skills MUST NOT be inlined into `oas.json` (enforced by `oas-skill-free.test.ts` in monorepo)

**Wrap-object contract:**
- Agent output MUST be `{"draft":{…}}` — DataFlowEdge extracts the `draft` key

## Package.json Conventions

**`cinatra` block required fields:**
- `apiVersion: "cinatra.ai/v1"`, `kind`, `packageType`, `manifestVersion`, `type: "node"`, `riskLevel`, `hasApprovalGates`, `toolAccess`, `sourceTemplateId`, `sourceVersionId`, `sourceVersionNumber`

**Dependency rules (enforced by CI):**
- Host-internal `@cinatra-ai/*` packages MUST be `peerDependencies` only, marked `peerDependenciesMeta.optional: true`
- NEVER in `dependencies`, `devDependencies`, or `optionalDependencies`

---

*Convention analysis: 2026-06-09*
