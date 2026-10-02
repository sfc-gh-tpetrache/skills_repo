# CoWork-specific queries

Run these only when scope is **CoWork** or **Both**. Base usage view: `SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COWORK_USAGE_HISTORY` (time column `START_TIME`). Trace data: `SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS`. `{WINDOW_DAYS}` default 30. All read-only.

CoWork sections: agent breakdown, verified-query coverage + VQR candidates, model mix, semantic prompt clusters (see `clustering.md`) — plus the common blocks run with the CoWork substitution.

---

## CW1. Credits & requests by agent

```sql
SELECT
    SNOWFLAKE_COWORK_NAME,
    COALESCE(AGENT_NAME, '(no agent)') AS agent_name,
    ROUND(SUM(COALESCE(TOKEN_CREDITS,0)), 4) AS credits,
    SUM(TOKENS)                              AS tokens,
    COUNT(DISTINCT REQUEST_ID)               AS requests,
    COUNT(DISTINCT USER_NAME)                AS users
FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COWORK_USAGE_HISTORY
WHERE START_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
  AND START_TIME < CURRENT_TIMESTAMP()
GROUP BY 1, 2
ORDER BY credits DESC;
```

---

## CW2. Verified-query coverage

The agent SQL-execution span records whether the generated SQL came from a verified query. Field:
`RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.sql_execution.verified_query_used"` (boolean) on span `SystemExecuteSQLTool_system_execute_sql`, alongside `...object.name` (the semantic view/model).

**Exclude skill-based turns.** A turn whose trace contains a `ServerSkillTool_%` span is driven by a packaged **agent skill** that runs its own SQL — it is NOT an ad-hoc text-to-SQL question that a verified query would replace. Compute coverage (and VQR candidates) over **ad-hoc SQL turns only**, excluding any trace that invoked a skill. (Observed in this account: 101 of 124 SQL executions were skill-driven; excluding them changes the denominator from 124 to 23.)

Overall coverage (ad-hoc only):

```sql
WITH skill_traces AS (
    SELECT DISTINCT TRACE:trace_id::VARCHAR AS tid
    FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
    WHERE TIMESTAMP >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND RECORD_TYPE='SPAN' AND RECORD:name::VARCHAR LIKE 'ServerSkillTool_%'
),
sql_exec AS (
    SELECT TRACE:trace_id::VARCHAR AS tid,
           IFF(RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.sql_execution.verified_query_used"::BOOLEAN, 1, 0) AS verified
    FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
    WHERE TIMESTAMP >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND RECORD:name::VARCHAR = 'SystemExecuteSQLTool_system_execute_sql'
)
SELECT COUNT(*) AS adhoc_sql_executions,
       SUM(e.verified) AS verified_executions,
       ROUND(100.0 * SUM(e.verified) / NULLIF(COUNT(*),0), 1) AS verified_pct
FROM sql_exec e
LEFT JOIN skill_traces sk ON sk.tid = e.tid
WHERE sk.tid IS NULL;
```

By semantic object (where the gaps are), ad-hoc only:

```sql
WITH skill_traces AS (
    SELECT DISTINCT TRACE:trace_id::VARCHAR AS tid
    FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
    WHERE TIMESTAMP >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND RECORD_TYPE='SPAN' AND RECORD:name::VARCHAR LIKE 'ServerSkillTool_%'
)
SELECT
    COALESCE(e.RECORD_ATTRIBUTES:"snow.ai.observability.database.name"::VARCHAR, '(unknown)') AS database_name,
    COALESCE(e.RECORD_ATTRIBUTES:"snow.ai.observability.object.name"::VARCHAR, '(unknown)')   AS object_name,
    COUNT(*) AS sql_executions,
    SUM(IFF(e.RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.sql_execution.verified_query_used"::BOOLEAN, 1, 0)) AS verified_executions,
    ROUND(100.0 * SUM(IFF(e.RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.sql_execution.verified_query_used"::BOOLEAN, 1, 0))
          / NULLIF(COUNT(*),0), 1) AS verified_pct
FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS e
LEFT JOIN skill_traces sk ON sk.tid = e.TRACE:trace_id::VARCHAR
WHERE e.TIMESTAMP >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
  AND e.RECORD:name::VARCHAR = 'SystemExecuteSQLTool_system_execute_sql'
  AND sk.tid IS NULL
GROUP BY 1, 2
ORDER BY sql_executions DESC;
```

Per-trace verified flag (used to join prompts -> VQR candidates in `clustering.md`) — **ad-hoc SQL traces only** (skill traces excluded, so VQR candidates never include skill-driven questions):

```sql
WITH skill_traces AS (
    SELECT DISTINCT TRACE:trace_id::VARCHAR AS tid
    FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
    WHERE TIMESTAMP >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND RECORD_TYPE='SPAN' AND RECORD:name::VARCHAR LIKE 'ServerSkillTool_%'
),
sql_exec AS (
    SELECT TRACE:trace_id::VARCHAR AS tid,
           IFF(RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.sql_execution.verified_query_used"::BOOLEAN, 1, 0) AS verified
    FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
    WHERE TIMESTAMP >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND RECORD:name::VARCHAR = 'SystemExecuteSQLTool_system_execute_sql'
)
SELECT e.tid AS trace_id, MAX(e.verified) AS used_verified_query
FROM sql_exec e
LEFT JOIN skill_traces sk ON sk.tid = e.tid
WHERE sk.tid IS NULL
GROUP BY 1;
```

---

## CW3. CoWork prompt extraction (clustering input)

Clean prompt text (no reminder wrapper on CoWork) per turn, with agent, per-request credits, and the verified flag. This is the input to `clustering.md` and is already **collapsed to distinct prompts** to keep embedding cheap.

```sql
WITH agent_runs AS (
    SELECT DISTINCT TRACE:trace_id::VARCHAR AS trace_id,
           RECORD_ATTRIBUTES:"request_id"::VARCHAR AS request_id
    FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
    WHERE TIMESTAMP >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND RECORD:name::VARCHAR = 'Agent'
      AND RECORD_ATTRIBUTES:"request_id" IS NOT NULL
),
prompts AS (
    SELECT TRACE:trace_id::VARCHAR AS trace_id,
           TRIM(REGEXP_REPLACE(MAX(RECORD_ATTRIBUTES:"snow.ai.observability.agent.planning.query"::VARCHAR), '\\s+', ' ')) AS prompt
    FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
    WHERE TIMESTAMP >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND RECORD:name::VARCHAR LIKE 'ReasoningAgentStepPlanning-%'
      AND RECORD_ATTRIBUTES:"snow.ai.observability.agent.planning.query" IS NOT NULL
    GROUP BY 1
),
usage AS (
    SELECT REQUEST_ID, COALESCE(AGENT_NAME,'(no agent)') AS agent_name,
           SUM(COALESCE(TOKEN_CREDITS,0)) AS credits
    FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COWORK_USAGE_HISTORY
    WHERE START_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND START_TIME < CURRENT_TIMESTAMP()
    GROUP BY 1, 2
)
SELECT
    LOWER(p.prompt) AS norm_prompt,
    MAX(p.prompt)   AS sample_prompt,
    MAX(u.agent_name) AS agent_name,
    COUNT(*)        AS request_count,
    ROUND(SUM(u.credits), 4) AS total_credits
FROM prompts p
JOIN agent_runs r ON r.trace_id = p.trace_id
JOIN usage u      ON u.REQUEST_ID = r.request_id
WHERE p.prompt IS NOT NULL AND LENGTH(p.prompt) > 0
GROUP BY LOWER(p.prompt)
ORDER BY request_count DESC, total_credits DESC;
```

---

## CW4. Model mix

CoWork `CREDITS_GRANULAR` is a nested array (per model / underlying service). This sums credits per model, preserving `unknown`.

```sql
WITH flattened AS (
    SELECT
        COALESCE(NULLIF(cf4.key, ''), 'unknown') AS model_name,
        COALESCE(cf4.value:input::FLOAT,0) + COALESCE(cf4.value:output::FLOAT,0) +
        COALESCE(cf4.value:cache_read_input::FLOAT,0) + COALESCE(cf4.value:cache_write_input::FLOAT,0) AS credits
    FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COWORK_USAGE_HISTORY h,
         LATERAL FLATTEN(input => h.CREDITS_GRANULAR) cf1,
         LATERAL FLATTEN(input => cf1.value) cf2,
         LATERAL FLATTEN(input => cf2.value) cf3,
         LATERAL FLATTEN(input => cf3.value) cf4
    WHERE cf2.key != 'start_time'
      AND h.START_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND h.START_TIME < CURRENT_TIMESTAMP()
)
SELECT model_name, ROUND(SUM(credits),4) AS total_credits
FROM flattened
GROUP BY model_name
ORDER BY total_credits DESC;
```

> If CW4's nesting depth differs in your account, fall back to the simpler per-request credits from the usage view and skip the model split rather than guessing the shape.

---

## CW5. Cache economics — RETIRED

Replaced by conversation stats (CW6/CW7). Do not render. Kept below for reference only; the report no longer includes a cache-economics section (aggregate cache totals were descriptive, not actionable).

<details><summary>retired query</summary>

```sql
WITH c AS (
    SELECT
        SUM(COALESCE(cf4.value:input::FLOAT,0))             AS input_cr,
        SUM(COALESCE(cf4.value:output::FLOAT,0))            AS output_cr,
        SUM(COALESCE(cf4.value:cache_read_input::FLOAT,0))  AS cache_read_cr,
        SUM(COALESCE(cf4.value:cache_write_input::FLOAT,0)) AS cache_write_cr
    FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COWORK_USAGE_HISTORY h,
         LATERAL FLATTEN(input => h.CREDITS_GRANULAR) cf1,
         LATERAL FLATTEN(input => cf1.value) cf2,
         LATERAL FLATTEN(input => cf2.value) cf3,
         LATERAL FLATTEN(input => cf3.value) cf4
    WHERE cf2.key != 'start_time'
      AND h.START_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND h.START_TIME < CURRENT_TIMESTAMP()
),
t AS (
    SELECT
        SUM(COALESCE(tf4.value:input::FLOAT,0))             AS input_tok,
        SUM(COALESCE(tf4.value:cache_read_input::FLOAT,0))  AS cache_read_tok,
        SUM(COALESCE(tf4.value:cache_write_input::FLOAT,0)) AS cache_write_tok
    FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COWORK_USAGE_HISTORY h,
         LATERAL FLATTEN(input => h.TOKENS_GRANULAR) tf1,
         LATERAL FLATTEN(input => tf1.value) tf2,
         LATERAL FLATTEN(input => tf2.value) tf3,
         LATERAL FLATTEN(input => tf3.value) tf4
    WHERE tf2.key != 'start_time'
      AND h.START_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND h.START_TIME < CURRENT_TIMESTAMP()
)
SELECT
    ROUND(c.input_cr,4) AS input_credits, ROUND(c.output_cr,4) AS output_credits,
    ROUND(c.cache_read_cr,4) AS cache_read_credits, ROUND(c.cache_write_cr,4) AS cache_write_credits,
    ROUND(c.cache_write_cr / NULLIF(c.input_cr+c.output_cr+c.cache_read_cr+c.cache_write_cr,0) * 100, 1) AS cache_write_pct_of_credits,
    ROUND(t.cache_read_tok / NULLIF(t.input_tok+t.cache_read_tok+t.cache_write_tok,0) * 100, 1) AS cache_read_hit_pct
FROM c, t;
```

> If the nesting depth differs in your account, skip this section rather than guessing the shape.

</details>

---

## CW6. Conversation length & duration (scoped function — recommended)

A **conversation** = a CoWork / Snowflake Intelligence thread; a **user turn** = one user prompt = one `Agent` root span (one per `trace_id`). Group by `snow.ai.observability.agent.thread_id`.

**Why the scoped function (grounded in the docs):** The docs steer you away from the raw-table approach, because that is exactly where contamination happens — CoCo also emits `Agent` spans and stamps numeric `thread_id`s in the *same* value space as CoWork, so grouping the raw `AI_OBSERVABILITY_EVENTS` table by `thread_id` silently folds CoCo turns into "CoWork" threads (observed: raw-table max 44 turns vs 5 with the function). The recommended path is the **scoped table function** `GET_AI_OBSERVABILITY_EVENTS('<db>','<schema>','<agent>','CORTEX AGENT')`: it is per-agent scoped (enforces `MONITOR` on the agent) and therefore cannot see CoCo spans, so the contamination cannot occur. See [AI Observability for Cortex Agents](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-monitor) and the [LOCAL schema reference](https://docs.snowflake.com/en/sql-reference/local/ai_observability_events).

Because it is per-agent, account-wide coverage = enumerate every agent, then `UNION ALL`. Caveats:
- The **default SI object** (`SNOWFLAKE_COWORK_NAME='SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT'`, `AGENT_NAME IS NULL`) has no named agent and is **not reachable** via the function. Report CoWork conversations as **named-agents only** and note the excluded default-SI turns.
- Caller needs `MONITOR` on each agent; agents the role can't see are silently absent.

**Step 1 — agent inventory** (db/schema/name to iterate):

```sql
SELECT DISTINCT AGENT_DATABASE_NAME, AGENT_SCHEMA_NAME, AGENT_NAME
FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COWORK_USAGE_HISTORY
WHERE START_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
  AND AGENT_NAME IS NOT NULL;
```

**Step 2 — per-agent rows** (the skill emits one block per agent from Step 1 and `UNION ALL`s them into CTE `ev`). Template for one agent:

```sql
SELECT '{AGENT_NAME}' AS agent,
       RECORD_ATTRIBUTES:"snow.ai.observability.agent.thread_id"::STRING AS conv_id,
       TRACE:trace_id::STRING  AS tid,
       RECORD_ATTRIBUTES:"request_id"::STRING AS rid,
       START_TIMESTAMP AS st, TIMESTAMP AS et
FROM TABLE(SNOWFLAKE.LOCAL.GET_AI_OBSERVABILITY_EVENTS('{AGENT_DATABASE_NAME}','{AGENT_SCHEMA_NAME}','{AGENT_NAME}','CORTEX AGENT'))
WHERE RECORD_TYPE='SPAN' AND RECORD:name::STRING='Agent'
  AND TIMESTAMP >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
```

**Step 3 — account-wide summary** over the UNION (CTE `ev`):

```sql
-- WITH ev AS ( <Step-2 block> UNION ALL <Step-2 block> ... ),
WITH conv AS (
    SELECT conv_id, COUNT(DISTINCT tid) AS turns,
           DATEDIFF('second', MIN(st), MAX(et)) AS dur_s
    FROM ev WHERE conv_id IS NOT NULL AND conv_id <> '0' GROUP BY 1
)
SELECT COUNT(*) AS conversations, SUM(turns) AS total_turns,
       ROUND(AVG(turns),2) AS avg_turns, MEDIAN(turns) AS median_turns, MAX(turns) AS max_turns,
       ROUND(AVG(dur_s)/60,1)    AS avg_dur_min,
       ROUND(MEDIAN(dur_s)/60,1) AS median_dur_min,
       ROUND(MAX(dur_s)/60,1)    AS max_dur_min
FROM conv;
```

Duration = wall-clock span from first to last turn (idle gaps included). Window-clipped: a thread started before the window is counted only for its in-window turns.

---

## CW7. Most-expensive conversations (detail)

Same `ev` UNION as CW6, plus per-conversation **credits + tokens** (join each turn's `request_id` to `SNOWFLAKE_COWORK_USAGE_HISTORY`) and the **entry agent** (agent of the earliest span in the thread). Render **one top-10 table** ranked by `credits` (most expensive) with the enriched columns the report shows: **Thread | Agent (entry) | Prompts | Input tok | Output tok | Cache tok | Cache hit | Credits**.

- **Entry surface** is Snowflake Intelligence for every CoWork thread (single surface — state it in the caption, no column). The **agent** column is the entry/orchestrator agent.
- **Tokens** come from `TOKENS_GRANULAR` (nested ARRAY; flatten `tf1..tf4`, `WHERE tf2.key != 'start_time'`; keys `input` / `output` / `cache_read_input` / `cache_write_input`).
- **Cache tok** = cache_read + cache_write. **Cache hit** = `cache_read / (cache_read + cache_write)`.
- Format tokens with K / M suffixes per cell. Credits can be `NULL` when a turn's top-level `request_id` is not present in the usage view (e.g. sub-agent-only credit rows).

```sql
-- WITH ev AS ( <CW6 Step-2 union> ),
, turns AS (SELECT conv_id, COUNT(DISTINCT tid) AS turns
            FROM ev WHERE conv_id IS NOT NULL AND conv_id <> '0' GROUP BY 1),
rids AS (SELECT DISTINCT conv_id, rid FROM ev WHERE rid IS NOT NULL AND conv_id <> '0'),
utok AS (   -- per request: credits + token components from nested TOKENS_GRANULAR
    SELECT h.REQUEST_ID, ANY_VALUE(h.TOKEN_CREDITS) AS credits,
           SUM(tf4.value:input::INT) AS in_tok, SUM(tf4.value:output::INT) AS out_tok,
           SUM(tf4.value:cache_read_input::INT) AS cr_tok, SUM(tf4.value:cache_write_input::INT) AS cw_tok
    FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COWORK_USAGE_HISTORY h,
         LATERAL FLATTEN(input => h.TOKENS_GRANULAR) tf1, LATERAL FLATTEN(input => tf1.value) tf2,
         LATERAL FLATTEN(input => tf2.value) tf3, LATERAL FLATTEN(input => tf3.value) tf4
    WHERE tf2.key != 'start_time' AND h.START_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
    GROUP BY h.REQUEST_ID
),
agg AS (
    SELECT r.conv_id, SUM(u.credits) AS credits, SUM(u.in_tok) AS in_tok, SUM(u.out_tok) AS out_tok,
           SUM(u.cr_tok) AS cr_tok, SUM(u.cw_tok) AS cw_tok
    FROM rids r JOIN utok u ON u.REQUEST_ID = r.rid GROUP BY 1
),
entry AS (SELECT conv_id, agent FROM ev WHERE conv_id <> '0'
          QUALIFY ROW_NUMBER() OVER (PARTITION BY conv_id ORDER BY tid)=1)
SELECT LEFT(t.conv_id,14) AS thread, e.agent, t.turns AS prompts,
       a.in_tok, a.out_tok, (a.cr_tok+a.cw_tok) AS cache_tok,
       ROUND(100.0*a.cr_tok/NULLIF(a.cr_tok+a.cw_tok,0),1) AS cache_hit_pct,
       ROUND(a.credits,2) AS credits
FROM turns t JOIN agg a ON a.conv_id=t.conv_id LEFT JOIN entry e ON e.conv_id=t.conv_id
ORDER BY a.credits DESC NULLS LAST LIMIT 10;   -- most expensive
```
