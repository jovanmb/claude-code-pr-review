# Changelog

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
