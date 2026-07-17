# PrepSense — AI-Powered Inventory Intelligence for Independent Restaurants

> Stop guessing. Start selling what you have.

---

## The Problem

Independent restaurants waste 4-9% of revenue on food spoilage every year. Unlike enterprise chains with dedicated inventory systems and data teams, small restaurant owners make daily purchasing and menu decisions based on gut feel — often over-ordering perishables, missing aging inventory, and writing off losses that compound weekly.

There was no AI tool built for their scale, budget, or operational reality.

---

## What PrepSense Does

PrepSense takes a chef or owner's natural language question about their inventory and returns a specific, confidence-tiered recommendation they can act on today:

- **What to use first** — identifies aging inventory before it becomes waste
- **What to order** — demand-based purchasing suggestions grounded in real data
- **What to feature** — menu recommendation based on what needs to move
- **Confidence tier** — high, medium, or low based on data quality and grounding
- **RLHF feedback loop** — every approval or rejection improves future recommendations

---

## Why It's Different from Just Asking ChatGPT/Claude/Other Ai 

1. **Grounded in real restaurant data** — pulls live inventory, pricing, and weather context before generating a recommendation
2. **LLM-as-Judge** — a second LLM validates every recommendation for groundedness, safety, and actionability before the owner sees it
3. **RLHF-style approval loop** — owners approve or reject recommendations and report outcomes, creating a feedback dataset that improves the system over time
4. **Price Bandit layer** — multi-armed bandit algorithm optimizes ingredient pricing suggestions based on historical trial and reward data
5. **31 real eval cases logged** — not a demo, a product with real outcome data

---

## Architecture

```
Owner Question (Chat or Webhook)
    ↓
Extract Form + Config
    ↓
Weather (OpenMeteo) — contextual grounding (weather affects demand)
    ↓
Fetch Price Bandit Data (Supabase) — pricing intelligence
    ↓
Build Model Request
    ↓
Router Agent — classifies intent (use, order, feature, general)
    ↓
Parse Intent
    ↓
Load Restaurant Data (Supabase) — live inventory, menu, history
    ↓
HTTP Request — external data enrichment
    ↓
Build Recommendation Request
    ↓
Log to Supabase — pre-recommendation logging
    ↓
Recommendation Agent (GPT-4.1-mini) — generates grounded recommendation
    ↓
Guardrails — safety and policy checks
    ↓
Build Judge Request
    ↓
LLM Judge (GPT-4.1) — validates groundedness, safety, actionability
    ↓
Log Eval Scores — logs trust scores to Supabase
    ↓
Assemble Response — final structured output with confidence tier
    ↓
Respond to Webhook / Chat
```

**Separate RLHF feedback flow:**
```
Owner approves or rejects recommendation
    ↓
Outcome Report webhook
    ↓
Process Outcome Report → Update Supabase row
```

---

## Key Design Decisions & Tradeoffs

**Explicit rec_id over LLM memory**

Early versions relied on the LLM to track which recommendation it was evaluating feedback for. This produced inconsistent matching when conversations branched. Switching to explicit rec_id passed through the entire flow made the RLHF loop deterministic and reliable — a foundational architectural choice that everything else depends on.

**Tiered confidence logic**

Rather than a binary pass/fail, recommendations are tiered high/medium/low based on grounding quality, data completeness, and Judge scores. This gives owners calibrated trust signals — a high confidence recommendation on well-documented aging inventory is acted on differently than a medium confidence suggestion on sparse data.

**Weather as grounding context**

Demand for soups, hot dishes, and comfort food correlates with weather. Integrating OpenMeteo as a free real-time grounding signal improved recommendation relevance for seasonal specials and daily specials without adding API cost.

**Price Bandit for dynamic pricing suggestions**

Rather than static pricing rules, a multi-armed bandit algorithm in Supabase tracks trials and rewards per ingredient, surfacing pricing suggestions that have historically driven the best outcomes. This is the learning layer that compounds over time.

**LLM Judge catches hallucinations before they reach the owner**

Early testing showed the recommendation model occasionally suggested using ingredients that were not in the current inventory data. The Judge node — running a stronger model — was added specifically to catch these cases. Of 31 logged eval cases, the Judge caught and flagged 3 hallucinated recommendations before they reached the user.

---

## Eval Framework

Every recommendation is logged to Supabase with:

| Field | Description |
|-------|-------------|
| `rec_id` | Unique recommendation ID |
| `ingredient` | Primary ingredient referenced |
| `overall_score` | Trust score 0-1 |
| `pass` | Did recommendation pass all checks? |
| `groundedness` | Is recommendation grounded in inventory data? |
| `safety` | Did it pass safety guardrails? |
| `actionability` | Is the recommendation specific enough to act on? |
| `issues` | Array of flagged issues from Judge |
| `approved` | Did owner approve? |
| `outcome_note` | What actually happened |

**31 real eval cases logged** across Harbor & Vine restaurant test environment.

---

## What I Would Do Differently

The price bandit layer is the most exciting part architecturally but has the least data — 31 cases is not enough for meaningful bandit convergence. With more time I would run PrepSense in a real restaurant for 90 days to accumulate enough outcome data for the bandit to meaningfully optimize, and add a seasonal adjustment layer so summer and winter demand patterns don't interfere with the reward signal.

I would also add a structured onboarding flow that helps restaurant owners upload their existing inventory and menu data in a standardized format, rather than relying on manual Supabase entry which is the current bottleneck to real-world adoption.

---

## Stack

| Layer | Tool |
|-------|------|
| Orchestration | n8n |
| LLM — Recommendations | OpenAI GPT-4.1-mini |
| LLM — Judge | OpenAI GPT-4.1 |
| Weather | OpenMeteo API (free) |
| Database | Supabase (PostgreSQL) |
| Pricing intelligence | Multi-armed bandit in Supabase |
| Eval logging | Supabase insights table |

---
URL for Website:
https://praneethareddyvaddeman-cmd.github.io/Prepsense-ai/UI.html

## Build Log

- **v1**: Basic inventory recommendation with webhook trigger and Supabase logging
- **v2**: Added LLM Judge grounding checks and tiered confidence logic
- **v3**: RLHF approval loop with explicit rec_id tracking
- **v4**: Price bandit layer, weather grounding, Router agent for intent classification
- **v5**: Guardrails node, eval score logging, 31 real cases logged
- **v8 (current)**: Full agentic pipeline with outcome reporting loop, hallucination catch documented
