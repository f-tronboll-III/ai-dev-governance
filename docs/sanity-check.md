# The Sanity Check

> A blind spec meets live production *before* it runs. The Executor verifies every assumption the Strategist made against the real database and filesystem, and stops if reality disagrees.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Two Roles](two-roles.md)

---

## The wall

The Strategist writes a clean, confident spec. It reads beautifully. It is also *wrong*, because the Strategist has never seen your production data — it's reasoning from the schema and from whatever you told it, not from what's actually in the tables right now.

Here's the one that earned this protocol. A spec said, reasonably: *"Rename this billing record to match the new client name."* Sensible on paper. But that record had **already issued invoices**, and the invoice line items referenced it by name. The rename went through. The next morning, paid invoices were displaying under the wrong client. The fix was a manual data-repair job and an awkward email.

Nothing was wrong with the *plan*. The plan was wrong about the *world*. No amount of better prompting fixes this, because the Strategist structurally cannot see the world — so the check has to live with the role that can.

## The rule

**Every handoff is a design, not a law.** Before executing any spec, the Executor reconciles it against live production. If reality contradicts the spec, the Executor stops and reports a *delta* rather than following the spec off a cliff.

## How it works

The Executor runs a **pre-edit pass** — four quick checks, in order, before touching anything:

1. **Data-state check** — query the records the spec assumes. Do they exist? In the state the spec expects? *(Does this billing record exist, and has it issued anything?)*
2. **Conflict check** — naming collisions, unique constraints, foreign-key relationships, live references. *(Will this rename break a name-based reference somewhere?)*
3. **Reversibility check** — flag any step that touches something already **sent, paid, or deployed.** These are the steps that hurt when they're wrong.
4. **Structural-context check** *(on cross-cutting work only)* — did the spec's author consult the [architecture graph](architecture-graphs.md), and is that graph current relative to the repo's recent architectural commits? A design written against a six-month-old structure map is blind in a second, quieter way. See [the Graph Consultation Pass](architecture-graphs.md#the-graph-consultation-pass) for the triggers and what a missing consultation obliges you to do.

If all four pass, the Executor proceeds normally. If any fails, it emits a **delta** and waits:

```text
## Sanity Delta — Handoff H-042, Step 3

SPEC SAYS:     Rename billing record "Acme" → "Acme Holdings LLC"
PRODUCTION:    Record id=4471 has 6 issued invoices; 2 are paid.
               Invoice line items reference the record by display name.
RISK IF RUN:   Paid invoices re-render under the new name → client-facing
               discrepancy on already-settled documents.
CORRECTION:    Add a display-name alias instead of mutating the record;
               leave issued invoices bound to the original name.
NEW CRITERIA:  New invoices use "Acme Holdings LLC"; the 6 issued
               invoices are unchanged. Verified by re-rendering invoice #5.

→ Awaiting operator approval before proceeding.
```

The **Operator** approves, adjusts, or rejects. Only then does the work run. The delta — and the decision — gets logged in the repo's `AGENT.md` changelog *and* as a one-liner in the cross-family [Delta Log](delta-log.md), so the reasoning survives the session and the pattern becomes visible across repos.

## Worked example

**Before (spec as written):** mutate record `4471` in place.
**Pre-edit pass:** data-state check finds 6 issued invoices on `4471`; reversibility check flags 2 as paid.
**After (delta applied):** add an alias, leave issued invoices bound to the original name, new invoices pick up the new name.
**Outcome:** the rename the Strategist *intended* happens for everything going forward, and nothing already settled silently changes underneath the client.

The Strategist's goal was met. The Strategist's *method* was corrected by the only role that could see why it was unsafe.

## Two checks that catch what the schema hides

The data-state check has two specific shapes worth naming, because both catch designs that are *structurally* correct and *factually* empty.

### Census the data before designing on schema

A design frequently rests on "the data's already there — the table exists, the column exists, the relationship exists." Schema presence is not data presence.

The scar: a recon pass mapped a donor-attribution pipeline from schema and code and concluded the substrate was already built. Every table it named did exist. Then the census ran. Linking records: five rows across three entities, none with the funding reference the feature needed. Payments attributed to a donor: zero of seven. Records carrying the donor-facing copy the UI would render: zero of forty-two. The "quick win" depended on an asset that was 0% authored, and the immersive logged-in experience had an addressable audience of one.

**The rule: one aggregate row-count pass per load-bearing table, up front, before scoping UI or estimating effort.** A `count(*)` and a `GROUP BY` take a minute and routinely reorder the entire plan — here, from "build the personalized portal" to "author the copy and ship the public surfaces first, and revisit the portal once its upstream data actually gets captured."

### Verify public reachability empirically

For any claim that a surface is publicly reachable — an endpoint, a feed, an image, a webhook target — **hit it and look at what comes back.** Confirm the response's content type is what you expect, not an HTML login page returning 200.

The same engagement surfaced three silent production breaks with exactly this shape: routes that read as public in the source but weren't, thanks to handler auth-gate ordering in one case and a missing middleware bypass in another. Code-reading *implies* reachability; a request *proves* it, and costs one command.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| Treating the spec as authoritative | It was written blind; production is the source of truth | Reconcile against live data first |
| Silent deviation | The operator can't trust results they can't see reasoned | Always emit a delta; never "fix it quietly" |
| Deviating to "improve" the spec | Scope creep dressed up as diligence | Deviate only to *reduce risk* (see [Bounded Deviation](bounded-deviation.md)) |
| Running the check but skipping the log | The next session repeats the mistake | Record the delta in the changelog |
| Reading the schema and calling the data-state check done | The table exists and holds five rows; the feature needs five thousand | Run an aggregate census on every load-bearing table |
| Handing the reality check to the human as an Operator Action | Inverts the whole protocol — the Executor is the role that can see production | Reads belong to the Executor, in every environment |
| "It's public, I read the route" | Auth-gate ordering and middleware bypasses make source misleading | Hit the live URL and check the content type |

## Adapt it to your setup

- **Solo, one repo:** you're the operator. The check is just a habit — before your Executor runs anything destructive, have it print "here's what the spec assumes vs. what I actually see" and eyeball it.
- **No database:** the same three checks apply to files, config, and deploy state — does the file exist, will this collide, is it already shipped?
- **Higher trust over time:** as patterns recur, promote the safe corrections into your family `AGENT.md` so the Executor applies them automatically (within the [Bounded Deviation](bounded-deviation.md) limits) instead of pausing every time.

## Related

- [Bounded Deviation](bounded-deviation.md) — the exact limits on when the Executor may deviate without asking
- [The Delta Log](delta-log.md) — where deltas are tallied so recurring ones become standards
- [The Two Roles](two-roles.md) — why the check belongs to the Executor, and where the Executor itself stops via [the Operator's hard line](two-roles.md#the-operators-hard-line-what-even-the-executor-doesnt-do)
- [Durable Memory](durable-memory.md) — recalled memories are design claims; the sanity check validates them against live state before action
- [Model Orchestration](model-orchestration.md) — the pre-edit Sanity Pass is non-down-modelable by rule; it stays on the heavy tier, at depth, no matter the deadline
- [The Trust Gradient](trust-gradient.md) — every handoff gets this pass regardless of which actor authored it, and un-anchored "verified" claims are themselves delta-worthy
- [Architecture Graphs](architecture-graphs.md#the-graph-consultation-pass) — the fourth leg: was the design grounded in a current structure map?
- [Operator Clearances](operator-clearances.md) — what the Executor may act on once the pass is clean, and what still stops at the human
- Back to the [main playbook](../README.md)
