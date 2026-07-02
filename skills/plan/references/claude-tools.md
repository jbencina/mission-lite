# Tool map — Claude Code

The skill and its subagent prompts are written in **action language** so the same body runs on any coding-agent CLI. On Claude Code the actions resolve to native tools 1:1.

| Action (as written in the skill) | Claude Code tool |
|---|---|
| Read a file | `Read` |
| Create a file | `Write` |
| Edit a file | `Edit` |
| Run a shell command | `Bash` |
| Search file contents | `Grep` |
| Find files by name | `Glob` |
| Fetch a URL | `WebFetch` |
| Dispatch a subagent | `Agent` (with `subagent_type`) |
| Parallel dispatch | multiple `Agent` calls in one turn |

## Subagent model & scoping

- Per-role model comes from `state.models.<role>` (e.g. `opus`, `sonnet`) and is passed as the subagent's model. Claude Code honors it per dispatch.
- Per-subagent tool scoping is honored: a worker/scout/reviewer prompt that says "you cannot dispatch subagents" or "read-only" is enforced by not granting the corresponding tool.

## Instructions file

The target project's conventions live in `CLAUDE.md` (and `AGENTS.md` if present). Claude Code loads `CLAUDE.md` natively; workers and reviewers still read it explicitly because it may not be in a fresh subagent's context.

See `copilot-tools.md` for the GitHub Copilot CLI mapping and the fidelity caveats that apply there.
