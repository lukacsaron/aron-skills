# The review report — what it is

A review report is the artifact a code owner reads to decide. It has these parts, in this order.
Every part is present; a part with nothing to say says so in one line rather than disappearing.

```markdown
# <Change or window> — review

**Reviewed:** YYYY-MM-DD · **Reviewer:** did not write this code
**Reviewers dispatched:** Claude lenses ×<n> (<models>) · Codex-R `<job id>` · Codex-A `<job id>` — or `Codex: unavailable (<setup output>)`
**Subject:** <repo> `<base>..<head>` (N commits) — or the window table for a multi-repo audit
**Contract:** spec `<path>` · plan `<path>` · prior review `<path>` · `none — <why>`
**Gates run by this review:** <command> in <repo> → <exact result line>. One row per gate.
**Constraint honoured:** nothing edited, committed, pushed, merged, reverted, reset or applied;
production access read-only `SELECT`.

## VERDICT: <one of the four below> — <tally: N Important, N Low, N Info, N Pre-existing>

Two paragraphs at most: what is right and how you know; what blocks and why it is not a rejection
(or why it is). A reader who stops here knows whether to merge.

## A. Seams — where one task's output meets the next's input
## B. Contract — each INV-n / Global Constraint / kill-switch clause, checked against the diff
| Inv | Verdict | Where (file:line) | Pinned by (suite) |

## C. Closure — every must-fix item from the prior review, with the commit that closes it
| MF | Claimed closed in | Verified at (file:line) | Closed? |

## D. Findings
### Important
**I1 — <one-line claim>** `path:line`
Failing input/state → wrong outcome. Which INV or rule it breaks. Blast radius (measured, dated,
read-only). Fix shape in one sentence. How it was verified (trace, test, query).
### Low
### Info
### Pre-existing (not introduced by this change; reported, not counted against it)

## E. What this review did NOT verify
One line each: the step CI would run that could not be reproduced, the migration that was read but
not dry-run, the suite that could not load, the subagent report not independently re-derived, the
Codex run that did not complete (which one, and the last status line).

## What must happen before merge
Numbered. Only Important items and operational blockers. Nothing else belongs here.

## Rollout reminders
Operator steps the change depends on (flag defaults, migration order, canary), copied from spec §9
with anything the diff changed.

## Feedback
- CLAUDE.md: <rule to add, or "none">  — a mistake flagged for the second time
- Wiki source: <new failure class, or "none">
- Eval: <incident that needs a regression eval, or "none">
```

## The four verdicts

| Verdict | Meaning |
|---|---|
| **MERGEABLE** | No Important findings, contract holds, gates green in the right repo |
| **MERGEABLE-WITH-MUST-FIX-LIST (N)** | Important findings exist, each bounded, each with a fix shape; the design is right |
| **NOT MERGEABLE** | The contract is violated in a way a fix list does not cover, or the disabled path is not the legacy path |
| **UNREVIEWABLE — <reason>** | No contract, no base, or the tree reviewed is not the tree that ships. Say what is missing |

## For a window audit (many change-sets, several repos)

The report opens with the **window table** — one row per change-set: repo, commits, tier, contract,
prior review, mode (full / closure / spot) — and then repeats sections B–E **per change-set** in
tier order, highest first. The verdict is per change-set; the window as a whole gets a one-paragraph
summary and a single consolidated must-fix list, grouped by repo.

Where it goes: a single change reviewed in its repo →
`docs/reviews/YYYY-MM-DD-<slug>-review.md` (or `docs/plans/…-final-review.md`, follow the repo).
A cross-repo window audit → `ttmt-wiki/sources/audits/YYYY-MM-DD-change-window-review.md`, so
`/wiki-ingest` can carry the findings into bug and decision pages.
