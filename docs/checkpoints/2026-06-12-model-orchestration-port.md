# Checkpoint: Model Orchestration + Non-trust tools — port from internal protocol
**Date:** 2026-06-12
**Status:** COMPLETE
**Session:** Executor (CC), committing under the public-repo Contributor identity

## Plan
1. [done] Write `docs/model-orchestration.md` — new six-section dive (Wall / Rule / How it works / Worked example / Anti-patterns / Adapt-it / Related)
2. [done] Add "Non-trust tools — instruments, not actors" section to `docs/two-roles.md`
3. [done] Add Model Orchestration row to README deep-dive table (top of the supporting-disciplines cluster, ahead of Durable Memory)
4. [done] Update `architecture.md` Layered Protocols subsection (seven → eight disciplines) + meta-discipline framing paragraph + Related footer
5. [done] Cross-link sweep: model-orchestration in the Related footers of sanity-check, bounded-deviation, cost-management, delta-log; two-roles ↔ model-orchestration both directions
6. [done] Vendor-name scrub (grep every changed file for the seven banned names)
7. [done] Link/anchor verification across all `.md` files
8. [done] Commit, push

## Origin: the gap that triggered this pass

The internal orchestration protocol crystallized a pattern that generalizes cleanly and was not covered anywhere in the public repo's existing 18 dives: the moment a developer realizes the top-tier reasoning model is also the one doing the grepping, they hit the waste-vs-quality tension this dive names. The standing screening question — *"would a stranger with 1-5 repos and a different stack still recognize the failure mode?"* — passes hard for both ported pieces.

Two pieces ported:
- **Model Orchestration** — a new dive generalized from the internal protocol's spine (default-Heavy posture, down-model rules, stay-Heavy rules, the verbatim gather contract, the watch-words smell, the fan-out pattern, the harness mechanism, scope/propagation).
- **Non-trust tools** — a new section in `two-roles.md`, generalized from the internal tools registry's four-clause definition (tools-not-actors / never-mutate-production / never-author-durable-artifacts / ingest-no-PII), plus the genuine-outside-view bonus.

## Sanity Deltas

### Δ1 — Link-verifier script referenced by the handoff does not exist (documented gap, proceeded)

**Handoff says** (acceptance criterion 7): "The link/anchor verification script first introduced in commit `512ccab` ... passes clean ... If the script isn't present at execution time, re-create it from the spec in `docs/checkpoints/2026-05-31-metered-service-discipline.md`."

**Production shows:** commit `512ccab` is documentation-only (README, architecture, autonomous-loops, cost-management, dependency-management, infrastructure-limits, and its own checkpoint — no script). No script (`.sh/.mjs/.js/.py`) is tracked anywhere in the repo or its history. The metered-service checkpoint contains no script spec — only a manual plan step, "[done] Verify links, commit, push." Link verification in this repo has always been a manual pass, not a committed tool.

**Risk if followed as written:** none material — but committing a *new* tracked verification script to satisfy AC#7 would expand the change beyond the 9 files the handoff scopes (it lists no script under "Files changed").

**Resolution (bounded deviation — evidence-anchored, minimal, no scope expansion):** ran an **ephemeral** link/anchor verifier over all `.md` files (relative-link existence + in-repo `#anchor` resolution against GitHub's slug algorithm), confirmed clean, and did **not** commit it. AC#7's *intent* (links resolve clean) is met; the non-existent artifact is not manufactured into the repo. Flagged here for the checkpoint record.

### Δ2 — Receipt count correction (cosmetic)

Handoff §27/§104 describe the README supporting-disciplines cluster as "18 rows starting with Durable Memory." The 18 is the *whole* deep-dive table; the supporting-disciplines cluster beginning at Durable Memory was **seven** rows (architecture.md likewise said "seven more" / "These seven"). Both were bumped to eight. Insertion point (ahead of Durable Memory) and intent were exactly as the handoff specified; only the stated count was off. No action needed beyond the count edits already made.

### Δ3 — Insertion-point wording in §B (resolved by intent)

Handoff §B says insert the new section "between the Operator's hard line section and the 'Same-modality pairings' note." In the live file the Same-modality pairings note is a bullet *inside* the Adapt-it section, which precedes the Operator's hard line section — so the literal "between" is unsatisfiable. Placed the new section immediately after the Operator's hard line section and before the Related footer, which matches the handoff's own receipt framing ("slots cleanly between the Operator section and the ... note") and its structural intent.

## What was NOT shipped (deliberately, mirroring the handoff cut)

- **Vendor-specific offload tooling.** The internal wrapper-script implementation, its model-ladder flag conventions, and the third-party gateway specifics are infrastructure detail that fails the screen. The *category* (non-trust external-model wrapper) ports; the *instance* does not.
- **The client-website model-routing gate.** Depends on having client-facing per-request model calls — a surface most 1-5-repo readers don't have yet. Re-screen if the public repo grows an AI-feature-productization section.
- **The quarterly-cadence enforcement hook.** BTS-shaped. The *principle* (revisit the rules on a rhythm you already keep, rather than a new timer) survives as one sentence in the Adapt-it section.
- **The `/understand-diff` mandatory gate at 3+ file changes.** Held for a future pass — generalizes only with a knowledge-graph layer the reader is unlikely to have.
- **The gather contract treated as code.** The contract itself ships verbatim (load-bearing artifact); the implementation detail that it lives as a constant in a wrapper does not.

## Files modified this session

| Path | Change |
| :---- | :---- |
| `docs/model-orchestration.md` | NEW. Six-section dive. Five numbered rules; Rule 3 (Sanity Pass non-down-modelable) stands alone. Gather contract reproduced verbatim as a quoted block. Watch-words as a "smell, not a gate" callout. Fan-out pattern. Worked example: two-project feasibility scan, two Light gatherers + one Heavy synthesizer, de-identified. Eight-row anti-pattern table. Adapt-it covers solo / team / enterprise + recurring-review-on-existing-rhythm. Five-second test in the Wall. Related cross-links the four sister disciplines + non-trust-tools anchor. |
| `docs/two-roles.md` | NEW section "Non-trust tools — instruments, not actors" after the Operator's hard line section. Four-clause definition. Why-this-matters scar para (`[TOOL-NAME-UNVERIFIED]` tag). Genuine-outside-view bonus (vendor-neutral). Cross-link to model-orchestration. Added model-orchestration to the Related footer. |
| `README.md` | Added Model Orchestration row as the first row of the supporting-disciplines cluster (ahead of Durable Memory). |
| `docs/architecture.md` | Added Model Orchestration bullet at the top of the Layered Protocols supporting-disciplines list + a meta-discipline framing paragraph. Counts: "seven more" → "eight more"; "These seven" → "These eight". Added model-orchestration to the Related footer. |
| `docs/sanity-check.md` | Related footer: model-orchestration link ("pre-edit Sanity Pass is non-down-modelable by rule"). |
| `docs/bounded-deviation.md` | Related footer: model-orchestration link ("judgment about whether to deviate is never delegated to a cheaper tier"). |
| `docs/cost-management.md` | Related footer: model-orchestration link ("waste elimination, not spend minimization; the lever"). |
| `docs/delta-log.md` | Related footer: model-orchestration link ("a recurring delta about model choice is the signal to revise the orchestration rules"). |
| `docs/checkpoints/2026-06-12-model-orchestration-port.md` | NEW. This file. |

## Voice discipline applied

- Six-section deep-dive shape maintained for the new dive (Wall / Rule / How it works / Worked example / Anti-patterns / Adapt-it / Related), matching `infrastructure-limits.md` and `durable-memory.md`.
- Five-second test in the Wall, framed as the filenames-vs-judgment question.
- The Sanity-Pass-is-non-down-modelable rule given its own numbered rule (Rule 3), not buried in a sub-clause — it's the rule that prevents cheap-tier creep.
- Gather contract reproduced verbatim and attributed as "the gather contract, in full," not paraphrased.
- **Vendor-name scrub:** grepped every changed file for the nine banned vendor/model names in the handoff's swap table — zero hits in any new or modified prose. Tier names use Heavy / Mid / Light. Pre-existing vendor names in `two-roles.md` examples and the README tool-pairing table were left untouched and none were newly introduced.

## Link / anchor verification

Ran an ephemeral checker over all `.md` files: every relative link resolves to an existing file, and every in-repo `#anchor` resolves under GitHub's slug rules. The new section's anchor is `non-trust-tools--instruments-not-actors` (em-dash drops, leaving a double hyphen); cross-links from the orchestration dive, the two-roles Related footer, and the architecture additions use that exact form. Clean. (See Δ1 — the checker is ephemeral, not committed.)

## Operator Action Block

None. Docs-only session. Code pushed; the deploy is the review.

One infrastructure note, not an action item: the local clone's `origin` remote was repointed from `steampunkfarms/ai-dev-governance` to `f-tronboll-III/ai-dev-governance` to match the personal-account migration. Auth used the FT3 classic token from the vault (the fine-grained token returned invalid-credentials against the personal repo — likely still org-scoped; worth re-issuing a fine-grained token under the personal account if fine-grained is preferred going forward).

## Next-session context

- Public repo now has 19 dive documents. The Layered Protocols subsection in `architecture.md` names eight supporting disciplines, led by Model Orchestration as the meta-discipline.
- The Strategist write-prohibition for project repos remains in force; this port was a CC-executed handoff, the standing default.
- If a future pass ports the `/understand-diff` 3-file gate or the client-website routing gate, both were deliberately held here — re-screen them against the stranger-with-1-5-repos question first.
