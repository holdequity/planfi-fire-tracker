---
name: fire-tracker
version: 1.0.0
description: Track FIRE progress by orchestrating the public planfi MCP. Use whenever someone wants to know whether they're already financially independent or can coast, how their savings/net worth benchmark against peers, which net-worth milestones they've hit, or whether they can retire by a target age and which levers get them there — e.g. "am I already financially independent?", "can I coast to retirement?", "how do I compare to other people my age?", "what milestones have I hit?", "can I retire by 50?".
---

# FIRE Tracker

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
All the math — the already-won surplus/cushion test, peer benchmarking, milestone thresholds, and
the goal solver's lever ranking — lives server-side. This skill only gathers inputs and calls the
tools — it does **not** compute anything locally and bakes in no defaults of its own. Read-only.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_already_won`):
`analyze_already_won`, `analyze_fire_benchmark`, `analyze_milestones`, `solve_goal`, plus optional
`generate_financial_plan` (for `plan_id` chaining + a `share_url`). Use whichever name your
environment exposes (bare or `mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp — no auth.)

## Step 1 — (Optional but preferred) build a plan first to chain context + get a share link

If the user has (or wants) a full household model, call **`generate_financial_plan`** once and
**capture the returned `plan_id`**. **Three of the four** tools — `analyze_already_won`,
`analyze_milestones`, and `solve_goal` — accept `{ plan_id }` (plus an `overrides` object), so they
can resolve the household from the saved plan instead of you re-sending every figure.
`generate_financial_plan` returns a **`share_url`** (planfi.app). `analyze_already_won` also returns
a `share_url` whenever you pass a `plan_id` that resolves, and `solve_goal` returns a `share_url` on
every call (it always builds one from the resolved plan).

> **`analyze_fire_benchmark` is the exception: it takes raw inputs ONLY (NO `plan_id`).** Pass its
> figures (`annual_income`, `annual_savings`, `liquid_net_worth`) directly — do not try to chain a
> `plan_id` into it.

This step is optional: every tool below runs from raw inputs too. Prefer the plan path when the
session already has a model, when the user is asking more than one of these questions, or when they
want a sharable artifact.

> **Engine facts to bake in:** all rates/decimals are **fractions** (a 40% savings rate → `0.40`);
> all dollars are **today's (real) dollars**; any bracket/threshold figures are approximate **~2026**
> values.

## Step 2 — Route by intent

Pick the tool that matches what the user is asking. Pass `{ plan_id }` when you have one (except for
`analyze_fire_benchmark`); the raw fields below are still REQUIRED where noted.

### "Am I already financially independent? Can I coast?" → `analyze_already_won`
Returns whether the portfolio already covers spending — `is_over_funded`, `fire_pct`, `surplus`,
`cushion_years`, `current_safe_spend`, `one_more_year_safe_spend`, and
`marginal_safe_spend_per_extra_year`.
REQUIRED: `liquid_net_worth`, `annual_spend`. Optional: `safe_withdrawal_rate` (default 0.04),
`fire_target_net_worth` (else derived from `annual_spend / safe_withdrawal_rate`),
`annual_contributions` (default 0), `real_growth_rate` (default 0.05). Accepts `plan_id` (+ an
`overrides` object, + `name` for the share link).

```
analyze_already_won({ liquid_net_worth: 1600000, annual_spend: 60000 })
```

### "How do I compare to peers?" → `analyze_fire_benchmark`  *(raw inputs only — NO plan_id)*
Returns a `savings_percentile_band`, `cohort_framing`, `savings_rate`, `years_to_fi`, and
`net_worth_to_income_multiple` versus a peer cohort.
REQUIRED: `annual_income`, `annual_savings`, `liquid_net_worth`. Optional: `safe_withdrawal_rate`
(default 0.04), `real_return` (default 0.05). **There is NO `plan_id`, `savings_rate`, `net_worth`,
`income`, or `age` input** — savings rate and the percentile band are computed server-side from the
three required figures.

```
analyze_fire_benchmark({ annual_income: 140000, annual_savings: 49000, liquid_net_worth: 450000 })
```

### "What milestones have I hit?" → `analyze_milestones`
Returns the net-worth milestones crossed in `newly_reached[]` (each `{ type, label, headline,
net_worth }`).
REQUIRED: `net_worth`. Optional: `coast_pct` (enables the Coast FIRE milestone), `fire_pct` (enables
the FIRE milestone), `age` (stamped on headlines), `date` (ISO; defaults today), `already_logged[]`
(milestone types not to re-detect). Accepts `plan_id` (+ `overrides`) for context.

```
analyze_milestones({ net_worth: 450000 })
```

### "Can I retire by age X?" → `solve_goal`
Solves for a target FIRE age and ranks the levers (savings rate, retire age, spend, allocation) that
close the gap, **easiest-first**.
REQUIRED: `target_fire_age` (20–100), plus a plan — either a full inline plan (`earners[]`, `stocks`,
…) OR a `plan_id`. Optional: `levers[]` (`savings_rate` | `retire_age` | `spend` | `allocation`),
`overrides`, `max_iterations`, `max_engine_calls`, `time_budget_ms`. Always returns a `share_url`.

```
solve_goal({ target_fire_age: 50, plan_id: 'pl_abc123' })
```

## Step 3 — Surface results honestly

For whichever tool you called:
- **Lead with the headline** — the already-won verdict (`is_over_funded`) + `surplus` /
  `cushion_years` / `current_safe_spend`; the `savings_percentile_band` + `cohort_framing`; the
  `newly_reached[]` milestones; or the ranked, easiest-first `ranked_levers` `solve_goal` returns to
  hit the target age (plus `feasible` / `already_achieved` — whether the target is reachable at all).
- **Read back `assumed_defaults[]`** — every one of these tools returns a structured
  `assumed_defaults[]` array of `{ field, assumed_value, note }` listing each input you omitted and
  the value the engine assumed (e.g. `safe_withdrawal_rate` 0.04, `real_growth_rate`/`real_return`
  0.05, `annual_contributions` 0). Surface each so the user can correct any silent default.
  `disclosures.key_assumptions` is SEPARATE static prose (a short list of method notes), and
  `disclosures.not_advice` is a BOOLEAN, not a message — honor it by presenting results as planning
  estimates, not financial advice.
- **Follow `next_actions[]`** — each is `{ tool, why, prefilled_args }` (carrying `{ plan_id }` when
  available). Only some tools emit edges: `solve_goal` → `generate_action_plan`;
  `generate_financial_plan` → `generate_financial_insights`, `check_model_completeness`, and
  commentary tools. `analyze_already_won`, `analyze_milestones`, and `analyze_fire_benchmark`
  currently return no `next_actions` edges, so chain by intent yourself there.
- **For a share link:** `solve_goal` always returns a `share_url`; `analyze_already_won` returns one
  when given a resolving `plan_id`; `generate_financial_plan` always returns one. `analyze_milestones`
  and `analyze_fire_benchmark` do not — run `generate_financial_plan` (Step 1) for a sharable plan.

## Recommended call sequence (typical session)

1. (preferred) `generate_financial_plan` → capture `plan_id` (+ `share_url`) — note
   `analyze_fire_benchmark` won't use the `plan_id`.
2. Route by intent → one of the four tools (with `{ plan_id }` where accepted, plus the REQUIRED raw
   fields).
3. Read back the headline + the tool's `assumed_defaults[]`.
4. Follow any `next_actions[]` the tool returns (`solve_goal` suggests `generate_action_plan`); for
   tools that emit none, chain by intent yourself, or run `generate_financial_plan` for a share link.

## Fictional examples

**1.** *"I'm 45 with $1.6M invested and spend $60k a year — am I already financially independent?"*
→ `analyze_already_won({ liquid_net_worth: 1600000, annual_spend: 60000 })`. Lead with the verdict
(won / not yet via `is_over_funded`), the `surplus`, `cushion_years`, and `current_safe_spend`; if not
yet there, chain `solve_goal` to show the smallest lever to close the gap. Read back
`assumed_defaults[]` (the `safe_withdrawal_rate` / `real_growth_rate` assumptions that drive the
verdict).

**2.** *"Can I retire by 50 if I'm 38 with $400k invested and saving $3k/mo?"*
→ build a quick plan with `generate_financial_plan` (capturing `plan_id` + `share_url`), then
`solve_goal({ target_fire_age: 50, plan_id })`. Lead with whether age 50 is reachable (`feasible`) and
the easiest-first `ranked_levers` (e.g. bump savings rate, trim spend, push retire age) it returns;
surface the `share_url` `solve_goal` itself returns. Read back `assumed_defaults[]`.

*(Both examples use fictional figures — never reuse a real user's numbers in documentation.)*

## Notes

- All rates/decimals are fractions; all dollars are today's (real) dollars; any thresholds are ~2026.
- REQUIRED raw inputs that can't be inferred: `analyze_already_won` needs `liquid_net_worth` +
  `annual_spend`; `analyze_fire_benchmark` needs `annual_income` + `annual_savings` +
  `liquid_net_worth`; `analyze_milestones` needs `net_worth`; `solve_goal` needs `target_fire_age`
  (20–100) plus a plan (inline or `plan_id`). Ask for them before calling.
- Pass `{ plan_id }` to reuse a saved household model on `analyze_already_won`, `analyze_milestones`,
  and `solve_goal`; pass an `overrides` object for inline changes on top of the resolved plan.
  **`analyze_fire_benchmark` does NOT accept `plan_id`** — pass its figures raw.
- All four tools return a structured `assumed_defaults[]` (`{ field, assumed_value, note }`) — read
  it back to the user. `disclosures.key_assumptions` is separate static method prose;
  `disclosures.not_advice` is a boolean. `solve_goal` always returns a `share_url`,
  `analyze_already_won` returns one when given a resolving `plan_id`; for the other two, chain
  `generate_financial_plan` for a sharable link.
- Not financial advice. Planning estimates only.
