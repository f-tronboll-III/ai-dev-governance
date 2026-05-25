# The Two Roles

> One AI that both plans and executes is its own reviewer — which means it isn't reviewed at all. Split it into a Strategist that designs and an Executor that ships, and let nothing cross the line between them except a written handoff.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** none — start here

---

## The wall

Hand a single AI the whole job and you get one of two failures.

Give your chat assistant filesystem access and it edits production blind — no type-check, no conventions, guessing at data it has never seen. Or let your terminal agent freelance the architecture and it commits hard to a half-formed plan, three files deep, before anyone weighed the design. A single agent that both plans and executes has no independent check on its own reasoning. That's fine until the day it's confidently wrong.

## The rule

**Two roles, one hard line.**

- The **Strategist** designs from outside the codebase and **never commits.**
- The **Executor** works inside the repo and is the **only** role that commits.
- The only thing that crosses the line is a **handoff** — a written spec the Executor implements under real tooling. Not a verbal "go ahead," not a snippet the Strategist wrote and the operator paraded in.

## How it works

The **Strategist** is your researcher and architect — strong at long-context reading, weighing tradeoffs, and writing detailed specs. It has four hard *nevers*: no editing source, no editing env/config, no database commands, no commits.

The **Executor** lives in the repo. It reads the [`AGENT.md` cascade](context-cascade.md) at startup, runs the build, tests, and DB queries, and pushes. Because it's the only role that can change the world, it's also the role that runs the [Sanity Check](sanity-check.md) — the check belongs with the eyes that can actually see production.

**Tier routing** decides who owns a task:

| Tier | Owner |
| :---- | :---- |
| 0 Hotfix · 1 Quick fix · 2 Standard | Executor |
| 3 Strategic (architecture, cross-repo, protocol) | Strategist designs → Executor executes |

The handoff is the interface between them: file paths, signatures, env-var names, an ordered plan, acceptance criteria, and an Operator Action Block of manual steps. See [Checkpoints & Handoffs](checkpoints-handoffs.md).

## Worked example

A Tier-3 task: *"add multi-tenant support."*

- The **Strategist** researches approaches, then writes a spec: the data-model changes, the migration order, the env vars required, explicit acceptance criteria, and an Operator Action Block (*"run the migration — you, not the agent"*).
- The **Executor** picks it up, sanity-checks against the live schema, implements step by step, runs the [QA gate](qa-gate.md), and pushes.

Contrast the anti-pattern: mid-chat, the Strategist says *"this is easy, I'll just edit the schema file"* and writes a migration against a data model it has never queried. That's Lesson 1 in one sentence, and it's exactly the line this split exists to hold.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| One agent plans *and* executes | No independent check on the plan | Separate the roles — even two sessions of the same tool |
| Strategist has commit access | Blind edits reach production | Strategist writes specs only; Executor commits |
| Executor invents architecture mid-task | Commits to an unreviewed design | Escalate to Tier 3; let the Strategist design first |
| "Just this once" line-crossing | The exception quietly becomes the norm | Hold the line; write the one-line spec |

## Adapt it to your setup

- **Solo:** you're the operator. The two roles can be two tabs or two terminals — a planning session and an execution session. The discipline is keeping them separate, not owning two products.
- **One tool that can do both** (a terminal agent that also plans well): keep the *discipline* of the split anyway — a distinct planning pass that produces a written spec, then a distinct execution pass against it. The artifact is the point.
- **Choosing tools:** Strategist = strong research / long-context / architecture, no repo access needed. Executor = repo-native, runs your build, commits. See the [actor table](../README.md#-works-with-any-strategist--executor).

## Related

- [Checkpoints & Handoffs](checkpoints-handoffs.md) — the artifact that crosses the line
- [The Sanity Check](sanity-check.md) — why the Executor, not the Strategist, validates against production
- [The Context Cascade](context-cascade.md) — the rules the Executor reads at startup
- Back to the [main playbook](../README.md)
