---
name: wiki-ingest
description: Use when a new source file has landed in sources/ and needs to be folded into the wiki. Checkpoints the source's key takeaways with the operator before any drafting, then proposes changes and verifies each claim against the source with fresh context — before producing a diff plan for human approval.
---

# Wiki Ingest

You're about to fold a source document into the TTMT wiki. The wiki is a curated, LLM-maintained knowledge base where errors compound: a wrong claim cross-references itself into related pages before anyone notices. Your job is to extract what's *actually in the source* — nothing more — and integrate it carefully.

## Input

The user invokes this skill with either:
- A path to a single source file (e.g., `/wiki-ingest sources/postmortems/2026-05-12-foo.md`)
- `--all-new` to process every source not yet referenced in any page's `derived_from`

Resolve the input to a list of source paths before proceeding.

## Workflow

### Pass 0: Takeaways checkpoint

**Runs before any drafting, and stops.** The workflow's only human gate used to be Pass 3, where
the operator meets a fully drafted multi-page diff — so disagreeing with the *framing* throws away
every page that framing produced. Pass 0 moves the cheap disagreement to the front: takeaways are
cheap to produce and cheap to redirect. It **adds** a gate; Pass 3 stays exactly as it is.

For each source:

0a. **Read the source verbatim.** Nothing else yet — no greps, no candidate list, no drafts.
0b. **Emit at most 10 bullets of "what this source says."** The source's claims in the source's own
    terms — what happened, what changed, what it asserts. Not what you intend to write about it.
0c. **List the pages you expect to touch.** Existing pages by path; new pages as
    `NEW wiki/<dir>/<slug>.md`. A path list, not drafts — one line of intent each at most.
0d. **STOP and wait for a confirm-or-redirect.** Do not slide into Pass 1 on your own.

If the operator redirects — wrong framing, wrong pages, a takeaway that misreads the source —
revise the bullets and stop again. Only an explicit confirmation opens Pass 1.

Output of Pass 0: ≤ 10 takeaway bullets plus the expected page list. No page drafts, no frontmatter,
no edits, no lint.

### Pass 1: Propose

For each source (only after Pass 0 was confirmed):

1. **Load contracts.** Read `schema.md` at the wiki root. Note the required frontmatter, page templates, ingest rules.
2. **Read the source verbatim.** Don't summarize yet — just read.
3. **Identify affected pages.**
   - Read `wiki/index.md` to understand the topical map.
   - For every entity name, file path, function name, or slug mentioned in the source, grep `wiki/` for matches.
   - Read the `summary:` frontmatter of every candidate page.
   - Build a candidate list of ≤ 15 pages that may need updates.
4. **For each candidate page, compute the patch:**
   - "Patch" means the smallest set of *edits*, not the smallest *content*. New pages and substantial updates must hit the richness criteria in `schema.md` (600-1500 words, 2-5 concrete claims per H2, code examples, "why" framing, cross-links).
   - If the source confirms or extends existing claims → in-place update of the relevant section, deepening it with the source's specifics.
   - If the source adds new information that doesn't displace anything → append `## Update YYYY-MM-DD — <source-slug>` block.
   - If the source disagrees with existing wiki content → append `## Contradiction — <source-slug>` block. Do NOT resolve the contradiction — flag it for human review.

   **Pass 1 checklist** — before moving on from a page draft, confirm:
   - [ ] Each required H2 section has 2-5 concrete claims (numbers, file paths, function names, table/column names, error codes).
   - [ ] At least one fenced code block per page where the source describes a function signature, frontmatter shape, API contract, or command invocation.
   - [ ] "Why it exists" framing present (under `## Purpose` for architecture, `## Why It Exists` for concept).
   - [ ] Service touchpoints enumerate concrete files, not just service names.
   - [ ] Counter-patterns / "do not X" guidance included where the source documents an anti-pattern, framed as `> **Do not [X].** Reason: [Y]. Instead: [Z].`
   - [ ] Cross-links use real markdown only for pages that exist; italics for pending pages.
   - [ ] Page is 600+ words unless the entity is trivially small (rare for architecture/concept pages).
   - [ ] `description:` is set — **one sentence, max 200 characters**, and distinct from `summary`.
         `description` is the cheap scan surface `/wiki-query` loads for every page; `summary` stays
         the longer abstract it opens afterwards. Write the description, don't truncate the summary.
5. **Update frontmatter** on every edited page:
   - Append the source path to `derived_from`.
   - Bump `last_updated` to the source date.
   - If the summary is outdated (new info changes the page's primary concern), propose an updated summary.
   - Set `description:` if the page doesn't have one (one sentence, ≤ 200 chars). A re-ingest is the
     cheapest moment to fill it in — you already have the page loaded.
   - Set `generated: {by: <actor>, at: <ISO-8601 UTC>}` — you are the producer of the content you are
     about to write, so say so. `by` uses the actor convention from `schema.md` (`claude-opus-5/wiki-ingest`
     for this skill, `human:<id>` for a person, `process:<id>` for automation); `at` is an ISO-8601
     datetime with an explicit UTC offset, e.g. `2026-08-29T14:30:00Z`.
   - Do **not** touch `verified` here. Pass 1 has verified nothing (see Pass 2, step 9b).

   > **Do not append to `wiki/index.md`'s (or any generated `wiki/**/index.md`'s) `derived_from`.**
   > Reason: indexes are **navigation, not derived pages** — nothing in them is a claim traced back to a
   > source, so a provenance list on them is noise that grows once per ingest forever. `wiki/index.md`
   > has already accumulated 239 `derived_from` entries and 22,734 words this way. Instead: add or
   > refresh the index's *link row* for the page, and leave its frontmatter alone.

6. **Propose new pages** for entities not yet covered. Use the per-type template from `schema.md`. Choose the correct location (`wiki/services/<svc>/components/`, `wiki/concepts/`, `wiki/bugs/`, `wiki/decisions/`, etc.). On a new page also set:
   - `description:` (mandatory for anything this workflow creates) and `generated: {by, at}` as above.
   - `stale_after:` — an **absolute** ISO-8601 instant, per the per-type defaults in `schema.md`:
     `bug` => none (a postmortem is permanently true) · `channel-analysis` => +90 days ·
     `operation` => +90 days · deploy-state notes => +30 days · everything else => none.
     Compute the instant at write time; the reader just compares `now >= stale_after`.

Output of Pass 1: a list of proposed patches (page path → diff) and proposed new pages (path → full content).

### Ingesting a proposal source (`sources/proposals/`)

Proposal sources are the one **forward-looking** input — a feature idea or request, not a record of what the platform does. They ingest differently:

- **Target a `proposal` page** under `wiki/proposals/<slug>.md` (type `proposal`), distilling the design doc to 600–1500 words across the required sections (`## Problem`, `## Proposed Solution`, `## Scope & Size`, `## Implementation Status`, `## Open Questions`).
- **First body line is the mandatory stage banner** (`> **PROPOSAL — stage: {stage}.** ...`).
- **Set lifecycle frontmatter**: `stage` (REQUIRED), `size`, `effort_estimate`, `opened`. A fresh idea is `stage: proposed`. Only set `stage: shipped|partial` when the source documents that it was actually built — and then `coverage` + `shipped_as` (PR refs + the verified wiki pages it produced) are mandatory.
- **Update the register** `wiki/proposals/index.md` (add/refresh the row) and ensure `wiki/index.md` links the register.
- **Do NOT cross-link the proposal from as-built pages** as if it were real. Proposals are referenced only from the register, other proposals, or an explicit `## Proposed Enhancements` block.
- **Verification target differs (see Pass 2):** a proposal's *design* claims verify against the idea source, not the codebase; its *implementation* claims (`stage`/`coverage`/`shipped_as`) verify against the cited PRs/wiki pages.

When a previously-proposed feature later ships, re-ingest (or hand-edit via this workflow) to flip `stage` → `shipped`/`partial`, fill `coverage`/`effort_actual`/`shipped_at`/`shipped_as`, and add the `## Implementation Status` narrative. Never delete the proposal.

#### Pre-ingest duplicate check (Project #2)

**Before producing the proposal-page draft**, search Project #2 ("TTMT Development") for any existing card that already tracks this idea. A proposal that duplicates an existing card creates wiki drift (two pages chasing one tracking issue) and is worse than no page at all.

Run this once per new proposal source, BEFORE Pass 1 drafts the page:

```bash
# Whole-project search (the "Feature Requests" column is where ingested proposals land,
# but ideas can also sit in Idea/Todo/Implementing — search them all).
gh project item-list 2 --owner lukacsaron --limit 250 --format json \
  | python3 -c "
import json, sys, re
data = json.load(sys.stdin)
# Edit the keyword list to fit this proposal — pull nouns from the title/scope.
keywords = ['<KEYWORD-1>', '<KEYWORD-2>', '<KEYWORD-3>']
pattern = re.compile('|'.join(re.escape(k) for k in keywords), re.IGNORECASE)
for item in data.get('items', []):
    title = item.get('title', '')
    if pattern.search(title):
        url = (item.get('content') or {}).get('url', '') if isinstance(item.get('content'), dict) else ''
        print(f'[{item.get(\"status\")}] {title} → {url}')
"
```

Pick 3–6 keywords from the proposal's title and core scope (e.g., for a Mautic email proposal: `mautic`, `email`, `notification`, `crm`). The `gh project item-list --limit` ceiling is 250; if the project ever exceeds that, fall back to per-status GraphQL pagination.

**If a match is found:**
- Surface it to the operator in Pass 3's diff plan. Do NOT silently create a duplicate proposal page or duplicate issue.
- If the match is a closed card (Done/Closed for a related-but-different feature), proceed but link the closed card from the proposal's `## Implementation Status` as a related-prior.
- If the match is open and substantively the same idea, recommend updating the existing wiki proposal page (or filing a refinement) instead of creating a new one.

#### Post-ingest tracking-issue creation

After the operator approves Pass 3 and the proposal page + register update are written, this skill opens a GitHub issue and adds it to Project #2 with `Status = Feature Requests`. This is the **opening half** of the lifecycle loop (the closing half — flipping to Done on ship — still cannot be done by this skill; see below).

Repository convention (verified against #624–#639): proposal-tracking issues are filed in `lukacsaron/ttmt-frontend` with the `enhancement` label, regardless of which service the proposal ultimately touches. The frontend repo is the canonical TTMT roadmap surface.

```bash
# 1. Create the issue. Body template mirrors the convention in #624/#639:
#    - one-paragraph framing
#    - links to the wiki page + source doc
#    - Size + Effort + Approach line
#    - "Why now" bullets
#    - Scope checkboxes
#    - Open product questions
#    - Wiki-stage lockstep footer
gh issue create -R lukacsaron/ttmt-frontend \
  --title "<short imperative title>" \
  --label enhancement \
  --body "$(cat <<'EOF'
**Feature Request — design tracked in the wiki.**

<one-paragraph pitch from the proposal's ## Problem framing>

- Design (canonical): https://github.com/lukacsaron/ttmt-wiki/blob/main/wiki/proposals/<slug>.md
- Full source doc: https://github.com/lukacsaron/ttmt-wiki/blob/main/sources/proposals/<source-filename>.md

**Size:** <S|M|L|XL> · **Effort:** <effort_estimate>.
**Approach:** <one-line on the recommended approach if the proposal has one>.

### Why now
- <bullet>
- <bullet>

### Scope
- [ ] <checkbox from ## Scope & Size table>
- [ ] <checkbox>

### Open product questions
1. <from ## Open Questions>

_Wiki stage: `proposed`. Keep the wiki page `stage` and this issue's Project Status in lockstep._
EOF
)"
# → returns the issue URL; grab the issue number.

# 2. Add the issue to Project #2 and capture the project-item node ID.
gh project item-add 2 --owner lukacsaron \
  --url "https://github.com/lukacsaron/ttmt-frontend/issues/<NNN>" \
  --format json
# → response includes "id": "PVTI_..." — this is the project-item node ID, needed for step 3.

# 3. Set Status = Feature Requests.
gh project item-edit \
  --id <PVTI_... from step 2> \
  --project-id PVT_kwHOAPvG-s4A02t5 \
  --field-id PVTSSF_lAHOAPvG-s4A02t5zgqa6-4 \
  --single-select-option-id 887a1eed

# 4. Verify the status landed.
gh api graphql -f query='
query {
  node(id:"<PVTI_... from step 2>") {
    ... on ProjectV2Item {
      fieldValueByName(name:"Status") { ... on ProjectV2ItemFieldSingleSelectValue { name } }
      content { ... on Issue { number title url } }
    }
  }
}'
```

**Project #2 identifiers (cache these — they don't change):**

| Thing | ID |
|---|---|
| Project node ID | `PVT_kwHOAPvG-s4A02t5` |
| Status field ID | `PVTSSF_lAHOAPvG-s4A02t5zgqa6-4` |
| Status: Idea | `994fc8e8` |
| Status: Feature Requests | `887a1eed` |
| Status: Todo | `f75ad846` |
| Status: Implementing | `ed8b113c` |
| Status: Done | `98236657` |
| Status: Closed | `0fc0472e` |

If any of these stop working, re-run `gh project field-list 2 --owner lukacsaron` and `gh api graphql -f query='{ user(login:"lukacsaron") { projectV2(number:2) { field(name:"Status") { ... on ProjectV2SingleSelectField { options { id name } } } } } }'` to refresh them, then update this table.

**After issue creation, update the wiki:**

1. **Proposal page frontmatter:** add `tracking_issue: "lukacsaron/ttmt-frontend#<NNN>"` (immediately after `opened:`).
2. **Register row** (`wiki/proposals/index.md`): replace the `— (needs card)` cell with `[#<NNN>](https://github.com/lukacsaron/ttmt-frontend/issues/<NNN>)`.
3. **Bump `last_updated`** on both files to today.
4. **Re-run lint:** `npm run lint:scope -- --scope=wiki/proposals`.

If the operator declines tracking-issue creation (skill should ask once in Pass 3 for new proposals), leave `tracking_issue` unset and the register cell as `— (needs card)`. Don't silently skip — surface the choice.

**Closing the loop is still NOT done by ingest.** This skill opens the loop (issue + Project Status = Feature Requests) when a proposal is first ingested. It does NOT close the loop when the proposal later ships — the *execution half* (close the issue, move Status to Done) still has to happen via the implementing PR's `Closes lukacsaron/ttmt-frontend#NNN` keyword or a manual `gh issue close NNN --reason completed` + Project Status edit. On a re-ingest that flips `stage` → `shipped`/`partial`, flag this to the operator. On `rejected`/`abandoned`, close as **not planned** and Status → Closed. See `schema.md` → "Closing the loop when a proposal ships (or is killed)". A `stage: shipped` page whose tracking issue is still `open` is drift — surface it.

### Ingesting a decision source (`sources/decisions/`)

`sources/decisions/` holds 116 ADR files and, until the `decision` page type existed, had nowhere to
land: ADR content dissolved into `architecture` and `concept` pages, usually as one line under
`## Recent Changes`. That keeps the outcome and loses the reasoning — **the options considered and
rejected, which are the entire point of an ADR, evaporate on ingest.**

- **Target a `decision` page** at `wiki/decisions/YYYY-MM-DD-<slug>.md` (type `decision`). The date is
  the *decision's* date, taken from the source — not the ingest date.
- **Use the required sections, in order:** `## Context`, `## Options Considered`, `## Decision`,
  `## Why`, `## Consequences`.
- **`## Why` is load-bearing.** Everything else can be reconstructed from the code and the git history;
  the reasoning cannot. This is the section that currently evaporates — write it first, and give it the
  source's actual argument, not a restatement of the outcome.
- **`## Options Considered` must name the options that were rejected, and why.** A decision page listing
  only the option that won is an architecture page with a date on it.
- **Verification target differs (see Pass 2):** like `proposal`, a `decision` verifies against its
  **decision source**, not the live codebase. The question is *does the page faithfully record what was
  decided and why*, not *does the code still look like this*. Code drifts away from decisions; that
  doesn't make the decision page wrong, it makes the decision superseded — describe current behavior on
  the `architecture`/`component` page and link to it from `## Consequences`.
- **Reversals use `supersedes`.** A later reversal gets its own dated page and `supersedes:` the old one,
  which flips to `status: superseded`. Never delete a decision page.

### Pass 2: Verify against source

**This pass is non-negotiable.** It exists because Pass 1 may have hallucinated, inferred, or extrapolated claims that aren't in the source.

For each proposed change from Pass 1:

7. **Re-read the source verbatim.** Treat this as a fresh task — ignore reasoning from Pass 1.
8. **For each new or modified claim in the patch, ask: does this claim trace to specific text in the source?**
   - ✓ **supported** — quote the supporting text (or summarize the section that supports it).
   - ⚠ **inferred** — reasonable but not stated. Reword as `> Inferred from <source>: <claim>` blockquote rather than asserted prose.
   - ✗ **unsupported** — fabrication. Strip from the patch.

   **For `proposal` pages, the verification target shifts:** a *design* claim ("the proposed schema adds a `profile_id` column") verifies against the **idea source** — does the page faithfully distill what the proposal says? It does NOT need to match the live codebase (the feature isn't built). But any *implementation* claim — `stage: shipped`, `coverage: full`, a `shipped_as` PR/page — verifies against that cited PR or wiki page. If you can't confirm the proof, downgrade `stage` to `proposed`/`accepted` and strip the unsupported `shipped_as`.
9. **Verify frontmatter changes.**
   - `derived_from`: must include the source being ingested — and does *not* include any
     `wiki/**/index.md` page (indexes are navigation; see Pass 1 step 5).
   - `last_updated`: must match the source date or today (whichever is later — never older).
   - `summary` updates: must be supported by the source.
   - `description`: present, one sentence, **≤ 200 characters**, and supported by the source the same
     way any other claim is. Count the characters — 200 is a ceiling, not a target.
   - `generated`: `by` matches the actor convention, `at` is ISO-8601 with an explicit UTC offset.
   - `stale_after`: matches the per-type default, and is an absolute instant rather than a duration.

9b. **Record the verification.** Once this pass has actually completed for a page, APPEND an entry to
    that page's `verified` list: `{by: <this agent's actor>, at: <ISO-8601 UTC>}` — e.g.
    `{by: claude-opus-5/wiki-ingest, at: 2026-08-29T14:30:00Z}`. Append; never overwrite prior entries.
    A single entry may be written as a bare mapping — that is read as a one-element list.

    This is recorded because **the pass already happens and its result was previously thrown away.**
    Pass 2 has always checked every claim against the source; the wiki just had nowhere to put the
    verdict, so the next reader had no way to tell a verified page from an unverified one. Writing it
    down costs nothing and is the difference between `unverified` and `machine-confirmed`.

    > **`verified` is NEVER back-filled.** Do not add a `verified` entry to a page whose verification
    > did not actually run in this session — not to pages you didn't touch, not to pages you edited
    > without re-checking, not to the 690 existing pages. Fabricating a verification event is precisely
    > the failure this field exists to prevent. No key at all resolves to trust tier `unverified`, which
    > for those pages is the truthful answer.

Output of Pass 2: a verified version of each patch (with unsupported claims stripped, inferred claims demoted).

### Pass 3: Diff plan and apply

10. **Present the diff plan to the user.** For each page:
    - Show the page path and current `summary`.
    - Show the proposed changes (additions in green, deletions in red — use diff format).
    - Show verification verdicts (count of ✓ / ⚠ / ✗ from Pass 2).
    - For new pages, show the full proposed content.
11. **Wait for explicit approval.** Do not auto-write.
12. **On approval, apply the changes.** Use the Edit and Write tools. The approval is itself a
    verification event: APPEND `{by: human:<id>, at: <ISO-8601 UTC>}` to `verified` on each page the
    operator approved, after the machine entry from step 9b. That `human:` prefix is what lifts the page
    from `machine-confirmed` to `human-reviewed` — it is the only thing that does. Use the operator's
    actual id; if you don't know it, ask rather than invent one, and if the operator approves only part
    of the plan, only the approved pages get the entry.
13. **Run scoped post-ingest lint** on touched pages:

    Run this from the **wiki repo root** (the directory containing `schema.md`, `wiki/` and
    `package.json`) — never a hardcoded absolute path, which only resolves on one machine:

    ```bash
    npm run lint:scope -- --scope=<touched-page-path>
    ```

    Run once per touched page (or once for the common subtree if touched pages share one).

14. **Report final state.** Pages touched, new pages created, lint errors (if any), source now appears in `derived_from` of N pages.

    Then record the ingest in the ledger with the script — **never by editing `wiki/log.md` by hand**:

    ```bash
    npm run log -- ingest '<source-slug> — N pages updated, M created'
    ```

    It places the entry newest-first for you. Hand-editing has been got wrong twice by two different
    sessions in one hour: the log's own documentation contains a fenced `## [YYYY-MM-DD]` template, so
    "insert before the first `## [`" lands *inside the code fence*, where `grep "^## \["` will never
    find the entry again. `npm run lint` now fails on that corruption.

    One line, because `wiki/log.md` uses a `union` merge driver and a union merge is line-wise — an
    entry spanning two lines can be interleaved with another ingest's. Everything the entry needs must
    fit on its own line. `wiki/log.md` is generated navigation, so this is the one write in the whole
    workflow that isn't a wiki page: never give it frontmatter, and never edit an earlier entry — a
    wrong entry is corrected by appending a new one that says so.

## Hard rules

- **You never write a wiki page bypassing this workflow.** Direct `Edit`/`Write` on `wiki/**/*.md` outside the approved diff is a contract violation.
- **You never strip an existing `## Contradiction` block.** Those stay until a human resolves them.
- **You never reduce `derived_from`.** Sources only get added, never removed (ingest is append-only on this field).
- **You never back-fill `verified`.** A `verified` entry may only be written for a verification pass
  that actually ran, in this session, on that page. Every other page correctly reads as `unverified`.
- **You never append to an index's `derived_from`.** `wiki/index.md` and every generated
  `wiki/**/index.md` are navigation, not derived pages. Update their link rows; leave their frontmatter alone.
- **When uncertain, leave it out.** Gaps are recoverable. Wrong claims compound.

## Common patterns to expect

- A source describes a bug. The bug page gets created or updated. The component pages for affected files (which the bug touches) get `## Update — <bug-slug>` blocks in their `## Known Issues` section, with a link to the bug page.
- A source describes an architecture decision. If it is an ADR from `sources/decisions/`, the reasoning lands on a `decision` page at `wiki/decisions/YYYY-MM-DD-<slug>.md`; the relevant `wiki/architecture/<topic>.md` page gets a `## Recent Changes` entry linking to it, and affected component pages get cross-link updates. Do not dissolve the ADR into the architecture page — the options rejected are the part that gets lost that way.
- A source describes a new concept. A new `wiki/concepts/<slug>.md` page is created; the index page gets the new entry added under "Key pages" in the appropriate section.

## When to refuse to ingest

- Source content is too vague (no specific claims) — return a recommendation to triage to `archive/`.
- Source contradicts multiple wiki pages without resolution — surface the contradictions; ingest only the non-contradictory parts.
- Source is structurally malformed (no readable content) — report and skip.

In all of these, **do not produce a diff plan**. Tell the user what's wrong and stop.
