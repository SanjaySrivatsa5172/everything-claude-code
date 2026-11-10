---
name: agent-fold-or-spawn
description: Decision framework for when to fold a subagent into the main agent loop vs. keep it separate. Apply before any multi-agent architecture decision. Based on Anthropic's "Building Agents: When to Fold Subagents" guidance.
origin: ECC
tools: Read, Grep, Glob
---

# Agent Fold-or-Spawn

Use this skill before spawning any new subagent, before proposing a multi-agent architecture, and when reviewing an existing agent fleet for consolidation candidates.

## When to Activate

- You are about to add a new subagent to handle a subtask
- You are reviewing whether an existing orchestrator-subagent chain is justified
- You suspect context loss across agent handoffs
- Agent chains are slowing response time measurably
- Tool count in a single system prompt exceeds 30

## The Default: Single Loop First

A single agent loop with progressive disclosure beats a multi-agent setup in the majority of real cases. Before proposing a split:

1. Confirm the single loop is actually degraded — not just theoretically large.
2. Load skills on demand (progressive disclosure) and use ToolSearch with deferred loading rather than splitting context.
3. Batch parallel tool calls within the single turn — the model issues multiple tool calls per turn; parallel execution does not require separate agents.

A context window that looks big is not evidence of context bloat. Evidence is measurable quality loss: wrong tool selection, instructions contradicting each other, or eval scores declining as the prompt grows.

## Fold-or-Spawn Decision Tree

Answer the following four questions in order. Stop at the first "spawn" answer — but only after confirming via evals.

```
1. Does shared context improve answer quality?
   YES → Fold. The model benefits from seeing everything together.
   NO  → Continue to Q2.

2. Can the tasks run truly in parallel AND return compact, independent results?
   NO  → Fold. Sequential chains should stay in one loop.
   YES → Continue to Q3.

3. Does the system prompt contain conflicting instructions that cannot be resolved?
   NO  → Fold. Restructure the prompt; don't spawn to escape it.
   YES → Continue to Q4.

4. Is context bloat measurably degrading eval scores?
   NO  → Fold. Keep iterating on the single loop.
   YES → Spawn is justified. Proceed to Spawn Rules.
```

## Spawn Rules (when justified)

**Decompose by context, not problem type.**
Each subagent must need little from the rest of the system and return a compact, schema-conforming result. Splitting by domain (accessibility, database, TypeScript) creates a telephone-game degradation at every handoff.

Good candidates for separation:
- Independent research paths with no shared data needs
- Black-box verification (generator-verifier pattern)
- Parallel work where each unit has a clean interface and the result is discardable

Bad candidates:
- Agents that chain on each other's output (tightly coupled → fold)
- Agents that touch the same files or data
- Agents that are called together more than 80% of the time

**Use the subagent-as-tool pattern.**
The main agent keeps the conversation and calls subagents through a shared response schema. Full handoff (where the subagent talks directly to the user) is only justified when the subagent must conduct multi-turn dialogue with the user directly.

**Cap tool definitions at 30–50 per system prompt.**
Tool selection accuracy degrades past this threshold. Group tools by user intent rather than one tool per API endpoint. Per-user permissions belong in the tools themselves (returning plain refusals the model can read), not in the system prompt.

## Fold Candidates in This Repo

These pairs are commonly chained and are candidates for folding into a single loop:

| Pattern | Why fold | Status |
|---------|----------|--------|
| `code-explorer` → `code-architect` | Share the same codebase read; architect needs explorer context | Fold candidate |
| `planner` → `architect` | Plan and architecture touch the same requirements | Fold candidate |
| `code-reviewer` → `silent-failure-hunter` | Both operate on the same diff | Fold candidate |
| `pr-test-analyzer` → `code-reviewer` | Both operate on the same PR | Fold candidate |
| Language `*-reviewer` chain | Multiple reviewers on same file degrade via handoff | Prefer single reviewer per language |
| `gan-planner` → `gan-generator` → `gan-evaluator` | Generator-verifier: **valid** multi-agent pattern; keep separate | Justified |

## Eval-First Protocol

Before folding or spawning:

1. Define 10 golden examples with expected outputs.
2. Grade final answers against a rubric (not intermediate steps).
3. If using an orchestrator, track routing accuracy.
4. Record timestamps — production runs become future evals.
5. Compare scores before and after the fold/spawn change. Change one variable at a time.
6. Look at which tasks changed, not just the aggregate score — regressions can be intermittent.

## Organizational Checks

Run these before shipping any multi-agent config:

- [ ] Central team reviewed system prompt and end-to-end evals
- [ ] Each skill/subagent has a named DRI (directly responsible individual)
- [ ] No duplicate skills exist across agents (audit for overlap)
- [ ] Tool count per system prompt ≤ 50
- [ ] Skills use progressive loading — not pasted wholesale into every prompt
- [ ] Knowledge bases are accessed via search tool, not pasted into the prompt

## Migration Path (removing unjustified splits)

1. Identify tightly coupled pair (chain on each other's output, same data, called together often).
2. Convert the downstream agent's prompts into skills loadable by the main loop.
3. Convert its API calls into MCP tools.
4. Run evals with and without the merge. Compare per-task, not just aggregate.
5. Check every production chain and every hard constraint the removed agent enforced.
6. Run each task multiple times — regressions can be intermittent.
7. Discard the subagent once evals confirm parity or improvement.
