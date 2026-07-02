# Tool map - Codex

The skill and its subagent prompts are written in **action language** so the same body runs on any coding-agent CLI. On Codex the actions resolve to the tools below.

| Action (as written in the skill) | Codex tool |
|---|---|
| Read a file | `exec_command` with `sed`/`rg`/`find`, or equivalent file-read tools |
| Create a file | `apply_patch` |
| Edit a file | `apply_patch` |
| Run a shell command | `exec_command` |
| Search file contents | `rg` via `exec_command` |
| Find files by name | `rg --files` or `find` via `exec_command` |
| Fetch a URL | `web.run` when browsing is available and appropriate |
| Dispatch a subagent | `multi-agent` subagent tools when available |
| Parallel dispatch | multiple subagent tool calls in one turn, or `multi_tool_use.parallel` for independent local reads/commands |

## Install & invoke

Codex reads `.codex-plugin/plugin.json` and the shared `skills/mission/` directory. The manifest declares `skills: "./skills/"`, so the `mission` skill is available after the plugin is installed.

Invoke mission-lite by asking Codex to use the `mission` skill with a goal, for example:

```text
Use the mission skill to build a task tracker with auth, tests, and a responsive UI.
```

There is no namespaced slash command contract in this bundle for Codex; use the skill name or a goal that matches the skill description.

## Fidelity caveats

- **C1 - Subagent availability depends on the Codex surface.** When Codex exposes subagent tools, use them for workers, scouts, and validators. If the current surface has no subagent-dispatch tool, stop and tell the user this mission runner needs subagents; do not collapse worker and validator roles into the orchestrator.
- **C2 - Per-subagent model selection is advisory.** `state.models.<role>` stores role-tier intent. Codex surfaces may not honor per-dispatch model overrides, so run the session with the model tier that matches the mission risk, and treat the stored values as guidance unless the available subagent tool supports explicit model selection.
- **C3 - Tool scoping is instruction-level unless the surface supports scoped subagents.** Leaf prompts say "read-only" or "no subagents"; enforce that by how you dispatch the subagent when possible. If the tool cannot enforce it mechanically, the prompt still carries the rule and validators should assume the looser sandbox.
- **C4 - Behavior validation / Playwright.** For `behavior_tool: playwright`, use Codex's available browser or MCP tooling. If no browser automation tool is available in the current Codex surface, fail the behavior assertion with evidence explaining the missing tool rather than silently passing.

## Instructions file

The target project's conventions live in `AGENTS.md` for Codex. Workers and reviewers still read `CLAUDE.md` and `.github/copilot-instructions.md` if present for cross-tool compatibility, but `AGENTS.md` is the primary Codex convention file.

See `claude-tools.md` and `copilot-tools.md` for the other platform mappings.
