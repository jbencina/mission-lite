# mission-lite

A Claude Code skill bundle for long-running, self-correcting, multi-agent software engineering missions. Inspired by the orchestrator-worker-validator pattern described in [The Five Types of Multi-Agent Architecture](https://www.youtube.com/watch?v=ow1we5PzK-o) and by Factory's [Missions workflow](https://docs.factory.ai/cli/features/missions).

## What it does

You invoke the skill with a goal (e.g. "add OAuth login to the checkout flow"). Claude Code then:

1. **Plans** — decomposes the goal into features grouped by milestones, writes a validation contract, and persists everything to `<project>/.missions/<mission-id>/`.
2. **Executes** — spawns a fresh worker subagent per feature with a focused prompt and a handoff template.
3. **Validates** — at each milestone boundary, spawns adversarial validator subagents (scrutiny + behavior) with fresh context to look for regressions, missing tests, and broken behavior.
4. **Self-corrects** — when validators surface issues, the orchestrator appends follow-up features and re-validates rather than declaring success.
5. **Resumes** — all state lives on disk, so any future Claude Code session can pick the mission back up.

The orchestrator itself does not write production code. Workers do that. The orchestrator's edits are confined to mission state, plan, validation contract, and final summary.

Factory Missions can leverage existing skills and develop specialized skills during planning. This lightweight adaptation captures that guidance as per-feature `worker_skills` blocks in the mission plan and injects those blocks into each worker prompt.

## Installing

Drop the contents of this repo into a Claude Code skills directory:

```bash
git clone https://github.com/jbencina/mission-lite.git ~/.claude/skills/mission-lite
```

Then invoke from any project with `/mission-lite` or by asking Claude to start a mission.

See the [Claude Code skills docs](https://code.claude.com/docs/en/skills) for other install locations (project-level `.claude/skills/`, plugins, managed settings).

## Contents

| File | Role |
| :--- | :--- |
| `SKILL.md` | Orchestrator entry point. Phases 0–6: bootstrap, planning, execute, validation, self-correction, finalize. |
| `worker-prompt.md` | System prompt for worker subagents (one per feature). |
| `scrutiny-validator-prompt.md` | System prompt for the scrutiny validator; fans out to per-feature reviewers. |
| `code-review-subagent-prompt.md` | System prompt for per-feature reviewers spawned by the scrutiny validator. |
| `behavior-validator-prompt.md` | System prompt for the behavior validator (runs the app and checks observable behavior). |
| `config.md` | Default model assignments for orchestrator, workers, and validators. |
| `templates/mission-state.json` | Starter state file copied into each new mission directory. |
| `templates/handoff.md` | Worker → orchestrator handoff template. |
| `templates/validation-contract.md` | Template the orchestrator fills in during planning. |
