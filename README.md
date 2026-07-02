# mission-lite

A Claude Code plugin for long-running, self-correcting, multi-agent software engineering missions. Vibe-coded by the orchestrator-worker-validator pattern described in [The Five Types of Multi-Agent Architecture](https://www.youtube.com/watch?v=ow1we5PzK-o) and by Factory's [Missions workflow](https://docs.factory.ai/cli/features/missions).

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

This repo is a [Claude Code plugin](https://code.claude.com/docs/en/plugins) (and a single-plugin marketplace). Add the marketplace and install:

```text
/plugin marketplace add jbencina/mission-lite
/plugin install mission-lite@jbencina
```

Then invoke from any project with `/mission-lite:plan` or by asking Claude to start a mission.

To hack on it locally without installing, point Claude Code at the checkout:

```bash
git clone https://github.com/jbencina/mission-lite.git
claude --plugin-dir ./mission-lite
```

Run `claude plugin validate .` after editing the manifests.

## Contents

The plugin manifest lives at `.claude-plugin/plugin.json`, the marketplace catalog at `.claude-plugin/marketplace.json`, and the skill at `skills/plan/`:

| File | Role |
| :--- | :--- |
| `.claude-plugin/plugin.json` | Plugin manifest (name `mission-lite`, version, metadata). |
| `.claude-plugin/marketplace.json` | Marketplace catalog so the repo is installable directly. |
| `skills/plan/SKILL.md` | Orchestrator entry point (`/mission-lite:plan`). Phases 0–6: bootstrap, planning, execute, validation, self-correction, finalize. |
| `skills/plan/scout-prompt.md` | System prompt for the read-only codebase scout that maps an existing repo during planning. |
| `skills/plan/worker-prompt.md` | System prompt for worker subagents (one per feature). |
| `skills/plan/scrutiny-validator-prompt.md` | System prompt for the scrutiny validator; fans out to per-feature reviewers. |
| `skills/plan/code-review-subagent-prompt.md` | System prompt for per-feature reviewers spawned by the scrutiny validator. |
| `skills/plan/behavior-validator-prompt.md` | System prompt for the behavior validator (runs the app and checks observable behavior). |
| `skills/plan/config.md` | Default model assignments for orchestrator, workers, and validators. |
| `skills/plan/templates/mission-state.json` | Starter state file copied into each new mission directory. |
| `skills/plan/templates/handoff.md` | Worker → orchestrator handoff template. |
| `skills/plan/templates/validation-contract.md` | Template the orchestrator fills in during planning. |
