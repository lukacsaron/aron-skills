---
name: running-the-change-cycle
description: Conducts a TTMT change from raw idea to shipped and documented — sizing the work, agreeing the resources it deserves, stopping at the points a human must decide, and invoking the stage skills in order. Use when starting a piece of work end to end, when the user asks to run the process or the full cycle, when a change needs orchestrating across stages, or when it is unclear which skill comes next.
---

# Running the change cycle

Fifteen skills cover the stages. None of them says what order, what must be true before the next one
starts, or who has to answer a question before you may continue. That is this skill's only job.

**Dispatch, never describe.** Each stage below names the skill that owns it and what that stage must
hand over. It does not restate what those skills say — two copies of a method drift, and the stage
skill is the source of truth. If you find yourself explaining *how* to write a spec here, stop and
invoke `writing-spec-from-intent` instead.

## Step 0 — Size the work, before anything else

Ask the operator, with `AskUserQuestion`. Do not infer the tier from how the request was phrased; a
one-line request can be a Tier 3 change.

| Tier | What it is | What the cycle runs |
|---|---|---|
| **0 · Trivial** | Copy fix, a column, a doc. No behaviour change, instantly reversible. | Build → Review. No spec, no plan — say so out loud rather than skipping silently. |
| **1 · Contained** | One repo, no money or orders, reversible by a redeploy. | Light spec (header + §1/§3/§4b/§10) → plan → build → review → wiki source. |
| **2 · Behavioural** | Moves money or orders, or sits on a path every user traverses. | Everything, kill-switch mandatory, canary before fleet. |
| **3 · Irreversible** | Schema/migration, fleet roll, anything you cannot take back. | Tier 2 plus explicit human sign-off at each gate. The cycle **writes** the migration; it never applies it — see *The cycle does not touch a database* below. |

Two questions decide the tier, and both are about consequence, not size:
**can this move money or orders?** and **can we take it back in one deploy?**

> **Do not let the tier be set by the repo the change lands in.** Reason: "it's only frontend" has been
> wrong on this platform — a gate in front of every signup is higher-consequence than a user-service
> flag nobody has flipped. Instead: tier by blast radius and reversibility, and let a UI change be
> Tier 2 when it is one.

## Step 0b — Agree the resources

Ask once, up front, with `AskUserQuestion`. Getting this wrong is expensive in both directions: a
fleet of agents on a Tier 1 change wastes the budget, and a solo pass on a Tier 3 change misses things.

**Model.** Default is the session model — say so and move on. Only raise the question when the tier
justifies it:

- **Hard design or planning** (Tier 2–3, a wide solution space) → Fable is worth asking for.
- **Integration-heavy implementation** (many call sites, cross-service) → Opus.
- **Implementers and reviewers** → Sonnet is the floor. **Never Haiku for implementation.**
- **Never Fable inside a Workflow.** Fable is for hard planning and for an explicitly-assigned
  riskiest implementer or reviewer, not for fan-out.

**Parallelism.** Three options, and the third needs permission:

- **Solo session** — the default, and correct for Tier 0–1.
- **Subagents** — independent, read-heavy work: a caller inventory, a multi-service counter-pattern
  sweep, three independent review lenses.
- **A Workflow** — only when the operator has explicitly opted in. Never infer it from the work
  looking parallel. If it would help, say what it would cost and ask.

**Isolation.** A worktree when agents mutate files in parallel, or when the change must not disturb a
tree another session is working in. Frontend worktrees go **outside** the repo — `TTMTv2/wt-fe/<name>` —
because `.wt/` inside it collects zero tests.

**People.** Name the owner for §11 flagged concerns and the approver for the plan and the merge,
*before* you reach those gates. A gate with no named human is a gate that gets skipped.

## The stages

Each line: what runs, which skill owns it, and **what must exist before the next stage may start.**

| # | Stage | Invoke | Hands over |
|---|---|---|---|
| 1 | Plan | `capturing-intent` | A proposal page (or a linked repo intent), with open questions still open |
| 2 | Design — surface | `designing-the-user-surface` | The job, every named state, the data contract, settings placement — *only if it renders* |
| 2 | Design — spec | `writing-spec-from-intent` | A committed spec: `INV-n` that can fail, kill-switch contract, dated caller inventory, §11 with owners |
| 3 | Build — plan | `planning-before-building` | A committed plan, ordered tasks, each with a verification line — including the public-docs + MCP-content task whenever the change adds a feature or alters an existing one |
| 3 | Build — code | `distinguishing-transient-from-absent`, `ttmt-ui-design-system`, `capturing-workaround` as they apply | The diff, matching the plan |
| 4 | Test | `writing-the-failing-test-first` | A test that failed before the fix, and gate-list membership |
| 5 | Deploy | `running-review-passes` | A committed final-review with a verdict and a must-fix list |
| 6 | Maintain | `wiki-ingest`, then `authoring-incident-evals` for a bug | Pages a future `/wiki-query` can find, `eval_refs` on the bug page, and the stage-3 docs + MCP task closed rather than deferred |

Stage 2's two halves run in that order: the surface decides what the spec has to specify.

## The cycle does not touch a database

A migration is a **deliverable of this cycle, not an action within it.** Stage 3 writes the `.sql`
file, stage 5 reviews it, and stage 6 documents it. Nothing here applies it — not to production, not
to staging, and not to local.

> **Do not apply a migration as part of building or verifying a change.** Reason: applying it is a
> separate decision with a separate blast radius, and folding it into the dev loop means it happens
> as a side effect of "the task isn't done yet" rather than because someone chose it. `migration up`
> also applies every *other* pending migration in the tree, so in a multi-session repo the cycle
> silently adopts work it did not author. Instead: hand the operator the migration and let them run
> `.agent/workflows/db-migration.md` when they choose to.

This changes what a migration task's **verification line** may say. "Run `supabase migration up`" is
not a verification — it is an unrequested deployment. Verify a migration by reading it against the
live schema and, where it earns the effort, dry-running it inside a transaction that rolls back:

```bash
psql "<local-db-url>" -v ON_ERROR_STOP=1 <<'SQL'
BEGIN;
\i supabase/migrations/<file>.sql
-- assert here: query the changed table/view, check reloptions survived, etc.
ROLLBACK;
SQL
```

Then confirm nothing persisted. That proves the SQL is correct without deploying it.

The same rule covers every other irreversible operator action the cycle can describe but must not
perform: deploying a service, rolling the fleet, flipping a production kill-switch, running a
backfill. Spec §9 and the plan's Rollout section **document** these as ordered operator steps with
their preconditions. They are not a to-do list the agent works through.


## The four hard stops

Everywhere else you may proceed on judgement. At these four you stop and wait for a human.

1. **After Step 0/0b** — tier and resources confirmed. Everything downstream is scaled by this.
2. **Spec §11 flagged concerns** — each needs a named owner and a decision. *"A concern nobody owns is
   the one that becomes an incident."* You may not resolve one yourself, even when you could.
3. **Plan acceptance** — *"nothing is implemented without an accepted plan."* Not "does this look
   reasonable" — could someone who never saw the conversation implement from it?
4. **Merge** — review reports and gives a verdict; a code owner approves. The agent that wrote the
   code never approves it.

> **Do not carry a stop over because the answer seems obvious.** Reason: the four stops are exactly the
> points where an agent's context is thinnest — it cannot know who owns a pricing decision, what the
> operator meant to cut, or what else is mid-flight in the tree. Instead: ask, and let a short wait cost
> less than a wrong assumption.

## Running it

State the tier and the resource plan back before starting, in one short block, so the operator can
correct it cheaply. Then work the stages, announcing each skill as you invoke it.

**Resuming mid-cycle is normal.** Most real work enters at stage 2 or 3 with intent already settled.
Say which stage you are entering and what you are taking as given — an unstated assumption about a
skipped stage is how a cycle silently loses its spec.

**Skipping is allowed; skipping silently is not.** Tier 0 legitimately skips spec and plan. Write the
skip and its reason where the artifact would have been.

> **Do not run this as a checklist to completion.** Reason: the value is in the gates, not the
> sequence — an orchestration that races to stage 6 with four unanswered questions has produced
> artifacts nobody agreed to, and the rework costs more than the cycle saved. Instead: treat each stop
> as the point of the stage, and let the operator's answers change the plan.

## What this skill is not

It is not a replacement for judgement about *which* skills apply — the stage skills carry their own
triggers and will fire on their own. Its contribution is order, gates, sizing and resourcing.

It is also **not measured.** Like most of the stage skills it was written from the repos rather than
from a baseline. Treat its tier boundaries as a starting proposal to argue with, not a finding.
