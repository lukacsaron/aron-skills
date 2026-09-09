---
name: wiki-consolidate
description: Surfaces a correction that is already on a wiki page but buried in an Update or Contradiction block, up into the canonical sections and summary so the retrieval path can see it. Use when a page's canonical sections contradict or omit something a later block on the same page records, when /wiki-lint reports a buried invalidation, or when the user says a page is misleading, stale at the top, or that the truth is in the update blocks.
---

# Consolidating a buried invalidation

`/wiki-query` scans `description`/`summary`, then reads pages top-down. A correction that lives only
in the chronological `## Update` tail is invisible to that path: the page is complete and practically
misleading.

This is a **third sanctioned operation**, distinct from ingest and from direct authoring — see
`schema.md` → "Consolidation". `/wiki-ingest` cannot do this job: it requires a source, and a
consolidation has none. It relocates knowledge the page already carries.

An audit of 271 pages found **120 confirmed cases**, 17 of them materially wrong rather than merely
incomplete. Worst example: a page asserted no live trade had ever been placed by a client that had
been trading live for two months — inside a block headed "preserve in any derived page."

## The eight constraints

All mandatory. Seven are checked by `python3 scripts/check-consolidation.py`; the first is not, and
it is the one that matters.

1. **No claim that is not already on the page.** Restating the buried block is the whole job. A new
   file path, number, date or behaviour is a fabrication.
2. **The `## Update` / `## Contradiction` block survives verbatim.** Its substance is duplicated
   upward, never moved. A `## Contradiction` is never stripped — only a human resolves those.
3. **No `verified:` entry, ever.** No source was re-read, so no verification event occurred. Writing
   one silently promotes the page from `unverified` to `machine-confirmed` on a pass that never ran.
4. **`generated.by` names the consolidation**, e.g. `claude-opus-5/buried-invalidation-fix` — never
   `wiki-ingest`, which did not run.
5. **`derived_from` unchanged.** No new source was consulted.
6. **Stale canonical text is marked superseded in place**, not deleted:
   `**SUPERSEDED YYYY-MM-DD — see §Update YYYY-MM-DD.**`
7. **Independent review for new claims**, then the checker, before commit.
8. **One line in `wiki/log.md`**, op `refactor` — via `npm run log -- refactor '<subject>'`, never by
   editing the file (a hand edit lands inside its fenced template; `npm run lint` fails on that).

## Method

1. `grep -n '^## ' <page>` for its shape — canonical sections versus the Update tail.
2. Read the canonical sections and `summary:` in full, then every Update/Contradiction block.
3. For each block: does it invalidate, narrow or qualify something above it? If yes, is that already
   reflected in a canonical section or in `summary`? **Check, do not assume** — most candidates fail
   here, and that is the correct outcome.
4. Surface what survives: a sentence or two in the right canonical section, counter-patterns into
   `## Edge Cases`, and a `summary`/`description` revision so a scanner would know to open the page.
5. Run the checker. Commit with the deviation from rule 3 of `CLAUDE.md` stated plainly.

## Why review is not optional

On the first run of this operation the orchestrating prompt instructed 15 agents to write a
fabricated `verified:` entry. Fourteen of fifteen reviewers approved it **because the brief had
sanctioned it**; one went back to `CLAUDE.md` and caught it.

> **Do not review an edit against the instructions it was given.** Reason: a brief can itself be
> wrong, and a reviewer checking only compliance cannot catch a bad brief. Instead: judge every edit
> against `CLAUDE.md` and `schema.md`, and report a violation even when the instructions required it.
