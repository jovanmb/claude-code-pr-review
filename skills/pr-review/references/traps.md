# Trap catalogue

Recurring ways a change looks right and isn't. Used by `pr-review`, and useful to any reviewer, human or agent. Each entry earned its place by being missed at least once. Pass reviewers **only the sections relevant to the diff**, not the whole file.

## Money and external APIs
- **Envelope vs result.** HTTP 200 / "call succeeded" is not "payment approved": read the provider's result code.
- **Amount provenance.** The client can submit a price or total the server should compute.
- **Time-based dedupe keys.** A key built from the current time lets a retry charge or pay out twice.
- **Raw provider responses.** Never log or persist a payment provider's raw response.

## Writes and concurrency
- **Unchecked rows-affected.** A 0-row UPDATE still sends the confirmation email.
- **Inconsistent guards.** The fix lands on one write path; three sibling paths keep the bug. Grep the pattern repo-wide.
- **Lock release windows.** A `finally` that releases a lock after an external charge succeeded reopens a double-charge window.
- **Bypassed side effects.** A direct SQL write skips the events, notifications and audit the normal path fires. List every one.
- **Check-then-act races and double submits.**

## Auth and the web surface
- **State-changing GET**, or POST without CSRF protection or a role check.
- **IDOR.** The ownership filter exists only in the UI, not in the query.
- **Default auth floor.** A new endpoint silently inherits the weakest policy.
- **Pre-existing vs introduced.** A guard missing in the base branch too is a known gap, not this PR's regression: say which.

## Config and deploy
- **New required setting** present on some hosts and missing on one: that host fails on first read.
- **Removed or renamed key** fails exactly like a key never added.
- **Unresolved deploy placeholders** can take the whole service down at startup.
- **Per-environment flag hard-coded** so it can't be enabled in one environment.
- **Not every literal needs to be configurable.** Don't false-block constants.
- **Main branch is not production.** Check deploy state before claiming "the fix was live".

## SQL and data
- **Non-sargable predicates:** `CAST(id AS text) = x` defeats the index.
- **CTE recomputation**, load-all-then-paginate in app code, leading-wildcard LIKE, N+1 queries.
- **Join multiplicity:** a 1:many join inflates SUM and COUNT.
- **"This filter drops rows"** claims: an OR join can emit another row that still matches. Check the whole row set.
- **Dialect gaps:** constructs the target engine doesn't support; identifier case folding breaking "if not exists" guards.
- **Index builds** that lock hot tables; covering indexes so wide they slow every write.
- **Migrations:** before code; immutable once applied; expand/contract when another app shares the DB; reference data as a migration, not a local seed.
- **Check column names against the schema** before writing SQL. Don't guess from a similar table.

## Timeouts
- **A timer cleared when the headers arrive** leaves the body read unbounded: a stalled response hangs forever with no error and no retry. Keep the deadline armed until the body is read.

## Pagination and boundaries
- **Reading only the first page** of a capped API. (A list of 151 read as 100 picks the wrong "latest" item and raises no error.)
- Page 0, the last page, zero rows.
- Null, empty, negative, overflow.

## Dates and times
- Naive vs offset timestamps; converting one side of a comparison but not the other; DST; a timezone lookup that silently falls back to server-local time (common on cloud hosts set to UTC).

## Transactions and architecture
- Two connections inside one ambient transaction escalate or fail; reuse one connection for the main write and the audit write.
- A modified shared symbol: list every consumer at the PR ref.

## Frontend
- Cache-invalidation keys that don't match the read keys.
- Missing loading, error or empty states.
- Raw HTML from user input.
- Array-index keys on reorderable lists.
- Ad-hoc `toLocaleString` instead of the shared money/date formatters.
- Client and server validation disagree.
- **Regex anchors in multiline mode.** `$` matches at every line end, so a lazy capture stops at the first line.

## Tests
- Weak assertions ("not contains", "is success") prove almost nothing. Assert exact values.
- Cover every branch, including 401 and 403.
- **A failing test is a disagreement between two intents.** Decide which is right before changing either.

## Completeness of your own change
- Sweep your own diff for **other instances of the behaviour** you changed (a list view computing the same fee as the form you fixed).
- Removing a conditional makes the paths that never had it invisible to grep. Enumerate a coverage grid (type × surface × rendering) instead.
- "Is it correct?" and "is it complete?" are separate refutation passes.
- A correct function doesn't prove every call site uses it.
- When porting or cherry-picking, re-check that comments carried across are still true on the target branch.

## Tooling
- Verify against the PR ref, not a stale local checkout.
- Shell path rewriting (e.g. Git Bash on Windows) mangles `/`-prefixed arguments.
- **Inline shell heredocs eat escapes:** a `\b` can land in a source file as a literal backspace. Write edit scripts to files, then look at the bytes (`cat -A`) when output is odd.
- File-encoding guards (BOM, replacement characters) belong in the build.
