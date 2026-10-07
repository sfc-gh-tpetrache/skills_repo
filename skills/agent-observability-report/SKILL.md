---
name: agent-observability-report
description: "Generate an operational monitoring report for a single Cortex Agent over the last N days (default 30), driven by SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS. Covers usage & adoption, reliability (success/error + SLOW quality-of-service), latency percentiles (p50/p90/p95/max), conversation depth, tool & sub-agent execution, token/credit economics, and user feedback. Use when asked for: agent monitoring report, agent observability report, 30-day agent report, how is my agent doing, agent usage/latency/adoption, is my agent slow, agent health check over time, which tools/sub-agents does my agent use, agent feedback summary. Invoke for ANY request to monitor or report on a deployed agent's production behavior over a time window (not a single-request debug, not an eval run)."
allowed-tools: "*"
---

# Cortex Agent Observability Report

Produce a repeatable **operational monitoring report** for one deployed Cortex Agent over a time window (default last 30 days). Answers the questions an agent owner asks once an agent is live: is it being used, is it reliable, is it fast enough, what does it cost, which tools/sub-agents actually fire, and are users happy.

**When to invoke:** "agent monitoring report", "30-day report for my agent", "how is my agent doing", "is my agent slow", "agent adoption/latency/usage", "agent health check", "agent feedback summary".

**When NOT to use:**
- Debugging why a *single request* failed or returned a wrong answer → use `agent-studio` → `debug`.
- Inspecting a batch *evaluation run*'s scores → use `agent-studio` → `agent` → `monitor` (eval runs).
- Account-wide AI spend across CoCo + CoWork and all agents → use `ai-finops-report`.

## Golden rules

1. **Observability is the primary source.** All usage, reliability, latency, depth, tool, and feedback metrics come from `SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS` via `GET_AI_OBSERVABILITY_EVENTS(db, schema, agent, 'CORTEX AGENT')`. One row per turn = `RECORD_ATTRIBUTES:"ai.observability.span_type"::string = 'record_root'`.
2. **Observability does NOT carry token/credit totals.** For token economics, read the ACCOUNT_USAGE usage views — and branch on interface: **CoWork / Snowflake Intelligence traffic is in `SNOWFLAKE_COWORK_USAGE_HISTORY`, NOT `CORTEX_AGENT_USAGE_HISTORY`.** Query both and use whichever has rows for the agent (API/Teams/external land in `CORTEX_AGENT_USAGE_HISTORY`). This split is the single most-missed detail — see [reference/queries.md](reference/queries.md).
3. **This is read-only.** Only `SELECT` / `SHOW` / table functions. Never create, alter, or drop anything.
4. **Reconcile the two sources.** The distinct request count from observability should match the usage-view request count; if they diverge, say so (telemetry can drop/lag; ACCOUNT_USAGE lags up to ~1h).

## Inputs

- **Agent** (required) — fully qualified `DATABASE.SCHEMA.AGENT_NAME`. If the user gives only a name, resolve with `SHOW AGENTS LIKE '%<NAME>%' IN ACCOUNT;` and confirm. If `get_page_context` is available, read `metadata.database/schema/agentName` silently and skip the prompt.
- **Window** — number of days back. Default **30**. Only ask if the user implies a different period.

## Workflow

1. **Preflight.** Resolve the agent FQN (Inputs above). Confirm the active connection. The querying role needs **MONITOR on the agent + the `SNOWFLAKE.CORTEX_USER` database role**; direct `AI_OBSERVABILITY_EVENTS` access needs `ACCOUNTADMIN` or `SNOWFLAKE.AI_OBSERVABILITY_READER` + `IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE`; usage views need ACCOUNT_USAGE access. If a query fails on privileges, show the missing grant and stop.
2. **Probe data availability.** Run the span-inventory query ([reference/queries.md](reference/queries.md) §0) to confirm there is `record_root` traffic in the window and to see which tool/sub-agent spans exist. If zero turns, tell the user and stop.
3. **Run the metric queries** in [reference/queries.md](reference/queries.md) §1–§7: usage/adoption + reliability, daily trend, latency percentiles, conversation depth, tool & sub-agent execution, feedback. All scope to `record_root` except tool/feedback spans.
4. **Run token economics** (§8): query both usage views for the agent over the window; use the one with rows; reconcile request counts against §1.
5. **Assemble the report** per [reference/report-template.md](reference/report-template.md): headline KPI card, then the fixed sections, a daily-trend bar, a tool/sub-agent bar, and prioritized recommendations. Default output is in-chat markdown + `visualize_data` charts; offer an optional HTML export (follow the `html-authoring` skill) and `publish_report` if the user wants a shareable artifact.
6. **State caveats** that actually apply (see below) and offer to re-run for a different window or compare across agent versions.

## Caveats (include the relevant ones in the report)

- **Token source branch**: CoWork/Snowflake Intelligence traffic is only in `SNOWFLAKE_COWORK_USAGE_HISTORY`; API/Teams/external in `CORTEX_AGENT_USAGE_HISTORY`. A 0-row result in one view is expected, not an error.
- **`SLOW` is a quality-of-service flag**, not an error. 100% SUCCESS with 100% `SLOW` means the agent answers correctly but is latency-flagged — call this out; drive the latency section + recommendations off it.
- **Single-user traffic** (one distinct `snow.user.name`) usually means pre-production/testing, not real adoption — say so rather than over-reading the numbers.
- **Freshness**: ACCOUNT_USAGE views lag up to ~1h (so very recent turns may lack token data); observability spans can arrive late or drop. Reconcile and note gaps.
- **Latency is in milliseconds** on `snow.ai.observability.agent.duration` (INTEGER) — divide by 1000 for seconds.

## Reference files

- [reference/queries.md](reference/queries.md) — validated SQL for every section, plus the attribute-key index for `record_root` spans.
- [reference/report-template.md](reference/report-template.md) — fixed section layout, KPI card, and chart specs.
