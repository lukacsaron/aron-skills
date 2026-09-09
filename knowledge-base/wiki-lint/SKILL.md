---
name: wiki-lint
description: Use weekly or before big ingests to health-check the wiki. Runs mechanical validators (npm run lint), then performs LLM-driven contradiction and code-reference checks. Outputs a report to lint-reports/ and never auto-fixes.
---

# Wiki Lint

You're running a health check on the TTMT wiki. Lint never auto-fixes — it produces a report for human (or follow-up agent) triage.

## Input

Either no input (lint everything) or `--scope=<subtree>` (e.g., `--scope=wiki/services/user-service`).

## Workflow

### Pass 1: Mechanical (no LLM)

1. Run the test runner from the wiki repo root (the repo you are already in — never a
   hardcoded absolute path; the checkout lives somewhere different on every machine):

   ```bash
   npm run lint
   # or, scoped:
   npm run lint:scope -- --scope=<subtree>
   ```

2. Capture the output. It covers:

   **Schema & structure**
   - Required frontmatter fields (`type`, `status`, `summary`, `last_updated`, `derived_from`)
   - Valid enums and date formats
   - Broken relative cross-links
   - `derived_from` references that don't resolve to real source files
   - Required H2 sections per page type — including the `decision` type
     (`## Context` / `## Options Considered` / `## Decision` / `## Why` / `## Consequences`)
   - Recommended H2 sections, as warnings — notably `## Lessons` on `bug` pages. A postmortem
     whose only output is a diff teaches nothing; `## Lessons` is where the generalizable part
     lives. Most existing bug pages predate the convention, so expect a large backlog here.

   **Index reachability** (replaces the old inbound-link orphan check)
   - Breadth-first traversal from `wiki/index.md`. The question is not "does *something*
     link here?" but "can a reader or an agent NAVIGATE here from the index?" A page that
     only deep cross-references point at is invisible on the primary retrieval path and is,
     for retrieval purposes, an orphan. The old inbound-link count is preserved in the
     message, so "unreachable but cross-linked" stays distinguishable from "referenced by
     nothing at all".

   **Frontmatter contract** (see `schema.md` for the canonical definitions)
   - `description` missing — the one-sentence cheap scan surface. All 690 `summary:` fields
     together are ~327KB (~82k tokens), so "scan summaries first" is no longer cheap;
     `description` is what a reader can afford to load for every page.
   - `description` over-long — hard ceiling of 200 characters.
   - `stale_after` elapsed — `stale_after` is an ABSOLUTE ISO-8601 instant, so staleness is
     the plain comparison `now >= stale_after`. Per-type defaults: `bug` => none (a
     postmortem is permanently true) · `channel-analysis` => +90d · `operation` => +90d ·
     deploy-state notes => +30d · everything else => none.
   - Malformed `generated` / `verified` — both carry Actors `{ by, at }`. `at` is an ISO-8601
     datetime with an explicit UTC offset (e.g. `2026-08-29T14:30:00Z`). `by` follows the
     actor convention: `<producer>/<version>` for agents and tools
     (`claude-opus-5/wiki-ingest`), `human:<id>` for people (`human:alukacs`),
     `process:<id>` for automation (`process:weekly-bot`). A bare `verified` mapping is
     valid and means a one-element list.

   **Knowledge gaps** (three coverage checks — they say a hole exists, never that a page is wrong)
   - *Missing-page queue*: slugs named in `related:` that no page exists for, ranked by how
     many pages want each one. This is the wiki asking, in machine-readable form, for pages
     it knows it needs.
   - *Under-linked pages*: pages with no outbound cross-links to other wiki pages. The
     inverse of index reachability — reachable, possibly correct, contributing nothing to
     navigation.
   - *Under-integrated sources*: sources cited by only one or two pages via `derived_from` —
     filed, not integrated; the ingest most likely took the headline claim and left the rest
     on the floor. (Zero-citation sources are a different failure; the source-coverage check
     in Pass 2 owns those.) The list is capped and the cap is announced in the output — carry
     that announcement into the report rather than dropping it.

3. **Severity discipline.** Everything in the new-checks groups above lands as a **warning**.
   There are 690 existing pages; a new REQUIRED field or section would break all of them at
   once. `npm run lint` must still report **0 errors** — a non-zero error count is a
   regression to fix, not a backlog to file. A large warning count is expected and fine; it is
   the worklist. Report both numbers verbatim.

**Do not back-fill `verified`.** If a page has no `verified` key, its trust tier is
`unverified`, and that is the truthful answer — not a defect to paper over. Fabricating a
verification event is exactly the failure the field exists to prevent. Only `/wiki-ingest`
writes `verified`, and only from a pass it actually ran. Lint never writes it at all.

### Pass 2: LLM-driven

These checks require reading content, not just structure.

4. **Stale pages.** For every page with `last_updated` older than 180 days, or with an elapsed
   `stale_after`, scan its `code_refs:` and the wiki pages in its `related:` list. If the
   referenced code paths look obsolete (filename changed, function renamed) or the related
   pages contradict its claims, flag for review.

5. **Contradiction register.** 40 pages currently carry `## Contradiction` blocks (~47 in
   total; `wiki/services/mt-gateway/components/ea-bridge.md` alone has 4). There is no list, no
   age and no owner, so they accumulate indefinitely. The gap is **visibility, not policy** —
   fix the visibility, keep the policy.

   Enumerate every `## Contradiction — <slug>` block in scope. For each one record:
   - the page it sits on,
   - the `<slug>` it names,
   - the **age of the source that slug names** — resolve the slug to a file under `sources/`
     (most source filenames carry a `YYYY-MM-DD` prefix; otherwise use the source's own
     frontmatter date). If the slug resolves to no source at all, say so — an unresolvable
     contradiction is itself a finding.

   Report **oldest source first**: a contradiction against a source from months ago has been
   sitting unresolved longest and is the most likely to have quietly become wrong on both
   sides. Never resolve one. Never edit or delete a `## Contradiction` block. The LLM surfaces
   contradictions; the human decides.

6. **Contradiction scan (new pairs).** Walk cross-link clusters. For each cluster (a connected
   component in the link graph), read all pages and check for claims that conflict. Flag pairs
   of pages where claim A in page X contradicts claim B in page Y. These are *candidates* for
   new `## Contradiction` blocks — proposed in the report, written only by `/wiki-ingest`.

7. **Code-reference spot check.** Sample 10 % of pages (randomly), read their `code_refs:`
   entries, and verify the target files exist in the live service repos. Those repos sit
   alongside this one in the same checkout parent (e.g. `../ttmt-user-service/src/lib/...`);
   resolve them relative to the wiki repo root rather than assuming any absolute path. Flag
   dead references.

8. **Source coverage.** List every file under `sources/` and check whether any wiki page has it
   in `derived_from`. Sources with **zero** references are un-ingested — flag them. (Sources
   with one or two references are the under-integrated warnings from Pass 1; keep the two
   lists separate, they need different follow-up.)

### Pass 3: Report

9. Write the report to `lint-reports/YYYY-MM-DD.md` with this structure:

   ```markdown
   # Wiki Lint Report — YYYY-MM-DD

   **Scope:** [full | <subtree>]
   **Pages scanned:** N
   **Sources scanned:** N

   ## Errors (mechanical)
   [from npm run lint output — this section should be empty; anything here is a regression]

   ## Warnings (mechanical)
   [grouped by check: index reachability, missing/over-long description, elapsed stale_after,
    malformed generated/verified, missing recommended sections, knowledge gaps.
    Give a count per group, then the entries. Repeat any cap the validator announced.]

   ## Contradiction register
   [every `## Contradiction — <slug>` block in scope, OLDEST SOURCE FIRST:
    | source date | age | page | source slug | resolves? |
    Never resolved here — this is a visibility list.]

   ## Contradictions (LLM, new candidates)
   [pairs of pages with conflicting claims not yet captured by a `## Contradiction` block]

   ## Code-reference issues (LLM)
   [dead code_refs]

   ## Un-ingested sources
   [sources/ files with zero derived_from references]

   ## Proposed work
   [The part of lint that looks forward instead of backward. Three lists, each capped, and
    each cap stated inline — "top 10 of 137" — so a truncated list can never read as a
    complete one. No silent truncation anywhere in this section.

    ### Pages worth writing (top 10)
    Ranked from the missing-page queue: the slugs the most pages already ask for.
    For each: the slug, how many pages want it, and which page type it would be.

    ### Sources worth seeking (top 5)
    Gaps the wiki cannot answer from what it already has. For each: the question it would
    unblock, and where the source would plausibly come from (a repo, a channel, an ADR,
    a person). Do not invent a source that may not exist — say what you'd go looking for.

    ### Questions worth investigating (top 5)
    Things the corpus makes suspicious but cannot settle: a claim two clusters disagree
    about, a component nothing links to that services still reference, a source cited once
    whose remaining 90% looks load-bearing. Phrase each as a question, not a conclusion.]

   ## Summary
   N errors, N warnings, N contradictions (register), N new contradiction candidates,
   N dead refs, N un-ingested sources.
   ```

10. Append exactly ONE self-contained line to `wiki/log.md` with the script — **never by hand**:

    ```bash
    npm run log -- lint '<scope> — E errors, W warnings, C contradictions'
    ```

    Where `<scope>` is `full` or the subtree, and `C` is the size of the contradiction
    register. The script places it newest-first, matching the rest of the ledger — this step
    previously said "at the end of the file", which contradicted `CLAUDE.md` and `schema.md`.
    Hand-editing lands the entry inside the file's fenced `## [YYYY-MM-DD]` template, where
    `grep "^## \["` will never find it; `npm run lint` now fails on that corruption.
    One line per lint run, appended, never rewritten — `wiki/log.md` is a ledger.
    This is the only file lint is ever allowed to write outside `lint-reports/`, and appending
    to it is the only write it makes.

11. Print a one-line summary to stdout. Do not make any changes to wiki pages.

## Hard rules

- **Never auto-fix.** Lint surfaces issues; humans decide.
- **Never resolve a contradiction.** The register makes them visible and ages them; that is all.
- **Never write `verified`, and never back-fill it.** An absent `verified` key is an honest
  `unverified`, not a gap to close.
- **Never delete `lint-reports/`.** Old reports are historical record.
- **Don't run weekly LLM lint inside `/wiki-ingest`'s post-ingest scoped lint** — that one is
  mechanical-only and fast.

## When to use which lint

- After every `/wiki-ingest` — automatic scoped mechanical lint on touched pages only (built into ingest).
- On every commit to the wiki repo — CI runs `npm run lint`, blocks the commit on mechanical errors.
- Weekly, or before a big batch of ingests — run `/wiki-lint` (this skill) for the full mechanical + LLM pass.
