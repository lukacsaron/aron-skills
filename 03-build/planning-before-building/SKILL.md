---
name: planning-before-building
description: Writes a TTMT implementation plan as ordered tasks with per-task verification, the kill-switch reader first on gated changes, and the wiki source document as the closing task. Use before writing code for any non-trivial change, when the user asks to plan the work or for an implementation plan, when leaving plan mode, and whenever implementation departs from a plan already committed.
---

# Planning before building

> "Nothing is implemented without an accepted plan."

Plan mode is the starting point: read the codebase, change nothing, produce a plan the engineer
accepts. The plan is then **committed** — plan mode produces a conversation, and a conversation is
not something the review pass can check a diff against.

The acceptance test is not "does this look reasonable":

> iterate "until an engineer who has never seen the conversation could implement the change from the
> plan alone."

## Where it goes

`docs/plans/YYYY-MM-DD-<slug>.md`, committed **in the same commit as its spec**, before the first
feature commit: `docs(<scope>): design spec + implementation plan`. In `ttmt-admin`, and for anything
run through the superpowers workflow, that is `docs/superpowers/plans/` — follow the repo.

## The shape TTMT uses

```markdown
# <Title> — Implementation Plan

## Global Constraints
What must hold across every task: the invariants from spec §3, the flag's default, what may not
change, which existing assertions are expected to flip and when.

### Task 1: <the kill-switch reader>
### Task 2: <extract / retarget, if legacy code must be moved before it is changed>
### Task 3: <the new behaviour (TDD)>
### Task 4: <wire it in behind the gate>
### Task 5: <surfacing — trace events, PostHog, notification copy>
### Task 6: <env-var documentation rows>
### Task 7: <sibling-repo copy, if the UI describes this behaviour>
### Task 8: <public docs + MCP content — REQUIRED when the change adds a feature or alters one>
### Task 9: <wiki source document>

## Rollout
Manual ops after all tasks are green, per spec §9. **These are the operator's steps, not
the agent's** — ordered, each with its precondition and how to back it out. A migration apply, a
deploy, a fleet roll and a production flag flip all live here and are performed by a human. Writing
them down is the deliverable; running them is not.

## Self-Review (performed at plan time)
The seams you already know are weak, before anyone writes code.
```

Each task names **the files it changes** and **how it is verified** — the suite, the command, the
observable. The playbook's four headings map onto this: *Files that change* and *Order of work* are
the tasks; *Risks* is Global Constraints and Self-Review; *Proof* is the per-task verification plus
Rollout.

## The three orderings that are not negotiable

1. **When the spec carries a kill-switch contract, its reader is Task 1.** Every user-service
   behavioural change ships behind an env gate whose default and exact parsing are fixed in spec §5,
   with the disabled path byte-identical to the legacy one, and gates are common in frontend plans
   too. Writing the gate first means every later task has somewhere safe to land. A change with no
   behavioural risk to gate (UI copy, a column) skips this — deliberately, in the plan's words, not
   by omission.
2. **Extract before you change.** When legacy behaviour must move, extract it *verbatim* in its own
   task and retarget its tests at the real code first. A test that mirrors an implementation instead
   of testing it passes whatever you do next — TTMT has hit this and named it.
3. **The wiki source document is the last task, not an afterthought.** The change ends by writing a
   source into `ttmt-wiki/sources/`, which `/wiki-ingest` turns into pages. Skip it and the next
   incident starts from zero.
4. **The public docs and the MCP content ship in the same plan as the feature.** If the change adds a
   user-visible feature, or changes the behaviour, default, range, or dashboard location of an
   existing one, the plan carries a task that updates both in the same cycle — the `ttmt-docs` page(s)
   that describe it, and every surface in `ttmt-frontend` that describes it to the MCP: the settings
   registry row(s) in `src/lib/settings-registry/registry/*.ts` (a new settings key without a row fails
   the INV-1 guard test and `tsc`; a changed dashboard label fails the anchors parity test), the
   re-exported `settings-registry.json` (on a clean tree, in its own commit) and the regenerated
   `ttmt-docs` `reference/settings-contract` page, `src/lib/mcp/static/glossary.md`, the tool description
   and output schema of any tool whose response gains or changes a field, and — when a tool or resource
   is added or removed — the server instructions, `docs/api/mcp-server.md` and the dashboard's Connectors
   panel (`McpPanel.tsx`, which states the tool and resource counts). Its verification line names the
   docs page and each MCP surface that changed. A change that touches none of
   those (an internal refactor, a fix that restores documented behaviour) writes the skip and its reason
   in the task's place, in the plan's words. The wiki and the frontend code are the canonical sources;
   docs and MCP content are copies of them, and a copy that is updated "later" is the drift class behind
   `bugs/settings-snapshot-drawer-incomplete` (39 of 89 settings shown) and
   `bugs/profile-import-settings-key-drift` (59 of 67 keys recognised).

## Keeping the plan true

> "When implementation departs from the plan, update plan.md in the same commit. Consider using a
> hook to enforce synchronization between the two."

`running-review-passes` checks the diff against the plan, so a stale plan converts every honest
deviation into a silent one. Amend the plan in the commit that departs from it — not afterwards.

## Boundaries

> **Do not edit files while planning.** Reason: an agent that has already started implementing writes
> the plan that justifies what it did, and the engineer's acceptance stops being a decision.
> Instead: read only until the plan is accepted.

> **Do not restate the spec as a plan.** Reason: a plan whose tasks are the spec's sections renamed
> carries no file paths and no verification, so it fails the one acceptance test that matters —
> someone outside the conversation implementing from it. Instead: name real paths, in a real order,
> from code you actually opened.

> **Do not leave a task without a verification line.** Reason: a task whose "done" is the author's
> judgement cannot be checked by the next session or by review, and it is where scope quietly grows.
> Instead: name the suite, the command, or the trace event that shows the task landed.

> **Do not write a verification line that applies a migration or deploys anything.** Reason:
> `npx supabase migration up`, `db push`, a service deploy and a backfill are not proofs that a task
> landed — they are irreversible operator actions, and putting one in a verification line makes it
> happen because the checklist demanded it rather than because a human chose it. `migration up` also
> applies every other pending migration in the tree, so the plan quietly adopts another session's
> unfinished work. Instead: verify a migration by reading it against the live schema and dry-running
> it in a transaction that rolls back (`BEGIN; \i <file>; … ROLLBACK;`), then confirm nothing
> persisted — and put the actual apply in **Rollout**, where it belongs.

## Linkage back to this wiki

`plan_ref: "ttmt-user-service:docs/plans/YYYY-MM-DD-<slug>.md@<sha>"`. See `schema.md` → "SDLC
artifacts".
