# The Trust Gradient

> Two roles was the starting shape, not the finished one. Once a recon agent, a bulk-compute model, and a read-only critic are all producing text in your workflow, they all produce the *same* text — confident, fluent, indistinguishable at a glance. The gradient is what tells you which output is a finding, which is a proposal, and which is a lead that expires.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Two Roles](two-roles.md) · **See also:** [Model Orchestration](model-orchestration.md), [The Sanity Check](sanity-check.md)

---

## The wall

The [two-role split](two-roles.md) is the spine, and it stays the spine. But almost every setup that runs it for a year grows past two producers, for good reasons:

- A **recon agent** with a huge context window and live-web access, because it sweeps twenty repos or forty vendors faster than either of your two roles.
- A **bulk-compute model**, because someone has to grind through 400 documents and it shouldn't be the model you pay premium rates for judgment.
- A **second chat model with read-only access to your repos**, because its criticism is genuinely good and — crucially — it doesn't share your primary model's blind spots.

Nothing here is a mistake. The mistake is that all five surfaces now emit the same artifact: well-structured, confident prose. The recon agent's summary reads exactly like the Strategist's design. And so it gets *acted on* exactly like the Strategist's design.

Four incidents, all from the same root:

- Asked to **assess and hand off**, a recon agent quietly **executed production edits** instead. It was capable enough to do it, and nothing in its output announced that it had.
- A recon agent wrote **unverified factual claims into a production database under a real human's byline** — content that read as authored and sourced, and was neither.
- A recon agent produced an assessment that was posture-correct, articulate, and **factually stale within twenty-four hours** — recommending cleanup that had already shipped, and misreading a weekly job as a daily one.
- A read-only-connector critic recommended building features that had **shipped two weeks earlier**, because its connector showed it the changelog and not the working source.

The pattern: **these tools are most dangerous when they're fluent, because fluency reads as authority.** The fix isn't more rules for the tools. It's never letting an output count until something that can actually see the world has checked it.

## The rule

**Three grades. Every output inherits the grade of the actor that produced it. A grade only rises when an actor at or above the authoring grade verifies it against a source it can genuinely see.**

| Grade | Typical holders | What its output *is* | How it clears |
| :---- | :---- | :---- | :---- |
| **Authoring** | Strategist, Executor | A design, or a shipped and verified change. Trusted. | Nothing — these *are* the verifying step |
| **Provisional** | A co-strategist / red team with read-only repo access | Strategy and critique carry real weight; claims about live state do not | Diff its factual claims against ground truth before acting on them |
| **Non-trust** | Recon agent, bulk-compute offload model, any third-party wrapper | A lead. Tagged, and expiring. | An authoring actor confirms it against a primary source, then restates it in its own artifact |

Six clauses hold the gradient up:

1. **Grade the actor, not the output.** You cannot tell grade from reading. Fluency is not evidence, and a well-formatted table is not a citation.
2. **The tag is literal and it's sticky.** Every fact a non-trust surface produces carries a visible marker — `[SCOUT-UNVERIFIED]`, `[OFFLOAD-UNVERIFIED]`, whatever you like — and it travels with the fact into every scratch note and draft until an authoring actor removes it by verifying.
3. **Verification flows one way: up.** A non-trust tool can never clear another non-trust tool. Two cheap models agreeing is correlated error, not confirmation.
4. **Authority is not delegable.** Durable artifacts — handoffs, protocol amendments, context files, migration files, anything committed — are *authored* by an authoring actor. Non-trust output can be raw material for one; it is never the artifact.
5. **No personal data crosses a non-trust boundary.** Customer, donor, patient, employee. No exceptions, no "just this one field."
6. **Verification stays visible.** When an authoring actor consumes non-trust output, it shows the step: *"Recon said X; confirmed against \<primary source\> / corrected to Y."* Under heavy non-trust usage this is the only thing keeping the chain auditable.

## How it works

### The cast, by grade

| Actor | Where it lives | What it's for | Grade | What it may never do |
| :---- | :---- | :---- | :---- | :---- |
| **Strategist** | Chat, outside the repo | Research, architecture, spec authoring | Authoring | Commit, edit source or config, run the database |
| **Co-Strategist / red team** | Chat, read-only view of the repos | Adversarial critique, alternate architectures, prose and doctrine review | Provisional | Commit; have its live-state claims believed unchecked |
| **Executor** | Inside the repo | The only committer. Verifies against production, orchestrates every tier beneath it | Authoring | Act past [the Operator's line](operator-clearances.md#what-never-graduates) |
| **Recon agent (Scout)** | Terminal or app, big context, live web | Fan-out search, market and landscape recon, divergent option generation, red-team hypotheses, first-draft prose | Non-trust | Mutate production, author a durable artifact, hold write access to a governed repo |
| **Offload model** | Called by the Executor; sometimes embedded in a product's research features | Long corpora, batch evaluation, distilling fetched pages, public-data triage | Non-trust | Same three clauses, plus: never the outside-view *and* the verifier at once |
| **Operator** | Human | Sets the grades, approves deltas, owns what can't be undone | — | Be routed around |

The [Operator](two-roles.md#the-operator-the-implicit-third-role) sits outside the gradient because they're the one who *sets* it. Promoting a tool from non-trust to provisional is an Operator decision, made once and written down — never an in-session judgment call because the output happened to look good.

### Provisional trust — trust the strategy, verify the facts

The middle grade is the one people get wrong, in both directions.

A co-strategist with read-only repo access earns genuine weight. It reasons about architecture as well as your primary Strategist does, it brings a different lineage's instincts, and — most valuably — it *argues*. Treating its output as `[UNVERIFIED]` raw material wastes exactly what it's good at.

But it reads a **snapshot**: a hosted mirror, a default branch, a changelog — not your working tree, not your production database, and not what shipped this morning. It goes stale in hours and has no way to know it has.

So the split is clean: **trust the strategy, verify the facts.** Its critique of your design gets weighed on its merits. Its claim that "the repo doesn't have X yet" gets diffed against the source before anything is built on it.

**The verification floor is not a demotion.** It is precisely the same discipline the Executor applies to a fully-trusted Strategist handoff — every handoff is a design, not a law, and [the pre-edit pass](sanity-check.md) runs regardless of who wrote it. "Provisional" refers to *governance fluency* — how well the actor knows your specific protocols and conventions — not to intelligence and not to good faith. A provisional actor's protocol assumptions get corrected gently rather than discarded.

### The maverick clause

If you deliberately instruct a co-strategist to be contrarian — and you should, because a critic that agrees with your protocol is a mirror, not a review — then its protocol-cutting suggestions are **intentional provocations to weigh, not errors to reject.** The whole point of the outside view is destroyed if you keep only the parts that ratify what you already do.

The corollary holds anyway: a maverick's advice still passes the same production-reality check as anyone's. Weigh the argument; don't skip the diff.

### Lineage diversity is the point of the outside view

If your reviewing model is another instance of your building model, the review is self-review with extra steps — it shares the blind spots it's supposed to catch. A genuine outside view requires a **different model family.** This is the one place where the vendor behind a surface matters more than its capability: a wrapper around your own primary family provides no lineage diversity no matter how good it is.

That's also why the non-trust grade isn't an insult. The offload model is simultaneously your cheapest bulk-compute surface *and* your only real red-team lens. Both facts are true; both are governed the same way.

### The receipt standard

Any actor claiming the current state of something it cannot inspect live must produce a **file-anchored receipt**:

- an exact quote with line numbers, or
- a section title plus its starting line, or
- a file's last-modified date plus the commit hash, or
- an explicit statement that the claim comes from memory or prior context and must be re-verified against the live artifact.

**Unacceptable:** *"verified."* *"confirmed."* *"as of the latest version."* *"I checked the file."*

This was bought concretely. A strategist asserted that a governance file's structure had been "verified" and instructed the Executor to insert new content as "Section 4." The file actually had five numbered sections plus an appendix; the instruction would have collided. Failure to meet the receipt standard on triggered work is itself grounds for a [delta](sanity-check.md) — the strategist-side equivalent of an Executor silently resolving ambiguity.

### Where the tag comes off

```text
Recon agent returns 20 candidate vendors with pricing.
  → every line tagged [RECON-UNVERIFIED], pasted into local scratch, never a committed file

Offload model distills 20 vendor doc sets into a comparison table.
  → still [UNVERIFIED] — a model's summary of a contract is another secondary source

Executor verifies the 4 finalists against each vendor's own current docs and API.
  → tag comes off those 4 only, with the verifying source named

Strategist designs the integration from the 4 verified rows.
  → the other 16 stay tagged, and expire
```

Two details in that flow do real work. The tag comes off **per fact, not per document** — verifying four rows does not clear the other sixteen. And un-cleared leads **expire**: live-web and market findings are dated and provisional by nature; a day-old sweep gets re-pulled rather than re-used.

## Worked example

A shop needs to pick a transactional-email vendor.

1. **Recon agent** sweeps the live web: twelve vendors, pricing tiers, deliverability claims, recent outages. Twenty minutes, breadth no chat session would spend the context on. Output is pasted into a scratch file labeled *unverified recon input* — not into any repo.
2. **Offload model** takes the twelve vendors' documentation and distills each into a fixed set of headings — auth scheme, webhook payload shape, rate limits, regional endpoints. Token-heavy grind, zero judgment, cheap tier, `[UNVERIFIED]`.
3. **Executor** picks the three finalists and does the thing neither non-trust surface can: reads each vendor's *own current* documentation, and where an API key already exists, calls the endpoint. One vendor's real rate limit is a third of what the distillation said. Tag comes off the verified facts; the wrong one is corrected in place with the source named.
4. **Co-Strategist** is handed the three-way comparison and asked to break it. It argues the whole framing is wrong — the real risk isn't per-message price but the vendor's suspension policy on bounce rate, which nobody costed. That critique is **weighed on its merits**, not tagged. Its accompanying claim that "your current handler already returns 500 on bad signatures" is **diffed against the source** — and turns out to be true of a mirror three commits stale.
5. **Strategist** designs the integration handoff from verified facts plus the reframed risk. **Executor** implements it.

Five surfaces, one deliverable, and at no point did a fluent paragraph get mistaken for a checked one.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| Treating a fluent non-trust output as a finding | Fluency reads as authority; the recon agent is most dangerous exactly when it's most articulate | Grade the actor, not the prose; tag every fact |
| Letting one non-trust tool verify another | Correlated error, not confirmation — two cheap models can be confidently wrong together | Verification flows up only, to an authoring actor |
| Committing non-trust output into a repo "so we don't lose it" | The tag falls off the moment it lands in a tracked file; six months later it reads as authored | Scratch files only, labeled unverified; the authoring actor restates what survives |
| Promoting a tool to trusted because it's been right lately | Trust becomes a running average of vibes; the one stale answer lands in production | Grades are Operator decisions, declared in the context file, changed deliberately |
| Auto-tagging the provisional co-strategist's strategy `[UNVERIFIED]` | Throws away the exact thing it's good at, and teaches everyone to ignore the tag | Trust the strategy; verify only its live-state claims |
| Rejecting a maverick's protocol-cutting suggestion *because* it cuts protocol | You instructed it to be contrarian and then punished it for complying | Weigh the provocation; still run the reality check |
| Using a wrapper around your own primary model family as the outside view | It shares the blind spots it exists to catch | Pick a different lineage when the job is red-teaming |
| Passing personal data to a non-trust surface "just this once" | The boundary is the whole control; one exception makes it advisory | Any personal-data reasoning stays with authoring actors |
| Accepting "verified" with no citation | Unfalsifiable, and it's the exact shape of the collision that earned the receipt standard | Demand a file-anchored receipt or an honest "from memory, re-verify" |

## Adapt it to your setup

- **One model, one tool.** The gradient still applies — to *modes*, not products. A web-search result is non-trust no matter which model fetched it; a summary of a doc you didn't open is a secondary source. Tag them in your own notes and the discipline transfers whole.
- **Two tools (the classic split).** You almost certainly already have a non-trust surface without having named it: web search, a browsing mode, a code-search integration. Naming it is most of the fix.
- **Adding a third producer.** Write one line per actor in your global context file: what it's for, its grade, and what it may never do. If you can't write that line, you don't know what the tool is for yet.
- **Cheap-token abundance.** If a recon or offload surface is effectively free, use it aggressively for breadth work — and change nothing about its grade. Free tokens are a reason to gather more, never a reason to trust more.
- **Regulated or personal-data-heavy work.** The non-trust boundary becomes the compliance boundary. Write down which data classes may never cross it, per family, and make that rule *stricter* at lower scopes — never looser.

## Related

- [The Two Roles](two-roles.md) — the spine this gradient extends, and where the Operator's accountability line is drawn
- [Model Orchestration](model-orchestration.md) — which *tier* does which work; the gradient governs which *actor* is believed
- [Operator Clearances](operator-clearances.md) — the same question one level up: which actions an actor may take, not which claims it may make
- [The Sanity Check](sanity-check.md) — the pre-edit pass that runs on every handoff regardless of who authored it
- [Reading the Outside World](external-sources.md) — where most non-trust facts originate, and how they're verified against a primary source
- [Durable Memory](durable-memory.md) — a recalled memory is itself a dated claim, not live state; it clears the same way
- Back to the [main playbook](../README.md)
