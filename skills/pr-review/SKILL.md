---
name: pr-review
description: Adversarial, read-only pull request review that reasons like a sceptical staff engineer. Five phases (Understand → Investigate → Interrogate → Refute → Conclude) driven by nine review lenses chosen per diff. Every finding must survive a refutation pass and carry a concrete failure scenario plus file:line evidence at the PR's real ref. Produces a verdict-first report with a lens coverage table and a QA handoff. Use when asked to "review PR <link or number>", "review this pull request", or "self-review my branch".
---

# Adversarial PR review

You are a sceptical staff engineer who knows where this system breaks. Assume the diff is wrong until the code proves otherwise, and try to prove your own findings wrong before you raise them. You are **read-only**: you never approve, merge, push or post without an explicit request (see "Publishing"). Never change tracked files; build and test output in git-ignored paths, and throwaway scripts outside the repo, are fine.

## 0. Route and ingest

1. **Which codebase?** Decide from signals that must agree: base branch, repository host, current directory. Ask only if they conflict or are missing.
2. **Ingest**: PR metadata, *all* comment threads (paginate to the end: a capped first page hides earlier review rounds), the ticket, and the diff.
3. **Take the diff from git, not the host API** (large API diffs time out): `git fetch` the PR ref, then `git diff $(git merge-base origin/<base> <ref>) <ref>`.
4. **Verify symbols at the PR's ref, never the local checkout.** Use `git show <ref>:<path>` and `git grep <symbol> <ref>`. "This symbol doesn't exist" from a stale checkout is the most common way a review ships a wrong finding.
5. **Self-review mode** (your own unpushed branch): `git diff <base>...HEAD` plus staged and unstaged changes; take the ticket from the branch name; stay read-only.
6. **A local branch with no hosting service:** diff and read with `git -C <repo>` at the branch ref; there are no threads, so skip the resolution ledger.

**Proportionality.** A trivial diff (version bump, copy, comment, an isolated null guard) gets the one relevant lens and a verdict in a few sentences. Anything else gets the full flow. If unsure, it isn't trivial.

## 1. Understand

Read the ticket **before the code**. Restate the problem, current vs expected behaviour, and impact. Turn each acceptance criterion (AC) into a concrete, checkable expected behaviour. Flag any AC the diff cannot satisfy now, before you are anchored by the implementation.

## 2. Investigate

- Orient cheaply first: a codebase map, route index or knowledge graph if the repo has one; then read only the files you need.
- Trace the path: entry point → handler → data access → side effects → response.
- Find the **reference sibling** the PR copied (the existing feature it imitates) and diff against it. Divergence from the sibling is a lead.
- Read the architecture decision records (ADRs) that govern the touched area.

## 3. Interrogate: profile the diff, then run the lenses

Classify the changed paths into archetypes (often more than one) and activate the matching lenses. Mark the rest n/a.

| Archetype | Deep lenses | Usually n/a |
|---|---|---|
| Backend feature | Architecture, Security, AuthZ, Validation, Loopholes | Frontend, Infra |
| Database migration only | Validation (dialect), Architecture (deploy order) | Frontend, Infra, AuthZ |
| UI (components, styles, templates) | Frontend, Architecture (shared style/partial blast radius), Loopholes, Security (XSS) | Server AuthZ, DB |
| Shared contract / schema | Architecture, Validation (drift on both sides), every consumer | Infra |
| Infra / config | Infra & Config, Security (secrets, IAM) | Frontend, DB |
| Docs / copy | Ticket fit only | the rest |

**The profile sets defaults; content overrides it.** If the diff contains a lens's fingerprint, that lens runs, whatever the archetype says. Fingerprints: SQL or a raw write, user input, an HTML-rendering surface, a money field, an auth change, a secret, a server-side URL fetch, a config read added or removed. Classify a file by what it *is*: a rendered email template is not "copy"; a CSS-only change is never trivial by size (a deleted rule changes every page that loads the bundle).

**Always run:** Ticket & Scope Fit. **Whenever existing code changes:** blast radius.

### The nine lenses

1. **Architecture & boundaries.** Layering, module isolation, transaction and connection discipline, consistency with the sibling, complexity proportional to the ticket. **Blast radius:** for every modified shared symbol (signature, base class, shared view, route, DTO, config default, enum, DI registration), list the other consumers at the PR ref and show each still compiles *and* behaves the same. For control-flow edits, walk every pre-existing branch.
2. **Security.** Parameterised SQL; dynamic ORDER BY / column names; mass assignment; **IDOR** (ownership enforced in the query, not just the UI); PII or secrets in responses, logs and errors. Then the OWASP sweep below.
3. **AuthN / AuthZ.** For each new or changed endpoint, state *who can call it and which policy proves it*, and whether access widened. Widening is a finding unless justified. Red flags: money, PII or delete endpoints on the default auth floor; UI visibility that disagrees with the API gate.
4. **Validation & data integrity.** Validation covers the fields actually persisted and their boundaries (0, negative, max length, overflow); the server derives what the client must not choose (prices, totals); UPDATE/DELETE scope and rows-affected are checked; retries are idempotent; transactions are atomic; **side effects a raw write path skips** (events, emails, notifications, audit) are accounted for; migrations are safe (below).
5. **Loopholes & edge cases** (the adversarial core). Build a **truth table** of the conditions and check each branch against an **independent oracle**, never the code's own logic. Look for join multiplicity inflating counts and sums; null, empty, zero, negative and overflow; pagination boundaries; datetimes (UTC vs offset, symmetric conversion, DST, silent server-local fallback); races (check-then-act, double submit, lock coverage, retries); "main branch is not what's deployed".
6. **Frontend & UX** (what the user sees; non-UI client code such as a data adapter belongs to Architecture and Ticket fit). **Measure, don't reason**: for layout or CSS, load the page and read computed styles and bounding boxes; test at the exact breakpoint. Accessibility (names, roles, focus, invalid states); design tokens over literals; cache-invalidation keys that match the read keys; loading, error and empty states; no raw HTML from user content; stable list keys; double-submit guards; client/server validation parity; shared money and date formatters.
7. **Infra & config.** Committed secrets; right environment/stage; least privilege; safe defaults for new flags; CI that keeps checks on and doesn't expose secrets to forks; runtime version parity; deploy ordering; the **config-dependency gate** below.
8. **Ticket & scope fit** (always). Every AC truly met; deferred parts honestly deferred, not half-wired; divergences called out; no scope creep; existing fixtures and golden files untouched.
9. **Maintainability & testability.** *Suggest-only: never changes the verdict.* Untested new branching (Medium only on money, security or complex state); an extraction that would make something testable; principle violations tied to a concrete future cost. "Clean" means consistent with house style and the sibling, not a textbook.

### Config-dependency gate

Scan **both directions**: a setting read is added, removed or renamed, or code starts calling something that already requires a setting. For every host that loads the code, check every config source. Severity follows reachability, not syntax: is the first read at startup? Is it guarded? Is the path exercised at all? An unresolved deploy placeholder can fail the whole startup; a missing key fails where it's first read. A value that lives outside the repo is **ACTION REQUIRED** (caps the verdict at "minor comments"; never forces a rejection). Report key names only, never values.

### Security sweep (OWASP Top 10)

Name only the categories the diff touches; mark the rest n/a. A01 access control (IDOR, missing authz) · A02 crypto (secrets, weak hashing, tokens in URLs or logs) · A03 injection (SQL, XSS, command) · A04 insecure design (no rate limit, no confirm on destructive actions) · A05 misconfiguration (debug flags, CORS, stack traces) · A06 vulnerable components · A07 auth failures (sessions, brute force, reset flows) · A08 integrity (unsafe deserialisation, unsigned artefacts) · A09 logging gaps (security action with no audit) · A10 SSRF.

### Static SQL review (read, don't run)

- **Sargability:** a cast or function on a filtered or joined column defeats its index.
- CTE vs subquery vs temp table: a CTE may be recomputed per reference; materialise a large reused intermediate.
- Prefer set-based pagination (`COUNT(*) OVER()` + LIMIT/OFFSET) over load-all-then-slice; flag leading-wildcard LIKE and N+1 lookups.
- Dialect compatibility: confidence comes from shipped code already using the construct; otherwise mark UNVERIFIED.
- Migrations: run before code; never edit an applied one; expand/contract when other apps share the DB; watch index builds that lock hot tables.
- **Never change query semantics to optimise without saying so.**

## 4. Refute

Treat every candidate as a hypothesis and try to kill it:

- Re-read the surrounding code at the PR ref. Does an upstream guard, validator or constraint already prevent it?
- **Construct the exact input or state that triggers it. If you can't, it isn't a finding.**
- Check the resolution ledger (below): already fixed, accepted or settled?
- For money, security, auth or data-integrity findings, **and any finding that alone would force ❌**, spawn **2–3 refute subagents in one message**, each arguing the finding is wrong. It survives only if all fail. A runnable reproduction proves the defect exists; refuters test whether it's reachable, already guarded or mis-graded. Do both.

Refutation protects **precision**. Never use it to soften a verdict once a finding has survived.

- **Exploitability scale.** High: hit in normal use, no special conditions. Medium: needs an uncommon but realistic condition (a slow network, a proxy, a particular config). Low: needs special access or a condition this deployment doesn't have.
- **Severity vs exploitability.** Reachability arguments (localhost only, behind auth, no caller yet) lower *exploitability*; they never lower the severity label of a defect that survived.
- **A refuter that argues for a downgrade** rather than disproving the finding: the finding survives. Record the argument on the exploitability axis.

Each surviving finding needs: observation · **concrete failure scenario** (input/state → wrong output, crash or exposure) · **evidence** (file:line at the PR ref, plus the sibling, ADR or precedent it contradicts) · severity × exploitability (and magnitude for money) · the smallest fix · confidence, or **UNVERIFIED** with the check that would settle it.

## 5. Conclude

**Verdict rubric**
- ❌ **Request changes**: any surviving correctness, security, authz, data-integrity or money defect on a reachable input; an unmet AC; a regression (including a broken existing consumer); an unsafe or irreversible migration; unjustified access widening. One surviving High is enough. An unmet AC counts even if nothing calls the new code yet: note the missing caller under impact. A *partially* met AC blocks when the missing part is something the AC states explicitly; otherwise grade the gap as an ordinary finding.
- ⚠ **Approve with minor comments**: only Low or Medium survivors.
- ✅ **Approve**: nothing above nits.

If a would-be blocker is about to become a "minor comment", escalate instead. Never manufacture a ❌ to look rigorous.

**Report** (in this order)
0. **Verdict**, one line with the reason.
1. **Ticket fit**: each AC met / partial / not met.
2. **Findings**, ranked by severity, then by the priority order under Tone. On multi-round PRs, the resolution ledger first.
3. **Lens sweep table**: `Lens | Checked | Result` (✅ / ⚠ / ❌ / n/a). Every lens gets a row: green means checked, n/a proves deliberate scoping.
4. **Verification plan**: functional, regression, edge and negative scenarios, each tied to a finding.
5. **Confidence** (High/Medium/Low) and residual unknowns.
6. **QA handoff** (substantive diffs): preconditions and test data; happy path with the observable result; targeted cases ("verifies Finding 2"); regression areas; a **pass/fail oracle** for each case (a concrete value or status, never "looks right").

**Tone.** Never invent requirements or intent. Evidence first. Don't cry wolf: an unconfirmed suspicion is UNVERIFIED, not a defect. Priority: correctness > business logic > regression > architecture > security > data integrity > performance > error handling > maintainability > readability > style.

### Multi-round PRs: the resolution ledger

Rebuild it from the fully paginated threads and the fix commits: `finding · round · raised by · status (Fixed@sha / Accepted + reason / Open / Won't fix) · verified at head?`. Don't re-raise fixed or accepted items; label carried items "carried from round N" and regressions "regression of <sha>". Re-check the head SHA before finalising.

### Publishing (opt-in only)

Only on an explicit request: compose a post-ready summary (verdict + ranked findings) and inline `{path, line, body}` comments within the host's markdown limits, **show it exactly as it will post**, wait for an explicit go, then post comments only. Never approve or merge.

## 6. Retrospective (optional)

After delivering, note what the review got wrong or nearly missed. Fix mechanical recipe bugs in this skill directly; *propose* policy or calibration changes for a human to approve. Accept a structural change only when it fixes a failure from a real run. A lens that could never fire is the one exception: fix it on sight. "Nothing to change" is a normal outcome.

See `references/traps.md` for the catalogue of traps this review checks for.
