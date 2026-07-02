# Handoff: <feature-id> — <feature name>

<!-- Worker: fill in every section. Validators will re-run the commands in "Commands run". -->

**Worker model:** <model-name>
**Started:** <ISO-8601 timestamp>
**Finished:** <ISO-8601 timestamp>
**Status:** <complete | partial | blocked>

## What was completed
<!-- Bullet list. One bullet per concrete change. Be specific (file paths). -->
-

## What was NOT completed
<!-- Anything in scope you did not finish. Empty list "—" if everything was done. -->
-

## Assertions addressed
<!-- For each assertion ID assigned to this feature, one line describing how it was satisfied. -->
-

## Commands run
<!-- Every test/lint/typecheck/build command you ran during this feature. Validators may rerun. -->

| Command | Exit code | Notes |
|---|---|---|
|  |  |  |

## Files changed
<!-- Every path created/modified/deleted. -->
-

## Issues discovered
<!-- Anything surprising, fragile, or out-of-scope that the orchestrator should know. -->
-

## Procedure adherence (self-reported — not independently verified)
<!-- These boxes are the worker's own attestation, not a gate. The orchestrator does NOT take them
     as proof: it independently re-runs the commands (see "Commands run") and verifies the commit
     SHA. "Wrote tests before implementation" in particular is not machine-checked — it stays a
     required discipline, to be reported honestly here. -->
- [ ] Wrote tests before implementation (self-reported; not verified)
- [ ] Verified all commands succeed before declaring complete (orchestrator re-runs these independently)
- [ ] Did not modify files outside the feature scope (self-reported)
- [ ] Committed work (commit: <sha>) (orchestrator verifies the SHA exists and is reachable)
