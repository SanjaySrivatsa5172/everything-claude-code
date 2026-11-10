---
name: agentic-engineering
description: Operate as an agentic engineer using eval-first execution, decomposition, and cost-aware model routing.
origin: ECC
---

# Agentic Engineering

Use this skill for engineering workflows where AI agents perform most implementation work and humans enforce quality and risk controls.

## Operating Principles

1. Single loop first — don't spawn a subagent until evals confirm the single loop degrades.
2. Define completion criteria before execution.
3. Decompose work into agent-sized units by context isolation, not by domain or problem type.
4. Route model tiers by task complexity.
5. Measure with evals and regression checks.

## Fold-or-Spawn Check

Before spawning any subagent, answer these questions in order:

1. Does shared context improve answer quality? → yes: fold
2. Can tasks run truly in parallel with independent, compact results? → no: fold
3. Does the system prompt have irresolvable conflicting instructions? → no: fold (restructure instead)
4. Is context bloat measurably degrading eval scores? → no: fold

Spawn only when question 4 is yes and evals confirm it. Otherwise keep work in the main loop. Load `skill: agent-fold-or-spawn` for the full decision tree and migration path.

## Eval-First Loop

1. Define capability eval and regression eval.
2. Run baseline and capture failure signatures.
3. Execute implementation.
4. Re-run evals and compare deltas.

## Task Decomposition

Apply the 15-minute unit rule:
- each unit should be independently verifiable
- each unit should have a single dominant risk
- each unit should expose a clear done condition

## Model Routing

- Haiku: classification, boilerplate transforms, narrow edits
- Sonnet: implementation and refactors
- Opus: architecture, root-cause analysis, multi-file invariants

## Session Strategy

- Continue session for closely-coupled units.
- Start fresh session after major phase transitions.
- Compact after milestone completion, not during active debugging.

## Review Focus for AI-Generated Code

Prioritize:
- invariants and edge cases
- error boundaries
- security and auth assumptions
- hidden coupling and rollout risk

Do not waste review cycles on style-only disagreements when automated format/lint already enforce style.

## Cost Discipline

Track per task:
- model
- token estimate
- retries
- wall-clock time
- success/failure

Escalate model tier only when lower tier fails with a clear reasoning gap.
