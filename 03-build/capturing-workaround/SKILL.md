---
name: capturing-workaround
description: Records a deliberately accepted deviation as a workaround page, with the reason it is not the real fix and the condition for removing it. Use when shipping a stopgap, a temporary hack, a monkey-patch, a pin, a disabled check, or a "we will fix this properly later" decision, and when the user mentions a workaround, stopgap, band-aid, or accepted tech debt.
---

# Capturing a workaround

A workaround is a deviation the team **consciously accepted**: it works, it is not the real fix, and
it carries a condition under which it should be removed. Recording it turns an accepted risk into
something reviewable instead of something rediscovered.

At the time this type was added, 26 wiki pages mentioned a workaround and 7 a stopgap, with no
register — so "what are we carrying?" was unanswerable.

## The one distinction that matters

> **Do not file an unfixed bug as a workaround.** Reason: an unfixed bug is a `bug` page with
> `resolution: open` — it has no accepted deviation and no removal criteria, and filing it here
> launders "we did not get to it" into "we decided this." Instead: a workaround requires a
> deliberate acceptance, a named blast radius, and a condition for removal.

If you cannot name who accepted it and what would let you delete it, you do not have a workaround.

## Shape

`ttmt-wiki/wiki/workarounds/<slug>.md`, type `workaround`. Required frontmatter beyond the standard set:

```yaml
accepted_on: 2026-08-30      # when the deviation was accepted
review_by:   2026-11-30      # when the acceptance must be RE-TAKEN
accepted_by: "human:alukacs" # optional, actor convention
replaces_fix: "<slug or tracking ref of the proper fix>"   # optional
```

`review_by` is required because a workaround without a review date is not accepted, it is forgotten.
Lint warns once it passes — louder than `stale_after`, because what has gone stale is a decision
nobody has re-taken.

Required sections:

```
## Context
## The Workaround
## Why Not The Real Fix
## Blast Radius
## Removal Criteria
```

**`## Why Not The Real Fix` is load-bearing.** It is what stops the workaround being re-litigated by
the next person who finds it, and it is the section that goes stale first.

**`## Removal Criteria` must state a condition, not a wish.** "When mtapi.io ships per-symbol
subscriptions" is a condition. "Eventually" is not.

**`## Blast Radius` names what breaks if the workaround is wrong** — which services, which accounts,
which data. A workaround whose blast radius is unknown has not been assessed.

## Writing it

The page is a wiki page, so it lands through `/wiki-ingest` with a source under `sources/`, not by
direct edit. Where the workaround was accepted in a PR or an incident, that record is the source.

Then run `npx tsx scripts/generate-indexes.ts` to build `ttmt-wiki/wiki/workarounds/index.md` — the register
groups overdue reviews first, which is the point of having one. (No workaround has been ingested yet,
so neither the directory nor the register exists until the first one lands; the generator knows the
path.)
