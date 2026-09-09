---
name: writing-spec-from-intent
description: Writes a TTMT design spec — the problem, the invariants it must not violate, the kill-switch, and the test matrix — and flags the concerns a policy owner must resolve rather than resolving them. Use when starting a non-trivial change, when the user asks for a spec, a design doc, a technical design, requirements, or an approach, and before any implementation plan is written.
---

# Writing a spec from intent

The spec is the contract the plan, the diff and the review are all judged against. TTMT already
commits one per change; this skill is about writing it to the shape the later stages can actually use.

## Where it goes, and what it is called

`docs/specs/YYYY-MM-DD-<slug>-design.md` in the product repo — but check which convention the repo
uses before creating a directory: `ttmt-user-service` and `ttmt-frontend` have both `docs/specs/` and
`docs/plans/`; `ttmt-admin` has only `docs/superpowers/{specs,plans,audits}/`. Same shape and same
rules either way; follow the repo rather than standardising it in passing.

The spec and its plan are committed **together, before the first feature commit**, as
`docs(<scope>): design spec + implementation plan`. That ordering is what makes the plan reviewable
against the spec instead of against the diff.

## The header carries the risk

The fullest header form is user-service's, and it is the one to reach for whenever the change can
move money or orders (admin and frontend specs commonly run a lighter header — Date / Status /
Services touched / Route; keep the fields that apply, never invent a kill-switch for an ungated
change). **Let the change set the weight, not the repo** — a gate in front of every signup is not
lighter than a flag nobody has flipped:

```markdown
**Date:** YYYY-MM-DD
**Status:** APPROVED DESIGN — pending implementation plan
**Kill-switch:** `FEATURE_X_ENABLED` (default ON; only literal `false`/`0`, case-insensitive,
trimmed, disables → byte-identical legacy path)
**Incident refs:** trd_… / sig_… (date, account: what happened).
**Prior art (read YYYY-MM-DD):** the wiki pages consulted — see "Read what already broke here" below.
`none found` is a valid value; a blank line is not.
**Blast radius of the defect (audited YYYY-MM-DD):** N trades / M users affected.
```

Header surveyed 2026-08-30 across the three repos' `docs/specs` and `docs/superpowers/specs`.

**Audit the blast radius; do not estimate it.** Query it and date the query. "76 trades / 16 users where a single partial-close event
closed the entire trade" is what turns a bug report into a prioritised change, and it is the number
review will ask for. For a user-facing change the unit is the funnel, not the trade — "116 reach the
card gate, 60 clear it, 51.7%" is the same discipline; PostHog already holds it.

## Read what already broke here — before §1

The wiki carries counter-patterns in an enforced form — `Do not X. Reason: Y. Instead: Z.` — one per
thing that has already failed in the area you are about to change. For every service the change
touches:

```bash
grep -rn '^> \*\*Do not' ttmt-wiki/wiki/services/<svc>/ ttmt-wiki/wiki/bugs/
```

Counted 2026-08-31: 304 blocks for user-service, 309 frontend, 165 admin, 135 landing-page, 459 on
bug pages. Re-run the grep rather than citing those numbers.

Then ask `/wiki-query` for prior art on the behaviour itself. A hit is one of three things, and each
one lands somewhere specific:

| What you find | Where it goes |
|---|---|
| A constraint the change must not violate | §3, as an `INV-n` |
| An approach already tried and rejected | §2, as the alternative and why it lost |
| A `proposal` or `decision` page already covering this | Nowhere — link it and stop; the spec may not be needed |

`running-review-passes` runs the same grep against the finished diff. That is the backstop, not the
first line of defence: a counter-pattern found here costs a sentence, found at review it costs a
rewrite.

> **Do not design against the code alone.** Reason: code shows what the system does and never what it
> already did wrong — one read-safety defect shipped seven separate times in `ttmt-user-service`, each
> instance documented in the wiki and invisible to the session writing the next one. Instead: read the
> counter-patterns for the touched services before §1, and carry what you find into §2 or §3.

## Sections

```markdown
## 1. Problem
A numbered gap table — G1, G2, … — one row per divergence: mode, behaviour, requested → achieved.
Name the file and line range the behaviour lives in. Say whether existing tests PIN the wrong
behaviour, because those assertions flip with the feature and the plan must schedule that.

## 2. Decisions taken (user-approved YYYY-MM-DD)
Each decision, its alternative, and why the alternative lost.

## 3. Invariants (the contract)
INV-1, INV-2, … — the properties no rounding path, clamp, retry, degradation **or rendered
state** may violate. A UI invariant is as checkable as a numeric one: *"a user who declines the
card keeps their Telegram connection"*, *"no list renders without an empty and an error variant"*.
This is the section review checks the diff against line by line. Write it in terms that can fail.

## 4. Semantics
The new behaviour, per mode or per branch.

## 4b. Surface & states  (REQUIRED when the change renders anything)
The job in one sentence — when/I want to/so I can. Every state the user can reach, each with its copy:
empty, loading (incl. the >400ms case), partial, error, stale, permission, success. For data: the unit,
the sample size shown alongside, and what is never blended. For a setting: where it sits, what it is
called in the user's words, its default and its scope layer. `none — no rendered surface` is a valid
value; a blank section is not. `designing-the-user-surface` carries the method.

## 5. Kill-switch contract
Default, exact parsing, and what the disabled path is. "Byte-identical legacy behaviour" is the
standard TTMT holds; anything weaker is a second code path to maintain.

## 6. Surfacing
Trace events, PostHog constants, notification copy — how anyone will see this working in production.

## 7. Callers (inventoried YYYY-MM-DD)
Every call site, listed. Inventoried, not recalled.

## 8. Test matrix
The exact suite files that will cover this (`tests/unit/<name>.test.ts`), and which existing
assertions flip. For a rendered change add an interaction/visual row and a manual-verification row —
frontend CI runs ESLint, `tsc` and Jest, none of which sees a layout break or unreadable contrast. `writing-the-failing-test-first` covers making them actually able to fail.

## 9. Rollout
Ship order, the flag's default at each step, and how to back out. Written for the **operator to
execute** — migration applies, deploys, fleet rolls and production flag flips are recorded here as
ordered human steps with their preconditions, never performed as part of writing the spec or
building the change. Where ship order matters, say why: "user-service before admin, because the
admin gate suppresses on a NULL the user-service build is what populates" is a rollout step; "deploy
both" is not.

## 10. Out of scope
Stated explicitly, so the plan cannot quietly widen the change.

## 11. Flagged concerns
| # | Concern | Owner | Status |
|---|---------|-------|--------|
```

## The one thing TTMT specs do not do yet

TTMT resolves design questions in-session and records them under **§2 Decisions taken
(user-approved)** — which is stronger than the playbook asks for, because the approver and date are
on the record. What is missing is the other half:

> "Work through the flagged concerns first as they are the points an analyst would have escalated.
> The product owner resolves each one with its policy owner before engineering sees the spec."

Anything touching money movement, order execution, customer data, broker credentials, retention,
**pricing presentation, consent, or friction on a path every user traverses**, or
a limit someone else owns goes in **§11 Flagged concerns** with a named owner and `Status: open` —
even when you could answer it. A concern that moves to §2 records who approved it. A concern nobody
owns is the one that becomes an incident.

## Boundaries

> **Do not resolve a flagged concern yourself.** Reason: the flag exists because the decision belongs
> to whoever owns that policy, and a spec that answers it removes their only chance to say no before
> engineering commits. Instead: state it in §11 with an owner, and let it move to §2 when approved.

> **Do not write an invariant that cannot fail.** Reason: §3 is what review checks the diff against,
> so a goal phrased as an aspiration ("handles partial closes correctly") passes every review while
> constraining nothing. Instead: state the bound — "a PARTIAL close never closes the trade's last
> open volume; `plannedCloseVolume ≤ totalOpenVolume − volume_min`."

> **Do not list callers from memory.** Reason: the caller inventory is what bounds the blast radius
> of the change, and a missed call site is a path that silently keeps the old behaviour after the
> flag flips. Instead: grep for them, and date the inventory in the heading.

> **Do not plan the implementation here.** Reason: a task breakdown written before the codebase is
> read is a guess, and a plan embedded in a spec is never revised when the real plan changes.
> Instead: stop at semantics and the test matrix — `planning-before-building` writes the plan.

## Linkage back to this wiki

`spec_ref: "ttmt-user-service:docs/specs/YYYY-MM-DD-<slug>-design.md@<sha>"` on the wiki page for
this change. See `schema.md` → "SDLC artifacts". The wiki holds the link, never a copy.
