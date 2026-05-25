# Checkpoint: H-PUB-002 — Plant the remaining deep-dive docs

**Date:** 2026-05-24
**Status:** COMPLETE — all conditions matched; pre-authorization applied.
**Shipped commit:** see commit log entry for `docs: add remaining deep-dives …`.

## Pre-edit Sanity Check

1. `git remote -v` → `steampunkfarms/ai-dev-governance`; branch `main`; tree
   clean after pulling `LICENSE` from remote (operator added it via GitHub
   web UI between H-PUB-001 and H-PUB-002, commit `32f944b`). MATCHES.
2. Five new docs were delivered at `/Users/ericktronboll/Downloads/files 3/`
   (operator's drop location), copied into `docs/` as the prerequisite step.
   Not a Sanity Delta — same operator-prerequisite pattern as H-PUB-001.
3. `docs/sanity-check.md` and `docs/bounded-deviation.md` present. MATCHES.
4. README "📚 Deep dives" table had Sanity Check + Bounded Deviation rows
   ✅ and linked; the next five rows were 🚧 Planned. MATCHES.

## What shipped (one commit)

- `docs/two-roles.md`
- `docs/context-cascade.md`
- `docs/checkpoints-handoffs.md`
- `docs/roadmap-system.md`
- `docs/qa-gate.md`
- `README.md` — five Deep-dives table rows flipped to ✅ with links;
  license badge wrapped in `<a href="LICENSE">` (operator's separate request).

## Link verification

All real markdown links resolve. Sibling-doc references in the five new docs
(`two-roles.md` ↔ `checkpoints-handoffs.md` ↔ `context-cascade.md` ↔
`qa-gate.md` ↔ `roadmap-system.md` ↔ `sanity-check.md` ↔
`bounded-deviation.md`) all hit existing files. README anchor links from the
new docs match GitHub-generated heading slugs:

- `#-works-with-any-strategist--executor` ↔ `## 🧩 Works with any Strategist + Executor`
- `#layered-protocol--governance-sync` ↔ `### Layered protocol — Governance Sync`
- `#foundation-4--the-file-handling-protocol` ↔ `### Foundation 4 — The file-handling protocol`
- `#layered-protocol--the-roadmap-system` ↔ `### Layered protocol — The roadmap system`
- `#layered-protocol--post-execution-qa-tier-2` ↔ `### Layered protocol — Post-execution QA (Tier 2+)`

Em-dash in headings produces a double-`-` segment in the slug (`protocol--governance`),
which is correct GitHub behavior.

## Bundled non-handoff work (operator's separate ask)

- README license badge wrapped in `<a href="LICENSE">…</a>` so the CC BY 4.0
  shield is now clickable. Inline HTML matches the existing pattern of every
  `<p>` / `<img>` / `<h1>` tag in the README header; the markdownlint
  MD033 warning fires on all of them and is intentionally tolerated.

## Acceptance criteria — final

- [x] `docs/` contains `two-roles.md`, `context-cascade.md`,
  `checkpoints-handoffs.md`, `roadmap-system.md`, `qa-gate.md`.
- [x] All seven Deep-dives rows are now ✅ and linked.
- [x] No dangling internal links.
- [x] Clean commit pushed to `main`; working tree clean.
- [x] Checkpoint written (this file).

## Operator Action Block

No required action. Optional: glance at the five rendered pages on GitHub.

## Files modified this session

- `README.md` — Deep-dives table rows 3–7 + license badge anchor.
- `docs/two-roles.md`, `docs/context-cascade.md`, `docs/checkpoints-handoffs.md`,
  `docs/roadmap-system.md`, `docs/qa-gate.md` — new.
- `docs/checkpoints/2026-05-24-remaining-deep-dives.md` — this file.
