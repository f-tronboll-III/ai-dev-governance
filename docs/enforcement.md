# Enforcement, Not Attestation

> A rule an agent can break by forgetting is a promise, not a wall. Telling an agent *"you're read-only today"* changes what it believes, not what it can do. For the failures that are irreversible or silent, turn the rule into mechanism — an inventory of what each actor can actually do, a guard that reads posture from a file, a refusal that says how to proceed — and keep the prose for the *why*.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Trust Gradient](trust-gradient.md), [Operator Clearances](operator-clearances.md) · **See also:** [Environment Targeting](environment-targeting.md)

---

## The wall

The shop had a written answer to every question in this section. The answers were wrong in the same way.

- **The fallback with the run of the house.** The only trusted Executor might be down for a day, and the available fallback was a tool the [trust gradient](trust-gradient.md) explicitly graded non-trust. Its contract said: read-only, except in one sandbox repo. Its actual configuration: approve every action automatically, any shell command allowed, no hooks at all. It could write, commit, and push anywhere in the fleet. Nothing had gone wrong. Nothing was checking.
- **The assistant that was "forbidden."** A second assistant seat, used for mail and documents, was recorded in the roster as *forbidden to change any live system.* Asked to list its own tools, it reported write access to production database SQL and migrations, payment writes and refunds, invoicing, and email send — broadcasts included. It had used none of them. The roster sentence had described an impossibility; it was a promise. "It drafts, a human sends" had no enforcement on that side at all, and the failure it guarded against had already happened once elsewhere.
- **One door locked.** The first guard on the fallback blocked writes through the file-editing tool. Every shell route to the same files — redirecting output into them, in-place stream edits, copy, tee, delete — was open.
- **The wrong door locked.** The fallback existed for rescue work: a DNS outage, a vendor setting, a status page. None of that touches a repo, and the first guard only gated repos. Measured with the guard in place: a DNS record create, a DNS record delete, a hosting-platform environment write, and an email to a real client were all allowed.
- **The workaround that lapsed.** One client application corrupted the conversation whenever the agent used its interactive-question tool, and the corruption was unrecoverable — every later request replayed the broken turn. The rule was clear and written down: never call that tool. Mid-turn, the agent called it anyway.

Five rules, all correct, all depending on an agent remembering them at the moment it mattered.

## The rule

1. **Inventory capability, not intention.** For every actor, list what it can actually do. Record each write-capable tool it is forbidden to use as *holds, undertakes not to use* — and remove whatever has no purpose. Only removal is enforcement.
2. **Posture is a file, not a sentence.** Authority beyond an actor's default lives in a state file with a scope and an expiry, and the guards read that file. The default never changes.
3. **Mechanize the irreversible and the silent.** Put guards where a failure can't be undone or won't be noticed — and guard only what policy already forbids. A guard that fights standing policy gets switched off, and then it guards nothing.
4. **Choose each guard's fail direction, and write it down.**
5. **One verdict for every door.** A guard that covers the edit tool and not the shell covers nothing.
6. **Enforce, don't attest.** *"I read the rules"* is a claim. A log of the files actually read is evidence, and the first mutation waits for it.
7. **A workaround is a unit:** rule, hook, and removal condition — removed together.
8. **A refusal carries the guidance.** A guard that blocks the fix without saying how to proceed gets argued past or disabled.

## How it works

### Inventory capability

1. **Ask the actor what it holds**, from inside its own session. Its self-report is often the most useful output of the whole exercise.
2. **Split every tool into read and write.**
3. **Record the write side honestly:** *forbidden, undertaken against, not impossible.* Then remove every connector that has no purpose for that actor, at the vendor's own settings or consent screen.
4. **Until the removal is done, it's an open defect** in the [ledger](defect-disposition.md) — the true state, written down.
5. **Check scopes, not just tools.** A repo-read integration on a personal account often has no read-only role to grant; a consent screen that lets you pick repositories and lists permissions is a real control, while a blanket repository scope is *write access wearing a read-only intention.* A payment integration bound to one account will answer confidently about another business's data if asked — confirm which account it reaches from a response, not from its name.

### Posture is a file

The fallback's authority became a state file that every guard reads:

| Posture | Mutation allowed | Set by |
| :---- | :---- | :---- |
| **Default** | Nothing outside one sandbox repo | The *absence* of a state file |
| **Standing clearance** | One named repo | An Operator ruling, recorded once |
| **Drafts writer** | Files in one drafts folder; no git | An Operator ruling, recorded once |
| **Armed** | One scoped path, expiring in at most 24 hours | An arming command, with a reason |

- **Expiry is on a clock.** An expiry the Operator has to remember is not an expiry; a stale arming disarms itself at the next session start.
- **Arming is not a promotion.** The actor's grade doesn't change. An armed window is an explicit record that unverified work is being accepted, and how much.
- **Everything shipped in an armed window is reviewed afterward** by an authoring actor. Audit the window by *content* and by the actor's own execution log — never by file modification times. In one review, a sync tool had rewritten unchanged files with fresh timestamps, making two files look as if they'd been written hours after the window closed.

### Mechanize the irreversible and the silent

Some operations stay refused even while armed, because they are irreversible *and* silent: database resets, schema pushes that can drop columns, dropping or truncating tables, seeds and builds that migrate production, direct deploys, force-pushes, skipping commit hooks, broad `git add`, writes to environment files, and authoring any governance or context file.

Rescue surfaces need gates of *different* strengths, because a rescue guard that refuses the rescue is worse than no guard:

| Surface | Gate | Why this strength |
| :---- | :---- | :---- |
| Outward sends to real recipients | **Denied outright**, armed or not | An unverified actor does not speak to a client in your name. Drafting stays allowed; read tools pass |
| DNS record writes | Denied **until that zone's record list has been read this session** | Adding a record to a wildcard-served name can take the site offline while every lookup still resolves; it happened once for eighteen minutes and monitoring never fired |
| Vendor control-plane writes | **Require arming**; reads unrestricted | These *are* the rescue actions — make them deliberate and time-boxed, not banned |

**Gate on the result, not the command.** The zone-read gate checks that the read actually returned a record listing. A request that got a 401, a firewall page, or the wrong provider still exits cleanly — without this check, the agent would have "read the zone" without receiving one. Reading one zone doesn't unlock another.

### Fail direction

Different layers fail differently, and the difference decides where each rule lives:

- **Deny rules in the agent's permission config don't fail open** — they beat every allow rule and every auto-approve mode. Put the irreversible set there.
- **Hooks usually fail open** — a hook that crashes, times out, or goes missing lets the call through. Use them for *context* (which repo, which posture, was the cascade read), not as the only barrier.
- **Choose per guard.** An unknown database target fails *closed*. A guard's own crash fails *open*, so a broken hook doesn't wedge every session. Public abuse guards fail open, except a brute-force cap on a short code, which fails closed.

### One verdict for every door

Route shell writes through the *same* verdict function the file guard uses — one implementation, because two had already drifted. The function has to tell a write *target* from a path merely mentioned: a recursive search of a repo that redirects its output to a temp file writes to the temp file, and a guard that can't tell the difference blocks ordinary reading and gets switched off.

**It is a screen, not a proof.** It can't see through `eval`, a path held in a variable, or a script it invokes. A clean result means *nothing recognised*, never *audited*.

### Enforce, don't attest

The fallback agent discovered instruction files from the repo root downward. Every family-level protocol in the shop lived *above* the repo roots, so the rules that governed how work got done were never in its context — and nothing said so. The fix wasn't copying the family rules into every repo (copies drift invisibly). It was:

- A post-read hook logs every file the agent actually reads.
- A pre-mutation hook **refuses the first write until the required files appear in that log.** Reading one of two doesn't clear it. A new session starts empty.
- A command any session can run prints the honest answer to *what governs here?* — family, repo, posture, hazards, and the exact files to read.

An agent that reports *"context loaded ✓"* has attested. One whose first write was refused until the log showed the reads has been enforced.

### Quarantine by construction

The recon agent — non-trust by grade — is launched only through a wrapper script, never directly:

- **Deny rules per run**, by repo name rather than absolute path (a relative `cd` never contains the absolute path), including any scope currently armed for other work, so a launched run never inherits write access granted to something else.
- **Ground rules injected above the task:** write only the report, tag it unverified in its title, date and cite every live claim, include no personal data, and if a tool call is refused, note it and carry on — never route around it.
- **The wrapper, not the agent, writes the prompt as sent** at the top of the report, so the audit record isn't the agent's to edit.
- **The output path is the trust label.** Reports land in a drop folder outside every repo, writable in any posture. Nothing there reaches production without a person moving it, and a file is unverified by its location. **Findings only, never executables** — a script written there and run later is how unverified output becomes an action.

The drop folder exists because the first version blocked every write and left the recon agent nowhere to put its report. That isn't a safe default; it's a broken one, and it pushes the Operator toward arming the tool for work that needs no authority at all.

### The workaround unit

A rule that works around a tool's bug carries four things: **the rule**, with its reason, so the agent knows what to do instead; **a hook** that blocks the call deterministically, so a mid-turn lapse can't reintroduce the failure; **a removal condition**, as a named test (*"once a flow completes cleanly in the fixed build"*); and **an instruction to delete the rule and the hook together.** A prose-only workaround lapses; a hook with no explanation leaves the agent unable to recover or explain; a workaround with no removal condition becomes folklore that outlives the bug.

### A refusal carries the guidance

Write the refusal message as the instructions for doing it right: *read the zone first; create the alias before the record; check which provider serves the zone; verify both records after.* Forcing a hundred-kilobyte protocol read mid-outage to deliver knowledge that fits in the refusal is the friction that gets guards switched off. And **a guard argued past protects nothing** — the agent reports the refusal and what would clear it, and stops.

## Worked example

The primary Executor's service is down for an afternoon, and a client's booking page needs a DNS fix.

```text
1. Operator confirms the outage is real, then arms the fallback:
     arm <client-repo-path> 4h "booking page DNS — primary executor down"
   → state file written: scope = one path, expiry = 4h.

2. Fallback's first write attempt → refused: required context files not yet read.
   It runs the what-governs-here command, reads the three files it names → log updated.

3. It drafts a status email to the client → allowed (draft).
   It tries to send it → denied outright, armed or not. Reported to the Operator.

4. It tries the DNS record write → denied: zone not read this session.
   Refusal text: read the zone; alias before record; check the serving provider.
   It reads the zone → the gate confirms the output is a record listing → write allowed.

5. Four hours pass → the next session start finds the expired state file and disarms.

6. Next morning, the primary Executor reviews the window from the fallback's own
   execution log and the content diff: one DNS change (verified at the destination),
   one draft (sent by the Operator), two deferrals → ledger rows.
```

At no point was the fallback's grade changed, and at no point did anything depend on it remembering what it was allowed to do.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| "This actor can't write production" with no tool inventory | It can; the sentence is a promise | Inventory, record "holds, undertakes not to use," remove what has no purpose |
| Telling the agent its posture | Belief isn't capability, and it will believe the same thing next week | A state file with scope and expiry that the guards read |
| Promoting the fallback "for the outage" | Trust becomes ambient and never gets withdrawn | A bounded, expiring, Operator-armed exception, reviewed afterward |
| Guarding the edit tool but not the shell | The open door is the one that gets used | One verdict function for every write path |
| A guard that blocks everything near production | It fights the clearances, gets disabled, and protects nothing | Guard what policy already forbids — the irreversible and the silent |
| A gate satisfied by running a command, not by its result | An error page "satisfies" a read requirement | Check the output looks like what the gate requires |
| Relying on hooks alone for the irreversible set | Hooks fail open when they crash | Irreversible operations in deny rules; hooks for context |
| A prose-only workaround for an unrecoverable bug | One lapse reintroduces the failure | Rule + blocking hook + removal condition, as a unit |
| A drop folder that accepts scripts | Unverified output becomes an action when someone runs it | Findings only |
| "Context loaded ✓" | An attestation, from the actor being governed | Log the reads; refuse the first write until they appear |
| Auditing an armed window by file timestamps | Sync tools rewrite timestamps | Diff by content and the actor's own log |
| A refusal with no guidance | Gets argued past or switched off mid-incident | Put the how-to in the refusal text |

## Adapt it to your setup

- **Solo, one agent:** you don't need postures. You need two things: the irreversible set in your agent's *deny* configuration (not just its instructions), and one honest pass through your agent's integrations asking *what could this do that I've only told it not to?*
- **A second assistant seat or a teammate's account:** run the capability inventory on it today. It's the cheapest finding in this doc, and the most likely to surprise you.
- **A non-trust tool you run regularly:** a twenty-line wrapper script — deny rules, injected ground rules, the prompt prepended to the report, a fixed output folder — turns its etiquette into construction.
- **No hook support at all:** a wrapper for the dangerous commands, plus deny rules where your tool has them. Every rule you can move out of prose and into configuration is one fewer thing that depends on memory.

## Related

- [The Trust Gradient](trust-gradient.md) — grades say whose output is believed; this doc makes the grade's prohibitions real
- [Operator Clearances](operator-clearances.md) — "where the platform can enforce a gate mechanically, do that"; this is how
- [Environment Targeting](environment-targeting.md) — a worked guard: the marker, the blocked set, and the fail directions
- [Honest Checks](honest-checks.md) — a guard is a check that blocks; the same three states and fail directions apply
- [Model Orchestration](model-orchestration.md) — the recon agent's place in the tier routing, and harness defaults that stop delegation silently
- [Defect Disposition](defect-disposition.md) — where an un-removed capability lives until the audit is done
- Back to the [main playbook](../README.md)
