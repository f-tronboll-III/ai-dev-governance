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
2. **Diff audit** — only the intended files changed.
3. **Security scan** — no hardcoded secrets, no unauthenticated admin routes.
4. **Hygiene** — remove stray debug logging and leftover TODOs.
5. **Acceptance criteria** — verify each one PASS/FAIL against the handoff.
6. **One-line QA report** — what was checked, what passed.
7. **Operator Action Block** — list any human-only manual steps.
8. **Push to main; confirm the host build is green.**
9. If many files changed, **regenerate the architecture graph** for the shared registry.

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

## Adapt it to your setup

- **Solo:** have the Executor print the checklist results so you can eyeball them — you're the gate.
- **Lower tiers:** Tier 0/1 get a lighter gate (type-check + diff audit); reserve the full list for Tier 2+.
- **CI:** automate steps 1–3 in a pipeline so the gate runs even when you forget to ask for it.

## Related

- [The Two Roles](two-roles.md) — the Executor owns the gate
- [The Sanity Check](sanity-check.md) — the pre-edit bookend to this post-execution gate
- [Post-execution QA — full overview](architecture.md#layered-protocol--post-execution-qa-tier-2)
- Back to the [main playbook](../README.md)
