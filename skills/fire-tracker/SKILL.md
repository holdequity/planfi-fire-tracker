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
`analyze_milestones`, and `solve_goal` — accept `{ plan_id }` (plus inline overrides), so they can
resolve net worth, spend, and age from the saved plan instead of you re-sending every figure.
`generate_financial_plan` also returns a **`share_url`** (planfi.app) — none of the four tools below
emit a share link themselves, so this is the way to give the user one.

> **`analyze_fire_benchmark` is the exception: it takes raw inputs ONLY (NO `plan_id`).** Pass its
> figures (income, savings rate / net worth, age) directly — do not try to chain a `plan_id` into it.

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
Returns whether the portfolio already covers spending — the `surplus`, `cushion_years`, and a
`safe_spend` figure.
REQUIRED: `liquid_net_worth`, `annual_spend`. (Accepts `{ plan_id }` to resolve these.)

```
analyze_already_won({ liquid_net_worth: 1600000, annual_spend: 60000 })
```

### "How do I compare to peers?" → `analyze_fire_benchmark`  *(raw inputs only — NO plan_id)*
Returns a `savings_percentile_band` and `cohort_framing` versus a peer cohort.
Pass raw figures directly: `income`, plus `savings_rate` and/or `net_worth`, and `age`. **Do not pass
`plan_id`** — this tool does not accept one.

```
analyze_fire_benchmark({ income: 140000, savings_rate: 0.35, net_worth: 450000, age: 38 })
```

### "What milestones have I hit?" → `analyze_milestones`
Returns the net-worth milestones crossed, including `newly_reached[]`.
REQUIRED: `net_worth`. (Accepts `{ plan_id }`.)

```
analyze_milestones({ net_worth: 450000 })
```

### "Can I retire by age X?" → `solve_goal`
Solves for a target FIRE age and ranks the levers (savings rate, retire age, spend, allocation) that
close the gap, **easiest-first**.
REQUIRED: `target_fire_age` (20–100). Optional: `levers[]` (`savings_rate` | `retire_age` | `spend` |
`allocation`) to constrain which knobs to turn. (Accepts `{ plan_id }`.)

```
solve_goal({ target_fire_age: 50 })
```

## Step 3 — Surface results honestly

For whichever tool you called:
- **Lead with the headline** — the already-won verdict + `surplus` / `cushion_years` / `safe_spend`;
  the `savings_percentile_band` + `cohort_framing`; the `newly_reached[]` milestones; or the
  ranked, easiest-first levers `solve_goal` returns to hit the target age (and whether the target is
  reachable at all).
- **Read back `disclosures.key_assumptions`** verbatim. These tools do **not** emit a structured
  `assumed_defaults[]` array — instead they apply silent Zod defaults (e.g. withdrawal rate, return
  and inflation assumptions, peer-cohort framing) and expose what they assumed only as **prose in
  `disclosures.key_assumptions`**. Surface those so the user can correct any silent assumption.
- Honor `disclosures.not_advice` — present as planning estimates, not financial advice.
- **Follow `next_actions[]`** — each is `{ tool, why, prefilled_args }` (carrying `{ plan_id }` when
  available). These commonly chain — e.g. already-won → `solve_goal` (if not there yet) or →
  `analyze_milestones`; benchmark → `generate_financial_plan` for a full forecast. Use the
  server-suggested chains rather than guessing.
- **For a share link:** these four tools don't return one. If the user wants a sharable plan, run
  `generate_financial_plan` (Step 1) and surface its `share_url`.

## Recommended call sequence (typical session)

1. (preferred) `generate_financial_plan` → capture `plan_id` (+ `share_url`) — note
   `analyze_fire_benchmark` won't use the `plan_id`.
2. Route by intent → one of the four tools (with `{ plan_id }` where accepted, plus the REQUIRED raw
   fields).
3. Read back the headline + `disclosures.key_assumptions`.
4. Follow `next_actions[]` — FIRE questions naturally chain (already-won → solve_goal /
   milestones; benchmark → full plan), or back to `generate_financial_plan` for a share link.

## Fictional examples

**1.** *"I'm 45 with $1.6M invested and spend $60k a year — am I already financially independent?"*
→ `analyze_already_won({ liquid_net_worth: 1600000, annual_spend: 60000 })`. Lead with the verdict
(won / not yet), the `surplus`, `cushion_years`, and `safe_spend`; if not yet there, chain
`solve_goal` to show the smallest lever to close the gap. Read back key_assumptions (the withdrawal
rate / return assumption that drives the verdict).

**2.** *"Can I retire by 50 if I'm 38 with $400k invested and saving $3k/mo?"*
→ build a quick plan with `generate_financial_plan` (capturing `plan_id` + `share_url`), then
`solve_goal({ target_fire_age: 50, plan_id })`. Lead with whether age 50 is reachable and the
easiest-first levers (e.g. bump savings rate, trim spend, push retire age) it returns; surface the
`share_url`. Read back key_assumptions.

*(Both examples use fictional figures — never reuse a real user's numbers in documentation.)*

## Notes

- All rates/decimals are fractions; all dollars are today's (real) dollars; any thresholds are ~2026.
- REQUIRED raw inputs that can't be inferred: `analyze_already_won` needs `liquid_net_worth` +
  `annual_spend`; `analyze_fire_benchmark` needs `income` + (`savings_rate` and/or `net_worth`) +
  `age`; `analyze_milestones` needs `net_worth`; `solve_goal` needs `target_fire_age` (20–100). Ask
  for them before calling.
- Pass `{ plan_id }` to reuse a saved household model on `analyze_already_won`, `analyze_milestones`,
  and `solve_goal`; any field you also pass is a shallow override. **`analyze_fire_benchmark` does
  NOT accept `plan_id`** — pass its figures raw.
- These four tools surface assumptions as **prose in `disclosures.key_assumptions`**, not a
  structured `assumed_defaults[]`, and they do **not** return a `share_url` — chain
  `generate_financial_plan` for a sharable link. (Server follow-up tracked in `SKILL_AUTHORING.md`.)
- Not financial advice. Planning estimates only.
