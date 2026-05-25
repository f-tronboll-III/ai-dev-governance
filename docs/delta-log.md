# The Delta Log

> A delta logged in one repo is a story. The same delta logged in three repos is a missing family standard. The Delta Log is the cheapest way to spot that pattern before it costs you a fourth time.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Sanity Check](sanity-check.md), [Bounded Deviation](bounded-deviation.md)

---

## The wall

The [Sanity Check](sanity-check.md) catches a blind spec before it lands. [Bounded Deviation](bounded-deviation.md) lets the Executor correct the spec in narrow, file-anchored ways. Each individual delta gets logged in the affected project's `AGENT.md` changelog — and that's fine for one incident.

The problem shows up at scale. The same delta recurs in three different repos over six weeks: each time the Strategist writes a spec that assumes the old date helper, each time the Executor patches it to the new one, each time it gets logged into a different repo's changelog. No one notices the pattern because no one is reading all three changelogs in one sitting.

You haven't had three incidents. You've had *one missing family standard*, three times. The model says recurring deviations should be promoted into family-wide rules — but only if you can see they're recurring.

## The rule

**Every delta gets one line in the Delta Log**, in addition to its full entry in the project changelog. The Delta Log lives in the governance repo and exists for exactly one job: making *recurrence* visible.

- **One file**, governance-wide: `delta-log.md` in the root of the governance repo.
- **Append-only.** Newest at the top. Never edit past entries; supersede them with a new line.
- **One-line format**, parseable by eye and by grep.
- **The Strategist reviews it at session start**, looking for clusters. Three lines that rhyme = a candidate for promotion.

## How it works

### The line format

```
YYYY-MM-DD | <family>/<repo> | <handoff-id> | <one-phrase summary> | <link to full entry>
```

Example (with the billing-rename delta from the [Sanity Check](sanity-check.md) walkthrough):

```
2026-05-12 | family-b/billing-svc | H-042 | alias instead of mutate for invoiced records | family-b/billing-svc/docs/checkpoints/2026-05-12-h042.md
```

The discipline is **one line, one delta.** If the line wraps in your editor, the summary is too long. The full reasoning lives in the linked checkpoint; the log line exists only to let a reader scan a year of deltas in 30 seconds.

### When to write a line

Every delta written into a project changelog also writes one line to `delta-log.md`. This includes:

- **Sanity-Check deltas** that the Operator approved and the Executor then applied.
- **Bounded-deviation deviations** the Executor made on its own authority (all three gates passed).
- **Governance Sync conflicts** resolved by the tie-breaker rule (see [Governance Sync](governance-sync.md)).
- **Tier escalations or demotions** mid-task (see [The Two Roles](two-roles.md) — tier negotiation).

What *doesn't* go here: routine completions, clean handoff executions, anything where the spec was followed as written. The log is for the moments where reality made the spec change shape.

### The Strategist's session-start scan

At session start, after pulling the governance repo, the Strategist reads the top 20-or-so lines of `delta-log.md` looking for clusters. Three rules of thumb:

1. **Same summary phrase, different repos** — the textbook promotion candidate. Lift the correction into the family `AGENT.md` so future handoffs land right the first time.
2. **Same repo, three deltas in a month** — that repo's `AGENT.md` is missing context the Strategist isn't getting. Update the project's local rules.
3. **Same Strategist blind spot across families** — the Strategist is making the same kind of mistake regardless of where. Update the global `AGENT.md` or the handoff template.

When a cluster gets promoted, append a line to the log itself recording the promotion, with backlinks to the original deltas:

```
2026-06-01 | promotion | (n/a) | date-helper convention promoted to family-b standard | family-b/AGENT.md commit abc1234 — refs deltas 2026-05-12, 2026-05-19, 2026-05-26
```

That entry closes the loop: future deltas with the same shape shouldn't appear, because the rule that prevented them is now in the cascade.

### When the log itself is the warning sign

If the log is growing fast (say, more than ~2 deltas a week in steady state) and *nothing* is being promoted, something deeper is wrong — usually one of:

- The Strategist isn't reading the log at session start. (Fix the protocol.)
- The deviations look superficially different but share a root cause the summaries are hiding. (Promote at the root-cause level, not the surface level.)
- The Strategist *is* reading them but won't commit to a standard because each case "feels different." (That's a trust issue, not a model issue — the Operator decides.)

## Worked example

Three consecutive Friday checkpoints, three different `family-b` repos, three nearly identical deltas:

```
2026-05-12 | family-b/billing-svc | H-042 | alias instead of mutate for invoiced records | ...
2026-05-19 | family-b/crm         | H-051 | alias instead of mutate for customer-name field on contracts | ...
2026-05-26 | family-b/portal      | H-058 | alias instead of mutate for renamed account display | ...
```

At the next session start, the Strategist sees three lines that rhyme. The pattern: *renames against records with downstream references should add aliases, not mutate.* That's a `family-b` standard now. One commit promotes it into `family-b/AGENT.md`, and:

```
2026-06-01 | promotion | (n/a) | family-b rename policy: alias > mutate when downstream references exist | family-b/AGENT.md commit 89ab12c — refs deltas 2026-05-12, 2026-05-19, 2026-05-26
```

Future handoffs land right the first time. The Executor no longer has to deviate. The Sanity Check still runs — but it now passes instead of pausing for an Operator decision.

That's the whole loop. Bounded Deviation is the safety valve; the Delta Log is the meter that tells you when to fix the pressure source.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| Verbose entries | Defeats the scan-in-30-seconds purpose | One line. The checkpoint holds the detail. |
| Editing past entries | History stops being a history | Append; supersede via a new line if needed |
| Logging routine completions | Drowns the signal in noise | Only log shape-changes (deltas, deviations, escalations) |
| Reading it weekly instead of at session start | Promotions lag; the same delta recurs again before you act | Read at the top of every Strategist session |
| Never promoting | The log becomes a graveyard of repeated mistakes | If three lines rhyme, promote — or write down why you won't |

## Adapt it to your setup

- **Solo, one repo:** you don't need this yet. A single changelog is fine. Add the Delta Log the moment you have two repos that could share a rule.
- **Small team:** consider weekly Operator review of the log — the human is often the one who spots the pattern the Strategist normalizes.
- **Bigger setups:** the log can be split by family if it gets long, but resist that — the cross-family clusters are some of the highest-value promotions.

## Related

- [The Sanity Check](sanity-check.md) — where most delta entries originate
- [Bounded Deviation](bounded-deviation.md) — the three gates whose deviations the log counts
- [The Context Cascade](context-cascade.md) — where promotions land
- [Governance Sync](governance-sync.md) — sync conflicts also write a line to this log
- Back to the [main playbook](../README.md)
