---
name: authoring-incident-evals
description: Writes a regression eval for a fixed production incident and wires it to the bug page that records it. Use whenever a bug reaches resolution fixed, when ingesting a postmortem, when a wiki bug page has no eval_refs, when a security-scan finding is patched, or when the user asks for a regression eval, an incident eval, or an entry in the eval suite.
---

# Authoring incident evals

An eval is what stops a fixed incident recurring after the fix, the model, or a prompt changes.
The rule this implements:

> "Each production incident gets an eval, written by the team that owned the incident, and stays in
> the suite as a regression test."

**At the 2026-08-30 count, 135 bug pages in this wiki were `resolution: fixed` with no
`eval_refs`** (`npm run lint` reports the live number). Each is a fix with no
guard against its own return. Closing one of those is the most common reason to use this skill.

## Where evals live

`ttmt-wiki/evals/<slug>.json`. One file per incident class, named for the concept it guards,
not the ticket. The suite runs in CI on a schedule **and on any change to `CLAUDE.md`, skills or
hooks** — that configuration steers the agent and deserves the regression testing code gets.

## Deriving the eval from the bug page

A `bug` page already contains everything the eval needs. Map it directly — do not invent scenarios:

| Bug page section | Becomes |
|---|---|
| `## Symptom` | the `query` — the task that reproduces the situation |
| `## Root Cause` | the `trap` — the specific wrong move to detect |
| `## Fix` + `## Detection` | the `expected_behavior` list |
| `## Affected Files` | context to include in the query |

```json
{
  "id": "<slug, matching the bug page>",
  "skills": ["<skill this exercises, if any>"],
  "derived_from": "wiki/bugs/<slug>.md",
  "incident_evidence": "<what actually happened, from the bug page — trade ids, counts, dates>",
  "query": "<the task, phrased so the trap is a natural mistake, never hinted at>",
  "trap": "<the documented wrong move>",
  "expected_behavior": ["<observable behaviour>", "..."]
}
```

**`expected_behavior` entries must be observable in the output.** "Handles errors correctly" is not
scorable; "Destructures both `data` and `error`, not `data` alone" is. If you cannot say what would
prove it, the entry is not ready.

**The query must not leak its own answer.** An eval that names the trap tests reading comprehension,
not judgement. Phrase the task the way the engineer who hit the bug would have phrased it.

**And the harness must not leak it either.** Keeping `trap` and `expected_behavior` in the same file
as `query` is fine — handing the agent that file's *path* is not. Inject the query inline; give the
rubric to the judge only. This suite's first baseline scored a perfect 26/26 because the attempts
read the whole eval file, and a perfect baseline is the shape a leak takes: it argues the skill is
unnecessary. Carry a `contaminated` flag on the judge's verdict so a leak surfaces in the results.

## For a bug fix, the failing test comes first

The unit test guards the code; this eval guards the behaviour against a model or prompt change. The
test-side discipline — fail for the right reason before the fix, never weaken the check, gate-list
membership — is `writing-the-failing-test-first`, not restated here.

## Wire it back

Add the eval path to the bug page's `eval_refs` frontmatter. That is the thread between the incident
record and the guard, and it is what the `resolution: fixed but no eval_refs` lint warning checks:

```yaml
resolution: fixed
eval_refs: ["evals/order-fill-watchdog.json"]
```

Editing a wiki page goes through `/wiki-ingest`, or through the consolidation path in `schema.md`
when the eval is the only change.

## Checklist

- [ ] `query` reproduces the situation without naming the trap
- [ ] every `expected_behavior` is observable in the output
- [ ] `incident_evidence` cites what actually happened, from the bug page
- [ ] `derived_from` points at the bug page
- [ ] `eval_refs` added to that page
- [ ] a baseline was measured — an eval nothing ever failed proves nothing
