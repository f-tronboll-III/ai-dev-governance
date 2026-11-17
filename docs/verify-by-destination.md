# Verify by Destination

> A response describes the *request*. Only the destination describes the *result*. A 200 from a parked domain, a message id for an email that went to the wrong place, a push that deployed nothing — each one is a healthy answer from the transport, and the agent reports it as done. Before you call something done, name where it was supposed to land, and look there, from outside the system that told you it worked.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The QA Gate](qa-gate.md) · **See also:** [Honest Checks](honest-checks.md), [Reading the Outside World](external-sources.md)

---

## The wall

Five failures from one shop, none of which produced an error:

- **The parked domain.** A business directory listing's Website field pointed at a domain the business had let lapse. A parking service had picked it up, and it now fingerprinted each visitor and sold the click. It returned HTTP 200. Every link check passed.
- **The successful sitemap.** The only sitemap a search console had on file was a leftover from an old site builder. For fourteen months it had been a 404 listing thirteen URLs of a site that no longer existed — and the console reported its status as *Success* the whole time. The search engine had never been told about the real one.
- **The handler subscribed to nothing.** A payment webhook handler switched on six invoice events, with tests. The vendor endpoint was subscribed to none of them. Paid and voided invoices never reconciled, and collected money sat in the account invisible to the books. The repo could not have revealed this; the subscription lives at the vendor.
- **The push that deployed nowhere.** Three consecutive commits reported `pushed`. The host had built none of them: the agent had overridden the git author identity, and the host refuses to build commits by an author with no seat on the team. The deployment state read *blocked*, with no build log — because no build ever started.
- **The draft that sent.** The operator's standing rule was *"I read drafts before they go out."* A create-draft call returned a perfectly plausible id. The client reply had already gone to two recipients, unrecoverably. Later instances showed drafts flipping to *sent* seconds or minutes after creation, sometimes in creation order while no agent call was running. The cause is still unproven. The rule below doesn't depend on knowing it.

Every one of those was a correct answer to the question the agent asked — *did the request succeed?* — and a wrong answer to the question that mattered: *did it land where it was supposed to?*

## The rule

1. **Name the destination before you call it done.** Every action has one: the page a link lands on, the deployment a push produces, the mailbox a draft sits in, the database row an environment variable points at. If you can't name it, you can't verify it.
2. **Verify from outside the system that reported success.** The system that says "sent" or "saved" is the one place that can't tell you it was wrong.
3. **Assert where you landed before believing what you read.** Check the status and the final URL before trusting a page's content. Never report a count from a run in which some pages couldn't be read.
4. **Read back the irreversible ones at the moment you report them.** "Staged," "drafted," and "queued" are claims about a state *right now*; make them only from a read made right now.
5. **When the destination is wrong on something outward-facing, disclose first and investigate second.**

## How it works

### The destination for each thing you'll call done

| You did… | The transport says | Look here instead |
| :---- | :---- | :---- |
| Published or fixed a link, a listing field, a bio link | 200 | The final landing URL, after redirects, is yours — and it's the page you meant |
| Set up a redirect | 301 | Follow it with a hop cap. A redirect loop answers 301 forever; in a command-line fetch, "too many redirects" is the tell |
| Submitted a sitemap | "Success" | The submitted URL is the current one, and it fetches 200 with your pages in it |
| Shipped a webhook handler | Tests pass | The vendor's enabled-events list includes every event the handler switches on, and the signing secret matches |
| Pushed a fix | "pushed" | The host's latest deployment for *that commit* is ready. *Blocked* and *canceled* are the silent tells |
| Created a draft | An id | Read the message back; its state is *draft* at the moment you say so |
| Sent an email | 200 and a message id | The provider's delivery record for that id — a suppressed recipient can still return 200 and an id |
| Copied an environment value | 200 from the API | Read the row back individually; compare fingerprints, never by eye; resolve which endpoint it points at |
| Added a DNS record | 200 from the provider | That provider is the one actually serving the zone, and the name now resolves |
| Built an inbound mail forwarder | 200 per message | A real message from an outside address arrives *with its body* |

That last row earned its place. A forwarder running on a send-only key couldn't read inbound message bodies, so it forwarded blank mail for two days — and the spam filter downstream, starved of headers, scored every message clean. Both components looked healthy from every angle that normally gets checked. Downstream, an empty body is indistinguishable from an empty email, so a failed fetch must say so in the message itself.

### From outside

The repo can't tell you what a vendor is subscribed to, what a directory listing says, or what a search console has on file. A dashboard owned by the system that did the work can't tell you it did the work somewhere else. Check from the outside: fetch the URL yourself, read the vendor's own configuration through its API, send a real message from an outside address. **A green answer from the wrong destination is the most expensive green there is**, because it closes the investigation.

### Prove you fetched the page

A crawl, a link check, or a sweep that counts things must first prove each fetch actually returned the page. Assert the status and the landing URL *before* reading any content: a 404 page inherits the site's layout and carries a plausible title and canonical tag belonging to no real page; a firewall challenge page is a 200 with somebody else's content. If any page in a run was unreadable, the run's counts are a floor, not a total — and the run should exit non-zero to say so. See [Honest Checks](honest-checks.md).

### Read back the irreversible ones

For anything outward-facing — a draft, a scheduled post, a queued send — the safety of "prepared but not sent" is only as good as your last look at it:

- **After every create, read it back** and report the state the system shows. Claim "awaiting your send" only if a read made at that moment says *draft*.
- **Never edit an outward-facing draft in place.** An error like "message is not a draft" is the tell that it already left. Delete the old one by its exact id, confirm it's gone, then create the correction.
- **One draft per subject.** Two live drafts on one subject preceded one of the observed sends; and after creating any further draft, re-check the earlier one — a listing only shows that a draft stopped being listed, not why.
- **If a draft is missing, say so in the first line of your next message.** Disclose first, investigate second, and never call it "drafted" in the meantime.
- **Never "correct" an already-sent message on your own.** A follow-up is a second outward action, and it's the Operator's call ([the Operator's line](two-roles.md#where-the-line-actually-falls)).
- **Scoping an integration to drafts-only is a mitigation, not a guarantee.** It narrows what the agent can do; it says nothing about what the client application or a person does next.

### Pointers drift together

Business listings, social profiles, and contact fields are maintained by hand, over years, by whoever had the password. They drift as a set, so **one bad pointer is strong evidence that its neighbours are stale too.** On any handover, inventory the client's pointers before touching their site, and sweep each profile whole — bio link, contact fields, *and* factual claims, because a bio is a pointer to a capability. One such sweep found six bad pointers across three platforms: a bio link that redirected to itself fifty times, a contact button wired to a mailbox nobody could open, and a profile inviting ten thousand followers to shop at a store with no products in it. None of it was visible from the repo, and nothing had errored.

## Worked example

An Executor fixes the webhook handler from the wall and is about to report *"fixed and pushed."*

```text
1. Push                          → git: pushed. Not evidence. Next.
2. Deployment for this commit    → state: BLOCKED, no build log.
                                   Cause: the commit's author identity has no seat on the host.
                                   Recovery: one more commit by the configured identity → READY.
3. Endpoint, from outside        → POST without a signature → 400 (payment-class handler: correct).
4. Vendor subscription           → enabled events: 2 of the 6 the handler switches on.
                                   Diff applied through the vendor API; read back: 6 of 6.
5. A real event                  → vendor's test event sent; audit ledger row processed,
                                   invoice reconciled in the books.
6. Report                        → "Live: commit abc123 deployed (READY). Vendor subscription
                                   corrected from 2 to 6 events. Test event reconciled."
```

Steps 1 and 3 would have passed on their own. Steps 2 and 4 are where the work actually was — and both were invisible from the repo.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| "The link returns 200" | A parked domain, a login page, and a firewall challenge all return 200 | Assert the landing URL, and that the destination is yours |
| Status-code-only link checks | A redirect loop is a well-formed 301 forever | Follow redirects with a hop cap; treat "too many redirects" as failure |
| Reporting "fixed and pushed" | The deploy may be blocked or canceled with no build log | Report the deployment's state for that commit |
| Overriding the git author for convenience | Hosts can gate builds on the author's identity, silently | Commit as the configured identity; attribute roles in a trailer |
| Checking a handler without the vendor's subscription | A correct handler subscribed to nothing is dead code that reads as working | Diff the handler's events against the vendor's enabled list |
| "The tool returned an id, so it's drafted" | The message may already be sent | Read it back; report its state |
| Reading the repo to confirm an external listing | The value lives where the repo can't see | Fetch it from outside |
| Fixing one stale profile field | Its neighbours drifted with it | Sweep the whole set |
| Reporting a count from a run with unreadable pages | The count is a floor that reads like a total | Fail the run, or report it as partial |
| Investigating a wrong outward action before telling the Operator | A sent client email sits undisclosed | Disclose in the first line, then investigate |

## Adapt it to your setup

- **Solo, one site:** the habit is one extra fetch. After any change that's meant to be reachable, fetch it from your own machine and look at the final URL and the first lines of the body. After any push, look at the host's deployment state, not the push output.
- **Email-capable agent:** put "read every draft back and report its state" in your `AGENT.md`. It costs one call and is the only thing between "drafted" and "sent."
- **Client handovers:** the pointer inventory comes first, before any code — businesses predate their websites, and the first failures are usually in places the repo has never seen.
- **Many sites:** a small probe script that asserts status and landing URL, exits non-zero when any page was unreadable, and is the only thing allowed to produce a count. Run it from outside your own network where you can.

## Related

- [Honest Checks](honest-checks.md) — the same lesson for the checks themselves: an instrument must be able to say "I couldn't look"
- [The QA Gate](qa-gate.md) — "pushed and green" becomes "deployed and verified at the destination"
- [Reading the Outside World](external-sources.md) — the fetch discipline, and why a blocked source is not an absent one
- [Environment Targeting](environment-targeting.md) — the destination of every database write
- [The Two Roles](two-roles.md#where-the-line-actually-falls) — outward-facing sends, and why "prepared" needs a read-back to stay true
- [Autonomous Loops](autonomous-loops.md) — webhook response classes, and why a handler can't tell you what it's subscribed to
- Back to the [main playbook](../README.md)
