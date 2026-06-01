# Infrastructure Limits

> Every vendor in your stack has limits, quotas, and prices. **Your AI assistants are confidently wrong about most of them.** Their training data is months or years old; vendor pricing and limits change quarterly; the gap between what the AI remembers and what the vendor charges today is where the most expensive mistakes live. This file is about closing that gap with a dated, operator-verified, in-repo source of truth that wins against memory every time.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Context Cascade](context-cascade.md) · **See also:** [Cost Management](cost-management.md), [Autonomous Loops](autonomous-loops.md), [Dependency Management](dependency-management.md)

---

## The wall

A team consolidates 90 cron jobs in a single hosting project because the Strategist "knows" the platform caps crons at 40 per project. The Strategist is wrong: the real cap is 100 — the docs changed eight months ago, after the model's training cutoff. The consolidation work was real engineering that solved a problem that didn't exist. Two days of effort, half a sprint of churn, no actual capability gained.

The deeper failure shows up the same week. The crons consolidation was triggered by *symptoms* — jobs not running, expected emails not arriving, dashboards going stale. The actual cause was that several cron-triggered endpoints required an auth header and the auth secrets had drifted out of sync across projects. Crons fired on schedule; endpoints returned 401; the hosting platform's dashboard showed every cron as "executed" and never alerted on the non-200 response. **The cron count wasn't the problem. The auth was the problem. The Strategist diagnosed against a remembered limit and missed the actual cause for weeks.**

This is the failure mode this discipline catches: AI assistants reason about infrastructure from stale memory, and infrastructure is exactly the surface where stale memory bites hardest. Limits move. Prices change. Free tiers shrink. Plan names get rebranded. New billable line items appear. Old ones get deprecated quietly. The vendor's docs are current; your AI's memory is not; the gap between those two is invisible from inside any single chat session — *until the bill arrives.*

The fix isn't to use a smarter AI. The fix is to put the *current truth* in a file the AI must read before reasoning about infrastructure, and to make it a discipline that the file gets updated by whoever changes infrastructure.

## The rule

1. **Every repo (or every family of repos sharing infrastructure) has an `infrastructure-limits.md` file** that lists every paid or limit-bound vendor in the stack: plan tier, limits, current usage, billing date, last operator verification.
2. **The file is dated, operator-verified, and beats AI memory.** If the file says one thing and a tool remembers another, the file wins — explicitly and in writing.
3. **Every session that touches infrastructure must read the file before reasoning.** Every session that *changes* infrastructure must update it before the session ends.
4. **The file lists known gotchas alongside the limits** — silent failure modes, deprecated patterns, "we tried this once and it cost $X." The gotchas are the scar-tissue layer that limits alone don't capture.
5. **Verification has a cadence.** Quarterly is a reasonable default; volatile vendors get monthly. The file's "last verified" line on each vendor is the audit signal.

## How it works

### Rule 1 — One file, every vendor that bills or limits you

The file lives at a predictable path — `docs/infrastructure-limits.md` for a single repo, `infrastructure-limits.md` at the root of a shared governance repo for a multi-repo enterprise. The path matters less than the convention: every Strategist and Executor in the system knows where to find it without asking.

The minimum content per vendor:

```markdown
## <Vendor> — <Plan tier>

**Plan:** <name>
**Billing cycle:** <e.g. 5th of each month>
**Base cost:** <$X/mo included>
**Current usage:** <last invoice or live read>
**Last verified:** <YYYY-MM-DD> by <operator/role>

### Limits

| Resource | Limit | Current | Notes |
| :---- | :---- | :---- | :---- |

### Pricing

| Resource | Rate | Notes |
| :---- | :---- | :---- |

### Gotchas

- <Silent failure mode, deprecation, "we learned this the hard way" item>
- …
```

A single page per vendor is plenty. The point isn't completeness — vendor docs are the canonical reference for that — it's having the *parts that matter to your stack* in one place the AI can read in seconds. A team with seven paid vendors ends up with a seven-vendor file that fits comfortably in a Strategist's context window alongside the relevant `AGENT.md` cascade.

### Rule 2 — The file beats memory, explicitly

The file's preamble carries an explicit precedence rule. Something like:

> **This file is the source of truth for vendor limits, plans, and prices. If it contradicts your training data, this file wins. If it appears stale (older than 90 days), flag it before relying on it.**

This sounds bureaucratic. It is load-bearing. Without the explicit instruction, a Strategist will routinely answer questions like "what's the cron-per-project limit?" from memory and never check the file — *especially* if the memory feels confident. The five-second test: *am I answering this from a file I just read, or from a number I just remember?* If the latter, stop and read the file.

The same rule applies to the Executor: a handoff that says "we can fit 90 crons in this project" gets verified against the file before execution, not against the Executor's memory of the platform's docs.

### Rule 3 — Pre-flight reads, post-action writes

Two coupled disciplines.

**Pre-flight: read before touching.** Any session that proposes to add a cron, a function, an integration, a new vendor, a new database, a new metered API call — *reads the file first.* This is a hard rule because the alternative is "I'll check if there's a problem later," and "later" is the bill.

In practice, this slot in the Strategist's workflow lives right after the `AGENT.md` cascade read: cascade for *what we do*, infrastructure-limits for *what we're allowed to do.* For the Executor, it lives in the QA gate — a handoff that touches infrastructure surfaces is rejected if it didn't cite the relevant infrastructure-limits section.

**Post-action: update before the session ends.** Any session that *changed* something the file describes — a new cron landed, a new vendor integrated, a plan tier changed, a usage spike happened, a new gotcha discovered — updates the file as part of the session's commit. This is the same discipline as updating the checkpoint or the changelog; it's not optional and it's not deferred.

The compounding effect: the file accurately reflects reality because every infrastructure-changing session leaves it accurate. A file that drifts loses the AI's trust within a month and reverts to ignored prose.

### Rule 4 — Gotchas are first-class

Limits and prices are the easy half. The harder half is the institutional memory: the *silent* failure modes, the cost spikes that took weeks to diagnose, the patterns you tried once and abandoned. These are what the file's "Gotchas" subsection captures.

Examples of what belongs in a gotcha — written from real scars, not from documentation:

- *"Cron auth failures are silent. The platform does not alert on non-200 responses to cron-triggered functions. Crons fire, endpoints return 401, the dashboard shows green. Diagnosed only after expected jobs failed to produce output for two weeks."*
- *"Build-minute overages are the dominant variable cost. An automated dependency-bump bot ran weekly and added $X/month in build minutes before anyone noticed. The bot is now disabled; do not re-enable without first accounting for build cost."*
- *"Function duration default is 60s, extendable to 300s. A daily sweep that calls 12 downstream endpoints in series can chain-timeout. Either parallelize or split into per-endpoint crons."*
- *"Per-second-billed compute does not sleep if anything keeps a connection open. A pooled DB client at module scope, plus a hosting platform that keeps function instances warm, equals 24/7 billing on an idle database. See the [metered-service trap](cost-management.md#the-metered-service-trap)."*

Each gotcha is a paragraph; each one prevents a category of future failure. A two-year-old `infrastructure-limits.md` with 20 well-written gotchas is one of the highest-leverage artifacts in the entire governance setup — it's a written form of every expensive lesson the team has learned. New Strategist sessions inherit those lessons by reading the file; without it, they re-learn them on the bill.

### Rule 5 — Verification cadence

A dated file with no refresh discipline is a future trap. Each vendor entry carries a "last verified" line; the operator runs a verification pass on a cadence:

| Vendor volatility | Cadence | What to verify |
| :---- | :---- | :---- |
| Stable mature platforms (cloud DNS, basic storage) | Annually | Plan tier and price unchanged; no new gotchas |
| Mainstream PaaS / IaaS | Quarterly | Limits, on-demand rates, any new metered line items |
| AI/LLM APIs, new-category vendors, anything in active product churn | Monthly | Everything — these vendors change pricing fastest |
| Anything where you just had a cost surprise | Until two consecutive clean cycles | Treat as volatile until proven calm |

Verification is a 10-minute read: log into the vendor's billing dashboard, confirm the plan and the per-resource costs match the file, scan for new line items, scan for deprecation notices, update the "last verified" date. Cheap to do, expensive to skip.

When a vendor's reality has drifted from the file, the update *is the session*: bring the file current, log the delta in the changelog, and — if the delta would have changed a recent design decision — surface it to the Strategist for review.

## When the Strategist is tempted to "I remember the limit is X"

Same shape as every other Strategist temptation. The five-second test: *am I answering from a file I just read, or from a number I just remember?* If the latter, the answer goes through the file or it doesn't go.

The pattern instead:

- **First reach is the file**, not memory.
- **If the file doesn't cover the question**, the Strategist either reads the vendor's live docs (and updates the file as part of the session) or asks the operator to verify (and updates the file with what comes back).
- **The answer in the handoff cites the file**: *"per `infrastructure-limits.md` §3 (verified 2026-05-30), the per-team rate limit is 5 req/s."* The citation makes the source visible and the verification date auditable.

This adds friction. The friction is the feature. A team that adopts this discipline catches one stale-memory mistake a quarter, and the catches compound — each one is a bill that never arrived.

## Worked example

A small team integrating a new transactional-email vendor. Without the discipline:

The Strategist drafts a handoff that says "the per-second rate limit on this plan is 10 req/s," based on memory of a blog post. The Executor builds a batch-send routine that fans out 8 sends per second. In production, the sends start failing with 429 errors after the first burst — because the actual current limit on this plan is 5 req/s, was reduced from 10 last year, and the blog post the Strategist remembered described the old limit. The batch silently sheds 30% of its sends over a weekend; a customer support ticket on Monday surfaces the gap.

With the discipline:

1. **Pre-flight read.** Strategist opens `infrastructure-limits.md` to draft the handoff. The transactional-email vendor entry exists (a colleague added it three months ago). Plan tier: "Pro." Current rate limit: "5 req/s per team, shared across all projects on the team." Last verified: two months ago.
2. **Handoff cites the file.** *"Per `infrastructure-limits.md` §Email (verified 2026-03-15), the team-wide rate limit is 5 req/s. Implement the batch with a 250ms stagger and a per-second cap of 4 to leave headroom for ad-hoc sends from other projects on the same team."*
3. **Executor implements against the actual limit.** The batch ships and runs clean.
4. **A gotcha gets recorded.** During implementation, the Executor discovers that the vendor's SDK returns an error object on rate-limit hits instead of throwing — every send must explicitly check `result.error`. That's a gotcha. The Executor adds a one-line entry to the file's gotchas section in the same commit: *"SDK returns error objects on rate limits, does not throw. Every send must check `result.error` or failures are silent."* The next Strategist session reads this and writes safer handoffs as a result.
5. **Quarterly verification.** Three months later, the operator's quarterly pass checks the vendor's pricing page. The rate limit is now 8 req/s on the Pro plan — the vendor raised it. The file is updated; the "last verified" date moves; the next session that touches email batches has accurate information.

The discipline traded a 15-minute file read per relevant session for a 30% silent-failure rate on a critical send path, plus the institutional memory of the SDK's silent-error pattern that would otherwise have to be re-discovered.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| Treat vendor docs as the source of truth and skip the in-repo file | The Strategist won't open vendor docs mid-design; it'll answer from memory | The file is the gate; vendor docs feed the file |
| Maintain the file but never put dates on entries | Without a "last verified" line, every entry decays into the same trust level | Dated entries, with the cadence per vendor |
| List limits but skip gotchas | The expensive failures are almost never about the limits | Gotchas are first-class; written as scar tissue, not as policy |
| Update the file "when we have time" | The time is now or never | Updates ship in the same commit as the infrastructure change |
| One global file across an enterprise of mixed stacks | Becomes a 50-page document no one reads in full | One per repo, or one per family, with the cascade rule resolving overlaps |
| Trust an AI's recollection of "I remember Vercel allows X crons" | Training data is months stale on a quarterly-changing platform | Every infrastructure claim cites the file or doesn't make it |
| Skip verification because "nothing changed" | The vendor changed; you didn't | Cadence runs regardless of whether you think anything moved |
| Quote a price without the verification date | A six-month-old price is a guess | Every price citation carries the date the file was verified |

## Adapt it to your setup

- **Solo, one repo:** the file is `docs/infrastructure-limits.md` at the repo root. Even with three vendors, the discipline pays off — the gotchas you write to yourself today save future-you the rediscovery.
- **Few repos, shared vendors:** the file lives in a shared governance repo or the repo of your most-central project, and the [context cascade](context-cascade.md) points other repos at it. Per-repo `AGENT.md` files reference it: *"infrastructure limits live in `<path>`. Read before designing any change that adds a cron, function, or vendor integration."*
- **Polyrepo enterprise:** family-level files for family-specific vendors; an enterprise-level file for shared infrastructure (the central hosting account, the shared DB, the shared email account). The cascade resolves overlaps the same way the [`AGENT.md` cascade](context-cascade.md) does.
- **Heavy vendor usage (20+ paid services):** the file becomes a directory — `docs/infrastructure-limits/<vendor>.md` per vendor, with an index file at `docs/infrastructure-limits/README.md`. Strategist reads the index, opens the relevant per-vendor file. Same discipline, more files.

## Related

- [The Context Cascade](context-cascade.md) — where the pre-flight read rule lives in the session-start protocol
- [Cost Management](cost-management.md) — `infrastructure-limits.md` is the data; the budgets and reviews in Cost Management are the action layer
- [Autonomous Loops](autonomous-loops.md) — the watchdog pattern automates the verification rule for live-billable surfaces
- [Dependency Management](dependency-management.md) — the same "dated, in-repo source of truth" discipline applied to lockfiles
- [Secrets and Rotation](secrets-and-rotation.md) — the same dated, operator-verified shape applied to credentials
- Back to the [main playbook](../README.md)
