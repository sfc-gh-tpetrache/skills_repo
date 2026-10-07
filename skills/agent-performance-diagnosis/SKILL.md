---
name: agent-performance-diagnosis
description: "Diagnose WHY a Cortex Agent is slow and recommend fixes, implementing the SKE Cortex Agent Performance Optimization Guide. Prompts for which agent to diagnose and the window (7/14/30 days, default 30), profiles latency (p50/p90/p95) from SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS as a baseline, then runs the guide's aggregate root-cause diagnostics in order — router/nested-execution attribution, platform, too many Analyst calls, slow SQL, slow reasoning, multi-turn — drilling the 1-2 slowest turns' span timelines to confirm where the time went before mapping to guide fixes (diagnose nested sub-agent/skill, cross-region inference if not already set, auto model, VQRs, materialize base tables, trim instructions, parallelize tools, disable chart gen). Use when asked: why is my agent slow, diagnose agent latency, agent performance optimization, speed up my agent, reduce agent latency, agent is too slow, high p95 latency, planning/SQL/reasoning is slow. Read-only latency diagnosis — not a single-request bug debug (use agent-studio/debug) and not a general health report (use agent-observability-report)."
allowed-tools: "*"
---

# Cortex Agent Performance Diagnosis

Find out **why a deployed Cortex Agent is slow** and produce prioritized, guide-mapped fixes. Implements the *SKE Cortex Agent Performance Optimization Guide*: profile overall latency as a baseline, run the guide's **aggregate** root-cause diagnostics in order (router/nested-execution → platform → too many Analyst calls → slow SQL → slow reasoning → multi-turn), drill the **1–2 slowest turns'** span timelines to confirm where the time actually went (nested sub-agent/skill wait vs real parent reasoning), then recommend the guide's fixes with a before/after baseline.

**When to invoke:** "why is my agent slow", "diagnose agent latency", "agent performance optimization", "speed up my agent", "reduce p95 latency", "planning/SQL/reasoning is slow", "agent is too slow".

**When NOT to use:**
- Debugging why a *single request* failed or returned a *wrong answer* → use `agent-studio` → `debug` directly.
- A general operational *health/usage* report (adoption, cost, reliability) → use `agent-observability-report`.
- Inspecting a batch *evaluation run*'s quality scores → use `agent-studio` → `monitor`.

## Golden rules

1. **Always ask which agent to diagnose.** Discover agents that have observability events and let the user pick (Phase 0). Skip the prompt only if the user already named a fully-qualified agent.
2. **Source telemetry from the event TABLE via a reusable CTE, not the UDTF and not a temp table.** Define a `_obs` CTE over `SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS` with a `RECORD_ATTRIBUTES` filter (§0b) and read `FROM _obs` — **not** from `GET_AI_OBSERVABILITY_EVENTS()`. The UDTF runs internal LLM-evaluation logic that throws *"You must have the SNOWFLAKE.CORTEX_USER role to use LLM evaluations"* even when the caller holds `CORTEX_USER`; the table reads fine and is faster. A CTE (not a temp table) keeps the skill portable — a fully-qualified `SELECT` needs no current database and no `CREATE` privilege, and stays truly read-only. The agent filter is selective, so re-scanning per query is cheap.
3. **Read-only.** Only `SELECT` / `SHOW` / table reads. The per-turn `agent-studio/debug` drill-down runs in **analysis-only** mode — evidence + diagnosis, **no edits, no fix application**. Respect debug's reproduce-before-fix invariant: never propose a config change from a stale log alone; recommendations are hypotheses to validate, not applied changes.
4. **Rule out platform-side latency first.** Follow the root-cause order: platform (§1.3) → too many Analyst calls (§1.4) → slow SQL execution (§1.5) → slow reasoning/planning (§1.6) → multi-turn context accumulation (§3.7). Don't tune agent config before excluding a platform/warehouse cause.
5. **Baseline before, measure after.** Capture the current latency profile before recommending anything; every recommendation should name the metric to re-measure after the change.
6. **Use both latency keys for their correct purpose** (see below).

## Dual latency keys

The guide uses two different representations — use each where it belongs:

| Purpose | Record filter (`RECORD:"name"`) | Latency expression |
|---|---|---|
| Latency profile + slowest-turn ranking | `'CORTEX_AGENT_REQUEST'` | `VALUE:"snow.ai.observability.response_time_ms"::FLOAT` |
| SLOW/TAINTED flag, first-vs-follow-up | `'AgentV2RequestResponseInfo'` | `RECORD_ATTRIBUTES:"snow.ai.observability.agent.duration"::FLOAT` (ms) |

## Inputs

- **Agent** — fully-qualified `DATABASE.SCHEMA.AGENT`. If not supplied, discover and prompt (Phase 0).
- **Window** — days back. **Always ask** via `ask_user_question`, offering **7 / 14 / 30** days with **30 as the default** (telemetry is often sparse, so a wider default surfaces more turns). Skip the prompt only if the user already named a window.

## Workflow

### Phase 0 — Select agent & scope
1. If the user already gave a fully-qualified agent, confirm it and skip to step 3.
2. Otherwise run the **agent-discovery** query ([reference/diagnostic-queries.md](reference/diagnostic-queries.md) §0): list `db.schema.agent`, event-row count, and last-seen. Present via `ask_user_question` and let the user choose (or paste an FQN).
3. Ask for the **window** via `ask_user_question` — options **7 / 14 / 30** days, **default 30** (preselect it). Skip only if the user already specified a period. You may combine this with the agent-selection prompt in step 2 as a second question.
4. **Prereq check:** querying role needs **only `SELECT` on `SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS`** (`ACCOUNTADMIN`, or `SNOWFLAKE.AI_OBSERVABILITY_READER` + `IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE`) and MONITOR/OWNERSHIP on the agent. No current database and no `CREATE` privilege are required — the queries are fully-qualified and read-only. If a read fails on privileges, show the missing grant and stop. Do **not** diagnose a missing-grant as a performance problem.
5. **Scope check:** run the §0b sanity count (the `_obs` CTE with `COUNT(*)` of `CORTEX_AGENT_REQUEST` rows). If zero rows in the window, tell the user and stop. Every subsequent query prepends the same `_obs` CTE.

### Phase 1 — Overall latency profile (§1.2)
Run the profile query (§1): total requests, avg, p50/p90/p95, stddev (ms). Record this as the **baseline**. Also compute the SLOW/TAINTED share (§1.6 flag). Headline the result ("p95 = X s over N turns, Y% flagged SLOW").

### Phase 2 — Aggregate root-cause diagnostics, in the guide's order
Run the guide's **aggregate** diagnostics across the whole window (not a top-N sweep) and work the decision order in [reference/root-cause-playbook.md](reference/root-cause-playbook.md). First resolve **§R router / nested-execution** (does the agent route to sub-agents or invoke agentic skills?), then **§T tool-time** (is time in document/chart generation, code execution/file manipulation, or MCP?), then: platform (§1.3) → too many Analyst calls (§1.4) → slow SQL (§1.5) → slow reasoning (§1.6) → multi-turn accumulation (§3.7). Use the §-matched queries in [reference/diagnostic-queries.md](reference/diagnostic-queries.md):
- **§R:** `AgentRouterTool_%` span counts by sub-agent; `planning.tool_selection.type = agent_router`; presence of `ServerSkillTool_*` / `TaskTool_SWARM_*`. Quantify the router split with **§6c-A** (`pct_delegated` = router-own vs. delegated sub-agent seconds), **§6c-B** (fan-out: distinct sub-agents per question), and **§6b** (now with per-span `p95_s`) to rank the slowest sub-agent.
- **§6 / §T tool-usage taxonomy:** run the category rollup (§6a) + per-span detail (§6b) to see what the agent actually does — routing / SQL & semantic / retrieval / **code execution & file manipulation** / **document & chart generation** / **MCP** / other skills, with time per category. `pdf_generation`/`pptx` report ~0 duration — the render seconds are in the paired `CodeExecutionTool_*` (read categories 4+5 together). MCP time is external-dependency time, not model/SQL.
- **§1.3:** daily avg/p95/max (query §1c) — run it on its own, **not** UNION-ed with the §3.7 query (two different `GROUP BY` grains in one `SELECT` fails to compile). For a router, a low parent step count with high p95 is **not** a platform signal — it means delegation. Verify account settings before recommending them (e.g. `SHOW PARAMETERS LIKE 'CORTEX_ENABLED_CROSS_REGION'`).
- **§1.4:** planning-step distribution + tool/router selection breakdown. Use **§6c-C** for the window-level tool-calls-per-question distribution (avg/p95/max) — a high p95 is the §1.4 "too many serial calls" signal.
- **§1.5:** aggregate time budget — now four buckets (reasoning vs SQL-gen vs SQL-exec vs **tool-execution**) plus the **`unaccounted_ms`** residual, SQL count, VQR hit rate. A large tool-execution or unaccounted share redirects the diagnosis to §T / §R, not §1.6.
- **§1.6:** per-step planning duration + step-0 vs step-1+ tokens.
- **§3.7:** first-in-thread vs follow-up duration.

### Phase 3 — Targeted span-timeline drill (only to confirm attribution)
Pick the **1–2 slowest turns** (ranking query §2) and inspect each turn's **full span timeline ordered by `TIMESTAMP`** — not a top-10 debug sweep. The goal is to confirm *where the seconds actually went* before concluding: a long `planning.duration` either **wraps nested execution** (§R — a routed sub-agent's `CORTEX_AGENT_REQUEST`, or a `ServerSkillTool_*` / `TaskTool_SWARM_*` swarm, or `SqlExecution_*`) **or is a tool/generation step** (§T — a `CodeExecutionTool_*` render paired with a `ServerSkillTool_pdf_generation`/`pptx`, or an `ServerMCPTool_*` call). Only when neither is present is a long planning step genuine model reasoning. Key on `request_id` (`ai.observability.record_id`), not `trace_id` (`CORTEX_AGENT_REQUEST` has a null `trace_id`).
```sql
SELECT TIMESTAMP, RECORD:"name"::STRING AS span,
  RECORD_ATTRIBUTES:"snow.ai.observability.agent.planning.duration"::FLOAT AS planning_ms
FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
WHERE RECORD_ATTRIBUTES:"ai.observability.record_id"::STRING = '<REQUEST_ID>'
ORDER BY TIMESTAMP;
```
If a slow planning step wraps a routed sub-agent, **pivot the diagnosis into that sub-agent** (re-run Phase 1–2 against it). The optional per-turn `agent-studio/debug` sub-skill (analysis-only) is a **closing guideline** the user can run on a specific turn — not a step you execute for every turn. **No edits.**

### Phase 4 — Attribute the dominant root cause & report
Name the dominant cause from the §R→§3.7 decision order, backed by a concrete evidence pointer (a `RECORD_ATTRIBUTES` field, a span-timeline nesting, or a verified account setting). Map it to the guide's fixes via the playbook (diagnose the nested sub-agent/skill first for routers; cross-region inference **only if not already enabled**; auto model, VQRs, materialize base tables §2.5, trim instructions §2.7/2.8/4.5, parallelize tools §2.4/4.1, `cortex reflect` §4.7, disable chart generation §4.8). Assemble the report per [reference/report-template.md](reference/report-template.md): selected agent + window → baseline profile → aggregate diagnostics by root cause → slowest-turn span attribution → prioritized recommendations (each with the metric to re-measure). Default output is in-chat markdown + `visualize_data`; offer an optional HTML export (follow `html-authoring`) and `publish_report` for a shareable artifact. **Always close the report with the per-turn debug guideline** (report-template §7): name the specific slowest `request_id`(s) and recommend running the `agent-studio` → `debug` sub-skill (analysis-only) on each to validate before any fix — and for a router, recommend re-running this diagnosis against the sub-agent the slow turn routed to.

## Caveats (include the ones that apply)
- **`SLOW` is a quality-of-service flag, not an error.** 100% SUCCESS with 100% SLOW means correct-but-latency-flagged — that is exactly the case this skill targets.
- **Latency keys are in milliseconds** — divide by 1000 for seconds.
- **Recommendations are hypotheses.** They come from telemetry patterns, not a reproduced fix. Validate on a live re-test before applying (that's the `agent-studio/debug` write-path, out of scope here).
- **QUERY_HISTORY lags** up to ~1h, so a very recent slow `query_id` may not resolve yet.
- **Rule out platform first:** a slow turn whose budget is dominated by non-agent wait (queued warehouse, cross-region cold start) is a platform/infra fix, not an agent-config change.

## Reference files
- [reference/diagnostic-queries.md](reference/diagnostic-queries.md) — agent discovery, the reusable `_obs` CTE, and all guide SQL (profile, slowest-turn ranking, router/nested detection, planning steps/tools, time budget, planning tokens, first-vs-follow-up, feedback).
- [reference/root-cause-playbook.md](reference/root-cause-playbook.md) — the four root causes + multi-turn, the decision order, and the guide fix mapped to each.
- [reference/report-template.md](reference/report-template.md) — fixed report layout and chart specs.
