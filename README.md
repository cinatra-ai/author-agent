# Author Agent

Draft a brand-new Cinatra agent extension from a single natural-language description of what you want it to do. Describe the agent — its purpose, its inputs, its outputs, and what it should ask for human approval — and the Author Agent produces a typed draft artifact ready for the downstream agent-creation pipeline to validate, commit, and publish.

## Works with

- Cinatra agent-creation pipeline (dispatches the draft to `agent_source_*` primitives)
- Any Cinatra workspace via the `/api/llm-bridge` endpoint

## Capabilities

- Draft a new Cinatra agent extension (`kind: agent`) from a free-form description
- Accept `packageSlug` and `spec` as inputs; emit a `draft` wrap-object for the pipeline
- Choose the right OAS Flow node shape (ApiNode, AgentNode, FlowNode, InputMessageNode)
- Lay out inputs, outputs, and human-in-the-loop approval gates per your spec
- Optionally draft companion `SKILL.md` methodology files for the new agent
- Install: add `@cinatra-ai/author-agent` as a dependency in your Cinatra workspace manifest
- Configure: no credentials required; the agent runs through the platform LLM bridge
- Troubleshoot: if the draft is rejected, check that `packageSlug` follows `<slug>-agent` naming and that `spec` clearly states inputs, outputs, and any HITL requirements
