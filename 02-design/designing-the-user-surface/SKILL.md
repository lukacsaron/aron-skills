---
name: designing-the-user-surface
description: Applies TTMT's UX discipline to anything a user sees — research proportionate to risk, a named state for every branch, honest data display, and settings that land where someone would look for them. Use when adding or changing a screen, a dashboard, a chart or metric, a setting or toggle, onboarding, an empty state, notification or UI copy, and before writing the spec for any user-facing change.
---

# Designing the user surface

A user-facing change is specified, not improvised. The failure this exists to prevent is a screen that
is *correct* — right data, right permissions, no crash — and unusable, because nobody decided what the
user is trying to do, what they see while it loads, or where they would go looking for the setting.

**Research before pixels; states before styling; the data's honesty before its beauty.**

## 1. Start from the job, not the request

The request names a solution; the job names the goal. "Add a filter" is a request. "I need to know
which channel loses me money on my prop account" is the job — and a filter may not be the answer.

Write the job as one sentence with a *when*: **when** situation, **I want to** motivation, **so I can**
outcome. If you cannot write it, you do not yet know what to build, and step 2 is how you find out.

> **Do not design the solution the requester named.** Reason: a requester describes the fix they can
> picture, which is bounded by the interface they already know — build it and you inherit their
> workaround as your architecture. Instead: capture the job, then choose the surface.
> [`capturing-intent`](../capturing-intent/SKILL.md) writes this down properly.

## 2. Research proportionate to risk

Not every change earns a study. Pick the cheapest rung that answers the question, and go up a rung when
the change is irreversible, on a path everyone traverses, or about money.

| Rung | Cost | Use when | What it yields |
|---|---|---|---|
| **Read the data you already have** | minutes | Always first | TTMT ships 338 PostHog event constants (`ttmt-frontend/src/lib/analytics/events.ts`). Funnel drop-off, feature usage, and where people stop are already recorded. |
| **Heuristic evaluation** | ~1 hour | Any redesign of an existing screen | Walk the flow against Nielsen's 10, rate each finding 0–4. **3–5 evaluators** find materially more than one. |
| **Usability test** | days | New flow, money path, or anything you are guessing about | **5–8 participants per segment**, **3–5 research questions**, **5–8 task scenarios** with success criteria. Metrics: task success rate, time-on-task, error rate, SUS or SEQ. |

**Nielsen's 10, as a checklist:** visibility of system status · match to the real world · user control and
freedom · consistency and standards · error prevention · recognition over recall · flexibility and
efficiency · aesthetic and minimalist design · error recovery · help and documentation.

**Severity 0–4:** 0 not a problem · 1 cosmetic · 2 minor · 3 **major, important to fix** ·
4 **catastrophe, must fix before release**. A 3 or 4 goes on the spec's must-fix list, not a backlog.

*(Method parameters above follow the design-research and prototyping-testing conventions published in
`Owl-Listener/designer-skills`; the participant and evaluator counts are that repo's, not measured here.)*

> **Do not open a study when the answer is already in PostHog.** Reason: research spend is finite, and a
> study that re-derives a number you already log delays the change while teaching you nothing. Instead:
> query the funnel first, and use the study for *why*, which telemetry cannot answer.

## 3. The cognitive budget

Each of these is a constraint you can check a design against, not a slogan.

| Law | The constraint | Where it bites in TTMT |
|---|---|---|
| **Hick's** | Decision time grows with the number of simultaneous choices | Settings pages with every knob visible at once |
| **Miller's** | Chunk into groups of about **four** | Long settings columns; TP/layer configuration |
| **Fitts's** | Acquisition time depends on target size and distance | Destructive actions placed next to routine ones |
| **Jakob's** | Users expect this to behave like the tools they already use | Trading UI has strong conventions — deviate deliberately or not at all |
| **Doherty** | Keep response under **400 ms** to preserve flow | Anything reading positions, trades, or the broker |
| **Serial position** | First and last items are recalled best | Menu and list ordering is a decision, not an accident |
| **Peak-end** | An experience is remembered by its most intense moment and its end | Onboarding completion, a halt notification, a closed trade |
| **Tesler's** | Complexity is conserved — the product absorbs it or the user does | Every "let the user configure it" is a transfer, and it should be a choice |
| **Zeigarnik** | Incomplete tasks stay mentally active | Progress indicators, saved drafts, resumable onboarding |

Two apply hardest here. **Tesler's**, because TTMT's settings surface is large and every new toggle moves
complexity onto the user. And **Doherty**, because a 400 ms budget is a real constraint on a page that
reads live broker state — if you cannot meet it, that is a loading state you must design, not a
performance problem you can defer.

## 4. Every state gets named

The most common defect in a TTMT UI change is a state nobody decided. Enumerate all of them in the spec:

- **Empty** — never yet used. TTMT's house tone is *"cute and funny"*, friendly, avoiding dead space.
- **Loading** — including the >400 ms case, which needs a skeleton rather than a spinner.
- **Partial** — some accounts loaded, some failed. Common here and routinely forgotten.
- **Error** — what broke, and the next action. Never an apology, never a raw code.
- **Stale** — data arrived but is old. TTMT has price staleness everywhere; say so in the UI.
- **Permission / entitlement** — not allowed, or not on this plan.
- **Success** — and what the user does next.

> **Do not ship a state you did not name in the spec.** Reason: unnamed states get whatever the
> component does by default — usually a blank region or a spinner that never resolves — and it surfaces
> to the user, not to a test. Instead: list every state in the spec's surface section and give each one
> its copy.

## 5. Showing data

TTMT is a trading platform. A chart that is merely pretty and quietly misleading is a defect, and the
platform already has a written doctrine for this — the read-only MCP server's honesty norms. Hold UI to
the same bar:

- **Attach the sample size.** Report the trade count behind any conclusion, and hedge when it is small.
  A handful of trades is an anecdote, not a trend.
- **Never convert mixed currencies.** Per-account buckets are the honest view; a blended total is a
  fabricated number.
- **Realized history is not a backtest.** Label it, and frame any suggestion as advisory.
- **Compare on Avg R and win rate with n attached**, not raw P&L — raw P&L mixes position sizing into
  the comparison.
- **Surface the caveat flags.** When the data carries `notes`, `truncated` or `mixed_currency`, show it
  rather than silently dropping it.

Chart craft — mark selection, encodings, axes, categorical palettes, accessible colour — is covered by
the `dataviz` skill. Load it rather than improvising a palette.

**A display that must not drift gets a contract.** The precedent is
`wiki/concepts/traders-in-profit-display-contract.md`: a shared helper, a snapshot fixture and a CI check
in both repos, so one surface cannot quietly disagree with another.

> **Do not round, blend or extrapolate a number to make a card look tidy.** Reason: users make position-
> sizing decisions from these figures, so a tidied number is a wrong number with the authority of a UI.
> Instead: show the real value with its unit and its sample size, and design the layout around the data
> rather than the reverse.

## 6. Adding a setting

A setting is the most expensive thing you can add to a UI: it is permanent, it multiplies test states,
and by Tesler's Law it moves complexity onto the user. Answer all six before adding one.

1. **Does it need to exist?** A setting is a decision you are declining to make. If one value is right
   for almost everyone, ship that value. A setting nobody changes is pure cost.
2. **Where would someone look for it?** Place it by the task it affects, not by the module that
   implements it. Group by proximity — spatial closeness groups elements more strongly than any other
   cue — and keep groups near four items.
3. **What is it called?** Name it for the user's outcome, not the internal flag. TTMT precedent: the
   `sl_anchor_mode` setting is labelled *"Count the stop loss from"* — live copy in
   `EntryConfigSection.tsx`, `ProfileEditPage.tsx` and `ImportProfileClient.tsx`, not the flag name.
4. **What is its default, and its scope?** TTMT settings are per-account with a template row
   (`user_trading_settings` where `metaapi_account_id IS NULL`) and an optional per-profile override.
   Say explicitly which layer this lives on — the inheritance is invisible in the UI unless you design
   it. `OverridableSelectField` and `RadioCardGroup`
   (`ttmt-frontend/src/components/dashboard/settings/profiles/shared/OverridableFields.tsx`) are the
   existing components for exactly this.
5. **What happens when it cannot apply?** TTMT's convention is **visible but disabled, with the reason
   shown** — a knob that vanishes teaches the user it never existed.
6. **Is it reversible, and does the user know?** If flipping it changes live orders or open positions,
   say so at the point of change, not in a doc.

> **Do not add a setting to avoid a decision.** Reason: every toggle doubles the states the UI must be
> correct in and hands the judgement to someone with less context than you — and TTMT's settings surface
> is already large enough that a new knob is likelier to be misconfigured than used. Instead: pick a
> default and defend it; add the setting when a real user with a real account needs the other value.

## The design system is not optional

`ttmt-ui-design-system` covers colour, typography, icons, spacing, both component catalogues,
dark/light, motion, forms, empty states, modals, charts, accessibility and responsive rules for
`ttmt-frontend` and `ttmt-landing-page`. Load it before writing components. Semantic classes
(`text-page-title`, `text-body`, `icon-inline`) exist so hierarchy is consistent — use them rather than
ad-hoc sizes.

## Where this lands

[`writing-spec-from-intent`](../writing-spec-from-intent/SKILL.md) has a surface section that consumes
this: the job, the states, the data contract and the settings placement all belong in the spec, before
implementation. `running-review-passes` checks the built diff against it.
