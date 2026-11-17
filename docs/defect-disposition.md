# Defect Disposition

> Every warning, error, or failing check an agent observes gets one of three durable dispositions: **fix it, record it, or pin it under a ceiling that only goes down.** "Pre-existing, not my work" is not a disposition. In a shop where nobody reviews the diff, a defect mentioned only in a session summary lives exactly as long as the session.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The QA Gate](qa-gate.md), [Bounded Deviation](bounded-deviation.md) · **See also:** [Honest Checks](honest-checks.md)

---

## The wall

The completion summaries started to share a sentence: *"Type-check clean. There are some pre-existing lint warnings, not related to this change."* It was true every time. The operator noticed they were getting uneasy at *"error, but skipping"* — and worked out why.

**In a shop with pull-request review**, "noted, out of scope" in a PR description is a real disclosure: a human reads it and can act. **In a shop where the deploy is the review**, nobody reads that line twice. The warning was mentioned, the session ended, and the mention went with it. It isn't a lightweight disclosure; it's a no-op wearing the costume of one.

The second-order harm is the one that causes outages. Once a repo's baseline is 183 warnings, warning number 184 — the real one, introduced this afternoon — is invisible. Noise normalization isn't an aesthetic problem; it's a loss of the ability to detect anything. It's the same shape as an incident months earlier in which fifty-nine scheduled jobs failed silently for days: nothing errored, and that was the problem.

Writing the fix down required a baseline, and the baseline was worse than the observation. Across 36 repos: **162 lint errors and 772 warnings. No repo had a warning ceiling. One ran lint in CI. Nine had no linter installed; three had a lint configuration so broken that lint couldn't run at all.** The one piece of good news was load-bearing: no repo had disabled type errors at build time, so the deploy gate itself was intact. The exposure was lint-shaped.

## The rule

**Every defect you observe gets exactly one of three dispositions, and the completion summary names them.**

| Disposition | What it means | When it's required |
| :---- | :---- | :---- |
| **D1 — Fix it** | Repair it in this session | It's in a file you're already changing · it's an *error* (not a warning) in a repo you're committing to · the fix is mechanical and bounded |
| **D2 — Record it** | One row in the defect ledger | Everything you observe and aren't fixing now, that a ceiling can't hold |
| **D3 — Baseline it** | Encode today's state as a ceiling the build enforces | Countable classes — lint warnings, an allowlist, a disabled rule with a reason |

There is no D4. *"Mentioned in the summary"* and *"noted for later"* are not dispositions.

Two limits keep this from turning into a different failure:

- **It covers what you observed, not what you could find.** There is no duty to go hunting. Speculative sweeps aren't asked for.
- **It's a disposition, not a repair.** Choosing "not now" is fully legitimate. Choosing it *silently* is not. "Fix everything you see" is its own failure — unreviewed drive-by edits in files the task never touched, which [Bounded Deviation](bounded-deviation.md) exists to prevent, plus heavy-tier tokens spent on cosmetics.

## How it works

### What counts as observed

Anything that actually crossed the agent's screen during the session: lint and type errors and warnings; failing, skipped, or focus-only tests; deprecation warnings; runtime errors in logs it pulled; and **a check that didn't run at all** — a missing script, a broken config, absent tooling. That last one is the easiest to miss, because it produces no output to notice.

### D2 — the ledger

One file, one table, one row per defect:

```markdown
| Date | Repo | Location | What | Disposition | Why deferred |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 2026-11-12 | storefront | src/app/checkout/route.ts:88 | `'session' is possibly 'undefined'.` | Open | Fix needs the auth refactor in H-112 |
```

- **Quote the literal message**, never a paraphrase, so the row is findable by searching for the error text.
- **Delete the row in the commit that fixes it.** Not struck through, not marked *RESOLVED* and left in place — that's the costume again. Git history is the record.
- **When you're in a repo, clear the repo, not the row:** search the ledger for that repo and look at all its rows.
- **Group by repo, not by severity.** A severity-ordered file rots: every insert is a decision about where it goes, merges collide, and nobody reads it by severity anyway. Rank within a repo's block; generate a cross-repo severity view with a script if you want one, never by hand.
- **Rank by what the defect is doing, not how loud it is.** *Accruing* (a live surface is wrong now and costs more each day) · *blind* (something meant to detect or enforce isn't, and nothing says so) · *bounded* (real, but something would notice) · *debt* (ceilings, cosmetics). A generic P1–P4 scale files silent detection loss as low priority, because nothing is visibly on fire — which is exactly backwards.
- **Split sections by who can clear a row:** open (the agent could do it, however uncomfortable) · needs the Operator (the Operator's next action is required before anyone can proceed) · deferred by ruling (with the condition that reopens it) · standing notes (known, deliberate, don't fix).
- **Watch the ledger's own size.** One ledger grew to 258 rows and about 134,000 tokens — past the point any session could load it. Promote durable lessons out of long rows into the docs they belong in, and keep the rows short.

### D3 — ceilings that only go down

A ceiling turns discipline into enforcement. Today's warning count becomes the build's limit (`--max-warnings 183`); warning 184 fails the build. Discipline erodes; a ceiling doesn't.

The rules that make ceilings honest:

1. **Errors are never baselined.** Ceilings hold warnings only. The one carve-out: when adopting a shared lint preset in a repo that was never linted, a whole *rule* may be demoted to a counted warning — with the reason at the demotion site, a ledger row, and a ceiling. The test: *could you name the rule and the reason without naming a line number?* If the honest sentence is "this call site is inconvenient," fix the call site.
2. **Every repo that ships code has a lint script with an explicit ceiling.** A rule that fires "when a session happens to see warnings" never fires in a repo nobody opens.
3. **Re-pin on exit.** If the repo ends the session below its ceiling, lower the ceiling to the actual count in the same commit. Slack is how "we brought it down" becomes "we have room to add some back."
4. **The campsite rule, file-scoped.** A file you edit leaves with no more warnings than it arrived with. Bounded to the diff, so it doesn't reopen "fix everything."
5. **One whole rule per substantial visit — not N warnings.** Either fix every instance of one rule, or dispose of the rule itself (record a project to address it, or disable it with a reason and a ledger row). The decision then stays made.
6. **Suppression is not reduction.** An inline disable or an expected-error comment added to lower a count is forbidden without an inline reason and a ledger row.

**Why not a percentage target** ("reduce warnings 10% per session")? Because it's indifferent to *which* warnings it eats and gets more dangerous as it progresses. Measured on one repo with 104 warnings: none were truly auto-fixable — the 37 the linter called fixable were suggestions that broke compilation — and 27 instances of one rule were a single unmade architectural decision repeated 27 times. A quota exhausts the cheap rule first, then forces behavior-changing fixes, and the cheapest way to hit any numeric quota is a disable comment.

### Tests

- **Never skip a test without a ledger row**, cited in a comment on the same line as the skip.
- **Never commit a focus-only marker** (`.only` and its equivalents). It silently disables every other test in the file, and the suite goes green. It's always D1.
- **A suite that doesn't run in CI is D2 at minimum, D3 preferred** — wire it in, even if only as a ceiling on failures.

### The completion summary

Name the dispositions, or say none were observed:

```text
Defect disposition: 2 fixed (D1 — both in files this change touched), 1 recorded
(ledger: storefront, checkout/route.ts:88), lint ceiling re-pinned 183 → 179 (D3).
```

or

```text
Defect disposition: none observed.
```

Silence is indistinguishable from an unreported skip, so the line is never omitted.

### How it interacts with the rest

[Bounded Deviation](bounded-deviation.md) governs D1: a fix that would expand scope drops to D2. A defect found during the [Sanity Check](sanity-check.md) that contradicts the handoff is a delta first, then takes a disposition. And a defect in the *check itself* — a linter that can't run, a screen that passes the wrong thing — is its own class: see [Honest Checks](honest-checks.md).

## Worked example

An Executor finishes a checkout change. Its type-check and lint output shows three things.

```text
1. src/app/checkout/route.ts:41  warning  'retries' is assigned but never used
   → in a file this change touched → D1, fixed.

2. src/lib/tax.ts:12  error  Unexpected any. Specify a different type
   → an ERROR in a repo being committed to → D1 required.
     The correct type needs a tax-provider schema the repo doesn't have yet.
     Fixing it properly expands scope. → Bounded Deviation says stop:
     D2, ledger row with the literal message, flagged "needs Operator" because
     the schema decision is theirs.

3. 183 warnings repo-wide, ceiling 183; after fix 1, count is 182
   → D3, re-pin the ceiling to 182 in this commit.
```

Summary line: *"Defect disposition: 1 fixed (D1), 1 recorded as needs-Operator (tax.ts:12 — schema decision), ceiling re-pinned 183 → 182 (D3)."* The Operator sees one decision that's theirs, in one line, with somewhere to go read it. Nothing is lost when the session ends.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| "Pre-existing, not my work" in the summary | Nobody rereads the summary; the note dies with the session | Name D1, D2, or D3 for each observation |
| Fixing everything in sight | Unreviewed drive-by diffs in untouched files; tokens spent on cosmetics | Bounded D1; record the rest |
| A percentage-reduction target | Eats the cheap rule, then forces risky fixes or disable comments | One whole rule per visit |
| Leaving slack under the ceiling | Warnings creep back into the room you made | Re-pin to the actual count in the same commit |
| A disable comment to hit a number | Destroys the signal and leaves the code worse | Only with an inline reason and a ledger row |
| Baselining errors | An error ceiling is permission to ship errors | Ceilings hold warnings only |
| A skipped test with no row, or a committed focus marker | The green count hides it | Same-line ledger citation; focus markers are always fixed |
| A ledger row marked resolved and left in place | The costume again | Delete it in the fix commit |
| Ordering the ledger by severity | Rots on every insert and merge; nobody reads it that way | Group by repo; generate any severity view |
| Omitting the disposition line when nothing was found | Indistinguishable from not reporting | "Defect disposition: none observed." |

## Adapt it to your setup

- **Solo, one repo:** one `KNOWN-DEFECTS.md` at the root, one `--max-warnings` in your lint script, and one line in `AGENT.md`: *"every warning or error you observe gets D1, D2, or D3, named in the summary."* That's the whole protocol at day-one scale.
- **With pull-request review:** the summary line can live in the PR description, because a human will read it — but keep the ceiling, because reviewers normalize noise too.
- **Many repos:** one ledger for all of them, grouped by repo, beats one per repo — the cross-repo view is where "the same warning in five forks" becomes visible.
- **Adopting it on an old codebase:** don't start with cleanup. Measure, pin every repo's ceiling at today's count, and let the campsite rule and one-rule-per-visit bring the numbers down. The first week changes nothing visible, and from then on nothing gets worse.

## Related

- [Honest Checks](honest-checks.md) — when the defect is in the instrument: checks that can't fail, can't run, or pass the wrong thing
- [The QA Gate](qa-gate.md) — where the disposition line lands in every completion summary
- [Bounded Deviation](bounded-deviation.md) — why a scope-expanding fix drops from D1 to D2
- [The Sanity Check](sanity-check.md) — a defect that contradicts the spec is a delta first
- [The Delta Log](delta-log.md) — the same "count until it's a pattern" instinct, applied to spec corrections
- Back to the [main playbook](../README.md)
