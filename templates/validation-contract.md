# Validation Contract: <mission-id>

<!--
This file defines what "done" means. Written by the orchestrator during Phase 1,
BEFORE any worker is dispatched. Every assertion gets a unique ID (A-NN) and is
verified by exactly one lane.

Lanes:
  - behavior              — exercised against the live app by the behavior validator
  - scrutiny (test)       — a project test must cover it; scrutiny validator runs the test
  - scrutiny (code review) — an adversarial code reviewer must verify it from the diff
  - scrutiny (static)     — lint/typecheck must pass (rarely needed as its own assertion)

Coverage rules:
  - every feature has >=1 assertion assigned
  - every assertion has >=1 feature assigned
  - validators only check assertions in their lane
  - validator findings cite assertion IDs
-->

## M1 — <milestone name>

### A-01 — <one-sentence behavior that defines done>
**Verify via:** <behavior | scrutiny (test) | scrutiny (code review) | scrutiny (static)>
**Steps:** <for behavior: numbered click-by-click flow; for scrutiny (test): the test that must exist; for scrutiny (code review): what the reviewer must confirm>
**Assigned to features:** <feat-NNN>[, feat-NNN ...]
