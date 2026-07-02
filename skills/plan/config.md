# mission-lite config

## Default model per role

| Role | Default | Why |
|---|---|---|
| Orchestrator | `opus` | Planning and decomposition. Slow careful reasoning, high stakes if features are picked wrong. |
| Scout | `sonnet` | Read-only codebase mapping for existing repos (Phase 1). Needs strong code comprehension to find reuse and landmines; runs once. |
| Worker | `sonnet` | Fast code fluency. The rigid worker prompt enforces discipline. |
| Scrutiny validator | `sonnet` | Coordinates the reviewer fan-out, aggregates their reports, judges test coverage, and runs the milestone integration check, then sets the milestone pass/fail. The command *execution* is mechanical, but that *aggregation and integration judgment* is not — which is why it warrants a stronger model than command-running alone would suggest. |
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

## Models across platforms

The table above uses Claude tier names (`opus`, `sonnet`). The **role → tier intent** is what's portable — a strong model for the orchestrator and validators (careful reasoning, high stakes if features or verdicts are wrong), a faster model for workers (code fluency under a rigid prompt). The exact values are platform-specific:

- **Claude Code** honors `state.models.<role>` per dispatch (see `references/claude-tools.md`).
- **Copilot CLI**: per-subagent model selection is weak/undocumented, so these values are **advisory** — all subagents may run on the session model. Set it with `/model` or `--model`, and read `opus`/`sonnet` here as "strong tier" / "fast tier", mapping to whatever strong/fast models the session offers. See fidelity gap **G2** in `references/copilot-tools.md`.

The template ships Claude defaults; override per the two paths above for other platforms.

## Model-agnostic contract

The skill dispatches subagents via your platform's dispatch tool (Claude Code `Agent`, Copilot CLI `task` — see `references/`), and the same role contract is fulfilled identically on each:

- Subagent receives: a prompt file, a path to the mission directory, optional per-role context (feature spec, assertions, etc.).
- Subagent must: read its prompt, do its work, write its output file at the orchestrator-specified path (`handoffs/<feature-id>-handoff.md` for workers, `validations/<milestone>-<type>.md` for validators), and exit.
- Subagent must not: hold state across invocations, modify files outside its scope, or spawn deeper subagents unless its prompt grants the `Agent` tool.

Swapping a role to another CLI touches only this contract surface and the per-platform tool maps in `references/` — the skill body and prompts, written in action language, remain unchanged.
