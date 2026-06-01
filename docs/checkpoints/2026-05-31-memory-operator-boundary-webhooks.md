# Checkpoint — Memory, operator-boundary, and webhook discipline

**Date:** 2026-05-31 (second commit of the day)
**Scope:** Three pieces, one commit. Highest-value gap (durable memory) + two section additions that close concrete failure modes.

## What shipped

| Piece | Form | File |
| :---- | :---- | :---- |
| Durable Memory | New deep-dive (164 lines) | `docs/durable-memory.md` |
| The Operator's hard line | New section in existing dive | `docs/two-roles.md` |
| Webhooks: respond 200, then do the work | New section in existing dive | `docs/autonomous-loops.md` |
| Cost-slope / limits-cliff framing | One-line cross-link sharpening | `docs/cost-management.md` |
| Layered Protocols list + count | Added Durable Memory; six → seven disciplines | `docs/architecture.md` |
| Deep-dive table | Added Durable Memory row | `README.md` |
| Architecture footer link list | Added Durable Memory | `docs/architecture.md` |
| Cross-links | checkpoints-handoffs ↔ durable-memory; sanity-check ↔ durable-memory + operator's-hard-line; secrets-and-rotation ↔ operator's-hard-line | three dives |

## Why these three (and not the other two CC surfaced)

Applied the standing screen — *"would a stranger with 1-5 repos and a different stack still recognize the failure mode?"* — to all five candidates:

- **#1 Durable memory: SHIP.** Strongest single addition in the queue. The repo's opening hook is literally *"forgot everything from yesterday"* and there was no dive on the cross-session knowledge store. Backfills the README's own premise.
- **#2 Operator boundary: SHIP.** Real gap. `two-roles.md` defines the Operator role; `secrets-and-rotation.md` covers secret values; the boundary on *destructive / outward-facing actions* the Executor is otherwise authorized to do was uncovered. Best as a closing-counterpart section in `two-roles.md` — names the line at the same scope where the Strategist/Executor split is defined.
- **#3 Pre-flight / capacity ceilings: HOLD.** The substance is already in `infrastructure-limits.md` (Rule 3 pre-flight; Rule 4 gotchas-first-class). Promoted the sharpening framing — *"cost is the slope, limits are the cliff"* — as a one-line cross-link in `cost-management.md`'s Related section instead of a new dive. That was the high-leverage delta; the rest would have been a doubling-up on yesterday's commit.
- **#4 Webhook 200-only: SHIP.** Concrete, vivid, currently uncovered. Lives as a new section inside `autonomous-loops.md` because that dive already owns webhook handlers (the Adapt-it line mentions them) — it just hadn't named the suspension-cliff failure mode or the always-200 contract.
- **#5 Reuse-before-you-build: HOLD.** Fails the screen on closer read. The sharp version (*"authoritative join over derived array,"* *"parallel state across repos"*) is shaped by multi-repo enterprises where parallel state is a recurring failure. A stranger with 1-5 repos will recognize this as *"don't reinvent the wheel"* — true but generic; sharp version is BTS-specific. Skip until a sharper generalization presents itself.

## Voice signature preserved

- **Five-second tests:** durable-memory has one (*"is this memory telling me what to expect, or am I about to use it as the answer?"*); operator's-hard-line has one (*"if this action goes wrong, who's the one undoing it?"*); webhook section has one (*"if this handler errored on every request for an hour, would the vendor still consider my endpoint healthy?"*).
- **Scar-tissue openings:** all three new pieces open with a de-identified failure story before any rule.
- **Numbered rules with prose bodies:** preserved across all three.
- **Vendor-agnostic discipline:** strict. The durable-memory worked example uses *"the platform CLI"* / *"the hosting platform"* / *"the email provider"* / *"the vendor"* throughout. Initial draft had four "Vercel" mentions in the index example; all genericized before commit. The webhook section names no vendors at all.
- **Sibling-file framing:** explicit. Checkpoints bridge time within a task; memory bridges time across tasks. The Operator's hard line is the Executor-scale counterpart to the Strategist's no-writes rule. Cost is the slope, limits are the cliff.

## What was deliberately NOT shipped

- The full BTS memory store structure (typed sub-directories per family; per-repo MEMORY.md indexes nested under a global) — implies an enterprise shape the typical reader doesn't have. The dive presents the same shape at the smallest viable scale and notes the cascade as an "adapt it" step.
- The specific feedback files (`feedback_no_vercel_env_push.md`, `feedback_sanity_check_protocol.md`) as worked examples — too BTS-tagged. The principles ported clean; the artifacts didn't.
- A standalone pre-flight/capacity-ceilings dive (CC's #3) — folded into `cost-management.md`'s Related section as the slope/cliff framing.
- A reuse-before-you-build dive (CC's #5) — held; the sharp version is BTS-specific.
- Mechanical enforcement guidance for the operator's hard line (pre-commit hooks, IAM rules, tool blocklists) — mentioned in a paragraph at the end of the section but not specified per-tool, because the specifics are stack-dependent and the discipline is the point.

## Sanity delta

Initial draft of `durable-memory.md` used four "Vercel" references in the index worked example. The link verifier flagged two of them as broken file links (the markdown link syntax made them look like real references). Fixing them as broken links would have left the vendor-agnostic discipline violation in place. Genericized the index example to *"hosting platform"* / *"the platform's `env add` command"* and the broken links resolved as a side effect. Worth carrying forward: **the vendor-agnostic audit and the link verification are the same pass** — any "vendor name in a worked example" is *also* probably an unintended link target. Run the verifier even when you don't think you've added links.

## Repo state after this commit

- 18 dive documents in `docs/` (was 17): added `durable-memory.md`.
- Layered Protocols subsection in `architecture.md` now names seven supporting disciplines (was six): Durable Memory, Architecture Graphs, Infrastructure Limits, Autonomous Loops, Secrets and Rotation, Dependency Management, Cost Management. Durable Memory leads because it's the foundation the others all depend on for cross-session persistence.
- README deep-dive table position: Durable Memory inserted between The Delta Log and Architecture Graphs (its closest siblings on either side — Delta Log is the within-task ledger; Architecture Graphs is the structure map that loads at the same point in the session as the memory index).

## For the next session

- Standing rule (no direct project-repo writes; CC handoffs only) remains in force. The two metered-service-discipline + memory-discipline passes were public-repo deployments, both within the bounds of the suspension-by-task pattern.
- Operator will continue to bring candidate BTS protocols through the screen *"Would a stranger with 1-5 repos and a different stack still recognize the failure mode?"* before bringing them to the strategist for the port decision.
- The semper-vets watchdog-alert thread from earlier today remains BFOS Executor territory and is not strategist work.
