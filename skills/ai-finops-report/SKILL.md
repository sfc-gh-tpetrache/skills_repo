---
name: ai-finops-report
description: "Generate an account-wide AI FinOps HTML report for Snowflake CoCo (Cortex Code) and CoWork (Snowflake Intelligence) usage. Attributes credits per prompt, ranks top users with a high/median/low request distribution, breaks down by product/interface/agent, analyzes conversation length/duration and the most-expensive sessions (prompts, tokens, cache-hit rate, credits), reasoning steps per interaction, tool usage, model mix, semantically clusters popular prompts (AI_EMBED), and reports verified-query coverage with VQR-authoring candidates. Use when asked for: AI FinOps report, AI usage report, AI observability report, cost per prompt report, CoCo/CoWork usage or spend report, who is driving AI cost, which prompts should become verified queries."
allowed-tools: "*"
---

# AI FinOps Report (CoCo + CoWork)

Produce an account-wide **AI FinOps** HTML report for an admin: where AI credits go across **Snowflake CoCo** (Cortex Code) and **Snowflake CoWork** (Snowflake Intelligence / Cortex Agents), who drives them, how efficiently (conversation length/duration, reasoning steps per interaction, cache-hit rate on the most-expensive sessions, model mix), what people actually ask (semantic prompt clusters), and whether agent questions are covered by **verified queries**.

**When to invoke:** "AI FinOps report", "AI usage/observability report", "cost per prompt report", "CoCo/CoWork spend report", "who is driving AI cost", "which prompts should be verified queries".

## Golden rules

1. **Credits come from the ACCOUNT_USAGE usage views — they are the source of truth.** The event table `SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS` is best-effort telemetry (spans can drop or arrive late); use it only for prompt text, traces, turns, tool usage, and `verified_query_used`. Never present an event-table-derived number as spend.
2. **Join trace to usage on `REQUEST_ID`** to attribute a known cost to a prompt. CoCo and CoWork use different spans/keys — see the reference files.
3. **This is read-only.** Only `SELECT` / `SHOW` / `DESCRIBE`. Never create, alter, or drop anything in the user's account.
4. **The HTML report must follow the `html-authoring` skill** (sandboxed: inline JS only, vendored `/libs/`, provenance block, light-dark theming, responsive charts).

## Inputs

- **Scope** — the ONLY input. Ask with `ask_user_question`: **CoCo**, **CoWork**, or **Both** (default Both). No other products.
- **Period** — fixed at **last month (last 30 days)**, i.e. `{WINDOW_DAYS}=30`. Do NOT ask the user for a period.

## Workflow

See the step-by-step workflow in [reference/workflow.md](reference/workflow.md). In short:

0. **Permissions preflight FIRST** — before asking anything or running analysis, confirm the current role can read the usage views and the event table (zero-scan `WHERE 1=0` probes). If not, show the access message with the exact grants to request and stop.
1. Ask **scope only** (CoCo / CoWork / Both). Period is fixed at last month (30 days) — do not ask. Confirm the active connection.
2. Branch on scope (explicit if/else):
   - **CoCo** -> run [reference/queries-common.md](reference/queries-common.md) (CoCo variant) + [reference/queries-coco.md](reference/queries-coco.md).
   - **CoWork** -> run common (CoWork variant) + [reference/queries-cowork.md](reference/queries-cowork.md).
   - **Both** -> run everything, plus the CoCo-vs-CoWork comparison.
3. Cluster prompts semantically per [reference/clustering.md](reference/clustering.md) (prompts only).
4. Assemble the HTML report per [reference/report-template.md](reference/report-template.md) and save it to the current workspace.
5. Print a short text summary + top recommendations. Offer to schedule recurring runs via `cortex automation`.

## Reference files

- [reference/workflow.md](reference/workflow.md) — ordered execution steps and the scope if/else.
- [reference/queries-common.md](reference/queries-common.md) — scope-agnostic SQL (KPIs, top users, cost-per-prompt distribution, turns, tool mix, model mix, daily trend).
- [reference/queries-coco.md](reference/queries-coco.md) — CoCo-specific SQL (interface split, conversations + most-expensive sessions, model mix, CoCo prompt extraction).
- [reference/queries-cowork.md](reference/queries-cowork.md) — CoWork-specific SQL (agent breakdown, verified-query coverage + VQR candidates).
- [reference/clustering.md](reference/clustering.md) — `AI_EMBED` semantic clustering of prompts.
- [reference/report-template.md](reference/report-template.md) — HTML report structure and section map (defers to `html-authoring`).

## Caveats (state the relevant ones in the report)

- **Credit source of truth** is the usage view; event-table sums under-report.
- **CoCo vs CoWork differ**: CoCo prompts are wrapped in `<system-reminder>` boilerplate (clean with `POSITION`/`SPLIT_PART`; Snowflake regex is greedy-only, no lazy/lookahead); CoWork prompts are clean. Join keys differ (see reference files).
- **`verified_query_used` is CoWork/agent-only** — it lives on the `SystemExecuteSQLTool_system_execute_sql` span. CoCo has no verified-query concept, so the VQR section is hidden for CoCo-only scope.
- **Turns**: `PARENT_REQUEST_ID` is typically not chaining multi-turn threads, so request ~= turn; count step spans per `TRACE:trace_id` for agentic depth.
- **Empty cleaned prompts** are context-only turns (e.g. file-attach continuations) — label, don't drop.
- **Access**: direct reads of `AI_OBSERVABILITY_EVENTS` require `ACCOUNTADMIN` or `SNOWFLAKE.AI_OBSERVABILITY_READER` + `IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE`.
