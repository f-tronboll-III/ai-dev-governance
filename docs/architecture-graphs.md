# Architecture Graphs

> A Strategist designs from outside the codebase, so a map of a repo's structure is genuinely useful — **built fresh, for the one repo about to be designed on.** Kept as a standing duty across every repo, the same map goes stale unread and starts telling confident lies. Build views on demand, answer structural questions from the code, and keep the one part that actually catches defects: reading every consumer of what changes.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Two Roles](two-roles.md), [The Context Cascade](context-cascade.md) · **See also:** [The Sanity Check](sanity-check.md)

---

## The wall

There are two walls here, and the second one was built by solving the first.

**The first wall: the Strategist is blind.** It sits down to design a Tier 3 change — say, "move the billing logic out of the storefront into a shared package." A defensible handoff needs to know which files import the billing module, which routes call those files, which environment variables and external services those routes touch, and which scheduled jobs depend on any of it. Without a structural view, the Strategist greps blindly (slow, incomplete, and the context fills with file dumps), asks the Executor to enumerate (a human-paced round trip that comes back as prose), or guesses (the handoff ships incomplete and the gaps surface mid-execution as deltas approved in a hurry).

**The second wall: the maintained graph nobody reads.** The obvious fix is to give every repo a generated architecture graph, regenerate it as part of the [QA gate](qa-gate.md), and require every triggered handoff to cite it. An earlier version of this doc prescribed exactly that, and the shop it came from ran it for seven months. Then it measured what was happening:

- Strategist-authored handoffs fell from about two hundred a month to two, as more design moved to the Executor. Handoffs citing a graph fell from seventeen a month to one.
- Across more than a hundred Executor sessions over two months, **not one opened a graph or an onboarding doc.** The diff-analysis tool ran in six.
- The retrieval index built over the graphs had answered six queries in its lifetime.
- More than half the graphs were six months old, because the quarterly refresh never ran. Several described retired repos. Several live repos had none.
- One live repo's graph was 124 commits and roughly 700 code files behind. Refreshing it would have meant a near-complete re-analysis.
- Twice, an Executor recorded that the graph was blind to most of its diff: five of nine changed files absent in one case, three of five in another.

And the doctrine made it worse than useless. The consumption rules told every assistant to *prefer* the graph over grepping the code, so a Strategist session would have planned confidently from a picture of a different codebase. **A stale graph that is trusted is worse than no graph at all.**

There was exactly one real catch in the whole period. On a schema change, the graph flagged a file as a consumer of the changed table; reading that file found a "Pay Now" button still showing on an already-paid bill. But what found the defect was *reading every consumer of the change* — and a search of the current code does that without ever going stale.

## The rule

1. **Answer structural questions from the current code.** *What imports this, who reads this table, what calls this route* — search the tree. It is always current, and searching is cheap.
2. **A graph is a dated snapshot.** It describes the commit it was generated from, and its first line says which. A graph older than the repo's last commit describes a different codebase: never take a structural fact from one without checking the code, and never cite one as current.
3. **Build a fresh view on demand, for one repo, before design.** When a Strategist is about to design work on a repo, the Executor builds a view of *that* repo and hands it over. A single-file pack of the repo is usually enough; a full structural map is worth its cost occasionally.
4. **Generated, never hand-written.** A hand-maintained map lies within a week. If you can't generate it, don't draw it.
5. **The defect-catching part belongs to the Sanity Check.** On triggered work, the Executor finds every consumer of what changes — in the current code — and reads each one. That is the [blast-radius leg](sanity-check.md) of the pre-edit pass.
6. **No standing refresh duty without a reader.** A maintenance duty is only worth reinstating alongside evidence that something reads the result.

## How it works

### What a useful view contains

Skip the temptation to draw everything. A view earns its place by answering the questions a Strategist needs to reason about *changes*:

- **Modules and files**, grouped by directory, with their internal dependencies.
- **External surfaces** — route definitions (path → handler file), CLI entry points, exported package symbols, scheduled jobs (schedule → handler), webhook endpoints.
- **External dependencies** — third-party packages with versions, environment variables read, external services called.
- **Shared-infrastructure references** — pointers into any orchestrator, registry, or other shared resource.

Not every variable, function body, or test. The view is for reasoning about *wiring*; behavior lives in the code, and the view tells you where to look.

### Two kinds of fresh view

| View | What it is | When it's worth it |
| :---- | :---- | :---- |
| **Repo pack** | The repo's source, filtered and concatenated into one file the Strategist can load whole | The default. Minutes to build, cheap, and it *is* the code — nothing to go stale between building and reading |
| **Structural map** | A generated analysis — layers, dependency edges, a guided tour of the main flows | A large unfamiliar repo, a cross-cutting redesign, onboarding a new Strategist to a codebase |

Either way, the generated artifact carries its generation time and the commit it was built from in its first line, so any reader can tell at a glance whether it still describes the tree.

### The handover

1. The Strategist (or the Operator) says which repo the next design touches.
2. The Executor builds the view from the current tree and hands it over — pasted into the session, or written to a scratch location the Strategist can read.
3. The Strategist designs against it, and cites it under the [receipt standard](trust-gradient.md#the-receipt-standard): the artifact's name plus its timestamp and commit. *"Per the repo pack built at `a1b2c3d`, 2026-11-14 …"*
4. The Executor's [pre-edit pass](sanity-check.md) checks the design's structural claims against the code as it stands *at execution time*, which may be days later. The view grounded the design; it doesn't excuse the Sanity Check.

The view is disposable. Keep it only as long as the design it served; next time, build another.

### What stays maintained

Retiring the graph refresh does not mean retiring every shared reference. Small, curated, cross-repo files that sessions *actually read* stay current exactly as before — the [infrastructure-limits file](infrastructure-limits.md), a map of every scheduled job across the fleet, a short registry of which repos exist and what they expose. The test is the same one that retired the graphs: **does something read it?** A registry that every infrastructure session consults earns its upkeep. A per-repo graph that nothing opens doesn't, however well-generated it is.

## Worked example

A shop runs five repos. The Strategist is asked to extract the billing logic from the storefront and the membership site into a shared package.

1. **The view.** The Executor packs both repos from the current tree — a few minutes — and hands the two files over, each headed with its commit.
2. **The design.** Two minutes of reading shows the storefront has four routes and one scheduled job importing `lib/billing`; the membership site has three routes and no jobs. Both read the same payment-provider secret, but the membership site also reads a price identifier the storefront doesn't, and calls the provider's REST API with `fetch()` rather than its SDK. The handoff is designed against that picture: shared package first, storefront migrated first (more usage, surfaces integration problems early), then membership, with environment handling specified per repo.
3. **The blast-radius leg.** Three days later the Executor runs the pre-edit pass. The trigger fires (billing), so it searches the *current* code for every importer of the billing module and every reader of the invoice table. The search turns up a customer-portal file added the day after the packs were built — it renders an invoice's payment state and would have broken on the new package's types. One added step in the delta, approved, executed.

The pack made the design fast. The search at execution time made it correct. Neither required a graph that someone had to remember to refresh.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| A standing duty to regenerate graphs after every change | Costs a near-full re-analysis per repo for output nobody opens, and the duty quietly stops running | Build a view on demand for the one repo being designed on |
| A doctrine that prefers the graph over the code | A stale graph becomes a confident wrong plan | Code is the source of truth; a graph is a dated hint |
| Mandatory graph citations in every triggered handoff | Becomes ritual: citations written, nothing read | Require the blast-radius read instead — it's the part that finds defects |
| A refresh schedule ("quarterly") | Schedules for unread artifacts don't run; most graphs end up six months old | Make freshness a precondition at the point of use |
| Maintaining the graph by hand | Drifts within a week; the lie is worse than the absence | Generate it from the tree, or don't have one |
| Citing a view without its timestamp and commit | Can't be evaluated for staleness — it just looks rigorous | First line of every generated view names its commit; cite it |
| Treating the view as the source of truth for behavior | It contains wiring only | Use it to find files; read the files for behavior |
| Keeping the view after the design ships | The next design reads it as current | Discard it; build another next time |

## Adapt it to your setup

- **Solo, one repo:** you probably need no graph at all. When you want your Strategist to see the repo, pack it into one file and paste it in. Your Executor's search is the blast-radius check.
- **A few repos:** pack per repo, on demand. Keep one hand-curated table of what each repo is and exposes; it's short, it changes rarely, and every cross-repo design reads it.
- **Monorepo:** the "repo" you pack is the package being changed, plus the root listing of packages and their cross-package imports — which your monorepo tool can usually emit for free.
- **You already maintain graphs:** measure before you keep paying. Count how many sessions opened one last month, and how far behind the newest commit each one is. If the answers are "few" and "far," retire the refresh duty and keep the generator for on-demand use.

## Related

- [The Sanity Check](sanity-check.md) — the blast-radius leg, which replaced the graph-consultation step
- [The Two Roles](two-roles.md) — the read-only inspection window a fresh view extends
- [The Trust Gradient](trust-gradient.md) — the receipt standard behind "cite the view with its commit"
- [The Context Cascade](context-cascade.md) — the other Strategist input that travels with a fresh view
- [Durable Memory](durable-memory.md) — the same lesson for remembered facts: a dated claim, verified before use
- Back to the [main playbook](../README.md)
