---
name: writing-the-failing-test-first
description: Makes a bug fix provable — write the test, watch it fail for the right reason, add it to the test:gate list, and fix the code without weakening the check. Use when fixing a bug, reproducing a defect, responding to a production incident or regression report, adding a test to ttmt-user-service, or when a green suite is being treated as proof.
---

# Writing the failing test first

For a bug fix the order is fixed: **write the test, watch it fail, then fix the code.**

> "A test that existed before the fix, and that the agent couldn't rewrite, is proof the bug is gone."

A test written after the fix, by the session that wrote the fix, encodes the same misunderstanding
that produced the bug. It passes because both halves agree.

## What TTMT actually does, and what this skill asks for

Measured 2026-08-30 over the last 200 commits on `ttmt-user-service@develop`: **131 change tests
and source together; 5 are test-only.** The playbook's separate-commit form is not this repo's practice, and
this skill does not pretend otherwise.

What is non-negotiable is the *sequence*, not the commit boundary:

0. **Read the bug page first** — `/wiki-query`, or `ttmt-wiki/wiki/bugs/<slug>.md` directly. Its
   `## Symptom` is what your test must reproduce and its `## Root Cause` is what must make it fail;
   without them you are writing a test against your *theory* of the bug, and the theory is what the
   fix will then agree with. Skip this only when the bug has no page yet — in which case the page is
   the change's closing task anyway.
1. Write the test and **run it before the fix exists.** Watch it fail, and read the failure — it must
   fail with the bug's own symptom, not a `TypeError` or a missing import. A test that fails for the
   wrong reason will pass for the wrong reason.
2. Fix the code.
3. Re-run. If the fix commit also **changes an existing assertion**, say so in the commit message and
   in the PR. That is the case the separate commit exists to make visible, and stating it costs a line.

## What actually runs your test, per repo

Adding a test is half the work. What runs it decides whether it guards anything, and the three repos
differ. **This table is a snapshot (2026-08-30) — before relying on it, look:** `ls .github/workflows`
and read the workflow, check `.husky/`, and for user-service read the `test:gate` list in
`package.json`. CI wiring is exactly the kind of fact that changes underneath a skill.

| Repo | What runs on a PR | What that means for you |
|---|---|---|
| `ttmt-user-service` | `.github/workflows/gate.yml` → `npm run test:gate`, an **explicit file list** in `package.json` | A new suite runs in CI **only if you add it to that list** |
| `ttmt-frontend` | `.github/workflows/ci.yml` → two contract guards, ESLint, `tsc --noEmit`, full Jest; `.husky/pre-commit` + `pre-push` run one guard | The full suite runs; check your guard is one of the wired ones |
| `ttmt-admin` | **nothing — the repo has no `.github/` at all** | 157 test files, and no automation runs any of them. Green is whatever the last person ran locally |

As of the same survey, `ttmt-frontend` defined `check:unchecked-supabase-writes` — a guard written
after the 2026-08-19 Postgres triage — with **no hook and no workflow invoking it**. A guard that
nothing calls is a comment; grep for the script name before counting it as coverage.

## Four ways a user-service test lies about passing

All four are verified findings from `ttmt-user-service` — `ttmt-wiki/wiki/operations/user-service-test-runner.md`.

**1. The gate does not run it.** As above, `test:gate` is an explicit list, not a pattern.

> **Do not assume the file you are about to change is in `test:gate`.** Reason: gate membership is an
> explicit list, not a pattern, so the highest-traffic file in the service — `breakeven-listener.ts`,
> which fires per broker tick — sat outside it indefinitely, and 31 breakeven suites ran zero times in
> CI. Instead: check the list before relying on it, and add the suites for the file you are touching.

**2. It passes vacuously behind a broad `catch`.** A mock missing a method the implementation calls
raises a `TypeError` *inside* the try; the catch-all absorbs it; the function returns its failure
path; assertions written against a path that never ran still pass. One suite here went green while
asserting nothing until a `.select` was added to a mock chain — then it became 28 real tests.
**Mutation-check it:** break the implementation once and confirm the test actually goes red.

**3. `jest.mock()` silently no-ops.** Jest runs in native ESM (`useESM: true`), so a plain
`jest.mock()` does nothing and the suite runs against the real module while believing it is mocked.
Flagged repo-wide and **not fixed** — check before trusting a mock.

**4. The runner was wrong.** Never `npx jest`. Without `--experimental-vm-modules` any
SDK-importing suite fails to load with `Cannot use 'import.meta' outside a module` — a wrong-runner
artifact, not a real failure, and it produces ~250 phantom failures. Use `npm run test`, or:

```bash
node --experimental-vm-modules node_modules/jest/bin/jest.js \
  --runTestsByPath tests/unit/<name>.test.ts --runInBand
```

## When the test encodes the old, wrong behaviour

This is real here — specs routinely note that legacy behaviour is **PINNED** by an existing
assertion that flips with the feature. Take it deliberately:

- The spec's §1 says which assertions flip; the plan schedules that as its own task.
- Where legacy code is being extracted, retarget the mirror test at the real code **before** changing
  behaviour, in its own commit. A test that mirrors an implementation passes whatever you do next.
- Say it in the PR description, so review sees a decision rather than finding it in a diff.

## Boundaries

> **Do not edit a test to make a fix pass.** Reason: the test is the only independent statement of
> what correct means, so an agent that can rewrite it can make any fix succeed — including one that
> does not fix the bug. Instead: fix the code; or change the assertion first, alone, with the reason
> stated.

> **Do not skip, `.only`, or quarantine a failing test to get to green.** Reason: a disabled test
> reports as passing forever, so the regression it guarded returns silently. Instead: fix it, or open
> the bug and leave it red.

> **Do not remove a suite from `test:gate`.** Reason: the gate list is the repo's only pinned
> regression set, and shrinking it to get a PR green disarms the check for every future change, not
> just this one. Instead: fix the suite, and if it is genuinely obsolete say so in the PR.

> **Do not close a bug on a passing test alone.** Reason: the test proves the reproduction is fixed,
> not that the reported symptom is gone — the two differ whenever the reproduction was a guess.
> Instead: confirm the original symptom at its source before marking the wiki page `resolution: fixed`.

## After the fix

`authoring-incident-evals` turns a production incident into a regression case in `evals/` and links it
from the wiki bug page via `eval_refs`. The unit test guards the code; the eval guards the behaviour
against a model or prompt change.
