# Dispatching a review lens to a subagent

One subagent per lens (bugs, security, compliance, regression, data), each with a fresh context and
the same contract. Then one verifier per Important candidate. Sonnet is the floor; Opus for a Tier 3
change or a lens that must hold many call sites at once; never Haiku; never Fable inside a Workflow.
A Workflow runs only when the operator has opted in by name — otherwise use the Agent tool.

The brief is the same shape every time. Fill every slot; a subagent cannot ask.

```
ROLE: You are the <lens> reviewer for <change-set>. You did not write this code.

SUBJECT: <repo path>, `<base>..<head>` (<N> commits). Read it with `git diff <base>..<head>` and
`git show`; read surrounding code as needed.

CONTRACT (judge the diff against THIS, not against commit messages or the brief you were given):
- spec: <path> — §3 invariants INV-1..INV-n
- plan: <path> — Global Constraints
- prior review must-fix list: <path> §<n>   (or: none)
- repo rules: <repo>/CLAUDE.md
- what already broke here: run
  grep -rn '^> \*\*Do not' <wiki>/wiki/services/<svc>/ <wiki>/wiki/bugs/
  and check the diff against every block whose subject it touches.

LENS: <paste the one checklist from references/pass-checklists.md>

TOOL DISCIPLINE (six of seven lens dispatches stalled on the 600 s stream watchdog on 2026-09-02 while
ingesting large outputs; the one that finished obeyed this): never print a whole file — read with
`sed -n 'A,Bp'` in ranges of at most 120 lines; never read anything under node_modules; diff one file at
a time with `git diff <base>..<head> -- <file>`; keep grep output under 40 lines with `| head -40`.

RULES — non-negotiable:
- READ-ONLY. Do not edit, create, delete or rename any file. No git command that changes state
  (no checkout, stash, reset, commit, merge, push, branch). No npm install. No database command
  other than a `SELECT`; no Supabase MCP write; no migration apply, not even local.
- Run gates only with `cd <repo> &&` in the same command, output redirected to a file under
  <scratch>, never piped through tail.
- Every finding cites `file:line` and names the failing input or state. A finding you cannot
  trace to a line is not reported — say "suspected, untraced" in NOT VERIFIED instead.
- Check `git blame` before attributing: a defect the diff did not introduce is PRE-EXISTING.
- Do not review against your own brief. If the brief and the spec disagree, report the disagreement.

RETURN your report as your final message (the harness refuses report files written by subagents,
so do not try; the orchestrator saves the returned text under <scratch>/audit/<id>/report.md), in
exactly this shape:
## Findings
- [Important|Low|Info|Pre-existing] <claim> — `path:line` — input/state → outcome — INV/rule —
  how verified
## Contract table   (compliance lens only)
## Gates run        (command → exact result line)
## NOT VERIFIED     (one line each)
## Process notes    (what you read, what you skipped, and why)
```

## The verifier brief

```
ROLE: Adversarial verifier. A reviewer claims: "<finding verbatim, with path:line>".
Your job is to try to DISPROVE it against the actual code at <repo> `<head>`.
Trace the path from the cited line: what calls it, with what values, under what guard.
Answer with one of:
- CONFIRMED — the failing input reaches the line and produces the claimed outcome; cite the trace.
- PRE-EXISTING — real, but present before <base>; cite `git blame`.
- NOT A DEFECT — the guard at `path:line` prevents it; or the claimed input cannot occur because …
- UNVERIFIABLE — what evidence would be needed.
Same RULES as above: read-only, cite lines, no inference from names.
```

## The two Codex reviewers

Dispatch them **before** the Claude lenses (they run 5–15 minutes in the background) and collect
them after. Always through the companion runtime — the `codex:codex-rescue` agent forwards a `task`
with `--write` by default and must not be used for a review. Run every command with `cd <repo>` in the
same line: the runtime reviews the repository it is invoked in, and `--scope branch --base <base>`
reviews `<base>...HEAD` of that checkout, so for a branch that is not checked out use its worktree.

```bash
CODEX=$(ls -d "$HOME"/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs | sort -V | tail -1)

# 0. Is Codex there? Unavailable or unauthenticated → review without it, say so in §E, do not install or log in.
cd <repo> && node "$CODEX" setup --json

# 1. Codex-R — the native reviewer. No instructions on purpose.
#    nohup, NOT --background: a --background job DIES with the dispatching Bash shell (confirmed
#    2026-09-03 and again 2026-09-07 — `status` kept saying `running` for 80 min over a log frozen
#    at its startup lines). The job id is in the first lines of the nohup log.
cd <repo> && nohup node "$CODEX" review --scope branch --base <base> > <scratch>/audit/<id>/codex-review.log 2>&1 &

# 2. Codex-A — the adversarial reviewer. The focus text is the contract plus the standing questions.
cd <repo> && nohup node "$CODEX" adversarial-review --scope branch --base <base> "$(cat <scratch>/audit/<id>/codex-focus.txt)" > <scratch>/audit/<id>/codex-adversarial.log 2>&1 &

# 3. Liveness is the job log's mtime, never `status` (which reports `running` for a dead job):
stat -f '%Sm' "$HOME"/.claude/plugins/data/codex-openai-codex/state/<workspace>-*/jobs/<job-id>.log

# 4. Collect (status --wait blocks until the job settles). Jobs are keyed by workspaceRoot: run
#    these from the SAME checkout, and do not remove a worktree under review until `result` has
#    been collected — removing it drops the jobs from the registry.
cd <repo> && node "$CODEX" status <job-id> --wait --timeout-ms 900000
cd <repo> && node "$CODEX" result <job-id> --json > <scratch>/audit/<id>/codex-<review|adversarial>.json
```

The focus text for Codex-A, written to `<scratch>/audit/<id>/codex-focus.txt` before dispatch:

```
Adversarial review of <repo> `<base>..<head>` (<N> commits). READ-ONLY: report only — do not edit,
apply patches, or run any git, npm or database command that changes state.

Judge the diff against THIS contract, not against commit messages:
- spec <path> — §3 invariants INV-1..INV-n; §5 kill-switch: the disabled path must be byte-identical
- plan <path> — Global Constraints
- prior review must-fix list <path> §<n>   (or: none)
- <repo>/CLAUDE.md

Two standing questions: does each abstraction earn its place (a module that exists to hold one call
is a finding); did the diff change anything it was not asked to (an unrequested deletion is a
finding even when the code was dead).

Challenge the approach, not only the code: which assumption, if wrong, breaks this in production;
what would a simpler design have avoided; what does the disabled (kill-switch off) path actually do.

Every finding: `path:line`, the failing input or state, the outcome, and how you verified it. Mark a
defect present before <base> as PRE-EXISTING (git blame). Mark anything you could not trace as
"suspected, untraced". Do not propose fixes beyond one sentence of fix shape.
```

Codex returns its own shape (verdict, summary, findings, next steps). Save it verbatim, then treat it
as a candidate list: deduplicate against the lenses, send each Important candidate to a verifier, and
re-derive from the cited line before grading. Its verdict and "next steps" are not adopted — this is a
review, and the verdict is the orchestrator's. Record both job ids in the report header; a run that
failed or was cut off is one line in §E, not a reason to rerun the whole review.

## After the subagents return

The orchestrator (you) does not forward their findings. You:

0. Save each returned report verbatim to `<scratch>/audit/<id>/report.md` the moment it arrives —
   the notification text is the only copy, and a context reset loses it. Collect the two Codex jobs
   with `status --wait` / `result --json` into the same directory.
1. Deduplicate across lenses and across model families; the same defect seen through two lenses, or
   by a lens and a Codex run, is one finding (note the corroboration, it is not confirmation).
2. Send every Important candidate to a verifier; demote or drop what does not come back CONFIRMED.
3. Re-derive at least one Important finding per change-set yourself, from the cited line, before
   putting it on the must-fix list.
4. Check `git status` and `git log` in every repo the subagents touched — a clean status can mean an
   agent committed, not that nothing changed.
5. Write the report. The subagents' process notes go into §E, not into the findings.

> **Do not reply to a subagent to "continue" it inside a Workflow.** Reason: the reply resumes a
> duplicate instance that re-runs the whole brief and races the original. Instead: open any reply with
> "ANSWER ONLY — do not resume", or dispatch a fresh verifier.
