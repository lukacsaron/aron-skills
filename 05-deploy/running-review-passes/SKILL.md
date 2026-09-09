---
name: running-review-passes
description: Reviews a TTMT change — a branch, a PR, a merged head, or a window of recent commits across repos — against its spec, plan, prior review and the repo's own record of what broke, and hands a code owner a verdict with proven findings. Use when reviewing a pull request or a diff, when asked for a code review, a second opinion, a regression audit or an audit of what changed since a date, when verifying that a prior review's must-fix list was closed, when writing a final-review document, and after finishing an implementation a human will merge.
---

# Running review passes

Once agents write most of the diff, review stops being "read every line" and becomes **proven
findings, against a stated contract, by someone who did not write the code, feeding a human
decision.** Each clause is load-bearing, and each is where a review goes wrong.

Three measurements set the bar. Anthropic's internal reviewers went from 16% to 54% substantive
comments when agents had to write a proof that each finding was valid, and under 1% of posted
findings were marked incorrect once a verification step filtered candidates. On this wiki's first
consolidation pass, 14 of 15 reviewers approved fabricated provenance because the brief they were
given had sanctioned it. Proof, contract, independence.

References, one level deep: `references/pass-checklists.md` (what each pass asks),
`references/report-template.md` (what the artifact is), `references/subagent-brief.md` (how to fan
out and verify).

## 0. Scope — say exactly what is under review before reading a line

| Shape | Subject | What you produce |
|---|---|---|
| **Single change** | one branch or PR, `base..head` | one report with a verdict |
| **Merged head** | several changes combined on `develop`, each reviewed alone | an *interaction* review: does the combination behave differently from the parts? |
| **Window audit** | every commit since a date, across repos | a window table, then per-change-set reviews in tier order |

Fix the diff you are reading. When the branch base is a merge commit, `git diff base~1..head` pulls
in the other parent's whole lineage; review per commit with `git show <sha>`, or diff from
`git merge-base develop <head>`, and match the files you cite against `--stat`.

For a window audit, inventory first, review second:

```bash
git -C <repo> log develop --since='<date>' --date=iso-local --format='%h %ad %s' > <scratch>/log-<repo>.txt
git -C <repo> diff --stat $(git -C <repo> rev-list -n1 --before='<date>' develop)..develop -- . ':!docs'
git -C <repo> diff --name-only <base>..develop | grep -E 'migrations/.*\.sql'
```

Write the whole log to a file, count it, and build the table from the file. Partition commits into
change-sets by branch or topic, and **tier each by consequence** with the change cycle's two
questions: can it move money or orders, and can it be taken back in one deploy? Tier 3 is anything
with a migration, a fleet action, a deletion or a backfill. A window audit is a **budget**
decision: Tier 3, and any Tier 2 without a prior review, get the full passes; a reviewed Tier 2 gets
a closure audit (§7) and a seam check; Tier 0–1 gets a spot check. The table says which mode each
change-set got.

> **Do not scope from a truncated log.** Reason: `| head -80` on a 113-commit window hid the entire
> teardown programme, the coolify webhook and a 3,200-line billing fix on the first pass of the
> 2026-09-01 audit; absence from a page is not absence from the window. Instead: write the full log
> to a file, count it, and build the table from the file.

## 1. The contract — what the diff is judged against

In this order, all that exist: the spec's §3 invariants and §5 kill-switch clause; the plan's
Global Constraints and per-task verification lines; **the prior review's must-fix list**; the repo's
`CLAUDE.md`; and the wiki's record of what already broke in the touched area:

```bash
grep -rn '^> \*\*Do not' ttmt-wiki/wiki/services/<svc>/ ttmt-wiki/wiki/bugs/
```

**The contract may live in a sibling repo.** A change to admin's alert predicates was planned in
`ttmt-user-service/docs/plans/`; a landing-page consent gate was specified in `ttmt-frontend/docs/`.
Before concluding there is no spec or plan, search every repo's `docs/` for the subject and the
commit keywords, and `git log --all --grep=<keyword>` for a `docs(...)` commit:

```bash
grep -rl '<subject keyword>' */docs */docs/superpowers --include='*.md' 2>/dev/null
```

When none exists — legitimate for Tier 0–1 — the contract is `CLAUDE.md`, the counter-patterns and
the commit's own stated claim, and the report's header says so.

> **Do not review a change against the brief it was given.** Reason: a brief can itself be wrong, and
> a reviewer checking compliance with it cannot catch a bad brief — the 14-of-15 failure above.
> Instead: judge the diff against the spec, the plan, the prior review and the repo's rules; where the
> brief and the spec disagree, that disagreement is the first finding.

## 2. The ground — confirm the tree, the base and the gates

Before the first pass: which repo, which base, which head; whether the tree you read is the tree
that ships (`git merge-base --is-ancestor <sha> origin/main` for any "deployed" claim — on main is
not the same as running, and a stale `origin/main` ref is a stale answer); and what CI actually runs
here. **Open the config; do not recall it** — the wiring changes underneath every snapshot:

```bash
ls <repo>/.github/workflows; cat <repo>/.husky/pre-commit <repo>/.husky/pre-push
grep -o 'tests/unit/[a-z0-9-]*\.test\.ts' ttmt-user-service/package.json | sort -u | wc -l
grep -rn 'check:' <repo>/.github/workflows <repo>/.husky <repo>/package.json
```

Checked 2026-09-01: user-service runs `test:gate` — an explicit list of 48 suites plus `tsc`;
frontend CI runs three guard scripts, ESLint, `tsc`, full Jest and a DB-integration job, while
husky runs one guard and `check:unchecked-supabase-writes` is invoked by nothing; **admin has no
`.github/` at all**, so review is its only gate and the verdict says so.

Then run the gates yourself, in the right repo, with the output kept:

```bash
cd <repo> && npm run test:gate > <scratch>/gate-<repo>.log 2>&1; grep -E '^(PASS|FAIL)|Tests:' <scratch>/gate-<repo>.log
```

A green gate is evidence only if the changed files are in it. `writing-the-failing-test-first`
carries the four ways a TTMT suite lies about passing; the tests checklist in
`references/pass-checklists.md` turns them into questions.

## 3. The passes — run separately, judge together

Five lenses, each a checklist in `references/pass-checklists.md`: **bugs**, **security**,
**compliance** (the diff against the contract, including kill-switch byte-identity), **regression**
(callers, mirrors, mocks, docs, the wiki grep), and **data** when SQL or a backfill is present. One
combined pass finds the loud problems and stops; five separate passes each go looking for their own
class. On a Tier 2 or 3 change each lens runs in a fresh context — a subagent per lens with the brief
in `references/subagent-brief.md` — so the verdict is not coloured by the assumptions that produced
the code.

**Two Codex reviewers run beside the five lenses on every Tier 2–3 change.** They are a second model
family, and independence is the whole point of them: a Claude lens and a Claude author share
training and habits, so a class of defect they both find unremarkable stays invisible. The two are
deliberately different from each other, not two copies of one pass:

- **Codex-R — the native reviewer** (`review --scope branch --base <base>`). It takes no
  instructions at all: it has not read the spec, the plan, this skill or its checklists. What it
  finds is what a reviewer finds who has read nothing we wrote, which is what our checklists cannot
  ask for by construction.
- **Codex-A — the adversarial reviewer** (`adversarial-review --scope branch --base <base> <focus>`).
  Its focus text is the contract — spec invariants, plan constraints, the prior must-fix list,
  `CLAUDE.md` — plus the two standing questions below, and a brief to challenge the *approach*:
  which assumption, if wrong, breaks it in production, and what a simpler design would have avoided.
  The five lenses judge the diff; this one judges the decision.

Both are dispatched with the commands in `references/subagent-brief.md` — through the companion
runtime directly, **never through the `codex:codex-rescue` agent**, which forwards a `task` with write
access by default and is not a reviewer. Both run in the background and are collected with `status`
and `result`. Their output is a **candidate list, never a verdict**: every Codex finding goes through
the §4 gate like any other — a line, an input, a trace, `git blame` — and a finding both model
families raise is corroborated, not confirmed. If the Codex CLI is not installed or not
authenticated (`setup --json` says so), the review runs without it and §E says which reviewer was
missing; it does not block.

Two questions run through every lens. **Does the abstraction make sense**, not just does it pass —
Karpathy's standing complaint is that models bloat APIs and abstractions, so a module that exists to
hold one call is a finding. And **did the diff change anything it was not asked to** — models also
remove comments and code they dislike in passing; an unrequested deletion is a finding even when
the code was dead.

## 4. Verification — a finding is a proof, or it is not a finding

Every candidate from every lens goes through the same gate before it is graded:

1. **A line.** `file:line` in the head being reviewed, not a function name.
2. **An input.** The concrete state or value that reaches that line and produces the wrong outcome.
3. **A trace, not a name.** The path from a real caller to the line, read — not inferred from what
   the function is called.
4. **Provenance.** `git blame` the line; a defect the diff did not introduce is **Pre-existing**,
   reported separately and not counted against the change.
5. **Not already caught.** Something ESLint, `tsc` or a wired guard already rejects is not a finding.
6. **Verified by the cheapest read-only means that exists.** A `SELECT` through the Supabase MCP,
   a PostHog query, a scratch test written **outside the repo tree** and deleted after. If such a
   means exists and was not used, the finding is not Important yet.
7. **Blast radius, when it changes the grade.** Measured, dated, read-only — never estimated.
   "76 trades / 16 users" is a grade; "many" is not.

A candidate that fails the gate is dropped, or kept as Info marked *unverified* with the evidence
that would settle it. An Important finding is additionally sent to a **verifier** whose job is to
disprove it (`references/subagent-brief.md`), and the orchestrator re-derives at least one Important
finding per change-set from the cited line before it goes on the must-fix list.

> **Do not report a finding you have not traced to a line.** Reason: a plausible finding with no
> location costs the author a full investigation to disprove, and a few of those teach the team to
> skim the review. Instead: cite `file:line` and the failing input, or say "suspected, untraced" in
> §E and leave it out of the findings.

## 5. Grading — and the one class that is not a code finding

**Important** — wrong behaviour, a security hole, data loss, a broken invariant, or a disabled path
that is not the legacy path. Must-fix. **Low** — real and bounded; fix now or file it, explicitly.
**Info** — the author should know; no action implied. **Pre-existing** — real, not this change's.

**Operational blocker** — not a defect in the diff: a prerequisite a human must perform before the
head ships. A production row to set, a flag order, a migration precondition, a column the code
reads that no environment has yet. It goes on the must-fix list as "Action for a human:", the review
does not perform it, and it never outranks a code finding.

**Rollout state is not a finding.** "Merged to `develop`, not on `main`", "not deployed", "the
migration is not applied" describe where the change is in the pipeline, not what is wrong with it.
They belong under Rollout reminders. The verdict judges the diff. Both baseline arms on 2026-09-01
put "land this on `main`" at the head of their must-fix list and above two real defects; the rule
they cited (`ttmt-admin/CLAUDE.md`: production admin deploys from `main`) is a reminder that
`develop` is staging, not a finding against the code.

Every Important finding carries: the line, the input, the INV or rule broken, the measured blast
radius, the fix shape in one sentence, and how it was verified.

## 6. The report

`references/report-template.md` is the artifact — verdict first, tally on the same line, seams,
contract table, closure table, findings by grade, **§E what this review did not verify**, the
must-fix list, rollout reminders, feedback. It is committed next to the plan it reviews
(`docs/reviews/…` or `docs/plans/…-final-review.md`, follow the repo); a cross-repo window audit is
a wiki source under `ttmt-wiki/sources/audits/`.

§E is not optional. The step CI runs that you could not reproduce, the migration you read but did
not dry-run, the suite that would not load, the scratch test you wrote and where — each is one line,
because the code owner's decision depends on knowing where the review's coverage ends.

## 7. Closure audits — when a prior review exists

A change already reviewed is not re-reviewed from scratch. The prior must-fix list becomes the
contract: for each item, find the commit that claims to close it, open the line it changed, and
confirm the defect is gone — and that the closing commit did not weaken the test that pinned it.
A must-fix "closed" by a commit message is open. Then check the seams the prior review named and
anything the closing commits touched outside their stated scope. Read the prior review's own §E
first: what it did not verify is what this pass verifies.

## 8. Resourcing

Solo for Tier 0–1. For Tier 2–3: one subagent per lens, **plus the two Codex reviewers (`review`
and `adversarial-review`, §3)**, a verifier per Important candidate, Sonnet as the floor, Opus for a
Tier 3 change or a lens that must hold many call sites, never Haiku, never Fable inside a Workflow.
The Codex runs spend the Codex CLI's own quota, not Claude tokens, and are slow: measured 2026-09-02
on a 2-commit, 18-file diff, the adversarial run took ~55 minutes and the native reviewer's retry was
still running after two hours (its first attempt failed on model capacity). Dispatch them first, then
the lenses, and collect them last; write the report when the lenses are done and record an
unfinished Codex job in §E with its id — never wait on it. A Workflow runs only when the operator opted in by name. Budget
honestly: one three-commit review cost each baseline arm ~250k tokens and 11–13 minutes, so a
window of 180 commits is tiered or it is not reviewed. After the subagents return, `git status` and
`git log` in every repo they touched — a clean status can mean an agent committed. The orchestrator
writes the report; subagent output is evidence, not text.

## 9. Feedback

A mistake flagged for the **second** time goes into `CLAUDE.md` as part of this review —
`maintaining-claude-md` decides where. A **new class** of failure goes into the change's wiki source
document. A **fixed incident** gets a regression eval via `authoring-incident-evals`. A test that
lied — vacuous green, unloaded suite, mock no-op — goes to `writing-the-failing-test-first` and to
the gate list. Review also flags when the diff has made `CLAUDE.md`, `docs/kill-switches.md` or a
wiki page false.

The measure that a policy skill works: findings citing that policy fall toward zero. Where they do
not, the skill is not triggering or its text has drifted from the policy.

## Boundaries

> **Do not approve, merge, or resolve a review thread on your own authority.** Reason: separation
> of duties is what makes agent-written diffs reviewable, and it survives only while approval
> requires a person. Instead: report, give a verdict, and let the code owner decide. "I would
> approve it as written" is approval in the reviewer's voice; the verdict vocabulary in
> `references/report-template.md` is the whole permitted range.

> **Do not fix what you find.** Reason: a reviewer who edits the tree has become its author, and the
> next reviewer inherits the same blind spot; `--fix` is a separate, user-requested step by a
> session that is not this review. Instead: state the fix shape in one sentence and stop.

> **Do not send a review to Codex through `codex:codex-rescue`, and do not adopt Codex's verdict.**
> Reason: the rescue agent forwards a `task` with `--write` by default, so a "review" through it can
> edit the tree; and a verdict from either model family is still one reviewer's opinion. Instead:
> `review` / `adversarial-review` through the companion runtime, read-only by construction, and every
> Codex finding through the §4 gate before it is graded.

> **Do not run any git command that changes state in a checkout you do not own.** Reason: another
> session may have committed seconds ago; `stash`, `checkout`, `reset` and `merge` have each damaged
> a shared TTMT tree at least once. Instead: `git show`, `git diff`, `git log`, `git blame` only;
> a worktree outside the repo if you must build, and frontend worktrees outside the repo entirely
> because `.wt/` collects zero tests.

> **Do not touch a database to verify a change.** Reason: a migration apply is a deployment with its
> own blast radius, `migration up` adopts every other pending migration in the tree, and the MCP is
> bound to production. Instead: read SQL against the live schema, dry-run inside `BEGIN … ROLLBACK`
> on local only when the operator has sanctioned it, and use the MCP for `SELECT` alone.

> **Do not read a green gate as coverage.** Reason: `test:gate` is an explicit file list, a suite
> behind a broad `catch` passes while asserting nothing, and a gate run in the wrong repo reports
> clean over code it never saw. Instead: check gate membership for the changed files, confirm the
> `PASS <file>` line names your suite, and `cd` in the same command as the gate.

> **Do not reason from a dated snapshot in this skill.** Reason: every CI, wiring and count statement
> here carries the date it was true; the moment someone acts on it, the skill asserts a falsehood.
> Instead: run the verification command beside it and report what you found today.

> **Do not carry a stop because the answer seems obvious.** Reason: the reviewer's context is
> thinnest exactly at the merge gate — it cannot know what else is mid-flight or who owns the
> policy a finding touches. Instead: put it on the must-fix list with a named owner and wait.

## Rationalizations this skill exists to refuse

| Thought | Reality |
|---|---|
| "It's only on `develop` — that's the top finding" | That is rollout state. It goes under Rollout reminders; the verdict is about the diff. |
| "There's no spec in this repo, so there's no contract" | Search every repo's `docs/` and `git log --all --grep` first. The plan for an admin change lived in user-service. |
| "I know this repo has no CI" | Open `.github/` and `.husky/` now. Wiring is the thing that changes under a skill. |
| "The tests pass, so the change is fine" | Which tests, in which repo, and were the changed files in the list? Green is a claim to verify. |
| "The commit message explains the fix; I'll check it did that" | That is reviewing against the brief. The spec and the prior review are the contract. |
| "I can see it's wrong from the function name" | Naming is inference. Trace the caller to the line or it is not a finding. |
| "It's plausible but I'll mark it unverified and keep it Important" | If a `SELECT` or a scratch test can settle it, run it. Unverified is Info. |
| "It's only admin / only frontend / only copy" | Tier by blast radius: admin has no CI, so review is its only gate; a gate in front of every signup is Tier 2. |
| "This was reviewed yesterday, skip it" | Yesterday's must-fix list is today's contract. Closure is verified at the line, not in the log. |
| "I'll just fix this small thing while I'm here" | Then you are the author. Fix shape in a sentence; stop. |
| "The window is too big to review properly" | Then the window table is the deliverable and the tiers are the budget. Say what got a spot check; never silently narrow. |
| "A quick `migration up` locally would prove the SQL" | It would also apply every other pending migration. `BEGIN … ROLLBACK`, and only with sanction. |
| "The subagent said 10 tests passed" | Did the `PASS <file>` line name the suite? A filtered run that matched zero files also says passed. |
| "Codex found nothing, so the lenses were thorough" | Codex is a second family, not an oracle. Its silence is one data point; its findings still need a line and an input. |
| "Codex is slow — skip it this once" | The Codex runs are background jobs dispatched first and collected last; they cost wall-clock only while nothing else is running. Skip only when `setup` says Codex is unavailable, and say so in §E. |

## Red flags — stop and re-read the section

- A verdict written before the gates were run, or gates run without `cd`.
- A finding with a file but no line, or a line but no input.
- "Would merge" or "would approve" in the reviewer's own voice.
- "Land it on `main`" at the head of a must-fix list.
- "No spec exists" after searching one repo.
- A must-fix item marked closed because a commit says so.
- A report with no §E.
- An edit, a stash, a checkout, a fetch, or a `migration up` in the tool log.
- A window inventory built from a `head`-ed log.
- A Tier 2–3 report with no Codex job ids in its header and no line in §E saying why.
- A Codex finding on the must-fix list with no `file:line` re-derived by the orchestrator.

## Measurement

Baseline recorded 2026-09-01 at `evals/baselines/2026-09-01-review-skill-baseline.json`: one
three-commit admin change, one trial per arm, Sonnet. No skill 4.5/10 with four traps hit; the
2026-08-31 text of this skill 7.5/10 with two traps hit — both arms graded rollout state as their
top Important finding and both searched one repo for the contract. This revision was written against
those two misses.

First held-out use, same day: the 2026-08-27→09-01 window audit
(`ttmt-wiki/sources/audits/2026-09-01-change-window-review.md`), eleven lens agents over four repos.
Every Important finding carried a line and a failing input; no agent approved in its own voice;
rollout state was filed under reminders in all eleven reports; three agents found the governing plan
in a sibling repo. Both baseline traps stayed closed. Two things this text had not anticipated and
now does: the harness refuses report files written by subagents, and a session-wide concurrency cap
can refuse a dispatch — retry after the first completion rather than dropping the lens.

2026-09-02: the two Codex reviewers (§3, §8) were added to the Tier 2–3 fan-out. First held-out use
the same day (`ttmt-frontend/docs/reviews/2026-09-02-record-search-closures-review.md`): Codex-R
failed once on model capacity; its retry took ~2 h 15 m and returned four findings including **both
Important findings of that review** — a palette state regression and an unnamed-tab a11y regression —
that none of the Claude lenses, the orchestrator (who was also the author) or Codex-A had raised. That
is the un-briefed reviewer earning its place: it does not know what we consider settled. Codex-A
(~55 min) returned five findings — two Lows the lenses also found, one new mechanism that a verifier
confirmed (a scope count in trade units on a position surface), one pre-existing, one
accepted-by-decision; its "NO-SHIP" verdict was not adopted. Both Codex verdicts were treated as data. Six of seven fresh-context lens dispatches stalled on the platform's 600 s
stream watchdog; the one that completed carried a brief that limits tool output (sed ranges, no
node_modules, per-file diffs) — that discipline now lives in `references/subagent-brief.md`.
