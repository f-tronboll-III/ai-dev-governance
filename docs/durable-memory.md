# Durable Memory

> A chat session ends and the AI forgets. Not just the conversation — the corrections, the preferences, the architectural decisions, the three different times you explained why a particular approach is wrong. Tomorrow's session opens with the same blank slate as yesterday's, and the same suggestions come back. **Durable memory is the discipline that fixes this without re-importing the entire repo into every session.** It's one fact per file, dated and typed, sitting in a directory the agent reads at session start — the cross-session knowledge store that turns repeated corrections into permanent ones.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Context Cascade](context-cascade.md) · **See also:** [Checkpoints & Handoffs](checkpoints-handoffs.md), [Bounded Deviation](bounded-deviation.md)

---

## The wall

An operator corrects the same mistake for the third time in two weeks: *"don't use the CLI to add environment variables on this hosting platform — it silently splits the value into three separate per-environment entries, and we end up with the production secret missing from preview. Always give me a copy-paste block instead."* The Strategist apologizes, acknowledges, drafts a new handoff that uses the CLI. The Executor catches it on review. The cycle repeats next month.

Nothing the AI did was unreasonable in isolation. Each session, it had no memory of the prior correction — and the CLI is the documented happy path for that vendor. Without a cross-session store of the operator's hard-won corrections, every session is the AI's first day on the job, and the operator's role degrades from collaborator to repeat-corrector.

The deeper failure shows up in *preferences and decisions*, not just gotchas. The operator told the AI in February that they prefer commit messages without emoji. In April, they decided that a particular table is the authoritative source for customer assignments and the array on the related record is derived. In May, they explained the company's standard tone for outbound email. None of those decisions live in the repo's code — they live in the conversation history of the sessions where they were made, which is to say *they don't live anywhere the AI can find.* Each one will be re-litigated until it's written down somewhere the AI will actually read.

The scar tissue is specific: **the agent that keeps re-proposing an approach you rejected three times.** That's the failure mode this discipline catches. Not "the AI doesn't know things" — the AI knows plenty. **The AI doesn't remember the *corrections you made* to what it knew.**

## The rule

1. **Durable memory lives in files, not in the AI's context window.** One fact per file, in a known directory, typed by category in frontmatter, with a human-readable index that the AI reads at session start.
2. **Memory is for what the repo doesn't already record.** Code is the source of truth for *how things work*. Memory is the source of truth for *operator preferences, recurring feedback, and decisions whose rationale lives outside the diff.*
3. **A recalled memory reflects the moment it was written, not the moment it was read.** Before acting on a memory, verify that the file, flag, or condition it references still exists. Stale memory lies confidently.
4. **The operator writes (or approves) memories; the AI reads them.** Memory is the operator's leverage over future sessions — it must reflect the operator's actual preference, not the AI's summary of what it thinks the operator preferred.
5. **The index is loaded; the full files are read on demand.** A flat list of every memory in every session blows the context budget; an indexed cascade matches the [context cascade](context-cascade.md) shape and scales to hundreds of entries.

## How it works

### Rule 1 — One fact per file, in a known directory

Every memory is a single Markdown file in a predictable location — `memory/` at the root of a governance repo, or `docs/memory/` inside a project repo. Each file holds one fact, one preference, or one decision. The file's name is its slug; the slug is descriptive enough to read in a directory listing.

A minimum memory file:

```markdown
---
name: <short title>
description: <one sentence the index can quote verbatim>
type: <feedback | project | reference | user>
created: <YYYY-MM-DD>
---

<The fact, in prose. Why it exists. How to apply it. What it prevents.>
```

The frontmatter types matter. They're the difference between *"a preference the operator has expressed"* (feedback) and *"a stable fact about the world this repo lives in"* (reference) and *"an active project's state and constraints"* (project) and *"who the operator is and how they like to work"* (user). The AI uses the type to decide how much weight to give the entry and how aggressively to verify it before acting.

One file per fact is non-negotiable. The temptation is to batch related feedback into one "preferences.md" document; the trap is that the document grows past readability, the AI starts skimming, and individual rules get dropped on the floor. One file per fact keeps each entry short, scannable, and link-targetable from the index.

### Rule 2 — Memory is for what the repo doesn't already record

The hardest part of memory is knowing what *not* to store. Anything that the repo's code, config, or `AGENT.md` files already encode does not belong in memory — the AI will read the canonical source and any drifting memory copy will eventually contradict it.

What belongs in memory:

- **Operator preferences not visible in code.** Commit-message style. Tone for outbound communication. Which platforms the operator wants daily reports from. How aggressive the Executor should be about refactoring on the side.
- **Recurring feedback that's been given more than once.** The third time you correct the same class of mistake, write it down. The third time is the signal that the AI's default is wrong for your stack.
- **Decisions whose rationale lives outside the diff.** "We picked vendor X over vendor Y because Y's webhook reliability was poor in a prior incident." That rationale won't survive in code comments; it survives as a memory entry the next architectural discussion reads.
- **Operator-facts the AI should know.** Name, role, time zone, project list, the names of recurring collaborators or clients. The AI uses these to write handoffs and reports that don't read like form letters.

What does *not* belong in memory:

- The current set of routes, the current schema, the current cron list. Code is the source of truth; a memory copy will drift.
- Last week's session summary. That's a [checkpoint](checkpoints-handoffs.md), not a memory.
- A one-time instruction. *"For this handoff, use the dev branch."* That's a chat-turn instruction. Memory is for what should hold across sessions.

The five-second test on every candidate memory: *would I still want the AI to behave this way in a session six months from now, even after the immediate context is gone?* If yes, it's memory. If no, it's a turn instruction.

### Rule 3 — A recalled memory reflects when it was written

This is the rule that the discipline lives or dies on. A memory file dated three months ago says *"the `subscribers` table holds verified emails only; unverified emails are in `pending_signups`."* That was true when written. Then someone merged the two tables in a refactor; now there's a single `subscribers` table with a `verified` boolean. The memory file wasn't touched. The next session reads it, drafts a handoff against the old schema, and the Executor catches the conflict — if the [sanity check](sanity-check.md) is running. If not, the handoff ships and breaks.

The rule: **before acting on a memory, verify that the world it describes still exists.** A one-line check — grep the schema for the table name, look for the file the memory references, confirm the flag the memory cites is still in the codebase. If the world has moved, update the memory in the same session as the fix.

This is why each memory carries a `created` date and, optionally, a `last verified` date. A memory file with `created: 2025-11-04` and no recent verification is *suspect by default* — it's older than most schemas, longer than most product cycles, and the prudent assumption is that something it references has moved.

The same discipline applies to feedback memories: *"the operator prefers X."* If the feedback is six months old and the operator has shipped three projects since, ask before assuming it still holds. Preferences age, too.

### Rule 4 — The operator writes; the AI reads

A memory written by the AI from its own summary of the conversation is *the AI's interpretation of the operator's preference, frozen.* That's not the same thing as the preference itself. The Strategist's summary of *"the operator wants commit messages without emoji"* might quietly become *"the operator wants minimalist commit messages,"* which is a different rule that the operator never agreed to.

The rule: **the operator writes the memory, or the operator reviews and approves the AI's draft before it lands.** In practice the workflow is:

1. The AI notices a candidate memory — a preference expressed twice, a correction given a third time, a decision with rationale.
2. The AI drafts the memory file and presents it to the operator inline: *"I'd like to record this: \<draft\>. Save it?"*
3. The operator confirms, edits, or rejects. Only on confirmation does the file land in the memory directory.

This is friction. It is also the only thing that keeps the memory store from becoming a fanfic version of the operator's actual preferences. A small directory of operator-approved memories beats a large directory of AI-inferred ones — the small directory will be trusted and acted on; the large one will quietly be ignored as unreliable.

### Rule 5 — Indexed cascade, not flat dump

A few dozen memories is fine to list flat at session start. A few hundred is not — the index alone starts to eat the context budget, and the AI starts skimming. The fix is the same shape as the [context cascade](context-cascade.md): an *index* loads at session start with one-line summaries; the *full files* are read on demand when the conversation touches their topic.

A working index entry:

```markdown
## Hosting platform gotchas
- `no-env-cli.md` — Never use the platform CLI to add environment variables; it silently creates split per-environment entries. Provide copy-paste blocks instead.
- `deploy-blocks-on-vuln-deps.md` — Platform blocks deploys when package-manager deps have known vulnerabilities; the advisory in the build log names the minimum version.
```

The index is one line per entry — enough for the AI to know whether to open the full file. The full file lands in context only when the session topic actually involves that vendor. A 200-memory store with a 50-line index is sustainable; the same 200 memories dumped flat would consume more context than the conversation itself.

Index sections are organized by topic, not by date or type — *"Hosting Platform Gotchas," "Email Infrastructure," "Operator Preferences"* — because the AI's retrieval question is *"do I have memories relevant to this topic?"* not *"what was added in March?"*

## When the Strategist drafts a handoff against memory alone

Same temptation as every other Strategist shortcut. The five-second test: *am I about to design against a memory I haven't re-verified, when the underlying file or table or flag the memory describes might have moved?* If yes, stop and verify.

The pattern instead:

- **Read the memory file in full**, not just the index line. The index summarizes; the file has the *why*, and the why is what tells you whether the memory still applies.
- **Verify the memory's referents.** A memory that says *"the cron at `/api/cron/sweep` runs daily and assumes the `Job` model"* gets verified against the live route and the live schema before the handoff cites it.
- **Cite the memory in the handoff.** *"Per `memory/no-env-cli.md` (operator-verified 2026-04-08), env vars are provided as a copy-paste block; do not run the platform's `env add` command."* The citation is what makes the memory's authority visible to the Executor and to future Strategist sessions reviewing the handoff.
- **If the memory contradicts what you find in the live code or schema, the live state wins** — and you update the memory in the same session.

## Worked example

A solo operator's third session on the same project. The Strategist is designing a handoff to add a new outbound email type to the system.

Without the discipline:

The Strategist drafts a handoff that includes *"add `EMAIL_PROVIDER_API_KEY` to the hosting environment via `provider env add` so the CI run can pick it up automatically."* The Executor implements. The CI run still fails on the deploy — the env var was added to "production" only, not to "preview," and the preview deploy is what runs the smoke test. Two hours of debugging surface that the CLI created a single-environment entry instead of the "all environments" entry the operator had previously asked for. The operator sighs and explains, *for the third time,* that this is why they keep saying not to use the CLI. The handoff is rewritten, the env var is re-added through the dashboard, the deploy ships.

With the discipline:

1. **Session start.** The Strategist's session-start protocol loads the memory index. The index has a section titled "Hosting Platform Gotchas" with three entries; one of them is *"Never use the CLI to add environment variables — creates split per-environment entries."*
2. **Drafting the handoff.** The Strategist opens the full file: `memory/no-env-cli.md`. It's six lines: don't use `provider env add`, here's why, provide a copy-paste `KEY=value` block instead, operator pastes into the dashboard. Dated 2026-02-14. Operator-verified.
3. **Handoff cites the memory.** *"Per `memory/no-env-cli.md` (operator-verified 2026-02-14): provide env vars as a copy-paste block; do not run `provider env add`. Operator Action Block: paste the following into the hosting dashboard under 'All Environments': `EMAIL_PROVIDER_API_KEY=<value>`."*
4. **Executor implements against the right contract.** The handoff lands cleanly. The operator pastes the block. Deploy ships on first try.
5. **A new gotcha gets recorded.** During implementation, the Executor discovers that the email provider's webhook signing requires a *separate* env var that the SDK reads at runtime. The operator confirms the discovery is real. The Strategist drafts a new memory entry: *"This provider's webhook signing reads `EMAIL_PROVIDER_WEBHOOK_SECRET` separately from the API key; both are required for inbound. Discovered 2026-05-31."* The operator approves; the file lands; the next session that touches inbound email reads it and writes a correct handoff on the first pass.

The discipline traded a 30-second memory read per session for the elimination of a recurring two-hour failure mode, plus the permanent capture of every new lesson as it's discovered.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| Store everything the AI summarizes as a memory | The store fills with AI-inferred preferences the operator never approved; trust collapses | Operator-approved or operator-written only |
| Copy the schema or route list into memory | Memory and code drift; memory eventually lies about the current shape | Memory is for what the repo *doesn't* already record |
| One large `preferences.md` instead of one file per fact | The file grows past readability; individual rules get skimmed and missed | One fact per file, indexed by topic |
| Read the full memory directory at session start | Eats the context budget before the conversation starts | Load the index; read full files on demand |
| Trust an unverified memory because the index summary sounds right | The index is a summary; the file has the *why* and the date | Always read the full file before acting; verify referents |
| Skip the `created` date on memory files | Without dates, every memory looks current; stale memories survive forever | Dated entries; a memory older than six months is suspect by default |
| Let the AI silently land a memory file mid-session | Operator never approved the wording; future sessions act on it as if they had | Drafts presented inline; only confirmed memories land |
| Treat checkpoints as memory | Checkpoints are session bookkeeping; memory is durable preference | Checkpoints in `docs/checkpoints/`; memory in `memory/` |
| Update memory only when something breaks | The store falls behind reality; trust erodes | Update in the same session as the fact changes |
| Use memory for a one-shot instruction | "Use the dev branch for this PR" is a turn instruction, not a memory | Turn instructions live in the chat; memory is for cross-session rules |

## Adapt it to your setup

- **Solo, one repo:** `docs/memory/` at the repo root, with an index at `docs/memory/INDEX.md`. Reference it in the repo's `AGENT.md`: *"Read `docs/memory/INDEX.md` at session start; load individual files when the session touches their topic."* Even a dozen memories pays off — the operator stops re-correcting the same things.
- **Solo, multi-repo:** memory lives in a shared governance repo, organized by family or project. The per-repo `AGENT.md` files reference the shared memory directory through the [context cascade](context-cascade.md). Per-repo memories that are repo-specific live in `<repo>/docs/memory/`; cross-repo memories live in the governance repo.
- **Team:** add an author field to the frontmatter so memories carry attribution. Disagreements about a memory get resolved like any other governance disagreement — the operator who owns the affected surface decides. Tag-team review of new memory files before they land prevents one teammate's preference from becoming everyone's rule.
- **Heavy memory volume (200+ entries):** subdirectory by topic — `memory/vendors/`, `memory/preferences/`, `memory/projects/` — each with its own sub-index, and a top-level `INDEX.md` that points to the sub-indexes. The cascade resolves the same way the [`AGENT.md` cascade](context-cascade.md) does. At this scale, periodic pruning becomes its own discipline: memories older than a year get re-verified or archived, on a cadence.

## Related

- [The Context Cascade](context-cascade.md) — durable memory is loaded the same way as governance files: indexed at session start, full files on demand
- [Checkpoints & Handoffs](checkpoints-handoffs.md) — checkpoints capture *what happened this session*; memory captures *what should hold across sessions*
- [Bounded Deviation](bounded-deviation.md) — a memory is what a Sanity Delta becomes once it's promoted from a one-time exception to a standing rule
- [Sanity Check](sanity-check.md) — verifying a memory's referents against live code is the same shape as the pre-edit sanity pass
- Back to the [main playbook](../README.md)
