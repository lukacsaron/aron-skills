---
name: wiki-query
description: Use to answer a question against the TTMT wiki. Scans one-line page descriptions first, then reads the top-5 pages in full, and produces a cited answer. If no answer exists, reports a gap rather than fabricating.
---

# Wiki Query

You're answering a question against the TTMT wiki. The wiki is the answer surface — never fall back to reading raw sources directly when the wiki has a gap. If you can't answer from the wiki, report the gap.

## Input

The user invokes this skill with a natural-language question. Example: `/wiki-query "how does trade-executor handle the entry zone resolver?"`

## Workflow

1. **Grep first, LLM second.**
   - Search `wiki/index.md` for relevant section names.
   - Grep filenames under `wiki/` for entity names from the question.
   - **Scan `description:`, not `summary:`.** `description` is one sentence, ≤ 200 characters — the cheap surface, small enough to hold every candidate at once. Get it with `head -20` on candidate `.md` files (or read the first 20 lines) to pull frontmatter cheaply.
   - *Why the change:* across the 690 pages the `summary:` fields total ~327 KB ≈ 82k tokens (median summary 421 characters). Summaries became abstracts — fine for the second read, ruinous for the first. Scanning them now costs more than reading the five pages the scan selects, so summary-scanning is no longer the cheap step it was designed to be.
   - **Fallback:** `description` is new and not yet back-filled everywhere. A page without one is scanned on its `summary`, as before.
2. **Pick ≤ 5 pages to read in full.** Rank by:
   - Direct match on slug (highest)
   - Match in `description:` (or in `summary:`, for pages that have no `description` yet)
   - Reachable via cross-link from a high-confidence match
   - Read the longer `summary:` only for this shortlist — it is the second read, not the scan.
   - Don't read more than 5. If the answer isn't in 5 pages, the wiki has a gap.
3. **Synthesize the answer.**
   - Quote or summarize the relevant claims.
   - Every claim must trace to a wiki page — cite with relative paths: `[trade-executor](services/user-service/components/trade-executor.md)`.
   - For synthesis questions ("how do X and Y relate?"), pull from each page and explicitly state the connection.
   - **Surface trust and staleness** for every page you cite — see "Trust and staleness" below. Don't present an unverified or expired page flatly as fact.
4. **If no good answer exists**, do not fabricate. Report:

   > The wiki doesn't cover this. Closest pages are [X](path) and [Y](path).
   >
   > To fix the gap, you can:
   > 1. Locate or write a source that answers this question and drop it into `inbox/`.
   > 2. Triage it into `sources/`, then run `/wiki-ingest`.

5. **File a substantial answer back — through the source layer.** If the question + answer are substantial and not yet covered by a page:

   > This isn't covered by a page yet. I can write it up as `sources/notes/qa-2026-08-29-<slug>.md` — the question, the answer, and the pages it cites — then run `/wiki-ingest` on it. Want me to?

   The note carries: the question as asked, the answer as given, the wiki pages cited, and an honest split of what is known from those pages vs. what you inferred. Write the note when the user says yes; offer the ingest, don't run it unprompted.

   *Why through `sources/notes/` and not a drafted page:* an exploration that lives only in chat history is exactly the RAG behaviour the wiki pattern exists to beat — good answers are supposed to compound in the knowledge base just like ingested sources do. But drafting a wiki page directly skips the two-pass verification, and an unverified claim cross-references itself into related pages and compounds too. Routing through the source layer gets both halves: the exploration compounds, *and* it still passes Pass 2 verification before it touches a page. Nothing is bypassed — the answer enters through the front door.

6. **Log the query.** After answering, append the entry with the script — **never by editing `wiki/log.md` by hand**:

   ```bash
   npm run log -- query '"why does the zone resolver run at Step 0.5?" — answered from 4 pages'
   ```

   or, when you reported a gap in step 4:

   ```bash
   npm run log -- query '"why does the zone resolver run at Step 0.5?" — gap reported'
   ```

   It prints the line number it wrote, and places the entry newest-first for you. The script exists
   because hand-editing this file has been got wrong twice by two different sessions in one hour: the
   log's own documentation contains a fenced `## [YYYY-MM-DD]` template, so "insert before the first
   `## [`" lands *inside the code fence*, where `grep "^## \["` will never find the entry again.
   `npm run lint` now fails on that corruption.

   Never edit a past entry. This matters more than it looks: `last_updated` records when a page was *written*, never whether anyone read it. The query log is the only signal available for **which** of 691 pages actually get opened — which ones earn their keep, and which have never been read once.

## Trust and staleness

Trust is **derived from a page's `verified:` frontmatter at read time and never stored.** There is no `trust:` field.

| `verified` contains | Tier | How to cite it |
|---|---|---|
| no `verified` key at all | `unverified` | Say so: "*(unverified — no verification pass recorded)*" |
| only non-`human:` actors (`claude-opus-5/wiki-ingest`, `process:weekly-bot`) | `machine-confirmed` | Cite normally; note the tier if the claim is load-bearing |
| any `human:` actor (`human:alukacs`) | `human-reviewed` | Cite normally |

A page is **stale** when `stale_after` is present and `now >= stale_after`. Flag it: "*(stale since {stale_after} — may no longer hold)*". No `stale_after` key means the page has no known expiry, not that it is fresh.

> Most of the 690 existing pages carry no `verified` key and therefore resolve to `unverified`. That is the truthful answer, not a defect — and it is never an invitation to add one. `verified` is written only by `/wiki-ingest`, only from a verification pass it actually ran. Fabricating a verification event is exactly the failure the field exists to prevent.

## Hard rules

- **Never read `sources/` directly.** The wiki is the answer surface. If you find yourself reading sources to answer a question, stop — run `/wiki-ingest` first.
- **Never write a wiki page.** Knowledge pages change only through `/wiki-ingest`. The single exception is the one-line query entry in `wiki/log.md` — that file is navigation and bookkeeping, not knowledge. A new answer goes to `sources/notes/` (step 5), never straight into `wiki/`.
- **Never fabricate citations.** Every cited path must exist.
- **Never read more than 5 pages in full.** If 5 isn't enough, the wiki has a gap. Report it.
- **Use the right vocabulary.** When the wiki has a canonical name for something (e.g., "AccountRouter", "TP redistribution"), use it — don't invent synonyms.
- **`proposal` pages are NOT current behavior.** A `type: proposal` page describes a feature idea, not how the platform works today. If one lands in your answer set:
  - **Never state its claims as fact.** Label them: "*(proposed, stage: {stage})*" or "this is a proposal, not yet built."
  - For a "how does X work today?" question, a proposal is only relevant as "there's a proposal to add X" — answer from `concept`/`architecture`/`component`/`bug` pages for what *is*, and mention the proposal separately if useful.
  - If the ONLY page covering a topic is a proposal, say so plainly: "The platform doesn't do this today; there's a proposal for it: [slug](proposals/slug.md), stage: {stage}."

## Querying proposals (feature ideas / requests)

Some questions are about the *idea/feature backlog*, not current behavior — e.g. "what feature ideas do we have for the performance page?", "did we ever build profile-level stats?", "what's still just proposed?", "how big was the reviews feature?".

- **Start at the register:** read `wiki/proposals/index.md` (Title · Stage · Size · Coverage · Shipped-as) — it answers most "what exists / was it built / how big" questions in one read.
- **Filter on `stage`/`coverage` frontmatter** for "shipped vs proposed vs rejected" questions. A `shipped`/`partial` proposal's `shipped_as` points at the verified pages it produced — follow those for the as-built detail.
- **`rejected`/`abandoned` proposals are first-class answers** to "why didn't we build X?" — the reasoning is in `## Implementation Status`.

## Question patterns

| Question shape | Approach |
|---|---|
| "What does X do?" | Read X's component or concept page; quote the `## Purpose`. |
| "How does X work?" | Read X's mechanics section; describe data flow. |
| "What's the bug behind Y?" | Read the bug page Y; quote `## Symptom`, `## Root Cause`, `## Fix`. |
| "How do X and Y relate?" | Read both; describe the connection using `related:` frontmatter and cross-links. |
| "What changed in X recently?" | Read X's `## Recent Changes` (component) or `## Update` blocks. |
| "Where in the codebase is X implemented?" | Read X's `code_refs:` frontmatter. |
| "Why did we choose X over Y?" | Read the `decision` page under `wiki/decisions/`; quote `## Options Considered` (the options that were rejected, and why) and `## Why`. |
| "What feature ideas / requests do we have (for X)?" | Read `wiki/proposals/index.md`; filter by topic. These are proposals, not current behavior. |
| "Did we build X / is X shipped?" | Read the proposal's `stage` + `coverage`; if shipped, follow `shipped_as` to the verified pages. |
| "Why didn't we build X?" | Read the `rejected`/`abandoned` proposal's `## Implementation Status`. |
