# mission-lite improvements — implementation plan (WORKING DOC)

> **Status:** Draft for review. Temporary planning artifact on the `mission-improvements`
> branch — delete before merge. Synthesizes two review passes (the original six-change brief +
> a state/correctness-focused review), reconciled. Open questions flagged **[NEEDS DECISION]**.

All paths relative to `skills/plan/` unless noted. Each change = one logical commit. Apply in
the group order below (A → D); within a group, order is flexible. After all changes:
`claude plugin validate .` from repo root, then re-read `SKILL.md` for contradictions
(branch strategy × pause affordance × the two execution invariants).

**Cross-cutting rules:**
- Any field the orchestrator writes must be declared in `templates/mission-state.json` (or
  `templates/handoff.md`) — the template is the source of truth for shape.
- Any field read mid-loop must be persisted, not held in memory (resumability invariant).
- Domain-agnostic constraint holds: no per-domain packs. Lane-awareness (C1) keys off assertion
  *lanes*, which is structural, not domain-specific — consistent.

**State-shape additions introduced by this batch** (all into `templates/mission-state.json`):
`branch`, `base_sha`; per-feature `attempts` + `last_failure_reason`;
`policy.max_worker_attempts_per_feature`; a control-action log field (C-pause).

---

## Group A — Correctness fixes (small, low-risk, do first)

### A1 — Resolve the `blocked` vs commit-SHA contradiction
**Files:** `SKILL.md` (Phase 2 step 5, and the triage chain step 6).
**Problem:** Pre-flight (`SKILL.md:145`) requires a commit SHA for *every* handoff before
triaging, but a clean `Status: blocked` worker (`worker-prompt.md:54`) legitimately may have no
commit. Today such a worker is re-flagged `blocked` with reason "commit SHA not recorded,"
**overwriting its real block reason** — surfacing the wrong cause to the user.
**Edit:** make the SHA requirement status-dependent:
```
complete: commit SHA required.
partial:  commit SHA required iff files changed; none allowed iff no files changed.
blocked:  commit SHA optional; block reason required; files-changed empty or explicitly listed.
```
Pre-flight should read `Status` first, then apply the matching rule — never clobber a real
block reason.
**Acceptance:** a blocked handoff with no SHA preserves the worker's block reason; complete/partial
still demand a SHA.

### A2 — Behavior validator emits `skipped`, not `pass`, for `behavior_tool: none`
**Files:** `behavior-validator-prompt.md` (line 36).
**Problem:** the `none` branch marks `Result: pass`, but the orchestrator expects `skipped` for
exactly that case (`SKILL.md:201,209`) and the prompt's own output format already lists `skipped`.
**Edit:** change the `none` branch to write `Result: skipped`.
**Acceptance:** library/no-runtime missions report `skipped`; orchestrator's "pass or skipped"
branch handles it without ambiguity.

### A3 — Code-reviewer tool line: list `Write` explicitly
**Files:** `code-review-subagent-prompt.md` (line 47).
**Problem:** phrased as "No Write except to {{REVIEWER_OUTPUT_PATH}}" — mildly ambiguous for a
role whose entire output is a written report.
**Edit:** `Tools: Read, Bash (git show/log only), Grep, Glob, Write (only to {{REVIEWER_OUTPUT_PATH}}). No Agent.`
**Acceptance:** unambiguous allowed-tool list.

---

## Group B — State / resume robustness

### B1 — Explicit git branch strategy  *(was Change 2; merged with `.missions` guard)*
**Files:** `SKILL.md` (Phase 0, Phase 2 step 6b, Phase 6), `templates/mission-state.json`,
`worker-prompt.md`.
**Problem:** no mission branch is defined; workers commit to whatever HEAD was checked out at
Phase 0. The `git branch --contains <sha>` reachability check (`SKILL.md:153`) is only
accidentally coherent.
**Edits:**
1. Phase 0: create + checkout `mission/<mission-id>` from current HEAD; record `branch` and
   `base_sha`; announce the branch.
2. `templates/mission-state.json`: add `"branch": ""`, `"base_sha": ""`.
3. Phase 2 step 6b: reference `state.branch` explicitly instead of "the current working branch."
4. Phase 6 / `SUMMARY.md`: report the branch; tell the user the work is on `mission/<id>` for
   review/merge. **No auto-merge.**
5. **`.missions/` commit guard** *(convergence: both reviews flagged this)*: orchestrator adds
   `.missions/` to the project `.gitignore` at Phase 0, **and** `worker-prompt.md` gains a hard
   rule: never `git add`/commit anything under `.missions/**`; never use `git commit -a`; stage
   only feature files explicitly.

**DECISION — Phase 0 ordering (locked).** Reorder bootstrap to: **check working tree clean →
create `mission/<mission-id>` branch → write the `.gitignore` entry → create the mission dir.** The
dirty-tree gate inspects the user's project tree before any mission files exist.
**DECISION — `.missions/` guard (locked):** gitignore `.missions/` **and** add the worker hard rule
(both). **Branch name (locked):** `mission/<mission-id>`.

**Acceptance:** after Phase 0, `git branch --show-current` = `mission/<id>`; `state.json` carries
`branch` + `base_sha`; Phase 6 names the branch and does not merge; no `.missions/` content lands
in feature commits.

### B2 — Per-feature attempt tracking + worker-attempt cap
**Files:** `SKILL.md` (Phase 2 steps 1–2, 6; resume section), `templates/mission-state.json`.
**Problem:** Phase 2 step 2 flips a feature to `in_progress` *before* spawning, but resume
(`SKILL.md:276`) treats `implementing` as "still pending," and nothing caps a feature whose
worker keeps crashing/partial-ing → potential infinite re-spawn.
**Edits:**
1. Add per-feature `attempts` (int, default 0) and `last_failure_reason` (string|null) to the
   feature shape in `templates/mission-state.json`.
2. Phase 2: increment `attempts` when spawning a worker; on blocked/partial/failed triage, set
   `last_failure_reason`.
3. Add `policy.max_worker_attempts_per_feature` (**locked default: 2**) to the template; Phase 2 blocks the
   feature (→ Phase 5) when `attempts` exceeds it, with a clear reason.
4. Resume: reword the `implementing` entry to derive from `attempts`/`last_failure_reason`
   (re-spawn vs. inspect vs. block) instead of assuming "still pending."
**Scope note:** this is the *subset* of the larger "task ledger" proposal — NOT the full
`verification{}` subobject or assertion mirror (those are deferred, see bottom). The per-assertion
follow-up cap already exists implicitly in Phase 4's loop guard (`SKILL.md:220`); do not duplicate it.
**Acceptance:** a crash-looping feature blocks after `max_worker_attempts_per_feature`; resume
makes a deterministic choice from persisted state.

---

## Group C — Execution quality

### C1 — Honest handoff attestation  *(old Change 6b, fallback variant)*
**Files:** `worker-prompt.md`, `templates/handoff.md`.
**DECISION — keep strict:** always-test-first discipline is **retained**. GPT's lane-aware
relaxation is **rejected** — the worker still writes failing tests before implementation
universally (`worker-prompt.md:41`). This change is *only* about ending the unverified-attestation
theater, not about changing execution.
**Problem:** the handoff's "Wrote tests before implementation" checkbox (`templates/handoff.md:38`)
is self-attested and unverifiable, but the surrounding framing implies it's a checked gate.
**Edits:**
1. Relabel the Procedure-adherence section **"Self-reported (not independently verified)"**. Keep
   the genuinely-verified items where they are (commit recorded → checked in Phase 2 step 6a/6b;
   commands succeed → re-executed in step 6d); the tests-before-code box stays (still required under
   strict discipline) but is clearly marked self-reported.
2. Leave the truthful command claim (`worker-prompt.md:51`, about the re-executed Commands table)
   intact; ensure nothing implies the tests-before-code box is independently caught.
**Correction carried forward:** the "validators catch lies" line is about the *Commands run* table
(which *is* re-executed, Phase 2 step 6d) — only the tests-before-code box was theater.
**Note (out of scope per "keep strict"):** projects with genuinely no test runner remain handled as
today (empty test command → "skipped (not configured)"). Not expanding that here.
**Acceptance:** the handoff no longer dresses unverified attestation as a checked gate; execution
discipline is unchanged.

### C2 — Inject project conventions into workers & reviewers  *(was Change 6a)*
**Files:** `worker-prompt.md`, `scrutiny-validator-prompt.md`, `code-review-subagent-prompt.md`.
**Edit:** add an early step (worker: ~step 2, before implementing) to read the project's
`CLAUDE.md`/`AGENTS.md` if present and follow them; mirror in the scrutiny + code-review prompts so
reviewers judge against the same conventions.
**Caveat:** verify whether Claude Code already injects project memory into `Agent`-spawned subagents
— even if so, the explicit instruction is cheap insurance and the worker has Read.
**Acceptance:** worker + reviewer prompts reference `CLAUDE.md`/`AGENTS.md`.

### C3 — Milestone integration check in the scrutiny validator
**Files:** `scrutiny-validator-prompt.md` (procedure + output format).
**Problem:** code reviewers each see only one feature's diff (`code-review-subagent-prompt.md:3`),
so cross-feature integration bugs *within* a milestone can slip through.
**Edit:** after per-feature reviewers return, add a "Milestone integration check" step + report
section where the scrutiny validator inspects the combined milestone diff for cross-feature
conflicts (inconsistent API-contract changes, a rename another feature still imports, feature B
assuming unimplemented feature-A behavior, assertions passing only in isolation). No new agent.
**Acceptance:** the scrutiny report contains a milestone-integration section; integration conflicts
surface as findings.

### C4 — Scrutiny validator model `haiku` → `sonnet`  *(was Change 5; reinforced by C3)*
**Files:** `config.md` (line 9), `templates/mission-state.json` (`models.scrutiny_validator`).
**Problem:** `haiku` justified as "mostly running commands," but the role coordinates reviewer
fan-out, judges coverage/weak tests, and (with C3) does integration reasoning.
**Edits:** default to `sonnet` in both files; rewrite rationale — command *execution* is mechanical,
but reviewer-aggregation + integration *judgment* is not.
**Nuance:** the orchestrator independently re-derives the milestone verdict (`SKILL.md:175–187`), so
haiku wasn't catastrophic — but the judgment load (now larger after C3) warrants sonnet.
**Acceptance:** both files default scrutiny validator to `sonnet` with a matching rationale.

---

## Group D — User-facing

### D1 — Pause/redirect via `CONTROL.md`  *(was Change 3)*
**Files:** `SKILL.md` (Phase 2 loop, `continuous_execution_invariant`), `templates/mission-state.json`.
**Edits:**
1. Convention: user drops `<mission-dir>/CONTROL.md` with free-form instructions.
2. At each feature boundary (start of Phase 2 step 1): check for it; if present → read, treat as
   **trusted user input** (the one trusted mid-mission channel, contrasted with untrusted handoffs/
   validator reports), act, archive to `<mission-dir>/logs/<timestamp>-CONTROL.md` (no re-fire),
   record the action in `state.json` (add a control-action log field to the template), then continue.
3. Update `continuous_execution_invariant` to sanction this pause point, and note **sequential
   execution remains intentional** (parallelization deferred).
**Caveat:** a long-running worker isn't interrupted until its feature completes (acceptable, keep
documented).
**Acceptance:** a mid-mission `CONTROL.md` is detected at the next boundary, acted on, archived; the
invariant no longer forbids it.

### D2 — Cost estimate + plan summary at the approval gate  *(was Change 4)*
**Files:** `SKILL.md` (Phase 1 step 7).
**Edit:** before requiring approval, present milestone count, feature-count per milestone, and a run
estimate, alongside contract + plan. Keep the explicit "approved" requirement.

**DECISION — estimate formula (locked): fuller floor.**
`#features (workers) + #milestones (scrutiny) + #features (reviewers) + #milestones (behavior)`,
before follow-ups — presented as a floor that grows with follow-ups. (The reviewer fan-out is one
code-reviewer subagent per completed feature per milestone, `scrutiny-validator-prompt.md:28–33`.)
**Acceptance:** approval output includes feature count, milestone count, and the run estimate with a
floor caveat.

### D3 — Description rewrite for intent-based retrieval  *(was Change 1)*
**Files:** `SKILL.md` (frontmatter only).
**Edit:** lead `description:` with recognizable user situations (large multi-feature builds from one
goal, full-stack/whole-app requests, brownfield migrations + large refactors, ambitious prototypes),
then keep the mechanism as a secondary clause. Leave `name: plan` unchanged.
**Caveat:** retain a "long-running / multi-feature" qualifier so it doesn't fire on small asks; the
Phase 1 anti-pattern gate guards scoping, not triggering. Recall up, precision preserved.
**Acceptance:** "build me a complete task-tracker app with auth, tests, and a working UI" plausibly
matches on intent alone; a single-function ask does not.

---

## Deferred — "Structured execution state" (separate later batch)

Decided out of this batch (own plan to follow). Both attack the same root issue — the orchestrator
re-parsing markdown via LLM for load-bearing decisions — and each touches Phase 1/3/4 plus every
prompt's output format:
- **Mirror approved assertions into `state.json`** (status / last_verdict / last_evidence) so Phase
  3/4 read structured state instead of re-parsing `validation-contract.md`. Aligns with the
  resumability invariant.
- **Required machine-readable JSON block** at the top of handoffs and validator reports (prose stays
  below), so extraction is deterministic before the orchestrator's existing re-verification.

## Ignored for lite
- **Feature `depends_on`** — redundant with sequential execution (features run in order;
  `RECENT_FILES` already gives cross-feature context). Only pays off under parallelization (out of
  scope).
- **Full per-feature `verification{}` subobject** — the orchestrator already computes these checks
  (Phase 2 step 6a–d); persisting them is audit-nice, not load-bearing. B2 captures the high-value
  subset (`attempts`, `last_failure_reason`).
- Mission Control / dashboards / headless / worktree orchestration / cost tracking — out of scope by
  design (lite).

## Locked decisions (resolved with reviewer)
- **C1 — keep strict:** always-test-first retained; lane-aware relaxation rejected. C1 is only the
  attestation relabel.
- **B1 — Phase 0 ordering:** check clean → branch → gitignore → mkdir.
- **B1 — `.missions/` guard:** gitignore **and** worker hard rule.
- **Branch name:** `mission/<mission-id>`. **`max_worker_attempts_per_feature`:** 2.
- **D2 — estimate formula:** fuller fan-out-aware floor.
- **Sequencing:** land **Group A** (correctness fixes) on its own — small, safe, independently
  reviewable — then **B → C → D**.
- **Deferred batch:** GPT #2 (assertion mirror) + #4 (JSON handoff/report blocks) — separate plan.

No open questions outstanding. Ready to implement on this branch, Group A first.
