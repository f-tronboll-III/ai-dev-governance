# The Roadmap System

> "What's left to build" lives in several places that will never stay in sync. Stop fighting that. Separate the genres on purpose, let the surfaces drift, and keep one index that catches the drift on audit.

**Part of:** [AI Dev Governance](../README.md) · **Prerequisite reading:** [The Two Roles](two-roles.md)

---

## The wall

A real codebase grows multiple "to build" surfaces: a product roadmap (decisions, pricing, versions), an execution roadmap per repo (what ships next), and often a dashboard. Two failures follow. People either burn effort trying to keep every surface in lockstep — and lose — or they lose track of what's actually true. And they quietly conflate two different *kinds* of list, product decisions and execution steps, into one blob nobody can maintain.

## The rule

1. **Separate the genres.** A product roadmap is not an execution roadmap. Never merge them.
2. **Let surfaces drift.** Mismatches between surfaces are expected, not bugs.
3. **Keep one index.** A single entry point catches drift *on audit* instead of pretending everything is aligned in real time.

## How it works

**Two genres, never merged:**

| Genre | Lives in | Writer | Purpose |
| :---- | :---- | :---- | :---- |
| **Product roadmap** | The governance repo | Strategist | Decisions, pricing, version planning, deferred questions |
| **Execution roadmap** | Each repo's `docs/roadmap.md` | Executor | What's left to build, handoff IDs, completion timestamps |

When a handoff bundles product-shape items (decisions, renames) with execution-shape items (file paths, commit boundaries), the Executor **splits them** — product items to the governance repo, execution items to the repo's roadmap.

**One index, one entry point.** A single hand-maintained index catalogs every product and execution roadmap (with links) plus a staleness table. It's the first thing to read before sprint planning or cross-repo moves, and the place drift gets caught.

**Dialects by work-shape.** Different groups of repos evolve different execution-tracking conventions because their work is shaped differently — centralized rollup, per-project, umbrella constellation. These are deliberately *not* normalized; forcing one convention across differently-shaped work is its own anti-pattern. (Full treatment in the [architecture reference](architecture.md#layered-protocol--the-roadmap-system).)

### A third surface: the dashboard

Many teams eventually grow a *rendered* roadmap surface — a database-backed table shown as a dashboard widget, often for an audience that won't read markdown (a stakeholder, a client view, an executive at-a-glance page, an in-product roadmap). This is fine. It's also the place the model most commonly breaks if you're not deliberate about it.

The rule: **the dashboard is a curated, opinionated projection — not the source of truth, and not auto-synced from the markdown roadmaps.**

- **Opinionated.** The dashboard doesn't show everything in the roadmaps — it shows the subset its audience needs. "Refactor the date helper" doesn't belong on a customer-facing roadmap; "shipping multi-tenant in Q3" might.
- **Manually curated.** Items land on the dashboard by an explicit decision, not by a sync script trying to map every markdown bullet onto a row. The mapping is lossy by design.
- **Not the source of truth.** When the dashboard and the markdown disagree, the markdown wins for *what's actually shipping*; the dashboard wins for *what the audience was told.* Closing that gap is a curation task, not a sync task.
- **Audit, don't reconcile.** The roadmap index includes the dashboard in its staleness table. "Dashboard hasn't been touched in three weeks" is a curation backlog, not a sync bug.

The failure mode this prevents: every team that has tried to auto-sync a markdown roadmap into a database-backed view has ended up with a script that quietly lies — it gets the easy cases right and silently miscategorizes the rest, and the audience the dashboard exists *for* ends up trusting wrong information. Curated drift, surfaced on audit, is the lesser evil.

## Worked example

A feature ships to one repo's execution roadmap and is marked done; its product-roadmap entry closes too. But the dashboard still lists it as "planned."

Under a sync-everything regime, you'd have a script fighting to reconcile three surfaces continuously — and quietly getting it wrong. Here, the periodic **index audit** simply flags *"dashboard entry stale for feature X,"* and someone fixes it in one move. The drift was allowed; the index made it visible. That's the whole trade: cheap, honest mismatches instead of expensive, brittle synchronization.

## Anti-patterns

| Anti-pattern | Why it bites | Do instead |
| :---- | :---- | :---- |
| One mega-roadmap | Product and execution items tangle | Split by genre |
| Auto-sync scripts across surfaces | They fight reality and quietly lie | Reconcile via the index on audit |
| Conflating product & execution in a handoff | The Executor can't tell decisions from tasks | Split: product → governance repo, execution → repo roadmap |
| No index | Drift goes invisible until it bites | Maintain one entry-point index |
| Forcing one dialect on every group | Different work shapes need different tracking | Let each group's dialect fit its shape |

## Adapt it to your setup

- **Solo / one repo:** you may need only one execution roadmap plus your `AGENT.md` changelog. Don't build the index until you have two or more surfaces to reconcile.
- **Add the index** the moment a second roadmap (or a dashboard) appears.

## Related

- [Roadmap system — full overview](architecture.md#layered-protocol--the-roadmap-system)
- [Checkpoints & Handoffs](checkpoints-handoffs.md) — execution items originate in handoffs
- Back to the [main playbook](../README.md)
