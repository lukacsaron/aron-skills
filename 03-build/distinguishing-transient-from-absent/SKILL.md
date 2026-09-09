---
name: distinguishing-transient-from-absent
description: Applies TTMT's read-safety rule before any destructive write that depends on a row being gone. Use whenever code reads Supabase or PostgREST and then deletes, cancels, marks failed, overwrites a JSONB array, or flips a terminal status based on the row being missing — and whenever handling a MetaAPI NotFoundError from cancelOrder.
---

# Distinguishing a transient read from confirmed absence

A read that comes back empty is not proof the row is gone. In TTMT this has produced the same class
of production bug **7+ times** — redistribution-retry, breakeven-retry, set-tps-handler,
trade-modifier, set-breakeven-handler, close-trade-handler, order-actions.

Canonical page, which is the source of truth for this rule and carries every documented instance:
`ttmt-wiki/wiki/concepts/transient-vs-confirmed-absent.md`.

## The rule

**Only `PGRST116` confirms absence. Everything else is transient.**

```ts
const { data, error } = await supabase
  .from('trades').select('id')
  .eq('id', tradeId)
  .eq('user_id', userId)        // never omit — see below
  .is('deleted_at', null)       // never omit — see below
  .single();                    // .single() so absence arrives as PGRST116

if (error) {
  if (error.code !== 'PGRST116') return;   // transient: PGRST301, 42501, network, timeout
  // PGRST116 — PostgREST confirmed zero rows. Absence is established.
} else if (data) {
  return;                                   // present
} else {
  return;                                   // {data:null, error:null} — NOT absence. See below.
}

await supabase.from('redistribution_retries').update({ status: 'failed', /* ... */ });
```

## The three mistakes this exists to prevent

Measured on a no-skill baseline over this exact task (`ttmt-wiki/evals/transient-vs-confirmed-absent.json`),
each of these was made by a competent attempt:

**1. Inferring absence from `data === null` instead of confirming it with `PGRST116`.** *(missed in
2 of 2 trials)* Reaching for `.maybeSingle()` to avoid "string-matching an error code" feels cleaner
and removes the only positive confirmation you have. `.maybeSingle()` collapses *confirmed empty*
and *`{data:null, error:null}`* into one branch — and the second is the pool-saturation signature
that produced the `trd_6jt22p` cascade. Prefer `.single()` here and branch on the code.

**2. Treating `{data: null, error: null}` as absence.** *(1 of 2 trials)* PostgREST returns it under
connection-pool saturation. It is the one case that survives even after you correctly throw on
`error`, because there is no error to throw on. It must have its own branch, and that branch must
not write.

**3. Dropping the scope filters.** *(missed in 2 of 2 trials)* `.eq('user_id', userId)` and
`.is('deleted_at', null)` are part of this pattern, not optional extras — the 2026-05-20 cascade fix
added both as defence-in-depth on the user-isolation and soft-delete axes. Without
`.is('deleted_at', null)` a soft-deleted trade reads as present and the retry never terminates.

> **Do not weaken the check to make the read simpler.** Reason: every simplification here trades a
> confirmed signal for an inferred one, and the inference fails exactly when the database is under
> load — which is when destructive writes are most expensive. Instead: keep the positive `PGRST116`
> confirmation and let the transient branch do nothing, so the next tick re-evaluates.

## The MetaAPI variant

`connection.cancelOrder()` throwing `NotFoundError` may mean **the broker just filled this order**,
not that it cancelled cleanly. Writing `status='canceled', cancel_reason='*_already_gone'` from that
catch loses the resulting position. Disambiguate against the position book before recording a
cancel.

## The plural form

A LIST read whose failure collapses to an empty set — `const rows = data ?? []` — before a terminal
decision has the same defect and **no `PGRST116` to lean on**, because zero rows is a legitimate
result. There, `error` is the only discriminator: if it is set, do not treat the empty list as
authoritative.

## The deterministic half

`ttmt-frontend` ships `scripts/check-unchecked-supabase-writes.ts` (repo-relative), which bans Supabase mutations
whose result is thrown away. Its own header records why: *"Every one of the seven Postgres error
classes triaged on 2026-08-19 survived in production for the same reason: a write whose `error` was
never read."* This skill is advisory; that script is what would make the rule hold.

**Before relying on it, check it is armed:** `grep -r check:unchecked-supabase-writes .husky
.github package.json`. At the 2026-08-30 audit it was wired to nothing — defined in `package.json`
and referenced by no hook and no workflow, while `ci.yml` and husky ran the repo's other two guards.
If that is still true, arming it is one line in the husky hooks and one CI step, and doing so is
worth more than any review comment this skill will ever generate.

> **Do not treat a written-but-unwired guard as coverage.** Reason: a check that no hook and no CI
> job invokes has exactly the force of a comment, and its existence makes everyone believe the class
> of bug is handled. Instead: grep for the script name before relying on it, and wire it where it is
> missing.
