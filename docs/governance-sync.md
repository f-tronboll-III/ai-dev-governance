# Governance Sync

> Strategist and Executor never talk to each other directly. They share a repo of files — and the moment those files drift between roles, the whole model breaks. Governance Sync is the boring, load-bearing protocol that keeps the shared files honest.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Two Roles](two-roles.md), [The Context Cascade](context-cascade.md)

---

## The wall

The two-role split solves one problem (a blind planner editing code) by creating another: now you have two AIs running in two different processes, on two different schedules, both able to touch the *shared governance files* that define how the model itself behaves.

The Strategist updates a family `AGENT.md` with a new convention while the Executor is mid-session running the old one. The Executor finishes a phase and writes a new entry to the project changelog while the Strategist, two hours later, opens the same file from a stale clone and overwrites it. A cron-driven Executor session starts before the Strategist's last push landed and reads governance that's a day behind reality.

None of these are exotic. Each one quietly desyncs the two roles. The model assumes they share state *through files* — so when the files diverge, the model is silently running on two different realities.

## The rule

Treat the governance repo as a **distributed source of truth with a fixed merge discipline.** Both roles pull at session start, both push at session end, and the merge rule is declared up front so neither role has to invent it under pressure.

- **At session start:** pull the governance repo, **fast-forward only.** If FF fails, halt the session and run the conflict protocol below — do not start work on stale state.
- **At session end:** if any shared governance file changed, commit (with a role-tagged author) and push before declaring the session complete. No "I'll push it next time."
- **Per-repo `AGENT.md` files** live in their own project repos and are read in place; they are *not* synced through the governance repo.
- **Strategist wins on governance shape; Executor wins on execution state.** When the same file is touched by both and a non-FF merge is unavoidable, this is the tie-breaker rule.

## How it works

### Session-start sync

Both roles begin a session with the same three lines:

```bash
cd <governance-repo>
git fetch origin
git merge --ff-only origin/main
```

If `--ff-only` succeeds, the session proceeds. If it fails, the local clone has diverged from the remote — there are commits on `main` the local doesn't have, *and* local commits the remote doesn't have. That's the conflict case. Stop. Do not start work.

### Session-end sync

The Executor's session-end protocol already includes pushing any project repo changes. Governance Sync adds one step: if the session also modified any file in the shared governance repo (an `AGENT.md` cascade file, a handoff under `handoffs/`, a roadmap index entry, a Delta Log entry, the agent memory state), commit those changes with a role-tagged author and push.

Recommended author tagging makes the audit trivial later:

| Role | `user.name` | `user.email` |
| :---- | :---- | :---- |
| Strategist (Perplexity, ChatGPT, Claude web, etc.) | `Strategist (<tool>)` | a stable email you control |
| Executor (Claude Code, Codex, Cursor, etc.) | `Executor (<tool>)` | a stable email you control |
| Operator (human, when editing governance directly) | your normal git identity | your normal email |

The point isn't bureaucracy — it's that when you grep the log later asking *"who made this rule and when,"* the answer is one command away.

### The conflict-resolution protocol

When `--ff-only` fails, do not `git pull --no-ff`, do not `git rebase` blind, do not `git push --force`. Run the protocol:

1. **Fetch and diff.** `git fetch origin && git log --oneline HEAD..origin/main` and `git log --oneline origin/main..HEAD` to see exactly what's on each side.
2. **Classify each side's commits.** Each commit is one of two shapes:
   - **Governance-shape** — a rule changed, a convention added, a family `AGENT.md` edit, a new handoff spec. (Strategist territory.)
   - **Execution-state-shape** — a changelog entry, a roadmap completion, a Delta Log entry, a memory-state update. (Executor territory.)
3. **Apply the tie-breaker.**
   - If the conflicting hunks are *governance-shape*, the **Strategist's version wins.** The Executor's local commits are rebased on top, dropping any governance edits the Executor wrote on its own. (If the Executor needed to encode a governance change, that's a Strategist handoff, not a unilateral edit.)
   - If the conflicting hunks are *execution-state-shape*, the **Executor's version wins.** The Strategist was guessing at production state; the Executor saw it.
   - If they collide on the *same line* of the same file with different shapes, escalate to the Operator. Don't pick.
4. **Rebase, push, and log.** After resolving, push, and write one line to the [Delta Log](delta-log.md): *"Sync conflict on `<file>` — Strategist won on shape change / Executor won on state — see commit `<sha>`."* This is how you spot a pattern (the same file conflicting repeatedly = a governance shape problem, not a sync problem).

### When fast-forward is impossible by design

A few legitimate cases produce a divergence that isn't actually a conflict:

- **Out-of-band Operator edits.** The Operator fixed a typo or merged a PR on the GitHub web UI. Pull normally; no agent action needed.
- **Two agents in overlapping windows.** A cron-fired Executor session ran while the Strategist was mid-edit in a chat session. Whoever pushes second runs the conflict protocol; the one that already pushed is fine.
- **Forked workflow** (rare). Some teams deliberately keep a Strategist branch and an Executor branch that merge daily. The protocol above still applies at merge time; the only difference is the merge happens at a scheduled cadence instead of every session.

## Worked example

A Friday afternoon: the Strategist (Perplexity) opens a chat session and edits `families/family-b/AGENT.md` to promote a delta — *"all `family-b` repos now use the same date-formatting helper from `@org/shared-dates`."* It pushes the change.

Twenty minutes later, the Executor (Claude Code) starts a session — but its local clone is from this morning. It pulls and `--ff-only` succeeds because nothing local was uncommitted. Good. It reads the new rule and applies the helper in the project it's working on.

Now imagine the unlucky version: the Executor's last session, two days ago, also touched `family-b/AGENT.md` — adding a changelog line for a phase that shipped. That commit was made locally but never pushed (the previous session ended in a timeout before Governance Sync could run). When today's session starts:

```
$ git merge --ff-only origin/main
fatal: Not possible to fast-forward, aborting.
```

The Executor runs the conflict protocol:

- **Remote side** has the Strategist's date-helper rule (governance-shape).
- **Local side** has the changelog entry from two days ago (execution-state-shape).
- They don't collide on the same line — easy rebase. Local commit gets reapplied on top of the Strategist's; both land on `main`. The Executor logs one line to the [Delta Log](delta-log.md): *"Recovered unpushed Executor changelog commit during Friday sync; no shape conflict, clean rebase."*

Without the protocol, this is the session that quietly drops the changelog entry, or force-pushes over the Strategist's rule, or worst of all, picks a side based on whatever the agent guessed.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| `git pull` without `--ff-only` | Implicit merges hide drift; you don't know what was reconciled | `--ff-only` always; force the conflict to surface |
| `git push --force` to "fix" a sync issue | Erases the other role's work silently | Run the conflict protocol; let the tie-breaker decide |
| Skipping the push at session end | Next session starts on stale state; conflicts pile up | Push is part of "session complete," not an optional cleanup |
| Editing governance from the Executor's project repo | Splits the source of truth | Governance changes go *through* the governance repo, with the Strategist's hand |
| Reusing one git identity for both roles | The audit log can't tell who did what | Tag the author by role |
| Letting unpushed commits accumulate | Each one is a future merge conflict | Push at session end, every session, no exceptions |

## Adapt it to your setup

- **Solo:** the protocol is the same, but the conflict case is rarer because only one agent is active at a time. The author-tagging still pays for itself the first time you ask *"why did I add this rule?"*
- **Cron-fired Executor sessions:** these are the most common source of overlapping-window conflicts. Either serialize them with a lock file, or accept that conflicts will happen and trust the protocol to handle them.
- **No governance repo yet:** if you're one repo and one solo dev, you don't need this. Add it the moment a second repo starts copy-pasting rules from the first.
- **Bigger teams:** the role tags can become real GitHub bot accounts with their own SSH keys. The protocol doesn't change; the audit gets cleaner.

## Related

- [The Two Roles](two-roles.md) — the roles whose state this protocol keeps aligned
- [The Context Cascade](context-cascade.md) — the files most often touched by both roles
- [The Delta Log](delta-log.md) — where sync conflicts get logged so patterns become visible
- [Bounded Deviation](bounded-deviation.md) — the Executor's freedom is bounded partly *because* sync is reliable
- Back to the [main playbook](../README.md)
