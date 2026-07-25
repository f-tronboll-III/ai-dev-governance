# Checkpoint: Cast expansion, trust grading, and gate→clearance maturation

**Date:** 2026-07-25
**Status:** COMPLETE
**Session:** Executor, working directly from an Operator request (no Strategist handoff)

## Why this pass

The repo's last substantive update (`422157c`, 2026-06-12) described a two-actor system: one Strategist, one Executor, with non-trust tools as a single short section. Six weeks of operational change had left the public playbook describing a shape the shop no longer runs:

1. The cast grew to **five AI producers plus the human**, with three distinct trust grades — not two roles plus a footnote.
2. A **provisional grade** appeared that the old model had no room for: a co-strategist with read-only repo access whose *strategy* is trusted and whose *live-state claims* are verified. Deliberately instructed to be contrarian.
3. Several **human approval gates matured into agent clearances** — most visibly environment variables and schema migrations, both of which the old `two-roles.md` still described as permanently human-only. The doc was actively wrong, not merely incomplete.
4. **Tier and reasoning effort** became two independent axes; `model-orchestration.md` described only the first, and the natural misreading ("Heavy is the default" → "always maximum effort") is exactly the waste the discipline exists to prevent.
5. Four disciplines had no home in the repo at all: primary-source fetch discipline, source-watcher robustness, the graph consultation pass, and the census-before-designing rule.

## Plan

1. [done] New dive `docs/trust-gradient.md` — three grades, the tag lifecycle, provisional trust, the maverick clause, lineage diversity, the file-anchored receipt standard
2. [done] New dive `docs/operator-clearances.md` — prevent-vs-inform, the three retirement questions, the announce step, what never graduates, freeze windows
3. [done] New dive `docs/external-sources.md` — the fetch→render→distill ladder, primary vs secondary, the authenticated sidecar, scheduled-watcher rules, empirical reachability
4. [done] `two-roles.md` — cast-grows-past-two section; rewrote the Operator's hard line for the clearance model; tightened the non-trust section to point at the gradient
5. [done] `model-orchestration.md` — Rule 0 and the two-axes section; which-surface routing table; effort in the mechanism list; three new anti-pattern rows
6. [done] `architecture.md` — cast table under Foundation 3; three new supporting disciplines; lifecycle step 9/11 rewritten around announce + clearance
7. [done] `README.md` — seventh lesson; graded cast table; adoption prompt asks about other surfaces; deep-dive table; over-engineering rows; provenance count
8. [done] `sanity-check.md` — fourth leg (structural context); census-the-data; empirical reachability; three anti-pattern rows
9. [done] `bounded-deviation.md` — spec ambiguity counts as deviation
10. [done] `governance-sync.md` — two-domain files: never blind-copy, always union-merge
11. [done] `architecture-graphs.md` — the Graph Consultation Pass (both roles' obligations, triggers, staleness, failure modes)
12. [done] `secrets-and-rotation.md` — use-is-not-display; the credential store; four new anti-pattern rows
13. [done] `qa-gate.md` — announce block as step 7; Action-Block-padding anti-pattern
14. [done] `autonomous-loops.md` — watcher failure mode + cross-link; removed an internal-jargon leak ("CC handoff")
15. [done] `getting-started.md` — four new trigger rows; recon-agent adopt-it bullet
16. [done] Vendor-name scrub on all new prose; internal-jargon grep across the repo
17. [done] Link/anchor verification (ephemeral checker, GitHub slug rules) — clean
18. [done] Commit, push

## Deltas

### Δ1 — `two-roles.md` was describing a retired rule as current

The "four categories" section listed **environment-variable writes and production database migrations** as permanently human-only, with the do-instead being "copy-paste block, Operator pastes into the dashboard." Both have been agent-cleared operationally since 2026-06-16.

Rather than deleting the material, the pass separated the *principle* from the *instance*. The principle — the agent's authority is bounded by who undoes the damage — is unchanged and now stated as the section's opening test. The two retired instances are named explicitly as having graduated, with the reason preserved (the env-var gate existed because a CLI exposed fewer options than the dashboard; the fix was a better capability path, not a permanent human step). That framing turned out to be the most valuable content in the pass, because the *general* pattern — a gate that only informs is a ceremony — generalizes far past our two instances. It became `operator-clearances.md`.

### Δ2 — Lesson count changed (six → seven)

"The six lessons" is load-bearing branding across README, `architecture.md`, and `getting-started.md`. Adding a seventh was not a cosmetic call. It was made because the trust gradient is a *founding* lesson now, not a supporting discipline: the failure it prevents (fluent output from an ungraded actor being acted on as verified) is the same shape and severity as the original six, and three of the four incidents behind it were production-affecting. Renumbered in four places; all cross-references updated.

### Δ3 — Provenance figures were stale

README said "two AI actors across ~20 repositories" and "~20 repos across three families." Corrected to "a cast of AI actors" and "~30 repos across three families plus a shared services layer." Deliberately kept approximate — a hardcoded count is a maintenance liability and the canonical inventory lives elsewhere.

### Δ4 — Internal jargon leak found during the scrub

`autonomous-loops.md` worked example contained "opens a CC handoff" — an internal actor abbreviation that means nothing to a public reader. Replaced with "opens a handoff for the Executor." Grep across all prose confirms no other leaks (`CC`, `CChat`, family acronyms, internal repo names).

## What was NOT ported (deliberately)

- **Specific vendor and model identities.** The repo's voice is actor-agnostic; every new dive names roles and grades, never products. Verified by grep across all three new files — zero vendor names.
- **The fork-and-tailor / anti-multi-tenancy doctrine.** Genuinely load-bearing for us, but it's a *business-architecture* decision about how to structure client work, not AI-development governance. Fails the stranger-with-1-5-repos screen.
- **The specific ecosystem-level vendor ban.** The *category* ports as one sentence in the which-surface section ("a ban is a legitimate governance decision, but it belongs in a written decision record with an unban path"). The instance does not.
- **Per-family stricter-rule instances.** The floor-vs-stricter principle already ships in `model-orchestration.md`; naming which family added which PII rule is internal detail.
- **Named tooling for graph generation and consumption.** `architecture-graphs.md` describes the consultation pass in terms of what each role must do, not which tool produces the artifact.

## Files modified this session

| Path | Change |
| :---- | :---- |
| `docs/trust-gradient.md` | NEW. Six-section dive. Three grades; six clauses; cast table; provisional trust ("trust the strategy, verify the facts"); the maverick clause; lineage diversity; the receipt standard; tag-lifecycle diagram; nine-row anti-pattern table. |
| `docs/operator-clearances.md` | NEW. Six-section dive. Prevent-vs-inform; three retirement questions; the announce step with format; what never graduates (five categories); what typically does; freeze windows with sunset; before/after migration worked example; eight-row anti-pattern table. |
| `docs/external-sources.md` | NEW. Six-section dive. Three-rung ladder; primary/secondary table; distiller-is-not-a-browser; the authenticated sidecar split; three watcher rules; empirical-reachability corollary; nine-row anti-pattern table. |
| `docs/two-roles.md` | Added "When the cast grows past two" with the two-axis table. Rewrote "The four categories" → "Where the line actually falls" (custody/money, outward sends, foreign-data overwrites, unbounded destruction) plus the gate→clearance narrative. Renamed the duplicate `### Anti-patterns` heading. Adapt-it: multiple-strategists bullet rewritten, critic-on-purpose bullet added. Related +2. |
| `docs/model-orchestration.md` | Rule 0 (two axes) added; "Six rules" count. New "Two axes, not one" section (tier=risk, effort=difficulty; the effort floor on judgment; orchestration modes are not effort rungs). New "Which surface, not just which tier" section with routing table, don't-over-route and cheap-tokens-don't-buy-trust guardrails, and the ban-as-decision-record note. Effort added to the mechanism list. Three anti-pattern rows. Related +2. |
| `docs/architecture.md` | "The cast beyond two" table under Foundation 3. Supporting disciplines: eight → eleven, with Trust Gradient / Operator Clearances / Reading the Outside World; meta-discipline framing now names two. Lifecycle steps 9 and 11 rewritten around the announce block. Six → seven lessons. Related +3. |
| `README.md` | Seventh lesson (glance list + full Part I section). "…and it grows with you" graded cast table. Adoption prompt asks about other AI surfaces and their grades. Deep-dive table +3 rows, two descriptions updated. Two over-engineering rows. Transferable-core bullet on tagging. Provenance and intro counts corrected. |
| `docs/sanity-check.md` | Pre-edit pass: three legs → four (structural-context check). New section "Two checks that catch what the schema hides" — census the data, verify public reachability. Three anti-pattern rows. Related +4. |
| `docs/bounded-deviation.md` | New "Spec ambiguity counts as deviation" subsection under the rule. One anti-pattern row. |
| `docs/governance-sync.md` | New "The two-domain file: never blind-copy, always union-merge" section, including the single-source exception and the eliminate-the-second-domain note. Two anti-pattern rows. |
| `docs/architecture-graphs.md` | New "The Graph Consultation Pass" section — triggers, Strategist obligation (citations + timestamps + explicit no-signal), Executor obligation (diff-scoped analysis, targeted question, staleness-vs-commits), and the symmetric failure modes. Three anti-pattern rows. Related +2. |
| `docs/secrets-and-rotation.md` | Headline reframed from "nobody sees a secret" to "no value is ever displayed." Rule 2 rewritten as "Use is not display," with the per-role verb table. New "The credential store" section. Environment table +1 row. Four anti-pattern rows. Related +2. |
| `docs/qa-gate.md` | Announce block inserted as step 7 (checklist renumbered to 10). Two anti-pattern rows. Adapt-it bullet for setups with no clearances yet. Related +1. |
| `docs/autonomous-loops.md` | Watcher failure mode added under Rule 2 with cross-link. One anti-pattern row. Related +1. Internal-jargon fix (Δ4). |
| `docs/getting-started.md` | Four trigger rows (trust gradient, model orchestration, clearances, external sources). One adapt-it bullet. Seven-lessons reference. |
| `CONTRIBUTING.md` | Attribution note extended to cover the Executor identity. |
| `docs/checkpoints/2026-07-25-cast-expansion-and-clearances.md` | NEW. This file. |

## Verification

- **Link/anchor check:** ephemeral verifier over every `.md` file — relative links resolve to existing files, all in-repo `#anchor` fragments resolve under GitHub's slug algorithm (em-dash headings produce double hyphens; handled). Clean. Not committed, per the standing Δ1 precedent from the 2026-06-12 pass.
- **Vendor-name scrub:** zero vendor or model names in any of the three new dives. Pre-existing vendor names in the README pairing tables and the `governance-sync.md` worked example are illustrative-tool references and were left in place; one stale claim about which products the shop pairs was corrected to a role description instead.
- **Internal-jargon grep:** clean across all prose after the Δ4 fix.

## Operator Action Block

None. Docs-only session; pushed.

## Next-session context

- The repo now has **22 dive documents.** `architecture.md` names eleven supporting disciplines, led by Model Orchestration and the Trust Gradient as paired meta-disciplines.
- The lesson count is **seven**. If a future pass adds an eighth, the same four locations need renumbering (README glance list, README Part I heading, `architecture.md` ×2, `getting-started.md`).
- Held for a future pass, in rough priority order: (a) the handoff-lifecycle discipline — how a handoff is opened, amended, and closed as an artifact, which currently only appears implicitly across `checkpoints-handoffs.md` and `delta-log.md`; (b) a public-endpoint anti-abuse standard; (c) the content-generation governance shape (voice guardrails single-sourced and propagated), which is real and generalizable but reads as niche unless the reader ships generated prose.
- `_deep-dive-template.md` contains a deliberate placeholder link (`other-facet.md`) and is excluded from link verification.
