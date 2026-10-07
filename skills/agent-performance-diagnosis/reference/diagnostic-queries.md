# Diagnostic queries

All queries read from the **event table** `SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS`, **not** the `GET_AI_OBSERVABILITY_EVENTS()` UDTF (the UDTF trips an "LLM evaluations" entitlement gate even when the caller holds `CORTEX_USER`; the table does not).

**Working set = a reusable CTE named `_obs`, not a temp table.** A fully-qualified `SELECT` needs no current database and no `CREATE` privilege, so the skill stays portable (works in any account) and truly read-only. Prepend the `_obs` CTE (§0b) to each query below and read `FROM _obs`. The agent-name filter is highly selective (one agent ≈ a few hundred rows), so re-scanning the table per query is cheap — the materialize-once optimization the guide used was only needed because the *UDTF* was slow.

Placeholders: `:db`, `:schema`, `:agent`, `:window_days`. Agent identity lives in `RECORD_ATTRIBUTES`:
- `snow.ai.observability.database.name`
- `snow.ai.observability.schema.name`
- `snow.ai.observability.object.name`

Do **not** filter on `snow.ai.observability.object.type` — it appears with inconsistent casing (`Cortex Agent` and `CORTEX AGENT`).

---

## §0 Agent discovery (Phase 0)

List every agent that has observability events so the user can pick one. (No `_obs` CTE here — this is the pre-selection step, before `:db`/`:schema`/`:agent` are known.)

```sql
SELECT
  RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::STRING AS db,
  RECORD_ATTRIBUTES:"snow.ai.observability.schema.name"::STRING   AS schema,
  RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::STRING   AS agent,
  COUNT(*)        AS event_rows,
  MAX(TIMESTAMP)  AS last_seen
FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
WHERE RECORD_ATTRIBUTES:"snow.ai.observability.object.name" IS NOT NULL
GROUP BY 1,2,3
ORDER BY event_rows DESC;
```

## §0b The reusable `_obs` CTE

This is the canonical working-set definition. Prepend it to every query in §1–§5 and select `FROM _obs`. It replaces the old `_obs_events` temp table — **no `USE DATABASE`, no `CREATE TABLE`, no write of any kind.**

```sql
WITH _obs AS (
  SELECT *
  FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
  WHERE RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::STRING = :db
    AND RECORD_ATTRIBUTES:"snow.ai.observability.schema.name"::STRING   = :schema
    AND RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::STRING   = :agent
    AND TIMESTAMP >= DATEADD('day', -:window_days, CURRENT_TIMESTAMP())
)
-- Sanity: how many turns are in scope?
SELECT COUNT(*) AS request_rows
FROM _obs
WHERE RECORD:"name"::STRING = 'CORTEX_AGENT_REQUEST';
```
If `request_rows = 0`, stop and tell the user there is no traffic in the window.

> When chaining with a query that already has its own CTE (e.g. §4), just add `_obs` as the first CTE: `WITH _obs AS (...), turns AS (SELECT ... FROM _obs ...)`.

---

## §1 Overall latency profile (guide §1.2) — the baseline

```sql
WITH _obs AS (
  SELECT * FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
  WHERE RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::STRING = :db
    AND RECORD_ATTRIBUTES:"snow.ai.observability.schema.name"::STRING   = :schema
    AND RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::STRING   = :agent
    AND TIMESTAMP >= DATEADD('day', -:window_days, CURRENT_TIMESTAMP())
)
SELECT
  COUNT(*) AS total_requests,
  ROUND(AVG(VALUE:"snow.ai.observability.response_time_ms"::FLOAT),0) AS avg_ms,
  ROUND(PERCENTILE_CONT(0.5)  WITHIN GROUP (ORDER BY VALUE:"snow.ai.observability.response_time_ms"::FLOAT),0) AS p50_ms,
  ROUND(PERCENTILE_CONT(0.9)  WITHIN GROUP (ORDER BY VALUE:"snow.ai.observability.response_time_ms"::FLOAT),0) AS p90_ms,
  ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY VALUE:"snow.ai.observability.response_time_ms"::FLOAT),0) AS p95_ms,
  ROUND(STDDEV(VALUE:"snow.ai.observability.response_time_ms"::FLOAT),0) AS stddev_ms
FROM _obs
WHERE RECORD:"name"::STRING = 'CORTEX_AGENT_REQUEST'
  AND VALUE:"snow.ai.observability.response_time_ms" IS NOT NULL;
```

### §1b SLOW / TAINTED share (guide §1.6 flag)

```sql
WITH _obs AS (
  SELECT * FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
  WHERE RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::STRING = :db
    AND RECORD_ATTRIBUTES:"snow.ai.observability.schema.name"::STRING   = :schema
    AND RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::STRING   = :agent
    AND TIMESTAMP >= DATEADD('day', -:window_days, CURRENT_TIMESTAMP())
)
SELECT
  COUNT(*) AS turns,
  COUNT_IF(RECORD_ATTRIBUTES:"snow.ai.observability.agent.status.description"::STRING ILIKE '%SLOW%')    AS slow_turns,
  COUNT_IF(RECORD_ATTRIBUTES:"snow.ai.observability.agent.status.description"::STRING ILIKE '%TAINTED%')  AS tainted_turns
FROM _obs
WHERE RECORD:"name"::STRING = 'AgentV2RequestResponseInfo';
```

### §1c Daily latency (guide §1.3) — platform rule-out

Daily avg / p95 / max to spot a platform-wide slow day vs. a workload spike. **Keep this one grouping (per day) — do not UNION it with the §4 first-vs-follow-up query**; mixing two different `GROUP BY` grains in one `SELECT` is what triggers a "neither an aggregate nor in the group by clause" compile error. Run the two separately. With few rows/day, p95 ≈ max — that's expected.

```sql
WITH _obs AS (
  SELECT * FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
  WHERE RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::STRING = :db
    AND RECORD_ATTRIBUTES:"snow.ai.observability.schema.name"::STRING   = :schema
    AND RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::STRING   = :agent
    AND TIMESTAMP >= DATEADD('day', -:window_days, CURRENT_TIMESTAMP())
)
SELECT
  DATE_TRUNC('day', TIMESTAMP)::DATE AS day,
  COUNT(*) AS turns,
  ROUND(AVG(VALUE:"snow.ai.observability.response_time_ms"::FLOAT)/1000,1) AS avg_s,
  ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY VALUE:"snow.ai.observability.response_time_ms"::FLOAT)/1000,1) AS p95_s,
  ROUND(MAX(VALUE:"snow.ai.observability.response_time_ms"::FLOAT)/1000,1) AS max_s
FROM _obs
WHERE RECORD:"name"::STRING = 'CORTEX_AGENT_REQUEST'
  AND VALUE:"snow.ai.observability.response_time_ms" IS NOT NULL
GROUP BY 1
ORDER BY 1;
```
A single day much slower than its neighbors across *all* turns suggests platform (§1.3); a spike confined to turns that fired a heavy path (deep-research swarm, multi-sub-agent fan-out) is workload, not infra — correlate with §R.

---

## §2 Slowest-turn ranking (guide §1.2) — pick the 1–2 slowest to drill in Phase 3

Not a top-N worklist — this ranks turns so you can pick the **1–2 slowest** to inspect span-by-span (Phase 3). The aggregate diagnostics (§1.3–§1.6) run over the whole window, not this list.

> **Join key = `request_id` (`ai.observability.record_id`), NOT `trace_id`.** All spans of one turn share a single `record_id`; `CORTEX_AGENT_REQUEST` rows carry `record_id` but have a **null `trace_id`**. Key every per-turn query on `record_id`.

```sql
WITH _obs AS (
  SELECT * FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
  WHERE RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::STRING = :db
    AND RECORD_ATTRIBUTES:"snow.ai.observability.schema.name"::STRING   = :schema
    AND RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::STRING   = :agent
    AND TIMESTAMP >= DATEADD('day', -:window_days, CURRENT_TIMESTAMP())
)
SELECT
  RECORD_ATTRIBUTES:"ai.observability.record_id"::STRING AS request_id,
  TIMESTAMP                                              AS ts,
  ROUND(VALUE:"snow.ai.observability.response_time_ms"::FLOAT,0) AS response_time_ms
FROM _obs
WHERE RECORD:"name"::STRING = 'CORTEX_AGENT_REQUEST'
  AND VALUE:"snow.ai.observability.response_time_ms" IS NOT NULL
ORDER BY response_time_ms DESC
LIMIT 10;
```

---

## §3 Per-turn evidence (guide §1.4–§1.6) — run on the 1–2 slowest turns in Phase 3

### §3a Fast-path pull of one request's spans (agent-studio/debug entry)

Keyed directly on the request id, so no `_obs` CTE is needed.

```sql
SELECT *
FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
WHERE RECORD_ATTRIBUTES:"ai.observability.record_id"::STRING = '<REQUEST_ID>'
LIMIT 50;
```

### §3b Planning steps & tool calls per turn (guide §1.4)

```sql
SELECT
  RECORD_ATTRIBUTES:"ai.observability.record_id"::STRING AS request_id,
  MAX(REGEXP_SUBSTR(RECORD:"name"::STRING,'ReasoningAgentStepPlanning-([0-9]+)',1,1,'e')::INT) + 1 AS planning_steps,
  COUNT_IF(RECORD_ATTRIBUTES:"snow.ai.observability.agent.planning.tool_selection.name" IS NOT NULL) AS tool_calls
FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
WHERE RECORD_ATTRIBUTES:"ai.observability.record_id"::STRING = '<REQUEST_ID>'
GROUP BY 1;
```

Tool names/types selected during planning:
```sql
SELECT
  RECORD_ATTRIBUTES:"snow.ai.observability.agent.planning.tool_selection.name"::STRING AS tool_name,
  RECORD_ATTRIBUTES:"snow.ai.observability.agent.planning.tool_selection.type"::STRING AS tool_type,
  COUNT(*) AS selections
FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
WHERE RECORD_ATTRIBUTES:"ai.observability.record_id"::STRING = '<REQUEST_ID>'
  AND RECORD_ATTRIBUTES:"snow.ai.observability.agent.planning.tool_selection.name" IS NOT NULL
GROUP BY 1,2
ORDER BY selections DESC;
```

> §3b–§3d are already scoped to a single `<REQUEST_ID>`, so they query the base table directly — no `_obs` CTE needed. (You *may* wrap them in `_obs` for consistency; it makes no difference to the result.)

### §3c Time budget — where the seconds go (guide §1.5)

Notes on real attribute keys (verified against a live agent):
- All duration/token keys are **`snow.`-prefixed** (`snow.ai.observability.agent.tool.sql_execution.duration`, not `ai....`).
- Reasoning time lives on `snow.ai.observability.agent.planning.duration` across **both** `ReasoningAgentStepPlanning-%` **and** `ReasoningAgentStepResponseGeneration-%` spans — match `ReasoningAgentStep%` to get total reasoning.
- `sql_execution.duration` is duplicated on both the `SqlExecution_*` span and its `SystemExecuteSQLTool_*` wrapper — gate on `STARTSWITH(RECORD:"name"::STRING,'SqlExecution_')` only, or you double-count. **Do not use `ILIKE 'SqlExecution\_%'`** — Snowflake `LIKE/ILIKE` has no default escape character, so the `\` is treated as a literal backslash and the pattern matches **nothing** (silently returns 0). Use `STARTSWITH` (or add an explicit `ESCAPE '\'`).
- Agents that use **Cortex Analyst** emit `SqlExecution_CortexAnalyst` (+ `snow.ai.observability.analyst.sql_generation.duration`); agents using the system SQL tool emit `SqlExecution_SystemSQL` with **no** separate generation span. The `STARTSWITH(...,'SqlExecution_')` match covers both.
- **`tool_execution_ms` (non-SQL tools) is a required bucket.** Document generation, file manipulation, code execution, MCP, retrieval, and routing carry time on their own `tool.<family>.duration` keys — not on `planning.duration`. If you omit this bucket, that time is invisible and, worse, a long planning step that *wraps* a tool call makes `reasoning_ms` absorb it. Always compute `tool_execution_ms` and the `unaccounted_ms` residual so a document-generation / code-exec turn is not mis-tagged as "slow reasoning." (See §6 for the per-family duration keys and why `server_skill.duration` is ~0 for pdf/pptx — the render time is on the paired `CodeExecutionTool_*`.)

```sql
SELECT
  RECORD_ATTRIBUTES:"ai.observability.record_id"::STRING AS request_id,
  -- total reasoning time (planning + response generation steps)
  ROUND(SUM(IFF(RECORD:"name"::STRING ILIKE 'ReasoningAgentStep%',
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.planning.duration"::FLOAT, 0)),0) AS reasoning_ms,
  -- Analyst SQL generation time (0 for agents that use the system SQL tool)
  ROUND(SUM(IFF(RECORD:"name"::STRING = 'SqlExecution_CortexAnalyst',
      RECORD_ATTRIBUTES:"snow.ai.observability.analyst.sql_generation.duration"::FLOAT, 0)),0) AS sql_generation_ms,
  -- SQL execution time (STARTSWITH avoids the tool-wrapper double-count AND the ILIKE escape trap)
  ROUND(SUM(IFF(STARTSWITH(RECORD:"name"::STRING,'SqlExecution_'),
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.sql_execution.duration"::FLOAT, 0)),0) AS sql_execution_ms,
  -- non-SQL tool execution: code-exec/file, doc/chart gen, MCP, retrieval, routing, semantic context
  ROUND(SUM(IFF(
      STARTSWITH(RECORD:"name"::STRING,'CodeExecutionTool_') OR STARTSWITH(RECORD:"name"::STRING,'ServerSkillTool_')
   OR STARTSWITH(RECORD:"name"::STRING,'ServerMCPTool_')      OR STARTSWITH(RECORD:"name"::STRING,'CortexChartToolImpl')
   OR STARTSWITH(RECORD:"name"::STRING,'CortexSearchService_')OR STARTSWITH(RECORD:"name"::STRING,'SemanticContextTool_')
   OR STARTSWITH(RECORD:"name"::STRING,'AgentRouterTool_')    OR STARTSWITH(RECORD:"name"::STRING,'TaskTool_SWARM_')
   OR RECORD:"name"::STRING = 'ToolCall-FileRead',
      COALESCE(
        RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.code_execution.duration"::FLOAT,
        RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.server_skill.duration"::FLOAT,
        RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.server_mcp.duration"::FLOAT,
        RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.chart_generation.duration"::FLOAT,
        RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.cortex_search.duration"::FLOAT,
        RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.semantic_context.duration"::FLOAT,
        RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.agent_router.duration"::FLOAT,
        0), 0)),0) AS tool_execution_ms
FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
WHERE RECORD_ATTRIBUTES:"ai.observability.record_id"::STRING = '<REQUEST_ID>'
GROUP BY 1;
```

> **Residual check.** Compare the four buckets to the turn's `response_time_ms` (§2). `unaccounted_ms = response_time_ms − reasoning_ms − sql_generation_ms − sql_execution_ms − tool_execution_ms`. A large residual on a turn that fired a routed sub-agent or a `ServerSkillTool_*` swarm = inherited **wait** inside a parent planning step (§R) — not parent compute. Beware double-counting the other way too: when a tool runs *inside* a planning step's window, both `reasoning_ms` and `tool_execution_ms` can include it, so treat the buckets as attribution hints, then confirm with the §3-timeline / §R span ordering.

Slowest executed SQL in the turn → look up in QUERY_HISTORY:
```sql
SELECT
  RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.sql_execution.query_id"::STRING AS query_id,
  RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.sql_execution.query"::STRING    AS query_text,
  ROUND(RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.sql_execution.duration"::FLOAT,0) AS sql_execution_ms
FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
WHERE RECORD_ATTRIBUTES:"ai.observability.record_id"::STRING = '<REQUEST_ID>'
  AND STARTSWITH(RECORD:"name"::STRING,'SqlExecution_')
  AND RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.sql_execution.query_id" IS NOT NULL
ORDER BY sql_execution_ms DESC
LIMIT 5;

-- Then, for the slowest query_id:
SELECT query_id, warehouse_name, execution_status, total_elapsed_time,
       queued_overload_time, compilation_time, execution_time,
       bytes_scanned, partitions_scanned, partitions_total
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_id = '<QUERY_ID>';
```

### §3d VQR hit rate (guide §1.5)

VQR usage is flagged per SQL-execution span by `snow.ai.observability.agent.tool.sql_execution.verified_query_used` (BOOLEAN). (The guide names `snow.ai.observability.analyst.vqr_matched`; that only appears on `SqlExecution_CortexAnalyst` spans — `verified_query_used` is the one present on the system SQL tool. Use whichever the agent emits.) This one aggregates across the whole window, so it uses the `_obs` CTE:

```sql
WITH _obs AS (
  SELECT * FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
  WHERE RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::STRING = :db
    AND RECORD_ATTRIBUTES:"snow.ai.observability.schema.name"::STRING   = :schema
    AND RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::STRING   = :agent
    AND TIMESTAMP >= DATEADD('day', -:window_days, CURRENT_TIMESTAMP())
)
SELECT
  COUNT(*) AS sql_executions,
  COUNT_IF(RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.sql_execution.verified_query_used"::BOOLEAN) AS vqr_hits,
  ROUND(100 * vqr_hits / NULLIF(sql_executions,0),1) AS vqr_hit_pct
FROM _obs
WHERE STARTSWITH(RECORD:"name"::STRING,'SqlExecution_')
  AND RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.sql_execution.verified_query_used" IS NOT NULL;
```

### §3e Planning tokens & step-0 vs step-1+ (guide §1.6)

```sql
WITH _obs AS (
  SELECT * FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
  WHERE RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::STRING = :db
    AND RECORD_ATTRIBUTES:"snow.ai.observability.schema.name"::STRING   = :schema
    AND RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::STRING   = :agent
    AND TIMESTAMP >= DATEADD('day', -:window_days, CURRENT_TIMESTAMP())
)
SELECT
  IFF(RECORD:"name"::STRING = 'ReasoningAgentStepPlanning-0','step_0_first','step_1_plus') AS step_bucket,
  COUNT(*) AS steps,
  ROUND(AVG(RECORD_ATTRIBUTES:"snow.ai.observability.agent.planning.token_count.input"::FLOAT),0) AS avg_input_tokens,
  ROUND(AVG(RECORD_ATTRIBUTES:"snow.ai.observability.agent.planning.token_count.total"::FLOAT),0) AS avg_total_tokens
FROM _obs
WHERE RECORD:"name"::STRING ILIKE 'ReasoningAgentStepPlanning-%'
GROUP BY 1;
```
Base token floor is ~2,500 input tokens; production agents commonly land at 6,000–15,000 step-0 tokens, and each planning step adds ~5–8 s.

---

## §4 First vs follow-up turns (guide §3.7) — multi-turn accumulation

> **Do not cast `snow.ai.observability.agent.first_message_in_thread` to BOOLEAN** — in observed data it holds the user's **prompt text**, not a flag, and `...agent.messages` holds the full conversation JSON. Derive turn order robustly from `thread_id` + `TIMESTAMP` instead.

`_obs` is chained as the first CTE, then `turns` reads from it:

```sql
WITH _obs AS (
  SELECT * FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
  WHERE RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::STRING = :db
    AND RECORD_ATTRIBUTES:"snow.ai.observability.schema.name"::STRING   = :schema
    AND RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::STRING   = :agent
    AND TIMESTAMP >= DATEADD('day', -:window_days, CURRENT_TIMESTAMP())
),
turns AS (
  SELECT
    RECORD_ATTRIBUTES:"snow.ai.observability.agent.thread_id"::STRING AS thread_id,
    RECORD_ATTRIBUTES:"snow.ai.observability.agent.duration"::FLOAT   AS duration_ms,
    ROW_NUMBER() OVER (
      PARTITION BY RECORD_ATTRIBUTES:"snow.ai.observability.agent.thread_id"::STRING
      ORDER BY TIMESTAMP) AS turn_in_thread
  FROM _obs
  WHERE RECORD:"name"::STRING = 'AgentV2RequestResponseInfo'
)
SELECT
  IFF(turn_in_thread = 1,'first_in_thread','follow_up') AS turn_kind,
  COUNT(*) AS turns,
  ROUND(AVG(duration_ms),0) AS avg_duration_ms
FROM turns
GROUP BY 1
ORDER BY 1;
```
If follow-ups are materially slower than first turns, suspect context accumulation (§3.7). If they are similar or faster, rule §3.7 out and focus on the per-turn reasoning/SQL budget.

---

## §5 Negative feedback (optional corroboration)

```sql
WITH _obs AS (
  SELECT * FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
  WHERE RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::STRING = :db
    AND RECORD_ATTRIBUTES:"snow.ai.observability.schema.name"::STRING   = :schema
    AND RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::STRING   = :agent
    AND TIMESTAMP >= DATEADD('day', -:window_days, CURRENT_TIMESTAMP())
)
SELECT
  VALUE:"positive"::BOOLEAN AS positive,
  VALUE:"categories"        AS categories,
  VALUE:"feedback_message"::STRING AS feedback_message,
  TIMESTAMP
FROM _obs
WHERE RECORD:"name"::STRING = 'CORTEX_AGENT_FEEDBACK'
ORDER BY TIMESTAMP DESC;
```

---

## §6 Tool-usage taxonomy (aggregate) — what the agent actually *does*

Classifies every tool span into a semantic category and times each one. Run this in **Phase 2** alongside the latency profile: it answers "is this agent routing, querying, generating documents, calling MCP, or manipulating files?" and — critically — surfaces tool time that §3c would otherwise fold into `reasoning_ms`.

**Grounded facts (verified against live telemetry):**
- Each tool family carries its **own** duration key; there is no single universal one. Coalesce across all of them:
  `agent_router`, `sql_execution`, `semantic_context`, `cortex_search`, `code_execution`, `server_skill`, `chart_generation`, `server_mcp` — each as `snow.ai.observability.agent.tool.<family>.duration`.
- `server_skill.duration` is **often 0** for document generation (`ServerSkillTool_pdf_generation`/`pptx`): the real seconds live in the **paired `CodeExecutionTool_*`** span that renders the file. The taxonomy counts both, so doc-gen work shows up under *code execution* time even when the skill wrapper reports 0 — read the two categories together.
- Two span populations share the table: Cortex **Agent** spans and **CoCo CodingAgent** spans (`CodingAgent.Step-N`, `ToolCall-bash`, `CodeExecutionTool_snowflake_sql_execute`). The `_obs` agent filter (`db`/`schema`/`agent`) already excludes CoCo, so this query only classifies agent-scoped tool calls.
- Use `STARTSWITH`, never `ILIKE 'X\_%'` — Snowflake `LIKE/ILIKE` has no default escape, so a `\` matches literally and silently returns nothing.

### §6a Category rollup

```sql
WITH _obs AS (
  SELECT * FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
  WHERE RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::STRING = :db
    AND RECORD_ATTRIBUTES:"snow.ai.observability.schema.name"::STRING   = :schema
    AND RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::STRING   = :agent
    AND TIMESTAMP >= DATEADD('day', -:window_days, CURRENT_TIMESTAMP())
),
tools AS (
  SELECT
    RECORD:"name"::STRING AS span,
    RECORD_ATTRIBUTES:"ai.observability.record_id"::STRING AS request_id,
    CASE
      WHEN STARTSWITH(span,'AgentRouterTool_') OR STARTSWITH(span,'TaskTool_SWARM_')               THEN '1 routing/delegation'
      WHEN STARTSWITH(span,'SqlExecution_') OR STARTSWITH(span,'SystemExecuteSQLTool_')
        OR STARTSWITH(span,'SemanticContextTool_')                                                 THEN '2 sql/semantic data'
      WHEN STARTSWITH(span,'CortexSearchService_') OR STARTSWITH(span,'CortexSearchSingleToolImpl') THEN '3 retrieval/search'
      WHEN span = 'ServerSkillTool_pdf_generation' OR STARTSWITH(span,'ServerSkillTool_pptx')
        OR STARTSWITH(span,'ServerSkillTool_docx') OR STARTSWITH(span,'ServerSkillTool_chart')
        OR STARTSWITH(span,'CortexChartToolImpl')                                                  THEN '4 document/chart generation'
      WHEN STARTSWITH(span,'CodeExecutionTool_') OR STARTSWITH(span,'ToolCall-FileRead')           THEN '5 code execution/file manipulation'
      WHEN STARTSWITH(span,'ServerMCPTool_')                                                        THEN '6 mcp (external tools)'
      WHEN STARTSWITH(span,'ServerSkillTool_')                                                      THEN '7 other server skills'
      ELSE NULL
    END AS category,
    COALESCE(
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.agent_router.duration"::FLOAT,
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.sql_execution.duration"::FLOAT,
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.semantic_context.duration"::FLOAT,
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.cortex_search.duration"::FLOAT,
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.chart_generation.duration"::FLOAT,
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.code_execution.duration"::FLOAT,
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.server_skill.duration"::FLOAT,
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.server_mcp.duration"::FLOAT,
      0) AS dur_ms
  FROM _obs
)
SELECT
  category,
  COUNT(*)                                   AS calls,
  COUNT(DISTINCT request_id)                 AS turns_used_in,
  ROUND(SUM(dur_ms)/1000,1)                  AS total_s,
  ROUND(AVG(dur_ms)/1000,2)                  AS avg_s,
  ROUND(100*SUM(dur_ms)/NULLIF(SUM(SUM(dur_ms)) OVER (),0),1) AS pct_of_tool_time
FROM tools
WHERE category IS NOT NULL
GROUP BY category
ORDER BY category;
```

> **Doc-gen reading:** because `pdf_generation`'s own duration is ~0, category 4 may show few seconds while category 5 (code execution) holds the render time. A high category-5 total on an agent that also fires `pdf_generation`/`pptx` = document generation is the real workload, not generic SWE. Treat 4+5 together.

### §6b Per-span detail

Same metrics at the individual span-name grain — use it to see *which* sub-agent (routing), *which* MCP server, or *which* skill dominates, now with **per-span `avg_s` and `p95_s`** so you can rank by latency as well as volume. The router label is normalized so FQN and short forms don't collapse to "AGENT". The coalesced `dur_ms` is computed once in a CTE so `AVG` / `PERCENTILE_CONT` read a single clean number per span.

> **To rank sub-agents specifically** (the "rank sub-agents by P95 and volume" question), filter the result to the `route -> ` rows: those carry each routed sub-agent's call count, turns-used-in, and p95 of its delegated duration together. `total_s` here is delegated wall-clock (it includes the sub-agent's full nested run).

```sql
WITH _obs AS (
  SELECT * FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
  WHERE RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::STRING = :db
    AND RECORD_ATTRIBUTES:"snow.ai.observability.schema.name"::STRING   = :schema
    AND RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::STRING   = :agent
    AND TIMESTAMP >= DATEADD('day', -:window_days, CURRENT_TIMESTAMP())
),
tool_spans AS (
  SELECT
    -- normalize AgentRouterTool_FROSTBYTE_AI_PROD.AGENTS.HR_AGENT and AgentRouterTool_HR_AGENT alike
    CASE
      WHEN STARTSWITH(RECORD:"name"::STRING,'AgentRouterTool_')
        THEN 'route -> '||SPLIT_PART(REGEXP_REPLACE(RECORD:"name"::STRING,'^AgentRouterTool_',''),'.',-1)
      ELSE RECORD:"name"::STRING
    END AS tool,
    RECORD_ATTRIBUTES:"ai.observability.record_id"::STRING AS request_id,
    COALESCE(
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.agent_router.duration"::FLOAT,
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.sql_execution.duration"::FLOAT,
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.semantic_context.duration"::FLOAT,
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.cortex_search.duration"::FLOAT,
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.chart_generation.duration"::FLOAT,
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.code_execution.duration"::FLOAT,
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.server_skill.duration"::FLOAT,
      RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.server_mcp.duration"::FLOAT,
      0) AS dur_ms
  FROM _obs
  WHERE STARTSWITH(RECORD:"name"::STRING,'AgentRouterTool_')
     OR STARTSWITH(RECORD:"name"::STRING,'TaskTool_SWARM_')
     OR STARTSWITH(RECORD:"name"::STRING,'SqlExecution_')
     OR STARTSWITH(RECORD:"name"::STRING,'SystemExecuteSQLTool_')
     OR STARTSWITH(RECORD:"name"::STRING,'SemanticContextTool_')
     OR STARTSWITH(RECORD:"name"::STRING,'CortexSearchService_')
     OR STARTSWITH(RECORD:"name"::STRING,'CortexSearchSingleToolImpl')
     OR STARTSWITH(RECORD:"name"::STRING,'CortexChartToolImpl')
     OR STARTSWITH(RECORD:"name"::STRING,'CodeExecutionTool_')
     OR STARTSWITH(RECORD:"name"::STRING,'ServerMCPTool_')
     OR STARTSWITH(RECORD:"name"::STRING,'ServerSkillTool_')
     OR RECORD:"name"::STRING = 'ToolCall-FileRead'
)
SELECT
  tool,
  COUNT(*)                                        AS calls,
  COUNT(DISTINCT request_id)                      AS turns_used_in,
  ROUND(AVG(dur_ms)/1000,2)                        AS avg_s,
  ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY dur_ms)/1000,1) AS p95_s,
  ROUND(SUM(dur_ms)/1000,1)                        AS total_s
FROM tool_spans
GROUP BY 1
ORDER BY total_s DESC;
```

---

## §6c Router vs sub-agent split & fan-out — the router-attribution questions

Two aggregate queries that answer "how much latency is the router's own work vs. delegated to sub-agents?" and "how often does one question fan out to several sub-agents?". Both rely on the fact that `AgentRouterTool_<SUBAGENT>`'s `snow.ai.observability.agent.tool.agent_router.duration` **measures the delegated sub-agent run directly** — so no timestamp-window correlation is needed. Key per-turn work on `record_id`.

### §6c-A Router/orchestration time vs. delegated sub-agent time

Per turn, delegated time = sum of that turn's `AgentRouterTool_*` durations; the router's own time (dispatch + parent reasoning) = `response_time_ms − delegated`.

```sql
WITH _obs AS (
  SELECT * FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
  WHERE RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::STRING = :db
    AND RECORD_ATTRIBUTES:"snow.ai.observability.schema.name"::STRING   = :schema
    AND RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::STRING   = :agent
    AND TIMESTAMP >= DATEADD('day', -:window_days, CURRENT_TIMESTAMP())
),
delegated AS (   -- sub-agent (delegated) time per turn
  SELECT RECORD_ATTRIBUTES:"ai.observability.record_id"::STRING AS request_id,
         SUM(RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.agent_router.duration"::FLOAT) AS subagent_ms
  FROM _obs
  WHERE STARTSWITH(RECORD:"name"::STRING,'AgentRouterTool_')
  GROUP BY 1
),
turns AS (       -- total time per turn
  SELECT RECORD_ATTRIBUTES:"ai.observability.record_id"::STRING AS request_id,
         VALUE:"snow.ai.observability.response_time_ms"::FLOAT  AS total_ms
  FROM _obs
  WHERE RECORD:"name"::STRING = 'CORTEX_AGENT_REQUEST'
    AND VALUE:"snow.ai.observability.response_time_ms" IS NOT NULL
)
SELECT
  COUNT(*)                                                              AS turns,
  COUNT_IF(d.subagent_ms IS NOT NULL)                                   AS routed_turns,
  ROUND(SUM(t.total_ms)/1000,1)                                         AS total_s,
  ROUND((SUM(t.total_ms)-SUM(COALESCE(d.subagent_ms,0)))/1000,1)        AS router_own_s,
  ROUND(SUM(COALESCE(d.subagent_ms,0))/1000,1)                          AS subagent_s,
  ROUND(100*SUM(COALESCE(d.subagent_ms,0))/NULLIF(SUM(t.total_ms),0),1) AS pct_delegated
FROM turns t
LEFT JOIN delegated d USING (request_id);
```
A high `pct_delegated` is the quantified router verdict: the parent is a thin pass-through and the real latency lives in the sub-agents — pivot the diagnosis into the slowest one (§6b `route -> ` ranking) rather than tuning the router.

### §6c-B Fan-out distribution — distinct sub-agents per question

```sql
WITH _obs AS (
  SELECT * FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
  WHERE RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::STRING = :db
    AND RECORD_ATTRIBUTES:"snow.ai.observability.schema.name"::STRING   = :schema
    AND RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::STRING   = :agent
    AND TIMESTAMP >= DATEADD('day', -:window_days, CURRENT_TIMESTAMP())
),
fanout AS (
  SELECT
    RECORD_ATTRIBUTES:"ai.observability.record_id"::STRING AS request_id,
    COUNT(DISTINCT SPLIT_PART(REGEXP_REPLACE(RECORD:"name"::STRING,'^AgentRouterTool_',''),'.',-1)) AS distinct_subagents,
    COUNT(*) AS router_calls   -- > distinct_subagents ⇒ same sub-agent called more than once in the turn
  FROM _obs
  WHERE STARTSWITH(RECORD:"name"::STRING,'AgentRouterTool_')
  GROUP BY 1
)
SELECT
  distinct_subagents,
  COUNT(*)                                    AS turns,
  ROUND(100*COUNT(*)/SUM(COUNT(*)) OVER (),1) AS pct_of_routed_turns
FROM fanout
GROUP BY 1
ORDER BY distinct_subagents;
```
Fan-out is typically **serial** on these agents, so a turn hitting 3 sub-agents adds their durations (it does not overlap them). Correlate a high-fan-out tail with the §1c daily spike and §6c-A `pct_delegated` to confirm multi-sub-agent turns, not the platform, drive p95.

### §6c-C Tool calls per question — distribution across the window

How many tool calls a single question triggers, aggregated over all turns (§3b computes this for *one* turn; this is the window-level distribution that feeds the §1.4 "too many calls" verdict). Counts every planning tool selection (`tool_selection.name IS NOT NULL`), which includes routing, SQL/Analyst, search, skills, and MCP — i.e. **all** tool calls, not just routing (§6c-B is routing-only).

```sql
WITH _obs AS (
  SELECT * FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
  WHERE RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::STRING = :db
    AND RECORD_ATTRIBUTES:"snow.ai.observability.schema.name"::STRING   = :schema
    AND RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::STRING   = :agent
    AND TIMESTAMP >= DATEADD('day', -:window_days, CURRENT_TIMESTAMP())
),
per_turn AS (
  SELECT
    RECORD_ATTRIBUTES:"ai.observability.record_id"::STRING AS request_id,
    COUNT_IF(RECORD_ATTRIBUTES:"snow.ai.observability.agent.planning.tool_selection.name" IS NOT NULL) AS tool_calls
  FROM _obs
  GROUP BY 1
)
SELECT
  COUNT(*)                                                                    AS turns,
  ROUND(AVG(tool_calls),1)                                                    AS avg_calls,
  MEDIAN(tool_calls)                                                          AS p50_calls,
  ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY tool_calls),0)           AS p95_calls,
  MAX(tool_calls)                                                             AS max_calls,
  ROUND(100*COUNT_IF(tool_calls >= 3)/NULLIF(COUNT(*),0),1)                   AS pct_turns_3plus_calls
FROM per_turn;
```
A high `avg_calls` / `p95_calls` points at §1.4 (too many serial tool or Analyst calls) — the fix is parallelizing independent tools (§2.4/§4.1) or trimming instructions that over-trigger tool use. For a router, cross-check against §6c-B: if most of the calls are routing, the lever is sub-agent fan-out, not tool parallelization.
