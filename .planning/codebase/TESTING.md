# Testing Patterns

**Analysis Date:** 2026-06-09

## Overview

This is a content-only Cinatra extension repo (no TypeScript source files under `src/`). There are no test files in this repository. The CI pipeline explicitly skips standalone tests for repos with host-internal `@cinatra-ai/*` peer dependencies — tests for this extension are owned and run by the cinatra monorepo.

The repo does ship `extension-kind-gate.mjs` which exports pure validation functions designed to be testable in isolation; the monorepo is expected to cover these with its own test suite (referenced as `oas-skill-free.test.ts` in `skills/agent-authoring/SKILL.md`).

## Test Framework

**Runner:**
- Not applicable — no test runner configured in this repo
- `package.json` has no `scripts.test` entry
- CI runs `corepack pnpm test --if-present` which exits 0 (no-op) for this repo

**Assertion Library:**
- Not applicable

**Run Commands:**
```bash
# No standalone test commands — tests run in the cinatra monorepo
corepack pnpm test --if-present   # exits 0 (no test script present)
```

## Test File Organization

**Location:**
- No test files present in this repo (`*.test.*`, `*.spec.*` not found)
- Monorepo test coverage expected at `packages/*/src/**/*.test.ts` (external)

**Naming:**
- Not applicable for this repo
- Monorepo convention inferred from skill reference: `oas-skill-free.test.ts`

## What Is Validated (CI Gates)

Although there are no unit tests, the CI pipeline runs two kinds of validation:

**1. Dependency shape check** (`.github/workflows/ci.yml`, `Classify repo` step):
- Inline `node -e` script validates `package.json` structure
- Fails exit 2 if any `@cinatra-ai/*` package appears in `dependencies`/`devDependencies`/`optionalDependencies`
- Fails if any first-party peer is not marked `peerDependenciesMeta.optional: true`

**2. Agent OAS validation gate** (`extension-kind-gate.mjs`):
- Runs as `node extension-kind-gate.mjs --package-root .` in the `kind-gates` CI job
- Parses `cinatra/oas.json` and scans all LLM-visible fields (`system`, `user`, `description`) for banned primitives
- Pure functions with `string[]` return type — suitable for unit testing

## Testable Pure Functions (in `extension-kind-gate.mjs`)

All exported functions are pure (string input → string[] errors output) and can be unit-tested without any filesystem setup except where noted:

| Function | Input | Tests should cover |
|----------|-------|-------------------|
| `parseArgs(argv)` | `string[]` | `--package-root` flag parsing, missing value error |
| `validateAgent(packageRoot)` | path string | missing OAS (pass), malformed JSON (error), banned primitives (error), clean OAS (pass) |
| `validateWorkflowPackageShape(pkg)` | object | wrong kind, wrong name pattern, inline workflow, non-integer workflowVersion, unexpected keys |
| `validateBpmnSanity(xml)` | string | empty string, malformed tags, missing BPMN namespace, missing `definitions` root, missing `process`, valid BPMN |
| `findWorkflowSidecars(packageRoot)` | path string | single sidecar, duplicate sidecars, nested directories |
| `validateWorkflow(packageRoot)` | path string | missing `package.json`, missing `workflow.bpmn`, duplicate sidecars, combined shape + BPMN errors |
| `runGate(packageRoot)` | path string | `kind:"agent"` dispatch, `kind:"workflow"` dispatch, unknown kind (pass) |

## Mocking

**Framework:** Not applicable (no test suite in repo)

**For future monorepo tests of `extension-kind-gate.mjs`:**
- Filesystem functions (`readFileSync`, `existsSync`, `readdirSync`) should be mocked to avoid real I/O
- The self-contained, zero-dependency design makes the module easy to import and test directly
- `validateBpmnSanity` and `validateWorkflowPackageShape` are fully pure — no mocking needed

## Fixtures and Factories

**Test Data:**
- Not applicable — no tests present

**Suggested fixture approach for monorepo tests:**
```js
// Inline minimal valid OAS for agent gate tests
const VALID_OAS = JSON.stringify({
  agentspec_version: "26.1.0",
  component_type: "Flow",
  // ... minimal fields with no banned primitives
});

// Inline minimal valid BPMN for workflow gate tests
const VALID_BPMN = `<?xml version="1.0"?>
<bpmn:definitions xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL">
  <bpmn:process id="proc" />
</bpmn:definitions>`;
```

## Coverage

**Requirements:** Not enforced in this repo

**Monorepo coverage:** Determined by monorepo CI configuration (not accessible from this repo)

## Test Types

**Unit Tests:**
- Not present; pure validation functions in `extension-kind-gate.mjs` are designed for unit testing

**Integration Tests:**
- Not present; the full OAS runtime-invariant validation runs marketplace-side at publish/install (per CI comments)

**E2E Tests:**
- Not applicable

## CI Test Execution

**Source mirror repos (this repo):**
- CI detects `@cinatra-ai/*` optional peers → skips standalone install, typecheck, and test
- Monorepo runs all tests in its own workspace

**Standalone repos (0 first-party deps):**
- CI runs `corepack pnpm install --no-frozen-lockfile`
- Runs typecheck via `tsc --noEmit`
- Runs `corepack pnpm test --if-present`

The `kind-gates` job always runs regardless of source-mirror status:
```bash
node extension-kind-gate.mjs --package-root .
```

## Banned Primitive Scan (Behavioral Contract)

The gate enforces that these strings do NOT appear in `system`, `user`, or `description` OAS fields:
- `lists_list`, `lists_get`, `lists_create`, etc. (retired CRM primitives)
- `@cinatra-ai/entity-accounts:account`, `@cinatra-ai/entity-contacts:contact` (legacy typeHints)
- `objects_list` combined with a CRM entity typeHint within 120 characters

Word-boundary matching is used (`(?<![A-Za-z0-9_])<token>(?![A-Za-z0-9_])`) to avoid false positives on substrings.

---

*Testing analysis: 2026-06-09*
