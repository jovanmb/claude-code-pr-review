# refute-first-review: adversarial PR review for Claude Code

A Claude Code plugin (one skill, `pr-review`) that reviews a pull request the way a sceptical staff engineer would. It assumes the diff is wrong until the code proves otherwise, and then tries to prove its own findings wrong before reporting them.

It is read-only. It never approves, merges, pushes or posts unless you ask, and even then it shows you the exact comment first.

## Why

AI reviewers are good at producing plausible findings. The hard part is making them **right**: no invented requirements, no "this symbol doesn't exist" from a stale checkout, and no severity inflation. This skill is built around that:

- **Every finding needs a concrete failure scenario** (this input or state leads to this wrong output, crash or exposure) and **file:line evidence at the PR's real ref**. If it can't construct the trigger, it doesn't raise the finding.
- **Every finding goes through a refutation pass.** For money, security and auth findings, it spawns 2–3 subagents whose only job is to argue the finding is wrong.
- **Lenses are chosen per diff.** A migration-only PR gets the data lens, a CSS change gets the frontend lens, and a security fingerprint (raw SQL, user input, a secret) switches its lens on whatever the file type.

## In a real run

I planted a SQL injection in a five-file PR and gave it to a fresh subagent running this skill, telling it nothing about the bug. It found the injection, proved it by running the SQL, and then found **two real bugs I hadn't planted**: an acceptance criterion the change didn't meet, and two storage backends that disagreed on wildcards. Its five points of confusion about the skill became the clarifications in version 1.0.

[Read the full case study →](https://jovanmb.github.io/ai-engineering-notes/pr-review-case-study.html)

A second run targeted a retry change whose tests all passed. It found the planted double-post, then a request-hangs-forever bug I'd written without noticing, and exposed a gap in when the skill demands independent refuters. That became version 1.1. [Read it →](https://jovanmb.github.io/ai-engineering-notes/retry-case-study.html)

## Install

As a plugin, in Claude Code:

```
/plugin marketplace add jovanmb/claude-code-pr-review
/plugin install refute-first-review@jovanmb-skills
```

It adds about 140 tokens to every session (the skill's description) and about 3,500 each time a review runs.

Or copy `skills/pr-review/` into `~/.claude/skills/` (or `.claude/skills/` in a project).

## Use

```
review PR 123
review https://github.com/acme/api/pull/123
self-review my branch
```

It needs `git` access to the repository. For PR metadata and comment threads it uses whatever your host offers (for example `gh` for GitHub).

## How it works

| Phase | What happens |
|---|---|
| 0. Route and ingest | Reads the PR, **all** comment threads (paginated to the end) and the ticket. Takes the diff from git, not the host API, and checks symbols at the PR ref. |
| 1. Understand | Reads the ticket **before** the code, and turns each acceptance criterion into a checkable behaviour. |
| 2. Investigate | Traces entry point → handler → data → side effects. Finds the existing feature the PR imitates and compares against it. |
| 3. Interrogate | Profiles the diff, then runs the matching lenses out of nine: architecture, security (OWASP sweep), authN/authZ, validation and data integrity, loopholes and edge cases, frontend, infra and config, ticket fit, and maintainability (suggest-only). |
| 4. Refute | Tries to kill each finding. Reachability arguments lower *exploitability*, never the severity of a defect that survived. |
| 5. Conclude | Verdict first. Then AC fit, ranked findings, a lens coverage table, a verification plan and a QA handoff with pass/fail oracles. |

[`references/traps.md`](skills/pr-review/references/traps.md) is a catalogue of ways a change looks right and isn't (a 200 that isn't an approval, a 0-row UPDATE that still sends the email, reading only the first page of a capped API). The skill passes reviewers only the sections relevant to the diff.

## Limits

- It costs more tokens than a quick review, by design. Trivial diffs (a version bump, copy, an isolated null guard) get one lens and a short verdict.
- Its static SQL review is reading, not running. Dialect claims without precedent in the codebase are marked UNVERIFIED.
- It is a reviewer, not a replacement for one. Treat its verdict as a well-argued second opinion.

## Licence

[MIT](LICENSE) © 2026 Jovan Blanco
