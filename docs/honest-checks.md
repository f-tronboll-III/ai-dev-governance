# Honest Checks

> Agents write a lot of checks — linters, screens, sweeps, audits, probes — and a check is only worth what its *exit code* is worth. A check must be able to say three things, not two: *clean*, *found something*, and *couldn't look*. It reports how much it examined, it is proven able to go red before anyone trusts it green, and nobody ever makes it green by removing the thing it measures.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The QA Gate](qa-gate.md) · **See also:** [Defect Disposition](defect-disposition.md), [Verify by Destination](verify-by-destination.md)

---

## The wall

Four failures, all from checks that were working exactly as written.

**The check that was too loud.** A prose-style guard — meant to catch canned AI openers in generated copy — started reporting 496 violations and made its gate unpassable. None were real. Its parser couldn't tell a string from a comment, so one apostrophe in a comment produced phantom entries, including a bare comma that matched nearly every line of code. Everyone concluded the check was noisy. The quiet half is the point: the same bug **silently dropped sixteen real entries — every one containing an apostrophe** — which were precisely the canned openers the guard existed to stop. The loud failure supplied a wrong explanation ("it's noisy"), and that explanation pre-discounted every real finding. A known-broken tool is where real failures go to hide.

**The check that passed the wrong thing.** A screen for public web forms reported a repo clean while four forms behind one route had no bot trap, no submit-timing check, and a rate limiter that enforced nothing. It passed because the route imported *a* guard: it asserted that something guard-shaped was present, not that the required set was. A later audit found the screen had never seen a whole class of endpoint at all — it walked route files, and the framework also creates public POST endpoints from server-side form actions that live nowhere near the routes directory. Roughly 230 such files across nineteen repos had never been screened. The ratchet read *zero findings* everywhere while three live contact forms had none of their three required guards and one repo charged cards from an unthrottled public action.

**The check that couldn't run.** An agent wrote a quick sweep to classify lint health across 25 repos, and every repo came back *tooling broken*. The sweep wrapped each run in a `timeout` command that didn't exist on that machine; every run exited 127, and the error-detection grep matched the path string in the shell's complaint. The result was a confident, well-formatted table — and it was very nearly reported as findings. An exit code is a signal, not a verdict.

**The check made green the wrong way.** A crawler reported seven errors on a client site: a 404 on an email-protection path, plus five pages linking to it. The cause was a CDN feature that hides staff email addresses from harvesters and only resolves under its own JavaScript. The agent's first framing offered disabling the obfuscation to clear the crawl. The Operator ruled that framing wrong. The fastest way to zero findings is always to remove the control that produced them, and the report goes green with nothing in error.

When the shop audited all 37 of its own check scripts: 22 had no guard comparing coverage against an expectation; 24 conflated *the tool failed* with *the content failed*; none had ever been proven able to fail; none compared coverage to a previous run. One computed that it was partial, printed `PARTIAL RUN` — and exited 0.

## The rule

1. **Three exit codes.** `0` checked and clean · `1` checked and found something · `2` could not check. State the contract in the script's header, and make the header true.
2. **Could-not-look outranks found-something.** Print a partial run's findings — they're a floor — but never let the exit code claim completeness.
3. **Report coverage, store it, and fail on an unexplained drop.** A check that reports *zero findings across zero inputs* must be structurally unable to exit 0.
4. **Prove it can go red before you trust it green** — a negative control at birth, for the exit-2 path as well as the exit-1 path.
5. **Presence is not sufficiency.** A screen that finds "a guard" is not a screen that finds "the guards this surface requires." Teach it the shape of each surface, and enumerate surfaces from what actually accepts input, not from where files happen to live.
6. **Record how a count was measured, alongside the count.** When the number moves, the instrument may be what moved.
7. **Never trade a control for a clean report.** A green report is an instrument, not a goal; disabling the thing it measures is the one repair that never counts.

## How it works

### Three exit codes, all the way up

The contract has to survive every layer above the script. A CI step, a composite action, or a wrapper that collapses exit 2 into its "violations found" message tells the author the opposite of the truth — that the content is broken, when the check couldn't read it. Each layer either propagates the three states or says which one it collapsed.

And the counter-caveat: don't make "zero found" an exit-2 everywhere by reflex. Ask what zero *means* in this check's denominator. A counter of *every send site in the repo* that reads zero means it couldn't read the repo. A counter of *non-conforming* send sites that reads zero means the repo conforms — failing it would teach everyone to ignore the check.

### Coverage is an output

Every run prints what it examined: *"examined 41 forms in 12 files; skipped 3 (unparseable: listed below)."* Store that number between runs and fail when it drops without explanation. Baselines usually pin the *findings* count and never the *scan* count, so a screen that quietly starts reading half as many files looks like an improving codebase.

### Negative controls

At birth, point the check at something it must flag — a fixture with the defect — and confirm exit 1. Point it at something it can't read and confirm exit 2. A check that has never been seen to fail is a check you have no evidence for. The existing intuition — *a check that can't fail isn't a check* — only fires when a monitor goes quiet. The too-loud failure above never tripped it, because it got louder.

### Presence is not sufficiency

A static screen reads imports and file shapes. It can say *something guard-shaped is here*. It cannot say *the floor is met*. Closing that gap:

- **Each surface type has its own required set.** A public lead form, a model-backed chat endpoint, a payment action, and a webhook each need different guards; the screen must know which surface it's looking at and assert the whole set for it.
- **Enumerate by what accepts input.** Frameworks create endpoints that aren't in the routes directory — a server-side form action is a public POST to the page's URL, and must be judged *per exported action*, not per file. A middleware matcher scoped to API paths doesn't cover it.
- **Audit by the unit being protected.** One repo owned exactly the right guard and had wired it into one of its seven public forms. That one form took zero spam; the other six took all of it. Counted by route, it was "2 of 4 guarded." Counted by form, it was "1 of 7."
- **A catch-all gate counts only if it demonstrably gates** — returns 401 or 403, redirects, reads a session. A catch-all that only sets a header is not a gate, and a screen that credits it downgrades real findings.
- **Check the guard's inputs, not just its presence.** One unthrottled chat endpoint keyed its rate limit on a session id supplied by the client — free for a bot to rotate. A limiter is only as good as the value it counts.
- **An ancestor's gap is one per descendant.** Three of six findings in one sweep were the same two routes, copied into three repos forked from a common ancestor. Fix the ancestor before forking it again.

**A clean screen is evidence, not proof.** It means *nothing obvious*, never *audited*. Verify the required set by hand on the surfaces that matter most, and treat every clean run recorded before a screen was repaired as unproven.

### The instrument changes the count

One shop's baseline for a pacing screen moved from 45 to 46 to 54 findings over a few weeks — and every move was the instrument getting better, not the codebase getting worse. Another moved its environment-marker baseline the same way. Record the instrument's version and method with every count, or a trend line will tell you a story about the code that is really a story about the tool.

### Never trade a control for a clean report

When a report flags something a security control produces:

1. **Establish whether the finding is real**, verified from outside.
2. **If it's an artifact of the control working**, the disposition is *record*, not *remove*. Say plainly that the control is doing its job and the report is wrong about it.
3. **If it genuinely costs something** — real visibility, real traffic — quantify the cost and bring both options to the Operator: mitigate with the control kept, or accept the finding. Never present "turn it off" as the recommendation, and never act on it unasked.

The same shape shows up everywhere a number can be gamed: a firewall baseline kept on projects that have never been attacked; a lint warning silenced to hit a count; a private site that deliberately lets the major crawlers in so they can read its no-index header. Suppression is not remediation, in any of them.

### Before adding a detector, check the reading

One shop's external monitors produced three alarming reports in a week. All three were misleading — not wrong about what they saw, but wrong in what they implied. Detection was never the bottleneck; interpretation was. **Don't answer a detection gap by adding another detector** until you've confirmed the ones you have are being read correctly.

## Worked example

An Executor is asked to write a screen that every public form in a repo carries its required guards.

```text
Header:        exit 0 = screened, all forms conform
               exit 1 = screened, findings listed
               exit 2 = could not screen (unparseable files listed)

Negative control 1  → fixture form with a bot trap only          → exit 1 ✓
                      ("contact-form: has 1 of 3 required: missing timing, rate limit")
Negative control 2  → fixture directory with a syntax error       → exit 2 ✓
Negative control 3  → fixture with a server-side form action and
                      no route file                               → exit 1 ✓ (enumerated)

First real run      → "examined 7 forms (5 page actions, 2 API routes) in 9 files; skipped 0"
                      → exit 1: 6 of 7 forms missing their guards
Coverage stored     → 7

Two weeks later     → "examined 3 forms in 4 files"   → exit 2: coverage dropped 7 → 3, unexplained
                      (a refactor moved the actions to a directory the walker didn't cover)
```

Without the coverage line, the two-weeks-later run would have reported *three forms, all clean* — an improvement on paper, and four unguarded forms in production.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| One nonzero exit for both "found something" and "broke" | A tool fault gets reported as a fault in the code | Three exit codes, documented in the header |
| A wrapper that collapses exit 2 into "violations found" | Tells the author the opposite of the truth | Propagate the three states through every layer |
| Pinning only the findings count | A screen reading half as many files looks like progress | Store and ratchet the scan count too |
| Living with a perpetually red check because it's "just noisy" | Pre-discounts the one real failure inside the noise | Repair the instrument; never lower the signal |
| Adding a check with no proof it can fail | It may be structurally unable to | Negative controls at birth, for exit 1 and exit 2 |
| Trusting a sweep's table without confirming it ran | Exit 127 from a missing binary reads as a finding | Confirm execution first; an exit code is a signal, not a verdict |
| "A guard is imported, so the surface is guarded" | Presence isn't the required set | Assert the set required by each surface type |
| Enumerating endpoints by file layout | Framework-created endpoints are invisible | Enumerate from what accepts input; judge per export |
| Reading a count as a trend without the instrument's version | The tool improved and the code looks worse, or the reverse | Record method and version with every count |
| Disabling a control to zero a report | Green report, protection gone, nothing errors | Record as not-a-defect; bring real costs to the Operator |
| Adding a detector because the last report was confusing | More signal you also can't read | Fix the interpretation first |

## Adapt it to your setup

- **Solo, a couple of scripts:** the whole discipline is four lines at the top of each script — what 0, 1, and 2 mean, and what it prints as coverage — plus one run against a fixture that should fail.
- **CI-heavy setups:** make exit 2 a distinct, visible state in the pipeline rather than a generic failure, so "couldn't check" never gets read as "the code is bad" and waved through as flaky.
- **Agent-written checks:** add one line to `AGENT.md` — *"any check you write uses exit codes 0/1/2, prints its coverage, and is run against a failing fixture before it's trusted."* Agents write checks quickly; this is the cheapest moment to make them honest.
- **Security tooling:** put "never remove a control to clear a finding" in the global context file, where every session sees it. The pressure to make the audit pass arrives mid-task, which is the worst time to discover the rule.

## Related

- [Defect Disposition](defect-disposition.md) — what happens to what a check finds; this doc is about whether the check can be believed
- [Verify by Destination](verify-by-destination.md) — the same lesson for actions: a healthy response from the wrong place
- [The QA Gate](qa-gate.md) — the gate is itself a set of checks, and inherits all of this
- [Reading the Outside World](external-sources.md) — the fetch-side version: a blocked source is not an absent one
- [Enforcement, Not Attestation](enforcement.md) — a guard is a check that blocks; the same fail-direction choices apply
- Back to the [main playbook](../README.md)
