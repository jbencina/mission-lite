# mission-lite config

## Default model per role

| Role | Default | Why |
|---|---|---|
| Orchestrator | `opus` | Planning and decomposition. Slow careful reasoning, high stakes if features are picked wrong. |
| Worker | `sonnet` | Fast code fluency. The rigid worker prompt enforces discipline. |
| Scrutiny validator | `haiku` | Mostly running commands and checking outputs. Precise instruction-following over creativity. |
| Code reviewer (fan-out) | `sonnet` | Needs to reason about a diff and find real bugs. |
| Behavior validator | `sonnet` | Needs to drive Playwright or interpret runtime output. |

These defaults are baked into `templates/mission-state.json` under the `models` key. The orchestrator copies that template at Phase 0 bootstrap.

## Overriding

Two override paths. Pick one.

**1. In the goal prompt.** When invoking the skill, include a `models` block:

```
goal: build a CLI that reverses strings
models:
  worker: opus
  scrutiny_validator: sonnet
```

The orchestrator parses this during Phase 0 and writes it into `state.models` before Phase 1.

**2. Edit `state.json` before approving the plan.** After Phase 1 produces the plan, the orchestrator pauses for user approval. The user can edit `state.models` directly at that point. The orchestrator re-reads state before Phase 2 begins.

## Model-agnostic contract

The skill spawns subagents via Claude Code's `Agent` tool, but the same role contract could be fulfilled by any external CLI:

- Subagent receives: a prompt file, a path to the mission directory, optional per-role context (feature spec, assertions, etc.).
- Subagent must: read its prompt, do its work, write its output file at the orchestrator-specified path (`handoffs/<feature-id>-handoff.md` for workers, `validations/<milestone>-<type>.md` for validators), and exit.
- Subagent must not: hold state across invocations, modify files outside its scope, or spawn deeper subagents unless its prompt grants the `Agent` tool.

A future override could swap one or more roles to an external CLI by editing this contract surface alone — `SKILL.md` remains unchanged.
