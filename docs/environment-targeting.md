# Environment Targeting

> Before an agent runs anything that writes — a dev server, a seed, a backfill, a migration, a test script — something has to answer *which database is this?* In most small shops the honest answer is "production," and nothing in the environment says so. Every environment file states its target in a key a machine can read; no marker means production; and a guard reads the marker before the irreversible commands run.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Sanity Check](sanity-check.md), [Secrets and Rotation](secrets-and-rotation.md) · **See also:** [Enforcement, Not Attestation](enforcement.md)

---

## The wall

An Executor is asked to add a column and "test it locally with some seed data." It reads the repo's local environment file, finds a database URL, runs the migration and the seed, and reports success. Everything it did was correct. The URL pointed at production.

That is not a rare configuration. One shop audited every local environment file across its repos that held a database credential: **of 33 files, 18 pointed at production** — the majority. Eleven pointed at a development branch, two at databases that no longer existed, two at something nobody could identify. Nothing in any file said which was which. The variable had the same name everywhere, and a local dev server, a seed script, and a "quick test" all read it.

The audit kept finding second and third versions of the same hazard:

- **Preview deploys wrote production too.** Most projects had one database row in the host's environment store, scoped to *all* environments. Every preview deployment of every branch wrote the production database. Fixing the local files fixed nothing there.
- **A second URL under a name no search matched.** One repo's migration tool read a *direct-connection* URL — a variable whose name contained none of the words a grep for database settings looks for. Splitting the main URL to a dev branch and missing that one left the migration CLI writing production while the app wrote dev. It was found only by scanning every value that started with the database's connection scheme.
- **A renamed key that failed silently.** A reporting script looked up the production URL under the name the notes recorded. The key had been renamed, the lookup found nothing, and the script fell back to the dev URL — then reported dev numbers as production's. It was caught only because the script printed the endpoint it had connected to.
- **A shared database across three repos.** A seed run in the smallest repo wrote the user table that gated paid access on the other two.
- **The checker was blind to a whole class.** The first marker audit keyed on one variable name. Repos using a different database client had no such variable, so the audit silently skipped them — four files, three holding production keys that bypass row-level security, one sitting loose in a directory outside any repo. They never showed up as a problem, because they never showed up at all. **Invisible is worse than wrong.**

None of this errored. Every command reported success, against the wrong database.

## The rule

1. **Every environment file that holds a database credential states its target** in machine-readable keys beside the credential: `DB_TARGET=prod|dev|dead|unknown`, plus the provider's identifier for the endpoint, the project, the branch, and an optional note.
2. **No marker means unknown, and unknown means production.** Label every file — both sides. A scheme that marks only the dev files leaves the dangerous ones silent, and silence must never read as safe.
3. **Record the target beside the credential, never inside its name.** The variable name is a binding contract with committed code; renaming it breaks things in both directions.
4. **Generate markers from the provider's own API; verify them offline against the credential.** A stale marker is worse than none — a confident label on the wrong database.
5. **A guard reads the marker** and refuses the operations your policy already forbids on production — schema push, database reset, development-mode migrations, seeds, builds that migrate — and nothing else.
6. **Prove which database you're on by identity** — the endpoint identifier, or a marker row — never by row counts, string comparison, DNS, or an unauthenticated probe.
7. **Preview is production until proven otherwise.** Split it in the host's environment store, and find every database URL by its *value*, not its name.

## How it works

### The marker block

```text
DATABASE_URL=postgres://…          # untouched — committed configs bind to this name
DB_TARGET=prod
DB_ENDPOINT=<provider's id for this compute endpoint>
DB_PROJECT=<provider project>
DB_BRANCH=main
DB_NOTE="Shared with two sibling repos — a seed here writes their users table"
```

**Keys, not comments.** Nothing binds to these names, so they cost nothing, and a comment is invisible to every mechanical reader — the guard included. When a file holds two credentials with different fates (a dead URL beside a live production key for another service), mark it **prod**: between an over-cautious label and a falsely reassuring one, take the over-cautious one, because a file labelled *dead* invites exactly the script that lands on live data.

### Why not just rename the variable

The obvious proposal is a per-environment variable name — `DEV_DATABASE_URL` locally, production moved somewhere else. One shop measured it and rejected it: more than twenty committed schema and ORM config files named `DATABASE_URL`, roughly six hundred source files referenced it, and the host bound the same name at build time. Rename it locally and every tool breaks; rename the binding and production breaks; alias both and the bare key is still there to be grabbed.

Moving production strings out of local files into a separate store looks like a fix and is a smaller one than it seems: both files are local, gitignored, and readable by anything with a shell. What it buys is friction — production stops being the ambient default. Worth doing where a dev branch already exists. Where one doesn't, deleting the string just stops local development from running. **No naming scheme substitutes for cutting the dev branch.**

### Generate, verify, re-verify

- **The generator** resolves each credential's endpoint against the database provider's API — the only authority on which branch an endpoint belongs to — and writes the marker block. Dry-run by default; backs each file up before writing.
- **The checker** runs offline with no key: it re-derives the endpoint from the credential and fails when the marker disagrees. Run it after any branch or environment change.
- **Know what the checker proves.** It proves the marker agrees with the credential. It does not prove the endpoint still sits on the branch the marker names — re-run the generator after branch work for that.
- **Report what you skipped.** Every audit prints how many files it examined and how many it skipped, by reason. A checker shaped for one kind of credential is silent about every other kind, and the count is the only thing that makes that silence visible. See [Honest Checks](honest-checks.md).

### The guard

A pre-command hook reads the marker for the repo the command is about to run in, and refuses the irreversible operations when the target is `prod`, `dead`, `unknown`, or absent:

| Blocked on a production or unknown target | Deliberately allowed |
| :---- | :---- |
| Schema push from an ORM (it infers and rewrites, and can drop columns to reconcile) | Reads, in any environment |
| Database reset; development-mode migrations | Targeted data changes with a tight predicate (cleared by policy) |
| Seed scripts | Applying a committed, idempotent migration file (cleared by policy) |
| A build command whose build script itself runs migrations | A compile check with the bare framework build command |

The allowed column matters as much as the blocked one. **A guard that fights standing policy gets switched off, and then it guards nothing.** It blocks only what the policy already forbids, and its header says so.

Three details decide whether the guard works:

- **Fail direction, chosen on purpose.** A missing or unknown marker is *blocked* (fail closed). A crash in the guard itself is *allowed* (fail open), because a broken hook must not wedge every session. Write both choices down.
- **Resolve the target from where the command runs.** An early version judged by the session's starting directory: a session opened at the parent directory judged every command by a stray file there, and a session opened in a dev repo let a seed through in a production repo. Resolve from an explicit `cd` in the command first, then the command's working directory.
- **It is one layer.** See [Enforcement, Not Attestation](enforcement.md) for why the rule in prose still matters when the hook exists.

### Preview is production too

- **Split the host's environment rows,** so preview and development point at a dev branch and only production points at production. Split the pooled and direct URLs *together* — splitting one gives a preview deployment a dev connection and a production one at the same time, which is worse than either.
- **Assert coverage, don't eyeball it:** every (variable, environment) pair covered exactly once — no gaps, no duplicates.
- **Read each row back individually** after writing, and resolve its endpoint. An API's 200 confirms the write, not the destination. Listing endpoints may return values in a form you can't compare.
- **Beware branch-pinned rows.** A preview row pinned to a git branch that is later deleted matches nothing, and the preview silently falls through to the all-environments row — production. Branches made by dependency-update bots are guaranteed to be deleted on merge.
- **Only split where the build doesn't migrate.** If the build script runs migrations, a preview build pointed at dev will migrate dev — and one pointed at production will migrate production.

### What proves which database you're on

| Probe | What it tells you | Why |
| :---- | :---- | :---- |
| Row counts match | Nothing | A copy-on-write dev branch starts identical to its parent |
| Connection strings compare equal or different | Nothing | A direct and a pooled URL to the same compute differ byte for byte |
| The hostname resolves | Nothing | Many providers wildcard their zone; every name resolves |
| An unauthenticated request | Nothing | Live and deleted endpoints can return the identical error |
| A branch exists | Not that it's usable | A branch with no compute endpoint accepts no connection |
| A real query with the credential | Live versus dead | Success versus an authentication failure |
| The endpoint identifier, compared | Which compute you're on | Identity, not resemblance |
| A marker row created on dev, confirmed invisible from production, then dropped | Isolation, end to end | The only proof that writes can't cross |

## Worked example

An Executor picks up a handoff: add a `consent_source` column and backfill it from existing rows.

1. **Pre-flight.** It reads the repo's marker: `DB_TARGET=prod`, with a note that two sibling repos share the database. The data-state check runs as read-only queries — allowed.
2. **The seed is refused.** It tries the local backfill script; the guard blocks it and prints why. No argument with the guard — the marker is doing its job.
3. **A dev branch.** It creates a branch from production through the provider's API, runs the generator (the new local file gets `DB_TARGET=dev` and the new endpoint id), and runs the checker: clean. It confirms isolation by creating a marker row on the branch and checking it is invisible from production, then drops it.
4. **The preview check.** The host has a single all-environments database row. It splits preview and development to the branch — pooled and direct together — reads each row back individually, and resolves both endpoints.
5. **Execution.** Backfill rehearsed on the branch; the committed idempotent migration applied to production under the existing clearance; one announce line names the column and the migration file.

The rule fired once, at step 2, and cost about ten minutes. Without it, step 2 is the backfill running against the production users table of three products.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| Renaming `DATABASE_URL` to carry the environment | It's a binding contract; you break tooling locally or production remotely | Add marker keys beside it |
| Labelling only the dev files | Silence on the production files reads as safe | Label every file; no marker means production |
| Marking the target in a comment | No tool can read it, including your guard | Make it a key |
| Trusting a marker written once | Goes stale on the first hand-edit or branch change | Offline re-check after any branch or environment change |
| A guard that blocks everything touching production | It fights standing clearances, gets disabled, and then guards nothing | Block only what policy already forbids |
| Fixing local files and calling the fleet fixed | Preview deploys still write production through one shared row | Split the host's environment rows too |
| Finding database URLs by searching variable names | A second URL under an unexpected name is invisible | Enumerate by value — every string with the connection scheme |
| "Counts match, so we're isolated" | A copy-on-write branch matches by construction | Compare endpoint ids, or prove it with a marker row |
| An audit that scans only the credential shape you thought of | Every other shape is skipped silently | Count and report what was skipped, by reason |

## Adapt it to your setup

- **Solo, one repo:** add the marker keys to your one local file and one line to `AGENT.md` — *"read `DB_TARGET` before anything that writes; no marker means production."* Then make the real fix: give yourself a dev database. Everything else here is for when that fix is incomplete.
- **No hook support in your agent:** the rule lives in `AGENT.md`, and seed and migration commands go through a tiny wrapper script that refuses on a production marker. A wrapper the agent must call is weaker than a hook, but far stronger than prose.
- **Several repos:** generate markers with a script, verify them in CI or at session start, and put the guard at the user level, not per repo — a per-repo guard is one more thing a new repo forgets.
- **A database you genuinely can't branch:** keep the marker anyway. The guard can't make production safe, but it makes every write to it a deliberate act rather than the ambient default.

## Related

- [The Sanity Check](sanity-check.md) — the data-state check is only as good as its knowledge of which database it queried
- [Enforcement, Not Attestation](enforcement.md) — guards, fail directions, and why a rule in prose is a promise
- [Secrets and Rotation](secrets-and-rotation.md) — the credential store, and why sourcing two environment files into one shell is how a dev URL lands in production
- [Operator Clearances](operator-clearances.md) — the cleared operations the guard deliberately leaves alone
- [Verify by Destination](verify-by-destination.md) — an API's 200 after an environment write confirms the write, not where it points
- [Honest Checks](honest-checks.md) — why an audit must report what it skipped
- Back to the [main playbook](../README.md)
