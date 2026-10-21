# Changelog

## 1.4.0 — 21/10/2026

From a third real run, the first through the packaged agent, on a design problem (see the [case study](https://jovanmb.github.io/ai-engineering-notes/design-case-study.html)):

- A documented house rule written as a description ("business rules live in `core`"), including README architecture sections and module header comments, is graded on its concrete cost and never dropped to suggest-only for its phrasing.
- The refuter downgrade rule now explicitly covers design and architecture findings.
- Never write a refuter's conclusion before its report arrives.
- New step: mutation-check the PR's own tests in a scratch copy.
- README: refutation scope matches the skill (data integrity and any blocking finding).

## 1.3.0 — 20/10/2026

- **Now an agent.** `agents/skeptical-reviewer.md` runs the review in its own context window, preloads the `pr-review` skill, and can spawn refute subagents. The plugin is renamed `skeptical-reviewer`; installs under either earlier name migrate automatically.
- **Codebase-aware.** Before judging, the review reads the repository's written rules (instruction files, contributing guide, ADRs), its enforced rules (linter, formatter and analyser configs) and its unwritten ones (2–3 neighbouring files of each kind). The report opens with **Standards applied**.
- **Design principles lens.** Lens 9 now covers OOP and SOLID, encapsulation, cohesion and coupling, DRY versus the wrong abstraction, YAGNI and error handling, judged in the codebase's own paradigm. It runs on every code change. Breaking a documented house standard is graded like any other finding; a generic principle stays suggest-only unless its concrete cost can be named.

## 1.2.0 — 20/10/2026

- The plugin is now called `refute-first-review` (it was `pr-review`, which clashed with several similar names). The skill inside is still `pr-review`. Existing installs migrate automatically through the marketplace's `renames` entry.
- Added `.claude-plugin/plugin.json`, the standard plugin manifest, so the repo is both a plugin and a one-plugin marketplace.

## 1.1.0 — 20/10/2026

From a second real run, on a retry change (see the [case study](https://jovanmb.github.io/ai-engineering-notes/retry-case-study.html)):

- Refute subagents now also cover data-integrity findings and any finding that alone forces ❌. A reproduction and refuters are both required.
- A defined exploitability scale (High / Medium / Low).
- A partially met acceptance criterion blocks when the missing part is stated explicitly.
- Findings are ranked by severity, then by the stated priority order.
- "Read-only" made precise: never change tracked files; ignored build output and scratch scripts outside the repo are fine.
- New trap: a timeout cleared when the headers arrive leaves the body read unbounded.

## 1.0.0 — 20/10/2026

First public release.

- Five phases, nine lenses chosen per diff, OWASP sweep, config-dependency gate and static SQL review.
- Refutation with 2–3 subagents for money, security and auth findings.
- Clarified after a real run (see the case study): reachability lowers exploitability, never severity; a refuter arguing for a downgrade doesn't kill a finding; an unmet AC counts even with no caller yet; local branches without a hosting service are supported; non-UI client code belongs to the Architecture and Ticket-fit lenses.
