# Tool map — GitHub Copilot CLI

The skill and its subagent prompts are written in **action language** so the same body runs on any coding-agent CLI. On Copilot CLI the actions resolve to the tools below.

| Action (as written in the skill) | Copilot CLI tool | Permission kind |
|---|---|---|
| Read a file | `view` | `read` |
| Create a file | `create` | `write` |
| Edit a file | `apply_patch` / `edit` | `write` |
| Run a shell command | `bash` | `shell` |
| Search file contents | `rg` (ripgrep — Copilot has no `grep` tool) | `read` |
| Find files by name | `glob` | `read` |
| Fetch a URL | `web_fetch` | `url` |
| Dispatch a subagent | `task` (`agent_type: "general-purpose"`) | n/a |
| Parallel dispatch | multiple `task` calls in one turn | n/a |

There is **no documented `web_search` tool** on Copilot CLI; the skill does not use one.

## Install & invoke

```text
copilot plugin marketplace add jbencina/mission-lite
copilot plugin install mission-lite@jbencina
```

Copilot resolves the plugin manifest via a fallback chain that ends at `.claude-plugin/plugin.json`, and reads the shared `skills/plan/` directory (SKILL frontmatter `name` + `description` is the same schema). Copilot has **no custom slash commands**, so there is no `/mission-lite:plan`; trigger the skill by name (`/plan`) or by describing the goal so it autoloads on `description`.

## Fidelity caveats (differences from Claude Code)

These are inherent to the platform, not bugs in the port. Documented so nobody is surprised.

- **G1 — Per-subagent tool scoping is coarser.** Copilot's permission model works in *kinds* (`read` / `write` / `shell` / `url`), not per-`task` tool allowlists. A prompt that says "read-only" or "you cannot dispatch subagents" is followed as an **instruction**, but Copilot does not hard-enforce it the way Claude Code does by withholding the tool. The mission-lite scout, workers, and reviewers therefore run in a looser sandbox. To tighten it, launch with deny patterns, e.g. `--deny-tool='shell(git push)'` or restrict `write`.
- **G2 — Per-subagent model selection is advisory.** `state.models.<role>` uses Claude tier names (`opus`, `sonnet`). Copilot's per-`task` model control is weak/undocumented, so these values may be ignored and all subagents may run on the session model. Set the session model with `/model` or `--model`. Treat the role→tier intent (strong orchestrator/validators, faster workers) as guidance, not a guarantee.
- **G3 — Invocation.** No `/mission-lite:plan`; see "Install & invoke" above.
- **G5 — Behavior validation / Playwright.** On Claude Code the behavior validator drives the Playwright plugin/skill. On Copilot, configure Playwright as an MCP server in `~/.copilot/mcp-config.json`; the `bash`/`curl` behavior path needs no extra setup.

## Instructions file

The target project's conventions live in `AGENTS.md`, which Copilot reads natively from the repo root (it also reads `.github/copilot-instructions.md` and, for compatibility, `CLAUDE.md`). Workers and reviewers read whichever is present.

See `claude-tools.md` for the Claude Code mapping.
