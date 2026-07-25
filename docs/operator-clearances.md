# Operator Clearances

> A human approval gate does two different jobs: it *prevents* an action, and it *informs* the human that the action happened. Most gates are only doing the second. When you can't say which job yours is doing, it has become a ceremony — it slows every session and protects nothing.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Two Roles](two-roles.md) · **See also:** [Secrets and Rotation](secrets-and-rotation.md), [The QA Gate](qa-gate.md)

---

## The wall

Your Executor is capable and credentialed. It just shipped a feature cleanly. And the completion summary ends like this:

```text
## Operator Action Block
1. Set these 6 environment variables in the hosting console (values below).
2. Run this migration against production:
   ALTER TABLE contacts ADD COLUMN sms_consent_at timestamptz;
3. Transfer the domain registrar and update nameservers.
```

The operator pastes items 1 and 2. Every time. Nothing is ever caught, because there was nothing to catch — the variable names were right, the SQL was right. Six months of that produces two failures at once.

First, **the human becomes a bottleneck on their own infrastructure.** Work finishes at 4pm and waits until 11pm because a paste is outstanding. The Executor learns to design *around* the gate, batching changes to minimize pastes, which makes each paste bigger and less reviewable.

Second — and worse — **the gate stops being read.** Item 3 in that block is genuinely irreversible: a registrar transfer changes custody of a domain, out of band, on a multi-day clock. It sits in the same undifferentiated list as two rubber-stamp pastes and gets the same three seconds of attention.

The scar cuts both ways, which is why this is subtle. The gate was originally *correct*: a platform CLI that set environment variables silently created separate per-environment entries, while the platform's dashboard offered an "all environments" option the CLI didn't expose. A year of config drift started right there, and "the human pastes into the dashboard" was the honest fix at the time. But the *capability* problem got solved — the platform's REST API takes an explicit all-environments target — and the gate outlived the reason for it by a year.

## The rule

**Name what each gate is for. If it only informs, retire it into a clearance.**

A gate that genuinely *prevents* something — an action the agent could get wrong in a way the human would catch — stays, and you write down what it prevents.

A gate that only *informs* is replaced by a **clearance**: the agent acts directly, and three things take the gate's place.

1. **A capability path at least as safe as the human's.** The API call that exposes every option the dashboard exposes. The hand-written, idempotent migration file — not a schema tool that pushes whatever it infers.
2. **A durable, reviewable artifact**, committed alongside the code. The record has to outlive the session and be readable by someone who wasn't in it.
3. **An announce step** — one line in the completion summary naming exactly what changed in production. This is the awareness the gate was actually buying you.

Fail any of the three and the gate stays. And when a routine, cleared step shows up in an Operator Action Block anyway, **treat that as a bug in the clearance** — fix the clearance, not the operator's evening.

## How it works

### The three questions that retire a gate

Ask them in order. A "no" anywhere stops the process.

1. **Can the agent perform this at least as safely as the human?** Not "can it perform it" — *as safely*. This is where option-exposure lives: if the human's path shows five toggles and the agent's path silently defaults four of them, the answer is no until you find a path that doesn't. Solving this is usually a one-time engineering task, not a permanent property.
2. **Does something durable and reviewable record what happened?** A chat transcript is not a record. A committed migration file with guarded, re-runnable SQL is. An API call whose parameters are visible in the code that made it is. The test: could someone reconstruct the change six months from now without the session?
3. **Does the human still learn it happened without being in the loop?** If the answer depends on them reading a diff they don't read, it's no. It has to be pushed to them, in the summary, in one line.

### The announce step

This is small and it is the whole substitute for the gate. Format is unremarkable on purpose:

```text
## Applied to production this session
- DB (contacts): added `sms_consent_at timestamptz`, `sms_consent_source text`.
  Idempotent; migration file `migrations/0072_sms_consent.sql`.
- Env (project: storefront): set `RESEND_WEBHOOK_SECRET` on all environments.
```

One line per structural change. What, where, why-in-a-clause, and the artifact to go read. The operator gets the same fact the gate was giving them, at their leisure, with something to look at — which the paste never provided, because a pasted statement leaves nothing behind.

### What never graduates

Some gates are not ceremonies. These stay, permanently, no matter how capable the agent gets:

- **Custody and identity changes.** Registrar transfers, domain purchases, nameserver cutovers, account creation. Irreversible, out of band, and they change *who owns the thing.*
- **Money.** Billing changes, plan upgrades, purchases, anything that moves funds or commits to a recurring charge.
- **Organization-level permission grants.** An org-wide secret, an IAM role, a repository-permissions change. Blast radius beyond the work in front of you.
- **Outward-facing sends to real recipients.** Email blasts, social posts, transactional notifications to real inboxes. The agent may *prepare* the send — render, queue, dry-run against a sample — and stops. The unit of approval is the send, not the code that performs it, because there is no unsend.
- **Unbounded destructive operations.** Mass update or delete without a tight predicate, truncating a table, dropping something populated. Bounded and reversible is a clearance; unbounded is a gate.
- **Anything where the answer to the five-second test is the human alone.**

> **The five-second test:** *if this goes wrong, who is the one undoing it?* If the answer is "the Operator, not the agent," it does not graduate.

### What typically does graduate

The mirror image — and it's worth naming these explicitly, because they're the ones clogging most Action Blocks:

- **Environment variables**, via an API that sets every environment explicitly, sourced from the operator's own credential store, with values never printed.
- **Schema migrations**, where the agent hand-writes a guarded idempotent file (`IF NOT EXISTS`, wrapped conditionals), commits it with the code, applies it, and announces the structural change in one line.
- **Targeted data changes.** Small, tightly-predicated fixes and backfills. Free in development; in production, state the exact statement and the reason in one line first, then report rows affected.
- **Reads of any kind**, in any environment. A read that shows up as an Operator Action is always a bug — verifying live state is the Executor's job, and handing it to the human inverts the entire [Sanity Check](sanity-check.md).
- **Repository-scoped secrets** set through a CLI the agent is authenticated for, values sourced by path and never echoed. (Organization-scoped ones stay gated — see above.)

### Freeze windows: gates that are temporary by design

Not every gate is permanent-or-retired. A third shape is a **freeze**: a scoped, temporary gate with a named sunset.

*"While these three domains are mid-transfer, no new DNS records on them — except the ones the transfer playbook lists as preserved. Exception path: the Operator and Strategist ratify it in the decisions log. Sunsets when the transfer verifies."*

That's a well-formed freeze: a **scope** (three named domains), a **prohibition**, an **exception path**, and a **sunset condition**. A freeze without a sunset silently becomes a permanent gate that nobody remembers the reason for — and those are the hardest kind to retire, because the justification is gone and only the ritual remains.

### Why this isn't just loosening the rules

It reads like a relaxation. It's actually a redistribution: the ceremonial gates go away, and the attention they were absorbing moves to the gates that genuinely prevent something. An Action Block holding one irreversible item gets read. An Action Block holding six pastes and one irreversible item gets skimmed — and the whole point of the Operator role is that the irreversible item gets *read*.

Where the platform can enforce a gate mechanically — IAM, a required approval on a migration pipeline, an integration scoped to drafts-only — do that instead of writing it down. Every mechanical enforcement is a line that no longer depends on discipline.

## Worked example

**Before.** A feature needs two new columns. The Executor writes the SQL into the completion summary, the operator pastes it into a database console at 11pm, and the change exists nowhere but the chat transcript. Two months later a fresh developer environment can't be built, because nothing in the repo describes those columns.

**After.** The Executor:

1. Writes `migrations/0072_sms_consent.sql` with guarded, re-runnable SQL — `ADD COLUMN IF NOT EXISTS`, wrapped in a conditional block for the constraint.
2. Commits it in the same commit as the code that depends on it.
3. Applies it to production directly.
4. Announces: *"Applied to production: `contacts` — added `sms_consent_at timestamptz`, `sms_consent_source text`. Idempotent; migration file `migrations/0072_sms_consent.sql`."*

The operator learns the same fact, in one line, whenever they read the summary — plus a file they can actually open. And the migration is replayable against a fresh database, which the paste never was. The gate is gone; **the record is better than it was when the gate existed.**

Contrast the item that stayed: the registrar transfer. It's still in the Action Block, alone now, with the steps and the multi-day clock spelled out. It gets read, because it's the only thing there.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| A gate that exists because it always has | Nobody can say what it prevents, so nobody can say when it's safe to retire — it just accretes | Name the job (prevent / inform) at the moment you create the gate |
| Retiring a gate without the announce step | You removed the friction *and* the awareness; the human silently loses the plot on their own infrastructure | Clearance = capability + durable artifact + announce, all three |
| Using an auto-push schema tool as the "capability path" | These infer and rewrite without a reviewable file; they drop columns to reconcile | Hand-written idempotent SQL the agent then applies; the file is the record |
| One Action Block mixing six routine pastes with one irreversible action | The important one inherits the rubber stamp the routine ones taught | Routine → clearance; the block holds only what genuinely can't graduate |
| Granting a clearance mid-task because the gate is inconvenient right now | Clearances decided under deadline pressure are the ones that go wrong | Clearances are declared in the context file, in advance, by the Operator |
| A freeze with no sunset condition | Becomes a permanent gate whose justification has evaporated | Every freeze names its scope, its exception path, and what ends it |
| Handing the human a *read* as an Operator Action | Inverts the Sanity Check — the Executor is the role that can see production | Reads are always the Executor's, in every environment |
| Echoing a credential value to justify a clearance | The clearance was for *using* the credential, never for displaying it | Source by path; never print, log, or commit a value |

## Adapt it to your setup

- **Solo.** You're both roles, so the gate feels free — it isn't. Write the announce line anyway; it's for future-you reconstructing why production has a column the schema file didn't mention.
- **Small team.** The clearance list belongs in the global context file, one line per cleared action, with the capability path named (*"env vars: REST API, all-environments target"*). The announce lands in the changelog as well as the summary.
- **Just starting out.** Don't grant clearances on day one. Start with everything gated, and retire each gate individually the first time it fires with nothing to catch — that's the moment you have evidence.
- **Regulated environments.** Some gates are legally required. Those genuinely *prevent* (they prevent an unattested change), never graduate, and should have the regulation named next to them so nobody re-litigates it annually.
- **Multiple agents.** Clearances attach to the *role*, not the tool. A [non-trust surface](trust-gradient.md) never inherits the Executor's clearances no matter how capable it is — that's the difference between the two docs: this one is about which actions an actor may take, the gradient is about which claims it may make.

## Related

- [The Two Roles](two-roles.md) — the Operator's accountability line, of which this is the working edge
- [The Trust Gradient](trust-gradient.md) — the same question for claims rather than actions
- [Secrets and Rotation](secrets-and-rotation.md) — the use-versus-display distinction that makes credentialed clearances safe
- [The Sanity Check](sanity-check.md) — what runs *before* a cleared action, and why reads are never handed to the human
- [The QA Gate](qa-gate.md) — where the announce step lands in the completion summary
- [Autonomous Loops](autonomous-loops.md) — clearances for code that acts with no one in the room at all
- Back to the [main playbook](../README.md)
