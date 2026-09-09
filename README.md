# aron-skills

The canonical home for the TTMT development skills. Every `.claude/skills/` and `.agents/skills/` directory across the TTMT repos is a symlink into this tree — this is the only place the real files live.

Fifteen skills, organised by the stage of the change cycle they serve.

## Why it lives outside the repos

These skills describe *how work gets done*, not what any one service does. They used to live in `ttmt-wiki/.claude/skills/` with the other repos symlinking sideways into it, which made the wiki repo the de-facto owner of the whole platform's engineering process and coupled every skill edit to a wiki commit. Pulling them out here gives the process its own home: one copy to edit, one history to read, and no repo holding a dependency the others quietly rely on.

## Layout

Numbered folders mirror the six stages in `00-orchestration/running-the-change-cycle/SKILL.md`, so the directory order *is* the pipeline order. Two unnumbered folders hold skills that aren't bound to a single stage.

```
aron-skills/
├── 00-orchestration/
│   └── running-the-change-cycle          drives stages 1-6; sizes the work, picks the gates
├── 01-plan/
│   └── capturing-intent                  raw idea or incident finding -> intent document
├── 02-design/
│   ├── designing-the-user-surface        UX discipline; only when the change renders
│   └── writing-spec-from-intent          spec: invariants, kill-switch, test matrix
├── 03-build/
│   ├── planning-before-building          ordered tasks, each with a verification line
│   ├── distinguishing-transient-from-absent   read-safety before a destructive write
│   └── capturing-workaround              record an accepted stopgap and its removal condition
├── 04-test/
│   └── writing-the-failing-test-first    watch it fail for the right reason, then fix
├── 05-deploy/
│   └── running-review-passes             review verdict + must-fix list  (+ references/)
├── 06-maintain/
│   ├── wiki-ingest                       fold a source into the wiki, two-pass verified
│   └── authoring-incident-evals          regression eval wired to the bug page
├── knowledge-base/
│   ├── wiki-query                        the answer surface; used at every stage
│   ├── wiki-lint                         health-check the wiki
│   └── wiki-consolidate                  surface a buried correction into canonical sections
└── agent-config/
    └── maintaining-claude-md             decide whether a rule belongs in CLAUDE.md, a skill, a hook, or the wiki
```

`wiki-ingest` sits under `06-maintain/` because that is its stage in the cycle, while its three siblings in the wiki toolchain sit under `knowledge-base/` because they are not stage-bound. They share the wiki's frontmatter schema and are best edited together.

## How the symlinks work

Consumers link **flat**, from the top level of their own skills directory, straight to the categorised path here:

```
TTMTv2/.claude/skills/wiki-query          -> ../../../aron-skills/knowledge-base/wiki-query
TTMTv2/ttmt-frontend/.claude/skills/wiki-query
                                          -> ../../../../aron-skills/knowledge-base/wiki-query
```

The category folders exist **only in this repo**. A skills-discovery root never sees them, because the link at the discovery root is named for the skill, not the category. That matters: skill discovery reads `<skills-root>/<name>/SKILL.md` and does not reliably descend a further level, so pointing a discovery root at this tree wholesale would surface nothing. Always link the individual skill.

Links are **relative**, matching the convention already used across the TTMT repos. They resolve as long as `aron-skills/` and `TTMTv2/` remain siblings under the same parent — clone a service repo somewhere else on its own and its skill links dangle, which is the same trade the previous sideways-into-`ttmt-wiki` links made.

### Current consumers — 82 links

| Consumer | Links | Depth prefix |
|---|---|---|
| `TTMTv2/.claude/skills/` | 15 | `../../../` |
| `TTMTv2/ttmt-wiki/.claude/skills/` | 15 | `../../../../` |
| `TTMTv2/ttmt-frontend/.claude/skills/` | 13 | `../../../../` |
| `TTMTv2/ttmt-user-service/.claude/skills/` | 13 | `../../../../` |
| `TTMTv2/ttmt-admin/.claude/skills/` | 13 | `../../../../` |
| `TTMTv2/ttmt-landing-page/.claude/skills/` | 13 | `../../../../` |

The four service repos carry 13 rather than 15 — they omit `wiki-lint` and `wiki-consolidate`, which are wiki-maintenance tools with no use from inside a service checkout.

`TTMTv2/.agents/skills/` needs no entry of its own. It already links to `TTMTv2/.claude/skills/`, so it chains through to here. Chained links resolve normally.

## Editing a skill

Edit the file here. Every consumer sees the change immediately — no copying, no sync step.

The symlinks themselves are git-tracked in each consumer repo, so a **new** skill or a **moved** skill produces a commit in each repo that links it. Editing an existing skill's contents does not.

## Adding a skill

1. Create `<category>/<skill-name>/SKILL.md`.
2. Link it from each repo that should see it, at the correct depth:

   ```bash
   # from the repo root of a service repo
   ln -s ../../../../aron-skills/<category>/<skill-name> .claude/skills/<skill-name>
   ```
3. Confirm it resolves: `test -f .claude/skills/<skill-name>/SKILL.md`.

Keep the link name identical to the directory name and to the skill's `name:` frontmatter — that string is what gets invoked.

## Moving a skill between categories

Moving breaks every link that names the old category, and a broken skill link fails silently: the skill simply stops being offered, with no error. After any move, repoint every consumer and then check for dangling links:

```bash
find /Users/alukacs/Code/TTMTv2 -maxdepth 6 -path '*/skills/*' -type l ! -exec test -e {} \; -print
```

Empty output means every link resolves.

## Portability

Seven of the fifteen are generic and travel to any project as-is: `running-the-change-cycle`, `capturing-intent`, `writing-spec-from-intent`, `planning-before-building`, `capturing-workaround`, `maintaining-claude-md`, `running-review-passes`.

The rest are wired to TTMT and need editing before reuse elsewhere:

| Skill | What binds it |
|---|---|
| `wiki-query`, `wiki-ingest`, `wiki-lint`, `wiki-consolidate` | `ttmt-wiki/` paths and its frontmatter schema |
| `authoring-incident-evals` | the wiki's bug pages and `eval_refs` field |
| `writing-the-failing-test-first` | `ttmt-user-service`'s `npm run test:gate` list |
| `distinguishing-transient-from-absent` | Supabase/PostgREST reads and MetaAPI `cancelOrder` |
| `designing-the-user-surface` | TTMT's dashboard surfaces and settings model |

## Provenance

Extracted from `ttmt-wiki/.claude/skills/` on 2026-09-09, byte-for-byte — each of the 15 was verified with `diff -r` against its original before the original was replaced with a symlink.

Not extracted, and still living where they were: `stripe-best-practices` (vendored third-party, in `ttmt-wiki/.agents/skills/`), `ttmt-ui-design-system`, `trace`, `ship-log`, `find-skills`, `content-marketing` (all in `TTMTv2/.claude/skills/` or `TTMTv2/.agents/skills/`), and the five `marketing-*` skills in `ttmt-marketing/.claude/skills/`.
