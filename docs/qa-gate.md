# The QA Gate

> "Done" can't mean "the code is written." It means shipped and verified against a fixed checklist — because the cheapest place to catch a regression is before you call it complete. The deploy *is* the review.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Two Roles](two-roles.md)

---

## The wall

The agent says *"complete."* But the commit isn't pushed. Or there's a hardcoded key it added "just to test." Or it touched three files you never mentioned. Or the build is green locally and red on the host. Without a fixed definition, "done" means whatever the agent felt like in the moment — and the gap surfaces in production, the most expensive place to find it.

## The rule

Tier-2-and-up work isn't done until it passes a **fixed, ordered checklist.** No checklist, no "complete." And **never mark complete with unpushed commits** — if it isn't pushed and built, it isn't anywhere.

## How it works

The Executor runs these in order; any failure blocks "done":

1. **Type-check** to zero errors.
2. **Diff audit** — only the intended files changed, and nothing the feature depends on is left uncommitted (check for untracked files the new code imports).
3. **Security scan** — no hardcoded secrets, no unauthenticated admin routes, and every new public form, endpoint, or server-side action carries the guards its surface type requires.
4. **Hygiene** — remove stray debug logging and leftover TODOs.
5. **Defect disposition** — every warning, error, or failing check observed this session gets D1, D2, or D3; the report names them or says "none observed." See [Defect Disposition](defect-disposition.md).
6. **Acceptance criteria** — verify each one PASS/FAIL against the handoff.
7. **One-line QA report** — what was checked, what passed, and the defect-disposition line.
8. **Announce block** — one line per production-structural change actually applied this session: schema columns added, environment variables set, data fixes run. This is what replaces the approval gates the agent no longer needs; see [Operator Clearances](operator-clearances.md).
9. **Operator Action Block** — list any human-only manual steps. If a routine, cleared step is sitting in here, that's a bug in the clearance, not a step for the human.
10. **Push to main, then verify at the destination.** A successful push is never evidence that anything is live: read the host's deployment state for *that commit* — *blocked* and *canceled* are the silent tells — and, for anything meant to be reachable, fetch it from outside. See [Verify by Destination](verify-by-destination.md).

Step 3's scan and step 5's lint output are themselves checks, and inherit everything in [Honest Checks](honest-checks.md): a scan that couldn't run is not a scan that found nothing.

## Worked example

A feature looks done. The **diff audit** (step 2) shows a change to an auth-middleware file the task never mentioned — the agent "tidied" it while passing through. The gate blocks completion. The stray change is reverted, or escalated as its own task per [Bounded Deviation](bounded-deviation.md).

Without step 2, that change ships silently, and someone spends next Tuesday debugging a mysterious auth regression with no idea it rode in on an unrelated feature.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| "Done" = code written | Unpushed / unverified work breaks later | Done = passed the gate + pushed + build green |
| Marking complete with unpushed commits | The "done" work isn't anywhere | Push first, then claim done |
| Skipping the diff audit | Unintended changes ship silently | Always audit the diff |
| "Looks good to me" | Vibes aren't a gate | Run the fixed checklist, in order |
| Secrets in code "temporarily" | Temporary lives forever in git history | Block on the security scan |
| Applying a production schema change and not announcing it | The human loses the plot on their own infrastructure, quietly | One line per structural change, every session |
| Padding the Action Block with routine pastes | Teaches the operator to skim the one item that actually matters | Routine → clearance; the block holds only what can't graduate |
| "Fixed and pushed" | The host may have blocked or canceled the build, with no log | Report the deployment's state for that commit |
| "Pre-existing warnings, not related" | Nobody rereads the summary; the warning baseline normalizes | A disposition per observation, named in the report |
| Treating a scan that exited early as clean | "Couldn't look" reads as "nothing there" | The scan reports its coverage; a partial run isn't a pass |

## Adapt it to your setup

- **Solo:** have the Executor print the checklist results so you can eyeball them — you're the gate.
- **Lower tiers:** Tier 0/1 get a lighter gate (type-check + diff audit); reserve the full list for Tier 2+.
- **CI:** automate steps 1–3 in a pipeline so the gate runs even when you forget to ask for it.
- **No clearances granted yet:** step 8 collapses into step 9 — everything structural is still a human step. Adopt the announce block anyway; it's how you'll later notice which gates never catch anything.
- **Lint ceiling:** the cheapest piece of step 5 to automate — `--max-warnings <today's count>` in the lint script, lowered whenever the count drops.

## Related

- [The Two Roles](two-roles.md) — the Executor owns the gate
- [Operator Clearances](operator-clearances.md) — what belongs in the announce block versus the Action Block
- [The Sanity Check](sanity-check.md) — the pre-edit bookend to this post-execution gate
- [Defect Disposition](defect-disposition.md) — step 5 in full
- [Verify by Destination](verify-by-destination.md) — step 10 in full: what "live" means for each kind of change
- [Honest Checks](honest-checks.md) — the gate is a set of checks; this is how to keep each one believable
- [Post-execution QA — full overview](architecture.md#layered-protocol--post-execution-qa-tier-2)
- Back to the [main playbook](../README.md)
