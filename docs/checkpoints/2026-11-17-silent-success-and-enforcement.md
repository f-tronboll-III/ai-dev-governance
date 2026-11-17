# Checkpoint: Silent success, honest checks, and enforcement over attestation

**Date:** 2026-11-17
**Status:** COMPLETE
**Session:** Executor, working directly from an Operator request (no Strategist handoff)

## Why this pass

The last substantive update (`a110de4`, 2026-07-25) predates four months of operational change in the shop this playbook documents. A comparison of the private governance against every public doc — fanned out by area to cheap-tier gatherers, with every candidate judged and written on the heavy tier — found two classes of gap:

1. **The public playbook was prescribing two practices the shop had retired.** The Graph Consultation Pass and the standing graph-refresh duty were retired on measurement on 2026-10-25. The blanket "always return 200" webhook rule was refined into vendor classes on 2026-07-09 — *before* the last pass, which missed it. The second is actively dangerous to a reader: a blanket 200 silently drops every payment event after a signing-secret rotation.
2. **The dominant failure class of the period had no home in the repo:** *nothing errored.* A 200 from a parked domain, a draft id for a message that had already sent, a push that deployed nothing, a check that couldn't run and exited 0, a local environment file pointing at production, a written prohibition with no mechanism behind it.

## Plan

1. [done] Rewrite `architecture-graphs.md` — on-demand views, code as the source of truth, the measured reversal
2. [done] `sanity-check.md` — fourth leg becomes the blast-radius read, with its triggers
3. [done] `autonomous-loops.md` — webhook response classes replace the blanket 200 rule
4. [done] New dive `environment-targeting.md`
5. [done] New dive `verify-by-destination.md`
6. [done] New dive `defect-disposition.md`
7. [done] New dive `honest-checks.md`
8. [done] New dive `enforcement.md`
9. [done] Section updates across twelve existing dives (table below)
10. [done] README, `architecture.md`, `getting-started.md` — eighth lesson, tables, counts
11. [done] Verification: links and anchors, vendor-name scrub, internal-jargon grep, markdown lint on touched files
12. [done] Commit, push, confirm on the remote

## Deltas

### Δ1 — Two public docs contradicted current practice, and were corrected rather than appended to

`architecture-graphs.md` required every repo to keep a generated graph, regenerated in the QA gate, with mandatory citations in every triggered handoff. The source shop ran that for seven months, measured it (handoff citations from seventeen a month to one; no Executor session in two months opened a graph; more than half the graphs six months stale; one 124 commits behind), and retired it. The rewrite keeps what survived — generated-not-hand-written, timestamp and commit in the first line, wiring not behavior — and replaces the consumption doctrine with on-demand views plus the blast-radius leg of the Sanity Check, which is where the one real catch actually came from. The doc now says plainly that an earlier version prescribed the retired practice, because "we measured it and stopped" is itself the most transferable lesson in it.

`autonomous-loops.md` said external webhooks return 200 "regardless," including on a bad signature. That is right for email-class vendors and wrong for payment-class vendors, which retry and alert; there, a 400 on a bad signature is the alarm. Rewritten as three classes with a table, a required streak alert, and a class comment atop every handler. The checkpoint for the 2026-05-31 pass that introduced the original rule was left as written — checkpoints are history.

### Δ2 — Lesson count changed (seven → eight)

The previous pass set the bar for a founding lesson: a failure of the same shape and severity as the originals, with production-affecting incidents behind it. "Nothing erroring is not the same as working" clears it — client email sent unrecoverably, money collected and invisible to the books, three commits deployed nowhere, production written from a "local" test — and it is the organizing principle of the source shop's always-loaded memory file. Renumbered in the same four places the last checkpoint listed (README glance list and Part I heading, `architecture.md` twice, `getting-started.md`), plus the README's "lessons above" line.

### Δ3 — The QA gate was renumbered

Step 10 ("regenerate the architecture graph") was removed as part of Δ1. Defect disposition entered as step 5, and the push step became "push, then verify at the destination." The checklist is still ten steps. Internal references to step numbers inside `qa-gate.md` (the no-clearances bullet) were updated; nothing outside the file cited a step by number.

### Δ4 — Provenance figure updated (~30 → ~40 repos)

The source shop's most recent fleet-wide checks ran across about forty repos. Kept approximate, per the precedent in the last checkpoint.

### Δ5 — Commit identity for this pass

The previous Executor pass committed with a role-tagged author name. Since then the source shop adopted a rule against per-commit identity overrides, after role-tagged authors blocked deploys on a host that gates builds by author. This repo isn't deployed that way, but the rule is unconditional, so this pass commits as the configured identity and carries the role in the trailer. `governance-sync.md` now documents exactly this caveat.

### Δ6 — Pre-existing markdown lint in touched files, fixed under the new defect-disposition rule

Seven bare code fences across `architecture.md`, `checkpoints-handoffs.md`, and `governance-sync.md` got a language; one emphasis-as-heading in `operator-clearances.md` became a blockquote. All D1 — files this pass already changed, mechanical fixes. One inline-HTML warning this pass introduced (`<earlier doc>` in prose) was fixed before commit.

## What was NOT ported (deliberately)

- **Component models** (install shared components from a governed model, never port them out of a live client repo). Real and well-evidenced, but it pays off only with three or more repos sharing a component, and it amends the fork-and-tailor doctrine the last pass excluded as business architecture. Strongest candidate for the next pass.
- **Inference is a suggestion** — a machine-derived value lives in its own field with provenance until a human accepts it, and a feed about what is *happening* can't fill a field about what a place *is*. Generalizable, but thinner than the rest; held.
- **Plain web-engineering standards** — the per-endpoint anti-abuse floors, email authentication policy, firewall baselines, error-message UX, search-visibility standards, telephony compliance, database wake budgets. Only their governance halves were ported (screens that pass the wrong thing, look-alike controls, "never trade a control for a clean report"). The full standards fail the stranger-with-one-to-five-repos screen as AI-governance content.
- **Every identifier.** Account and endpoint ids, repo and client names, people, vendor names in new prose, hook and script paths, the actor roster's real names. Numbers survived only where they can't identify their source.
- **The defect ledger's rank-ceiling machinery and model-id debt.** Tool-specific and fast-aging; the generalizable parts are in `defect-disposition.md`.

## Files modified this session

| Path | Change |
| :---- | :---- |
| `docs/architecture-graphs.md` | Rewritten. Two walls (blind Strategist; the maintained graph nobody read, with measurements). Six rules. Two kinds of fresh view, the handover, what stays maintained. New worked example with the blast-radius read catching a post-pack consumer. Eight anti-pattern rows. |
| `docs/sanity-check.md` | Fourth leg is now the blast-radius check, with its triggers. Three anti-pattern rows; Related updated. |
| `docs/autonomous-loops.md` | Webhook section rewritten as three vendor classes with a table, the pattern, recovery, the subscription-diff note, six anti-pattern rows, a new five-second test. |
| `docs/environment-targeting.md` | NEW. Markers beside every credential, "no marker means production," why not to rename the variable, generate/verify, the guard and its fail directions, preview is production too, what proves which database you're on. Nine anti-pattern rows. |
| `docs/verify-by-destination.md` | NEW. Five-incident wall; destination table for ten kinds of "done"; from outside; prove the fetch; read back the irreversible ones; pointers drift together. Ten anti-pattern rows. |
| `docs/defect-disposition.md` | NEW. D1/D2/D3, the ledger, ceilings that only go down (six rules), why not a percentage, tests, the summary line. Ten anti-pattern rows. |
| `docs/honest-checks.md` | NEW. Three exit codes, coverage as output, negative controls, presence is not sufficiency, the instrument changes the count, never trade a control for a clean report, interpretation before detection. Eleven anti-pattern rows. |
| `docs/enforcement.md` | NEW. Capability inventory, posture is a file, mechanize the irreversible and silent, fail direction, one verdict per door, enforce don't attest, quarantine by construction, the workaround unit, guidance in the refusal. Twelve anti-pattern rows. |
| `docs/context-cascade.md` | New section: the router, with measurements and six mechanics. Five anti-pattern rows; two adapt bullets. |
| `docs/durable-memory.md` | Rule 5 gains the resident-file admission rule and the one-live-home rule. Three anti-pattern rows. |
| `docs/secrets-and-rotation.md` | Three new sections: two readers / duplicate names, a bad read looks like a bad secret, data plane vs control plane. Five anti-pattern rows. |
| `docs/trust-gradient.md` | Three new subsections: prohibition is not a capability limit, one label per surface, third-party targets. Quarantine-by-construction note. Three anti-pattern rows. |
| `docs/model-orchestration.md` | New section on host-appended "don't delegate" instructions and the standing request. Two anti-pattern rows. Registry wording de-graphed. |
| `docs/two-roles.md` | Outward sends gain the read-back rule; unbounded destruction gains "tight is not the test." Two anti-pattern rows. |
| `docs/external-sources.md` | Rules 5–6; per-host user agents and the hanging endpoint; new "Absence versus failure" section; operator types every password. Four anti-pattern rows. |
| `docs/infrastructure-limits.md` | New "Limits the code only pretends to respect" section. Three anti-pattern rows. |
| `docs/governance-sync.md` | Author-tagging caveat; path-scoped commits; piped-push exit codes; rules outrank docs outrank scripts. Four anti-pattern rows. |
| `docs/checkpoints-handoffs.md` | Two new sections: after the handoff ships, and when two actors have no channel. Three anti-pattern rows. |
| `docs/qa-gate.md` | Checklist revised (Δ3). Three anti-pattern rows; adapt bullet; Related +3. |
| `docs/operator-clearances.md` | Two anti-pattern rows (settled facts as questions; 404 read as permission). Freeze example as a blockquote (Δ6). |
| `docs/architecture.md` | Eight lessons; changelog and router note; Sanity Check and QA summaries; supporting disciplines eleven → sixteen; lifecycle steps 6, 7, 10; transferable core; Related. |
| `README.md` | Eighth lesson (glance list and Part I). Deep-dive table +5 rows, three descriptions updated. One over-engineering row. Transferable-core and adapting bullets. Provenance and intro counts (Δ4). |
| `docs/getting-started.md` | Eight lessons; six trigger rows; one adapt bullet. |
| `docs/checkpoints/2026-11-17-silent-success-and-enforcement.md` | NEW. This file. |

## Verification

- **Link and anchor check:** ephemeral verifier over all 43 markdown files — every relative link resolves, and every in-repo `#anchor` resolves under GitHub's slug rules (41 cross-file anchors). Clean. **Negative control:** run against a fixture with one missing file and one missing anchor → both reported, exit 1; the valid em-dash anchor in the same fixture passed. Not committed, per the 2026-06-12 precedent.
- **Vendor-name scrub:** zero vendor or model names in any added line or new file. The grep was itself controlled against a planted vendor name before its clean result was trusted.
- **Internal-jargon grep:** clean. The only hit is the pre-existing provenance URL in the README.
- **Markdown lint:** touched files fixed per Δ6; remaining diagnostics in the editor were stale and confirmed against the files on disk.

## Operator Action Block

None. Docs-only session; pushed.

## Next-session context

- The repo now has **27 dive documents** plus the architecture reference. `architecture.md` names sixteen supporting disciplines.
- The lesson count is **eight**. A ninth means renumbering the same five places listed in Δ2.
- The QA gate is ten steps; step numbers are cited only inside `qa-gate.md`.
- Held for a future pass, in rough priority order: (a) component models, (b) inference-as-suggestion, (c) the content-generation governance shape still held from the last pass.
- `_deep-dive-template.md` contains a deliberate placeholder link and is excluded from link verification.
