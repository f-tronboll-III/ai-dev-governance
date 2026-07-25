# Secrets and Rotation

> A cast of AIs and a human share a codebase. **No secret value should ever be displayed — in chat, in a handoff, in a log, or in a commit.** The discipline below is what keeps that true not just on day one, when everyone is careful, but two years in, when the operator has rotated 40 keys and onboarded three new repos.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Two Roles](two-roles.md), [The QA Gate](qa-gate.md) · **See also:** [Operator Clearances](operator-clearances.md), [File Handling](file-handling.md)

---

## The wall

Secrets leak in five ways, in roughly this order of frequency:

1. **Pasted into chat.** The Strategist asks "what's my Stripe key?" while debugging a webhook. The operator pastes it. The chat log now lives in the AI provider's system, the operator's local cache, and possibly a shared session transcript.
2. **Committed to a repo.** A developer adds a "temporary" `.env` to a commit, or an Executor writes a config file with a value the operator pasted earlier in the session. `git push --force-with-lease` does not delete the value from GitHub's history.
3. **Logged.** A debug `console.log(process.env)` ships to production. The value appears in the host's log aggregator, viewable by anyone with access.
4. **Echoed into a handoff.** The Strategist writes a handoff that says "set `STRIPE_SECRET_KEY=sk_live_…`" with the actual value. The handoff goes into the governance repo, which the Executor reads from, which is checked into git.
5. **Forgotten on rotation.** A key is rotated in the hosting console but not in the local `.env.local`, or vice versa, and the discrepancy is only discovered when something silently fails.

Each one is recoverable individually. The compound problem is that AI workflows multiply every leak surface: more chat transcripts, more handoff documents, more agent containers with their own filesystems, more places a value can end up resident in memory or on disk. The discipline below is shaped specifically for that environment — it assumes two AIs and a human, and it puts every secret behind a discipline that survives the *next* leak vector you haven't thought of yet.

## The rule

1. **Secret values never appear in chat, handoffs, checkpoints, logs, or any file in any repo.** Names appear; values do not.
2. **Separate *using* a credential from *displaying* one.** The Strategist never touches values at all. The Executor may *use* a credential it can never *show* — sourcing it by path from an operator-owned store to make an API call, and never printing, echoing, logging, or committing it.
3. **Secrets live in exactly one place per environment** — the hosting console for production, a single ignored local file for development — and are referenced everywhere else by name.
4. **A rotation playbook exists for every secret, lives in the repo's `AGENT.md`, and is exercised at least once before the secret is needed in anger.**
5. **A leak protocol exists for every secret category** — what to rotate, who to notify, how to confirm the leak window is closed — and is rehearsed on adoption, not on the incident.

## How it works

### Rule 1 — Names, not values

Every reference to a secret across the playbook uses the *name* — the env-var identifier — not the value. The Strategist writes a handoff that says:

> Set `STRIPE_SECRET_KEY` in the hosting console for this project. The value is in the operator's `1Password → Stripe → Live mode → Restricted key`. Do not paste the value into this handoff or into chat.

That's it. The Executor reads the name, the operator pastes the value into the hosting console, and the value is never resident in any AI's context, any chat transcript, or any committed file.

The same rule applies to `AGENT.md` files, checkpoints, the Delta Log, and every other artifact the playbook creates. If you find yourself about to type a value, stop and re-route through the operator.

### Rule 2 — Use is not display

The instinct is to say *"only the human ever handles a secret."* That's the right instinct pointed at the wrong noun. What actually needs protecting is **display** — a value rendered into a transcript, a log, a document, or a commit, where it becomes durable and copyable. **Use** — a credential read from a store and handed to an API client — leaves no residue at all.

Collapsing the two produces a rule that's simultaneously too strict and too weak. Too strict, because the human ends up pasting values into consoles for years on work an agent could do more reliably. Too weak, because "the human did it" says nothing about whether the value ended up in the session transcript on the way there.

So the line runs by role and by verb:

- **The Strategist touches values at all: never.** It designs from outside the repo and has no legitimate reason to hold a credential. Its handoffs name variables and point at where the value lives.
- **The Executor may use, never show.** It sources a credential by path from an operator-owned store, uses it, and never prints, echoes, logs, interpolates into a message, or commits it. Not into a summary, not into a debug line, not "redacted except the last four" unless the last four is genuinely all that's needed.
- **The Operator owns the store**, and remains the only role that *originates* a value — generating it at the vendor, placing it in the store, and rotating it.
- **Non-trust surfaces get nothing.** A recon agent or bulk-compute wrapper has no credential access of any kind ([Trust Gradient](trust-gradient.md)).

The five-second test changes shape accordingly: *is this value about to become durable somewhere?* If it lands in a transcript, a log line, a file, or a commit — stop, regardless of who's typing.

### The credential store

Once the Executor may *use* credentials, it needs somewhere to get them, and "ask the operator each time" is the bottleneck this whole distinction exists to remove. The shape that works:

- **One operator-owned file**, gitignored, outside any repo's tracked tree, holding the full-permission credentials for every provider across every project. It is the store; it is never the deployment target.
- **Referenced by path, never by content.** A handoff or a context file may say *"source `PROVIDER_API_TOKEN` from the credential store"* and name the store's path. It never quotes a value, and neither does any completion summary.
- **It is not production secrets.** Production values live in the hosting console; the store holds what the *agent* needs to operate the fleet — and those are usually the more powerful ones, which is exactly why they're gitignored and never echoed.
- **Enumerating it is a normal read; printing it is a leak.** An agent may check that a key exists, and may redact when reporting what it found. The moment a value is rendered, treat it as the incident it is.

This is the [clearance](operator-clearances.md) pattern applied to credentials: the human keeps the part only a human can hold — origination, custody, rotation — and the mechanical part moves to the agent, with the value never becoming visible on the way.

### Rule 3 — One source of truth per environment

Per environment, the values live in exactly one place:

| Environment | Source of truth | Read by |
| :---- | :---- | :---- |
| Production / staging | Hosting provider's env-var store, written via its API or UI | The deployed app at runtime |
| Local development | A single ignored file (`.env.local`) at the repo root | The local dev server |
| Shared / cross-repo | A secrets manager (1Password, AWS Secrets Manager, Doppler, Infisical) | The operator, manually copying out |
| Agent operations | The operator-owned credential store, gitignored, outside every repo | The Executor, by path, without ever displaying a value |

Anything else — a `.env.example` checked into the repo, a `config.local.ts`, an init script — contains *names only* (and optionally placeholder values like `STRIPE_SECRET_KEY=sk_test_REPLACE_ME`). The repo never contains a real value, in any branch, in any commit, ever.

A short script (`scripts/check-secrets.sh` or similar) runs in CI on every PR and greps the diff for high-entropy strings and known prefixes (`sk_live_`, `whsec_`, `xoxb-`, AWS access key shapes, `pk_live_`, `eyJ` JWT shapes, etc.). The script fails the PR if it finds one. This is the mechanical backstop for rule 2.

### Rule 4 — Rotation, in advance

Every secret has a documented rotation procedure that lives in the repo's `AGENT.md`. Minimum content:

- **Where to generate the new value** — vendor console URL, role/permission to use.
- **Where to set the new value** — hosting consoles for every environment, secrets manager.
- **What to test after rotation** — a smoke test that confirms the app works with the new value.
- **What to revoke** — the old value's location in the vendor's console, and the timing rule for revocation (e.g. "wait 24h after the new value is live, then revoke the old one").

The procedure is rehearsed *before* it's needed in anger. Pick a non-critical secret, rotate it on a calm afternoon, walk the procedure, fix anything that didn't work, update the `AGENT.md`. The rehearsal is the only way to discover that step 4 actually requires admin access the operator doesn't have, or that the smoke test takes 20 minutes when you assumed it took 2.

Rotation cadence varies by category. A reasonable starting point:

- **Long-lived API keys (Stripe, Twilio, SendGrid):** rotate annually, or immediately on any suspected exposure.
- **OAuth client secrets:** rotate annually.
- **Service-account credentials:** rotate semi-annually, or as the platform requires.
- **Database passwords:** rotate semi-annually, or as the platform requires.
- **Internal HMAC / signing secrets:** rotate annually, unless the algorithm is broken and forces sooner.

Rotation events go in the changelog of the repo that uses the secret, with the rotation date and the new key's last-four (for identification, not for use).

### Rule 5 — A leak protocol per category

A leak is when a secret value lands somewhere it shouldn't — a chat transcript, a public log, an accidental commit, an over-shared screenshot. When a leak happens, the playbook is:

1. **Revoke immediately.** Not "after we figure out scope." The first action is to make the leaked value useless. Revoking takes seconds; reasoning about blast radius can take hours.
2. **Generate the replacement.** Follow the rotation procedure from rule 4.
3. **Inventory the leak surface.** Where did the value end up? Chat? Git? A log file? Each surface is a separate cleanup task.
   - Chat: clear the conversation if the platform allows; otherwise note that the value was once visible there.
   - Git: rewrite history if the value is in a few commits and the repo is small; otherwise rotate and accept the historical exposure (the revocation makes it harmless).
   - Logs: scrub if the platform allows; otherwise note in the incident.
4. **Document the incident.** A checkpoint at `docs/checkpoints/<DATE>-leak-<slug>.md` with the trigger, the value's location-history, the rotation timeline, and one line for the Strategist on whether the leak path should become a new prohibition.
5. **If the leak is third-party-scoped (a vendor key)** — notify the vendor's security team. Most vendors care more than you'd expect and will sometimes accelerate the rotation on their end.

The principle: **the cost of an unnecessary rotation is small; the cost of a delayed rotation is unbounded.** When in doubt, rotate.

## When the Strategist needs to "test" something

This is the equivalent of the [Strategist's "just fix it quick" temptation in file handling](file-handling.md#when-the-strategist-is-tempted-to-just-fix-it-quick), and the same shape as [the Operator's hard line in two-roles](two-roles.md#the-operators-hard-line-what-even-the-executor-doesnt-do) — the Executor can ship; the Operator owns the irreversible step. The Strategist is debugging a webhook signature mismatch and asks the operator for the signing secret so it can compute a hash by hand. Don't.

The pattern instead:

- The Strategist writes a handoff that asks the **Executor** to compute the hash inside the repo, with the value pulled from env at runtime.
- The Executor runs the test, returns the result (the *result*, not the secret), and the Strategist reasons forward from the result.
- The operator confirms the result matches what the vendor's tooling produces.

This adds a 5-minute round trip and removes a vector that has bitten real teams. The five-second test: *would I be comfortable if this secret value ended up in tomorrow's chat log?* If no, don't put it there today.

## Worked example

A small team adds support for sending transactional email via Resend. The naive path: get the API key, paste it into a chat to ask why the SDK is failing, paste it into a `.env` that gets accidentally committed, force-push to remove it, congratulate yourself.

The disciplined path:

1. **Setup.** Operator generates a Resend API key in the Resend console; stores it in 1Password under "Resend → Production." Adds `RESEND_API_KEY` to the Vercel project's env vars (production + preview). Adds `RESEND_API_KEY=re_test_REPLACE_ME` to `.env.example` and commits that. Adds `RESEND_API_KEY=<real test key>` to local `.env.local`, which is gitignored.
2. **`AGENT.md` entry.** A four-line rotation procedure: where to generate, where to set, smoke test (`curl -X POST https://api.resend.com/emails --auth …` → expect 200 with test-mode response), revocation timing.
3. **Strategist handoff.** Writes "the handler reads `RESEND_API_KEY` from env." Does not include the value. The handoff is committed to the governance repo without risk.
4. **Executor implements.** Reads env at runtime, ships, the deploy works.
5. **Six months later, rotation.** Operator generates a new key on a Tuesday afternoon. Walks the rotation procedure from `AGENT.md`: new value in Vercel for production *and* preview, smoke test passes, old key revoked 24h later. Two-minute log line in the changelog.
6. **A year in, a leak.** A debug log inadvertently echoed the env to the host's log aggregator for ten minutes. Operator revokes immediately, rotates, scrubs the logs, writes a checkpoint at `docs/checkpoints/2027-04-12-leak-resend.md`. The Strategist updates `AGENT.md` to add a one-line prohibition on `console.log(process.env)` in handlers, and the post-deploy QA grep is updated to flag it. Total time from discovery to resolution: 15 minutes, of which 10 were waiting for the new key to propagate.

The discipline traded a 30-minute setup for a leak resolution that took 15 minutes instead of half a day.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| Paste a secret into chat "just for debugging" | The transcript is durable; the value is now in the AI provider's system | Reference by name; have the Executor compute the test |
| Put real values in `.env.example` | `.env.example` is committed; the value is now public | Names + placeholder values only (`sk_test_REPLACE_ME`) |
| Hand-edit the hosting console's env every time | Drift between environments; no rotation log | A rotation playbook in `AGENT.md`; rehearse it |
| Commit `.env.local`, then `git revert` | `git revert` doesn't remove from history; rotation is required regardless | Gitignore from the start; never commit |
| "We'll rotate after the launch" | Launch never produces a calm moment; the rotation gets indefinitely deferred | Rotate on a fixed cadence regardless of launch state |
| Strategist writes "set `STRIPE_SECRET_KEY=sk_live_…`" in a handoff | Value is now in the governance repo, even after edit | Handoff says only the name and points to the operator's password manager |
| Log `process.env` for debugging | Production logs are searchable by anyone with access | Log only the values you need, by name, redacted by length |
| Revoke "once we understand the scope" | Scope-reasoning takes hours; revocation takes seconds | Revoke first, then investigate |
| One shared API key used by every repo | A single leak forces a rotation that breaks every repo simultaneously | One key per repo or per service boundary; rotate independently |
| Echoing a credential to prove it was found | The proof is the leak; the transcript is durable | Report presence and location, never the value |
| Routing a credential through the human because "an AI shouldn't touch secrets" | Conflates use with display, and often puts the value in the transcript en route | The Executor uses without showing; the Operator originates and rotates |
| Giving a recon or bulk-compute surface a credential "just for this one call" | Non-trust surfaces are outside the boundary by definition; there is no small version of this | Credentialed work is an authoring actor's, always |
| Writing a credential into `.git/config` or a shell history to avoid re-typing it | Both are durable, both get read, neither is where you'd look after a leak | Use it ephemerally, per invocation, from the store |

## Adapt it to your setup

- **Solo, no secrets manager:** a single passworded `secrets.md` file in an iCloud-synced folder or a password manager's secure note is enough. The principle (names in code, values in operator-only storage) is identical.
- **Team:** add a secrets manager (1Password Shared Vault, Doppler, Infisical, AWS Secrets Manager). The operator-paste discipline is enforced by team norm, backed by the CI grep for known prefixes.
- **Compliance-bound (HIPAA, SOC 2, PCI):** rotation cadences shorten, leak protocols become legally significant, and the audit trail in the changelog becomes the basis for an evidence record. The shape of the discipline is identical; the cadences and documentation tighten.
- **Heavy third-party integration (many vendors):** the rotation playbook becomes a per-vendor table in the repo's `AGENT.md`, not a paragraph. Generate-test-revoke steps are identical per vendor; the URLs differ.

## Related

- [The Two Roles](two-roles.md) — where the Operator's accountability line falls, and why the Strategist holds no values at all
- [Operator Clearances](operator-clearances.md) — the general form of "the human keeps origination; the agent gets the mechanics"
- [The Trust Gradient](trust-gradient.md) — why non-trust surfaces get no credential access under any circumstances
- [File Handling](file-handling.md) — the same "verify before trusting" discipline applied to writes
- [The QA Gate](qa-gate.md) — where the post-deploy secret scan lives
- [Autonomous Loops](autonomous-loops.md) — loops are a high-leak surface because they read secrets unattended
- Back to the [main playbook](../README.md)
