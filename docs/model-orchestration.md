# Model Orchestration

> Your top-tier reasoning model is also the one doing the grepping. Down-model the *gathering* and keep the *judging* on the heavy tier, and the same agentic loop goes from waste-prone to disciplined — without losing an ounce of quality on the decisions that matter.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Two Roles](two-roles.md) · **See also:** [Cost Management](cost-management.md), [The Sanity Check](sanity-check.md), [Bounded Deviation](bounded-deviation.md)

---

## The wall

You point your best model at a task — "figure out how auth works in this repo and tell me whether the new SSO flow fits." It's the right model for the *whether*: that's judgment, and judgment is what you pay top-tier rates for. But before it can judge, it spends twenty tool calls finding files. It greps for `auth`. It lists the routes directory. It reads six middleware files to find the two that matter. It enumerates env vars. Every one of those calls runs on the heavy model, at heavy-model cost, and — worse than the cost — every file it reads to *locate* the answer is now sitting in the context window, crowding out the room it needs to actually reason about the SSO question when it finally gets there.

So you've paid the premium tier twice over: once in tokens to do mechanical work a far cheaper model does just as well, and once in quality, because the model arrives at the actual judgment with its context half-full of grep output. The synthesis — the part that needed the good model — happens on a tired, cluttered context. The expensive model did the cheap work *and* did the expensive work worse for having done it.

The fix isn't a smarter model or a bigger context window. It's a division of labor. The heavy model should never have been the one finding filenames.

**The five-second test, before any task:**

> *Am I about to run the top-tier model to find filenames, or am I about to run it to make a judgment? If it's filenames, that's the cheapest model's job — and I'm paying heavy rates for the privilege of cluttering the context window.*

## The rule

Five rules, one principle. The principle: **optimize for waste elimination, not spend minimization.** Cost isn't the enemy — the heavy tier is worth every token when the task is judgment. Unjustified token burn on mechanical work, and quality drift from a cluttered context, are the enemy.

1. **Heavy is the default; you justify routing DOWN, never staying Heavy.** The burden of proof runs one direction only. The absence of a reason to down-model *is* the instruction to stay on the heavy tier. This makes the safe choice the lazy choice and removes any need for a "keep this on the good model" flag.
2. **Down-model the gathering; keep the judging on Heavy.** Search, grep, read, enumerate, bulk-extract, reformat — that's the cheap tier's job. Decide, prioritize, weigh tradeoffs, synthesize — that stays on the heavy tier. (Sometimes the gathering belongs to a *different vendor* entirely — see [Non-trust tools](two-roles.md#non-trust-tools--instruments-not-actors).)
3. **The Sanity Pass is non-down-modelable, full stop.** There is no deadline, token budget, or time pressure that licenses running the [pre-edit Sanity Check](sanity-check.md), a delta judgment, or a [bounded-deviation](bounded-deviation.md) decision on a cheaper tier. It is judgment on production reality; it stays Heavy by construction of Rule 1.
4. **One gather contract, parameterized — not a template library.** There is exactly one set of instructions that turns a model into a gatherer. You vary the *task* (what to collect, under what headings), never the contract. Per-task-type variants are a drift surface; resist them.
5. **Fan out for breadth without cross-dependency; never for work that needs shared context between agents.** Parallel cheap-tier agents are for independent targets each can answer blind. The moment two agents need to see each other's findings, it's one agent's job — or a workflow with a barrier — not a fan-out.

## How it works

### When to route DOWN (and how far)

Route a subagent or stage to the **Light** tier (the cheapest) when **all three** hold:

- The task is read-only, or its output is non-authoritative (the heavy model verifies before using it).
- Success is mechanical and pattern-matchable — *find X, list Y, extract Z* — with no architectural judgment.
- A wrong result is cheap to catch when the heavy model reviews the return.

Route to the **Mid** tier when the work needs real reasoning but not top-tier judgment: a bounded-diff code review, drafting copy or docs to a clear spec, moderate summarization, self-contained per-item transforms in a pipeline.

Keep it on **Heavy** — no delegation, or delegate only sub-parts — when **any** of these holds:

- The output mutates production, or gets acted on without a downstream verifier.
- It's a Sanity Check step (Rule 3 — never down-modeled).
- It's architecture, cross-project design, or a decision that triggers a knowledge-graph or registry update.
- It requires holding large cross-file context and reasoning over contradictions.
- It's ambiguity resolution that, if wrong, expands scope or risk.

**The rule of thumb that subsumes the rest:** down-model the *gathering*, keep the *judging* on Heavy. Search and read with the cheap tier; reason and decide with the good one.

### The gather contract, in full

There is one contract, reproduced here verbatim. Every down-modeled gather — a subagent, a workflow stage, or an [external non-trust tool](two-roles.md#non-trust-tools--instruments-not-actors) — is dispatched with this prepended, and only the task parameterized:

> You are in GATHERING MODE ONLY. You collect, extract, and report raw material. You MUST NOT reason about significance, recommend, prioritize, rank, judge tradeoffs, propose architecture, or draw conclusions. Report facts and direct quotes with locations. If asked for an opinion, return the relevant raw material instead and state you are in gather mode. Your output is unverified input to a verifier — never a decision.

Don't reword it to "improve" it. The wording is the function: every clause closes a specific way a cheap-tier gatherer leaks judgment into what should be raw material. When you dispatch a gatherer, prepend this and specify only *what to collect, under what headings* — never write a second variant for a second task type.

### Judgment-leakage watch-words — a smell, not a gate

A gatherer will sometimes editorialize despite the contract. Treat these words in a cheap-tier agent's output as a **smell** that judgment leaked in — when you see them, use the surrounding text as raw material only, and re-derive the judgment yourself on the heavy tier:

> *recommend · should · prioritize · best · optimal · I suggest · the right approach · architecture · tradeoff · instead we could · more important*

This is a heuristic for your eye, not an automated linter. Don't build tooling around it — the moment it becomes a gate, gatherers learn to launder the same judgment into words not on the list.

### The fan-out pattern

When the work is breadth across many independent targets — files, directories, sub-questions — with **no cross-dependency** (each can be answered blind to the others), the shape is:

- One cheap-tier gatherer per independent target, each running under the gather contract.
- The parent prompt specifies **exact output headings** so the returns compose cleanly.
- The heavy tier then synthesizes across all returns — and *that* synthesis is the judging, so it stays Heavy.

Don't fan out work that needs shared context between agents, and don't let any fan-out agent synthesize across the others' findings. Breadth fans out; judgment converges.

### The mechanism, whatever your harness

- **Subagents:** pass an explicit model tier on the delegation. Default mechanical fan-out to the cheap tier unless a stay-Heavy rule trips.
- **Workflows:** assign a tier per stage — finder and reader stages cheap, synthesis and verify stages Heavy. Record the per-stage tier where the workflow's shape is written down, so the routing is auditable.
- **Disclosure:** when a result that fed a decision came from a down-modeled agent, note the provenance. The human reading the conclusion should know which tier gathered the evidence under it.

## Worked example

A feasibility scan across two unrelated projects: *"can each of these two codebases support a new third-party integration, and what would it cost in refactoring?"* Two projects, no overlap between them — a textbook fan-out.

```text
Heavy (orchestrator) dispatches two independent gatherers:

  Gatherer A  →  Project 1, gather contract, fixed headings:
                   [Integration points] [Auth model] [Blocking constraints] [Relevant env vars]
  Gatherer B  →  Project 2, gather contract, same fixed headings
                 (Light tier · read-only · blind to each other · no cross-talk)

         ↓ both return raw material under identical headings ↓

Heavy synthesizes: compares the two returns, weighs refactor cost,
                   makes the feasibility call. THIS is the judging — Heavy.
```

The shape is the whole lesson: two cheap gatherers + one heavy synthesizer + zero cross-talk between the gatherers. Each gatherer reads its own project and reports facts under fixed headings; neither is asked whether the integration is *a good idea* — that question never leaves the heavy tier. The scan costs a fraction of running the heavy model over both repos, and the synthesis lands on a clean context holding exactly the comparable material it needs.

The anti-pattern version: one heavy agent reads both repos end to end, arrives at the feasibility question with both codebases jammed into its context, and bills the premium tier for forty file-reads it never needed to do itself.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| A cheap-tier agent *recommends* an approach | Recommendation is judgment; the cheap tier wasn't given the context to make it well, and its confidence reads the same as the heavy tier's | Gatherer returns raw material; heavy tier recommends |
| A cheap-tier agent *ranks* options | Ranking is prioritization — judgment wearing a list's clothing | Gather the options with their attributes; rank on Heavy |
| Down-modeling the synthesis step | The synthesis is the one step that needed the good model; down-modeling it is down-modeling the whole point | Synthesis is always Heavy, no exceptions |
| Using a gatherer's embedded opinion as a conclusion | The contract forbids opinions for a reason; a leaked one is unverified and wrong-shaped | Treat any opinion in gather output as a smell; re-derive on Heavy |
| Building a template library of gather prompts | Every variant is a drift surface; they fall out of sync and encode subtly different rules | One contract, parameterize the task only |
| Fanning out work with cross-dependency | Agents blind to each other produce incoherent partial answers that don't compose | One agent, or a workflow with a barrier |
| Down-modeling the Sanity Pass under deadline pressure | The pass is judgment on production reality; a cheap-tier miss here is a production incident | Sanity Pass is Heavy by construction — Rule 3 |
| A project file that *weakens* the floor instead of tightening it | A lower-scope file that permits a down-model the floor forbids quietly reopens the failure mode everywhere it applies | Lower scopes may only add *stricter* rules, never loosen |

## Adapt it to your setup

- **Solo, one repo:** the rule lives in your one `AGENT.md` as a sentence or two — *"mechanical fan-out (search, read, enumerate) goes to the cheap tier; judgment, synthesis, and the Sanity Pass stay on the heavy tier."* You don't need a tier ladder document; you need the habit of asking the five-second test before you spend the good model on grep.
- **A team:** put the orchestration rule in the enterprise-level (global) context file as a **floor**. Per-project files may add *stricter* constraints for their context — a project handling personal data forbids passing that data to any down-model gatherer at all — but no project file may *weaken* the floor to permit a down-model the floor forbids. Lower scopes tighten; they never loosen.
- **Enterprise scale:** the floor is global, the stricter rules are per-family, and the discipline extends past your own model family — gathering can be handed to an [external non-trust tool](two-roles.md#non-trust-tools--instruments-not-actors) under the same gather contract and the same "output is unverified input, never a deliverable" posture. Revisit the routing rules on a rhythm you already keep (a quarterly review you already run), rather than standing up a separate timer to remember to.

The smallest version that still works is one sentence in one file: *down-model the gathering, keep the judging on the heavy tier.* Everything above is that sentence, with the failure modes that justify each clause.

## Related

- [Non-trust tools](two-roles.md#non-trust-tools--instruments-not-actors) — extending "down-model the gathering" outside your own model family; the gather contract and the unverified-input posture apply unchanged
- [The Sanity Check](sanity-check.md) — the pre-edit pass that is non-down-modelable by rule
- [Bounded Deviation](bounded-deviation.md) — judgment about whether to deviate is never delegated to a cheaper tier
- [Cost Management](cost-management.md) — the right framing is waste elimination, not spend minimization; model orchestration is the lever
- [The Delta Log](delta-log.md) — a recurring delta about model choice on a class of task is the signal to revise these routing rules
- Back to the [main playbook](../README.md)
