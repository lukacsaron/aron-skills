---
name: capturing-intent
description: Turns a raw idea, complaint, or incident finding into a well-formed intent document in the originator's own words, ready for the design pass. Use when someone describes a problem they want solved, asks to file a feature request or idea, wants to write a proto-spec or PRD, or when a monitoring finding or triage decision needs to re-enter the development loop.
---

# Capturing intent

Intent is captured **once, in the originator's own words**, as a version-controlled artifact the next
stage can act on — instead of passing through backlog entries, user stories and refinement meetings
that leave what reaches engineering several steps removed from what was meant.

The originator need not be technical and no formal language is required.

## Method

1. **Check whether it has already been asked.** `/wiki-query` for the problem — not the solution — and
   scan `ttmt-wiki/wiki/proposals/index.md`. If a proposal already covers it, **do not open a second
   one.** Add this requester, their words, and their evidence to the existing page through
   `/wiki-ingest`. A second independent request is the strongest signal a proposal can carry, and
   filing it separately destroys exactly that signal.
2. **Let them describe the problem in their own words.** What they cannot do today, who is affected,
   what better looks like, what is out of scope.
3. **Ask what an analyst would ask** — scope, users, constraints, what success looks like — until the
   idea is concrete. This is the part that earns the artifact; do not skip to the template.
4. **Write it to the template.** Their words, tightened; not your paraphrase of their words.
5. **Have them correct anything you misunderstood.** Their correction, not your defence.
6. **File it** — see "Where it goes". Author and timestamp join the record.

## Template

```markdown
# Intent: <short title>
Author: <name (team)>. Status: draft.

## Problem
What cannot be done today, and who it affects.

## Proposed outcome
What is true after this ships, in observable terms.

## Affected users and systems
People, services, and data touched.

## Constraints
Hard limits — data classification, existing auth, no new PII, budget, deadline.

## Open questions
The things you genuinely do not know yet. Leaving these blank is a tell that step 2 was skipped.
```

## Where it goes

**The wiki, by default.** Write `ttmt-wiki/sources/proposals/<slug>.md` in the originator's words and
run `/wiki-ingest`; it becomes a `proposal` page. `proposal` **is** TTMT's intent artifact —
forward-looking, verified against its idea source rather than the codebase, carrying a `stage`
lifecycle (the enum and its ship/kill rituals live in `schema.md` → "proposal"; don't restate them
here), a `size`, and a `tracking_issue`, all listed in one register at
`ttmt-wiki/wiki/proposals/index.md`.

Three reasons this is the default rather than a service repo:

- **Intent precedes scope.** Every other artifact in the chain — spec, plan, review — is written after
  you know which service changes. Intent is written before. Filing it under one repo picks a service
  while the design pass is still supposed to be free to choose, which is the same mistake as writing
  the solution.
- **The originator often isn't an engineer**, and this skill says so. Asking them to choose among
  service repos, one of which files specs under `docs/superpowers/` instead, is asking the wrong
  person a question with no right answer.
- **Rejected intent needs somewhere to rest.** `stage: rejected` is a real state with pages in it. A
  service repo has no equivalent, so a declined idea becomes untracked debris or gets deleted along
  with the reason it was declined.

**A service repo, by exception.** `docs/intent/YYYY-MM-DD-<slug>.md` is worth it only when the scope is
already known, single-repo, and the spec follows in the same sitting — there the `git log` locality
beside `docs/specs/` and `docs/plans/` is real. Check the repo's convention first: `ttmt-admin` uses
`docs/superpowers/`.

Taking the exception means **one home, not two.** File the intent in the repo, then reference it from
the wiki with `intent_ref: "<repo>:docs/intent/YYYY-MM-DD-<slug>.md@<sha>"` (see `schema.md` → "SDLC
artifacts"). The wiki holds the link, never a copy.

> **Do not keep the intent in two places.** Reason: the two copies answer to different reviewers and
> neither is authoritative, so the one that gets updated is whichever the next person happens to open —
> and a stale intent is worse than none, because it looks like a decision. Instead: pick the wiki or the
> repo, and let the other side hold only an `intent_ref`.

## Boundaries

> **Do not write the solution.** Reason: intent that arrives pre-solved removes the design pass's
> only degree of freedom, and the constraint that would have ruled the solution out never gets
> stated. Instead: capture the problem and the outcome; the spec chooses the approach.

> **Do not resolve the open questions yourself.** Reason: an open question is the cheapest thing in
> the lifecycle to answer and the most expensive to discover after build. Instead: carry them
> forward — the design pass answers them or escalates them to a policy owner.
