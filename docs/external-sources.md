# Reading the Outside World

> Every fact your agents pull from outside the repo — a vendor's API contract, a webhook payload shape, a regulator's docket, a competitor's pricing — arrives through a fetch that can silently hand back an empty shell, a stale mirror, or a firewall block dressed as a page. The tool reports success either way.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Trust Gradient](trust-gradient.md) · **See also:** [Model Orchestration](model-orchestration.md), [Autonomous Loops](autonomous-loops.md)

---

## The wall

Three failures, one root.

**The shell.** An agent fetches a vendor's documentation. The response is HTTP 200 and 4KB of navigation chrome, because the content renders client-side. The agent reads what it got — no rate limits mentioned — and designs around a limit that is documented, on that exact page, to anyone whose browser runs JavaScript.

**The scavenge.** The primary docs are a single-page app and awkward to read, so the agent goes looking for an easier copy: an SDK's README, a community wiki, a blog post walking through the same integration. It finds one, and it's clear and well-organized and a year stale on precisely the webhook payload shape being implemented. The API data was right the whole time; the convenient secondary reference is what cost the day.

**The silent watcher.** A scheduled monitor polls twelve sources. On its first live run, five return 403 — all four government pages behind a web application firewall, plus a federal docket — because the fetch went out with a default, bot-shaped user agent. Nothing throws. The monitor's heartbeat is green. It reports healthy while **watching nothing on exactly the five highest-value sources**, and you find out when a change you should have caught slips through.

The root under all three: **the fetch succeeded and the content was wrong**, and nothing in the tool's response distinguishes that from success. Agents are exceptionally good at reasoning over whatever text they were handed, which is precisely the problem — a confident answer built on an empty shell looks identical to a confident answer built on the real page.

## The rule

1. **Go to the primary source and render it.** Don't hunt for an easier-to-read copy. Secondary references go stale even when the primary is sound.
2. **Escalate the tool; don't downgrade the source.** Plain fetch → a browser that executes JavaScript → optional cheap-tier distillation of what the browser rendered.
3. **A distillation is a summary, not a contract.** Contract-critical facts — parameters, payload shapes, auth and signature schemes, required headers, limits — are checked against the rendered primary before anything is built on them.
4. **Authenticated, interactive work runs in the human's real browser session**, driven by a down-modeled agent under a spec the heavy tier wrote and verifies.
5. **Every request an agent writes or makes — scheduled watchers and one-off vendor calls alike — sends a browser-like user agent, fails per-source rather than per-batch, and verifies that what it extracted exists in the bytes it fetched.**
6. **Every fetched fact has three outcomes, not two:** found, confirmed empty, or *could not check.* A blocked, failed, or never-successful source is never reported as absence.

## How it works

### The ladder

**Rung 1 — plain fetch.** Cheap, fast, correct most of the time. Try it first. What matters is recognizing when it failed *invisibly*: a body far shorter than the page's apparent scope, a literal "enable JavaScript" string, a navigation shell with no content beneath it, a 200 whose body is a login form.

**Rung 2 — a browser that executes the page's JavaScript.** This is the only rung that renders a client-built page, and you go straight here the moment rung 1 returns a shell. The trap is the detour: an agent that hits a shell and then starts *searching for an alternate plain-text copy of the docs* has silently swapped a tool problem for a source problem, and will land on a secondary reference of unknown age.

**Rung 3 — distillation (optional).** Hand the rendered DOM to a cheap or [offload](trust-gradient.md) tier to crunch it into clean markdown or extract named fields. This is [down-model the gathering](model-orchestration.md) applied to a fetched corpus, and it answers the real complaint about rung 2 — that raw rendered output is tedious and token-heavy on an expensive model.

One thing about rung 3 that has to be said plainly, because agents reason past it: **the distiller is not a browser.** It cannot fetch, navigate, or render anything. It only processes text that a browser already retrieved. An agent that "asks the offload model to check the docs" is asking a model to invent them.

### Primary versus secondary

| Primary | Secondary |
| :---- | :---- |
| The vendor's own current documentation | An aggregator, wiki, or blog post |
| The API's own schema / OpenAPI document | An SDK's README |
| The live endpoint's actual response | Another model's summary of any of the above |
| The regulator's own docket page | Your own notes from three months ago |

A secondary reference is acceptable as a *convenience* only when it demonstrably matches the primary — which means you read the primary anyway, so the convenience is mostly imaginary.

And distilled documentation is itself a secondary source. A model's summary of a contract is structurally the same object as the stale blog post: readable, plausible, and unaccountable. It carries the [unverified tag](trust-gradient.md) until an authoring actor checks the contract-critical fields against the rendered primary. Everything else in the distillation — orientation, structure, what section to look in — can be used freely. It's the parameters and payload shapes that have to be checked, because those are the ones that fail silently at runtime.

### Authenticated and interactive work: the sidecar

The headless browser is the right tool for unauthenticated fetch, render, scrape, and scriptable navigation. It is the wrong tool for two things: sessions behind a login, and interfaces built on canvas or drag-and-drop, where element-guessing heuristics misfire on visual placement.

For multi-step work behind a login — pulling a report from a SaaS portal, placing fields on a document template, driving an editor — drive **the operator's real, already-authenticated browser** through a sidecar agent, and split the work by the orchestration rule:

- **Heavy tier writes the spec** — precise, ordered, unambiguous about what to click and what a correct result looks like.
- **A mid tier executes** the mechanical browser work in the operator's session.
- **Heavy tier verifies the outcome**, through an API wherever one exists rather than by trusting the executing agent's report.

The operator invokes it, because it runs in their session — and **the operator types every password and every two-step code, never the agent.** This is the same shape as every other delegation in the playbook: the judgment stays up, the mechanics go down, and the verification comes back up.

### Scheduled watchers

Anything that polls public sources on a schedule — a regulatory monitor, a price or availability watcher, an aggregator feed, a change detector — carries three obligations beyond the ordinary [autonomous-loop](autonomous-loops.md) discipline.

1. **Send a browser-like `User-Agent` and `Accept-Language`.** Many sites — government pages behind firewalls especially — return 403 or 405 to a default agent string and 200 to a normal browser one. A bot-shaped user agent makes the watcher silently watch nothing on exactly the sources that were worth watching.
2. **Be robust per source, not per batch.** Sources block, move, 404, and change shape. Log and skip a failed fetch rather than crashing the run; *correct* a bad URL instead of retrying it forever; prefer a structured feed (RSS, JSON API) over scraping HTML wherever one exists.
3. **Verify what the fetch yielded before it drives state.** An extracted or proposed URL must actually appear in the bytes that were fetched. Without that check, a model asked to "find the docket link" will produce a plausible one that has never existed.

The failure mode these prevent is the worst one a monitor has: **the silent no-op.** Nothing errors, the heartbeat updates, and the absence of alerts reads as "no changes" rather than "not looking." Test a new watcher against the real sources on its first run, never against fixtures — fixtures cannot 403.

The user-agent rule applies to one-off calls too, not just watchers — and "browser-like" is chosen per host, not fleet-wide. One government mapping service didn't return a 403 at all: it **hung**, with no error and no rate-limit response, on any user agent containing an email address, a URL, or the word "bot" — exactly the contact details a polite client is told to include. Retry-with-backoff turned each hang into a minute and a half per record. A timeout from a government host means: bisect the user agent against one fixed URL before blaming the server.

### Absence versus failure

A fetch has three outcomes — *found*, *confirmed empty*, and *could not check* — and most failures in this section come from collapsing the third into the second.

- **An OR over two sources hides the dead one.** A hazard page combined two public feeds. One had 404'd since the day it was wired in — the repo had never once fetched it successfully — and the page still showed a fresh timestamp and *"no nearby incident,"* because the read path OR-ed the dead source with the healthy one. That's a false negative about an active hazard, on a hazard page. Persist per-source status, last success, and last attempt; distinguish *never succeeded* from *stale* from *empty*; and surface a dead source by name rather than routing around it.
- **Success with nothing inside.** A bulk-data API's newest dataset returned zero files with `status: successful`, so taking the latest date produced a confident empty answer. Assert a non-empty parse before accepting an empty result.
- **An error inside a 200.** A geographic query with the wrong field-name case returned its error *wrapped in a 200*, which read as "outside coverage" and reported *no parcel* for an entire property portfolio. Read the body, check for an error object, and look up the service's own field list before querying it.
- **Rate limits that read as findings.** A naive loop produced eighteen source failures on one 28-record run, each one indistinguishable from a real gap in the data. Pace the loop, back off on 429, report every failed source by name, and exit non-zero.
- **401 means a bad credential; 403 often means a firewall.** A vendor API behind a web application firewall returned 403 with a bare `error code: NNNN` — the firewall's format, not the vendor's JSON — to a default HTTP client, while `curl` got 200 on the same key and endpoint. The 403 was read as *"this key lacks permission,"* which produced a false blocker and two unnecessary replacement keys. Re-issue the request from a second client before concluding anything; when two clients disagree, the problem is the client.
- **Blocked is not absent.** In external recon, a firewall 403, a social network's rejection, or an empty search page is evidence about *your client*, not about the target. Four sources a recon pass recorded as blocked all held real data that the operator retrieved by hand in under a minute. When a source is blocked, record it as *unverifiable* and ask — an operator's screenshot outranks a blocked probe.

### The empirical-reachability corollary

The same discipline turned inward. For any claim that one of *your own* surfaces is publicly reachable — an endpoint, a feed, an image, a webhook target — **hit it and look at what comes back.** Confirm the response is the expected content type, not an HTML login page with a 200 on it.

This is not theoretical: three silent production breaks in one engagement all had the same shape — a route that read as public in the source but wasn't, because of handler auth-gate ordering in one case and a missing middleware bypass in another. Code-reading implies reachability. A request proves it. One `curl` is cheaper than the outage.

## Worked example

Implementing an inbound webhook for a vendor whose documentation is a single-page app.

```text
1. Plain fetch of the webhook docs page
     → 200, 6KB, mostly <nav>. No payload schema anywhere in the body.
     → Recognized as a shell. Do NOT go looking for a friendlier copy.

2. Headless browser renders the same URL
     → full page: signature scheme, header names, payload examples, retry policy.

3. Rendered DOM handed to the offload tier under the gather contract
     → clean markdown, tagged [UNVERIFIED], ~90% fewer tokens on the heavy model.

4. Executor verifies the contract-critical four against the rendered primary:
     signature header name · signing algorithm · timestamp tolerance · retry count
     → three match; the timestamp tolerance in the distillation was wrong.

5. Handler implemented against the verified contract.
     Endpoint hit live after deploy: confirms 200 + expected content type,
     not a platform auth page.
```

The distillation did the token-heavy grind. The browser did the rendering. The Executor did the four checks that would have produced a silent signature-verification failure in production if it had trusted the summary. Each rung of the ladder did exactly the work it was suited for, and nothing was believed because it was convenient.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| Reading a fetched shell as if it were the page | 200 plus empty content is indistinguishable from success in the tool's response | Recognize the shell signals; escalate to a rendering browser |
| Hunting for an easier-to-read copy when the primary is awkward | Swaps a solvable tool problem for an unsolvable staleness problem | Escalate the tool; the source stays primary |
| Treating a distilled summary as the contract | A model's summary of an API is another secondary source, with the same failure mode | Verify parameters, payloads, and auth against the rendered primary |
| Asking the offload model to "look up" something | It has no browser; it will produce something plausible instead of something real | Fetch with a browser first; the offload tier only processes retrieved text |
| Driving an authenticated portal with a headless browser | Session handling and canvas interaction are exactly where it's weakest | Sidecar in the operator's real browser; heavy writes the spec and verifies |
| Shipping a watcher with a default user agent | Silently watches nothing on firewalled sources; heartbeat stays green | Browser-like UA plus `Accept-Language`, verified against real sources |
| One failed source aborts the whole polling run | A single 404 blinds you to the other eleven sources indefinitely | Log and skip per source; correct dead URLs rather than retrying them |
| Testing a watcher only against fixtures | Fixtures never 403, never move, and never change shape | First run goes against the real sources, and you read the results |
| "The route is public — I read the code" | Auth-gate ordering and middleware bypasses make the source misleading | Hit the live URL; confirm the content type of what comes back |
| OR-ing two sources and showing "nothing found" | The dead source disappears behind the healthy one | Per-source status; a dead source is a named gap |
| Reading a 403 as missing permission | A firewall answers in the same status code | Re-issue from a second client before concluding |
| Retrying a hanging government endpoint | The hang may be triggered by your user agent | Bisect the user agent on one fixed URL |
| Recording a blocked source as a negative finding | "Couldn't look" becomes "nothing there" | Mark it unverifiable, and ask the operator |

## Adapt it to your setup

- **Solo, no browser tooling.** The ladder still applies with one rung fewer: when a fetch looks like a shell, *you* open the page in your own browser and paste what matters. Slower, same discipline.
- **Occasional doc lookups only.** The one rule to keep is primary-source-first. The distillation rung earns its place when raw rendered pages start crowding your context window, not before.
- **Heavy research workloads.** All three rungs, with the distiller under the [gather contract](model-orchestration.md#the-gather-contract-in-full) so it returns raw material instead of conclusions.
- **Running scheduled watchers.** The three watcher rules go into your global context file, not into each watcher's own code — they're a property of every future watcher, and the second one is always written by someone who didn't build the first.
- **Compliance-sensitive sources.** Add provenance to the record: which URL, rendered at what time, by which rung. A distilled claim that later matters legally needs to trace back to a fetch, not to a summary.

## Related

- [The Trust Gradient](trust-gradient.md) — why a distillation is a secondary source and what clears it
- [Model Orchestration](model-orchestration.md) — the gather contract the distiller runs under, and the heavy-writes-spec / mid-executes split
- [Autonomous Loops](autonomous-loops.md) — the general discipline for anything running with no one in the room; watchers add three rules to it
- [Infrastructure Limits](infrastructure-limits.md) — where verified vendor limits and gotchas are written down so the next session doesn't re-fetch them
- [The Sanity Check](sanity-check.md) — the same instinct applied to your own production state rather than someone else's docs
- [Verify by Destination](verify-by-destination.md) — the outward-facing half: confirming where your own links, sends, and deploys actually landed
- [Honest Checks](honest-checks.md) — the three exit codes, which are this section's three outcomes applied to a check script
- Back to the [main playbook](../README.md)
