# Checkpoint: Metered-service discipline — infrastructure-limits + trap + watchdog
**Date:** 2026-05-31
**Status:** COMPLETE
**Session:** Perplexity Computer (CChat), as Contributor

## Plan
1. [done] Write `docs/infrastructure-limits.md` — platform-limits-as-dated-truth discipline
2. [done] Add "The metered-service trap" section to `docs/cost-management.md`
3. [done] Add "The watchdog pattern" section to `docs/autonomous-loops.md`
4. [done] Update README deep-dive table with infrastructure-limits row
5. [done] Update `architecture.md` Layered Protocols subsection (now six disciplines, not five)
6. [done] Cross-link new and existing docs (cost ↔ watchdog ↔ infra-limits, dep-mgmt ↔ infra-limits)
7. [done] Verify links, commit, push

## Origin: the gap that triggered this pass

Operator surfaced a real internal scar: a per-second-billed serverless DB fleet that silently burned $700 over two billing periods because nothing in the system noticed the DBs weren't reaching their idle state. The remediation produced a body of internal work (per-repo health-route fixes, a hosting-level platform-limits doc, a watchdog agent). Operator asked whether any of this belongs on the public repo.

The screening question I proposed and the operator accepted as the standing filter for future candidates:

> **Would a stranger with 1-5 repos and a different stack still recognize the failure mode?**

For the metered-service work, the answer is yes — every developer running a per-second-billed serverless dependency is one keepalive away from this exact bill. The lesson generalizes; the BTS-specific implementation (24 projects, org-soft-union-24949984, bts-brain as the ops home, BFOS family standards) does not. So this pass ports the *lesson*, not the *artifact*.

## Sanity Delta (against the operator's request)

| Operator request | Shipped as | Notes |
| :---- | :---- | :---- |
| The platform-limits-as-source-of-truth discipline | `docs/infrastructure-limits.md` (NEW) | Generalized from BTS's `platform-limits.md`; vendor names removed; "Vercel", "Neon", "Resend" replaced with vendor categories. Pre-flight protocol generalized. Gotchas as first-class is the load-bearing pattern. |
| The Neon "scale-to-zero" specific lesson | New "The metered-service trap" section in `cost-management.md` | Names the failure pattern by mechanism (keepalives, warm pools, polling crons), not by vendor. Worked example uses "a serverless Postgres" rather than naming Neon. Five-second test for self-diagnosis. |
| The watchdog agent shape | New "The watchdog pattern" section in `autonomous-loops.md` | Generalizes the Phase 3 agent into a reusable shape: poll → classify → alert → opt-in mitigate. Includes the "anti-overengineering" guard with the exact list from the BTS handoff's Phase 3h Δ#2 lesson. |

What was *not* shipped (deliberately):

- **The 24-project Neon map and per-repo audit.** Pure BTS-internal; vendor-specific; not generalizable.
- **The bts-brain-as-ops-home pattern.** Implies an enterprise shape (separate ops repo) that doesn't match most public readers. The watchdog pattern says "lives in a scheduled loop with its own owner" without prescribing where.
- **The cleanpunk-shop / studiolo / BFOS-fleet per-repo remediation specifics.** Same reasoning.
- **The CWS unauth/auth health-route branch split as a named pattern.** The principle (split liveness from health) is in the metered-service-trap section as rule 2; the specific CWS implementation shape would feel arbitrary on the public repo without the BTS context that produced it.
- **The semper-vets currently-alerting incident.** Not the right surface — that's Executor-domain (BFOS, CC), not Strategist-domain (and certainly not public-repo).

## Deviations from the plan

One. The original plan said "five layered disciplines" in `architecture.md`'s Layered Protocols subsection. Adding `infrastructure-limits.md` makes it six, which required two prose edits in that section (the "four more disciplines become load-bearing" → "six more" and the closing "These five layered disciplines" → "These six"). Caught and corrected before commit.

## Files modified this session

| Path | Change |
| :---- | :---- |
| `docs/infrastructure-limits.md` | NEW. Five rules: one-file-every-vendor / file-beats-memory / pre-flight-read-post-action-write / gotchas-first-class / verification-cadence. Five-second test. Worked example using an unnamed "transactional-email vendor." |
| `docs/cost-management.md` | Added `## The metered-service trap` section between "## When the Strategist is tempted…" and the Worked Example. Five sub-rules. Five-second test. Updated Related section with infra-limits and watchdog links. |
| `docs/autonomous-loops.md` | Added `## The watchdog pattern` section between "Rule 5 — Dry-run the first cycle" and "## When a loop misfires". Five elements of shape. "Why alert-default / mitigate-opt-in" asymmetry. Anti-overengineering list. Five-second test. Updated Related section with cost-management and infra-limits links. |
| `docs/architecture.md` | Added `infrastructure-limits.md` bullet to Layered Protocols subsection in the proper position (after architecture-graphs, before autonomous-loops). Updated counts: "four more" → "six more"; "These five" → "These six". Added infra-limits to the Related footer. |
| `docs/dependency-management.md` | Added cross-link to infra-limits in Related section ("same dated-and-verified discipline"). |
| `README.md` | Added `infrastructure-limits.md` row to deep-dive table in proper position. |

## Voice discipline applied

- Six-section deep-dive shape maintained for the new file (Wall / Rule / How it works / Worked example / Anti-patterns / Adapt-it / Related).
- The two section additions use the embedded-section pattern (no full six-section structure; they live inside an existing dive and follow its voice): a problem statement, sub-rules, and a five-second test. This matches how "When the Strategist is tempted…" sections work in other dives.
- Five-second tests added to all three: infra-limits ("am I answering from a file I just read, or from a number I just remember?"), metered-service-trap ("if nothing real is happening, will this dependency still be billing me?"), watchdog ("if a resource silently regressed today, when would I find out?"). The recurring five-second-test pattern is becoming a recognizable voice signature across the layered-disciplines dives.
- Vendor-agnostic discipline preserved. The "metered-service trap" deliberately names mechanisms (per-second-billed serverless dependencies, per-token LLM APIs, per-invocation function platforms, per-GB egress, per-message queues) rather than vendors. The infra-limits worked example uses "transactional-email vendor" without naming Resend, SendGrid, Postmark, or any other.
- Scar-tissue tone applied: the infra-limits opening uses the 90-crons / silent-auth-failure failure mode (the canonical BTS lesson from `platform-limits.md`), de-identified to "a small team" with vendor names replaced by category nouns. The metered-service-trap opening uses the $700-over-two-billing-periods shape, de-identified to "several hundred dollars higher than expected."

## Cross-link audit

The three new pieces form a tight triangle, plus existing-doc connections:

- `infrastructure-limits.md` ↔ README, architecture (layered protocols + Related footer), context-cascade, cost-management, autonomous-loops, dependency-management, secrets-and-rotation
- `cost-management.md` "metered-service trap" → `infrastructure-limits.md` (gotchas storage) + `autonomous-loops.md` (watchdog pattern as prevention)
- `autonomous-loops.md` "watchdog pattern" → `cost-management.md` (the trap it prevents) + `bounded-deviation.md` (asymmetric-cost reasoning) + `infrastructure-limits.md` (tuning data)
- `dependency-management.md` → `infrastructure-limits.md` ("sibling files" framing — dependencies and vendor limits are the two halves of "outside your control")

## Operator Action Block

None for the public repo. The session was docs-only.

The operator separately committed to building a list of other internal optimizations/protocols and applying the "would a stranger with 1-5 repos and a different stack still recognize the failure mode?" screen before bringing them to a future session. That's a self-directed Strategist action; no support needed from this session.

## Next-session context

- Public repo now has 17 dive documents (the original 10, plus `_deep-dive-template.md`, plus 5 added in commit `6c71986`, plus `infrastructure-limits.md` added in this commit). The Layered Protocols subsection in `architecture.md` now names six supporting disciplines.
- The reinstated standing rule (no direct edits to project repos by the Strategist) automatically resumes after this session. The suspension for the public repo specifically lifts; CC handoffs are again the default for project repos.
- The operator is curating a candidate list of additional BTS protocols for public-repo evaluation. When that list arrives, the screening procedure is: per candidate, name the failure mode → judge whether a stranger with 1-5 repos and a different stack would recognize it → if yes, port the lesson (generalized) not the artifact (specific). Future Strategist sessions should follow this same pattern.
- One internal item worth surfacing back to the operator (not for public repo): the BFOS `semper-vets` Neon endpoint is currently generating frequent watchdog alerts (visible in `bts-governance/alerts/2026-06-01-*`). That's BFOS Executor-domain; if it persists, the right action is a CC handoff inside BFOS, not Strategist diagnosis. Noted here for awareness only.
