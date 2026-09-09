# Pass checklists

Each pass is run on its own, against the same contract, and produces candidate findings that go
through verification before grading. A checklist item is a question to answer with a `file:line`,
not a box to tick.

## 1. Bugs — does the code do what the contract says on every path the diff touches?

- Null, empty, zero, `-0`, `NaN`, `undefined`-vs-`null` — at each new branch and each new field read.
- Error branches: what happens when the call throws, times out, or returns `{data: null, error: null}`
  (PostgREST under pool saturation — the transient-vs-absent rule, `distinguishing-transient-from-absent`).
- Ordering and concurrency: which lock (`createKeyedLock`) covers the write; is a long-running sweep
  held inside it; TOCTOU across any gate that reads then writes.
- A value that is read once and used twice with different freshness (a price, a status, a count).
- Every new numeric comparison: units (pips vs price vs points), sign for BUY vs SELL, strict vs
  non-strict at the boundary the spec names.
- Retries: is the retry idempotent; does it re-read before acting; does it terminate.
- Fallbacks: does the fallback path receive the same validation the primary path did.
- PostgREST shapes: a to-one embed is an **object**, not an array; a list read past 1,000 rows is
  truncated unless paginated with a stable `.order()`; `data ?? []` hides a failed read.
- Time: DST-aware market hours (`isForexMarketOpen`), local-day bucketing, `Date` in UTC vs local.

## 2. Security — what can a caller reach now that they could not before?

- Every new read and write: is it scoped by `user_id`, by `metaapi_account_id` where financial, and
  by `.is('deleted_at', null)` on the soft-delete allowlist tables.
- New API routes: CSRF on frontend→backend; `X-Internal-Auth` per-instance secret on
  frontend→user-service; `withAdminAuth` on admin; bearer-auth admin routes listed in
  `API_TOKEN_PATHS` or the proxy 307s them to `/login`.
- New `public` tables: `ENABLE ROW LEVEL SECURITY`, policies or a stated service-role-only
  lockdown; every new FK has an index.
- `SECURITY DEFINER` functions: `SET search_path = ''`, `REVOKE ... FROM anon` where the design says.
- Secrets: nothing in the diff, in fixtures, in log lines, in error messages; no PII in logs.
- Kill-switch OFF path: **no new DB read, no new network call** — the disabled path is the legacy
  path, byte for byte.
- Anything that widens what the anon key can reach through PostgREST.

## 3. Compliance — the diff against the contract, not against the brief

- Spec §3: each `INV-n`, in a table, with the line that satisfies it and the suite that pins it.
  "Satisfied at the pure function, unpinned at the call sites" is a finding.
- Plan Global Constraints; each task's verification line actually run; plan amendments recorded in
  the same commit as the departure (`git log -S"AMENDMENT"`).
- Kill-switch: default and parsing exactly as spec §5 (`false`/`0`, case-insensitive, trimmed);
  env-var row present in `docs/kill-switches.md` + `CLAUDE.md` index (user-service) or the root
  `CLAUDE.md` (frontend).
- §10 Out of scope: nothing in the diff belongs there.
- §11 Flagged concerns: none resolved by the implementer.
- Surfacing (§6): trace event types, PostHog constants from `lib/analytics/events.ts`, copy.

## 4. Regression — what else did this change reach?

- Callers of every changed signature: `grep -rn '<name>(' src tests` — updated, or a stale call
  behind a broad `catch`.
- Old names extinct: the renamed function, the removed flag, the retired status value — in source,
  tests, mocks, fixtures, comments, docs, env examples.
- Mirrored pairs changed together (wiki `architecture/parity-contracts.md`): trade-preview ↔
  order-allocation, zone-resolver, FAILURE_PREFIXES, traders-in-profit helper, settings-validation
  (frontend ↔ admin — no pin exists; diff by hand).
- Settings keys: grep **both** casings (`global_signal_behavior` and `globalSignalBehavior`).
- Tests that changed: an assertion that flipped is either in the spec §1 PIN list or a finding.
- Mocks: `jest.mock()` no-ops under native ESM in user-service; `jest.fn()` inside
  `unstable_mockModule` factories is wiped by `resetMocks`; a Pattern-4 inline test exercises a
  copy, not the production wiring.
- `CLAUDE.md`, `docs/kill-switches.md`, `.env.example`: does the diff make a stated
  fact false?
- The wiki's own record: `grep -rn '^> \*\*Do not' ttmt-wiki/wiki/services/<svc>/ ttmt-wiki/wiki/bugs/`
  for each touched service — does the diff repeat something that already broke here?

## 5. Data and migrations — only when the diff carries SQL or a backfill

- Forward-only: no edit to an applied migration; a new file, later timestamp.
- Every new `public` table: RLS enabled. Every new FK: index. Soft-deletable: `deleted_at`.
- `CREATE OR REPLACE FUNCTION`: owner/ACL preserved; the body diffed against the previous version,
  not just read.
- Trigger/TypeScript parity: the predicate in SQL and the predicate in the service agree term by term.
- Backfill: bounded (time window, row cap), guarded on expected state so it is a no-op on drift,
  snapshot taken first, rollback SQL under `docs/plans/rollback/`.
- Ordering guard: a later migration that depends on an earlier one asserts it (`pg_get_functiondef`
  contains the expected clause) rather than assuming.
- Dry-run: `BEGIN; \i file; <asserts>; ROLLBACK;` against local — and confirm nothing persisted.
  Never `migration up`, never `db push`, never against anything but local. If the operator has not
  sanctioned even a local dry-run, read the SQL against the live schema and say the dry-run was not
  performed.
- Data migrations already applied to production and only now tracked: review them anyway; record
  what they did and whether the guard would have protected drifted rows.

## 6. Tests — did the proof actually run?

- Changed file in user-service: is a suite for it **in the `test:gate` list**? `grep -o
  'tests/unit/[a-z0-9-]*\.test\.ts' package.json | sort -u`.
- The suite loaded: the `PASS tests/…` line names the file; the test count matches the file.
- Right repo: `cd <repo> &&` in the same command as the gate; suite/file counts match the repo.
- Full output kept: `> /path/run.log 2>&1`, then grep; never `| tail` a run whose failure you may
  need to name.
- Frontend worktrees inside the repo (`.wt/`, `.worktrees/`, `worktrees/`) collect **zero** tests —
  `npx jest --listTests <dir> | wc -l` before trusting any run from one.
- A new load-bearing test: mutation-check it once (break the implementation, watch it go red).
- Vacuous green: assertions after a `try` whose `catch` swallows a `TypeError` from a missing mock
  method pass while asserting nothing.
- Frontend full jest can omit a failing suite from its summary — run the suites you care about with
  `--runTestsByPath` as well.
