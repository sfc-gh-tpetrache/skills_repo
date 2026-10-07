# Queries — Agent Observability Report

All queries use the observability table function unless noted. Substitute `{DB}`, `{SCHEMA}`, `{AGENT}` (the agent FQN parts) and `{DAYS}` (window, default 30). Durations are **milliseconds** — divide by 1000 for seconds.

```
SNOWFLAKE.LOCAL.GET_AI_OBSERVABILITY_EVENTS('{DB}', '{SCHEMA}', '{AGENT}', 'CORTEX AGENT')
```

> One row per turn is the `record_root` span. Tool/sub-agent calls and feedback are separate spans identified by `RECORD:name`.

## Attribute key index (on `record_root` spans)

| Metric | Attribute key (string unless noted) |
|--------|-------------------------------------|
| Turn marker | `ai.observability.span_type` = `'record_root'` |
| Request ID | `snow.ai.observability.agent.request_id` (also `ai.observability.record_id`) |
| Thread ID | `snow.ai.observability.agent.thread_id` (INTEGER; 0 = stateless) |
| Latency (ms) | `snow.ai.observability.agent.duration` (INTEGER) |
| Status | `snow.ai.observability.agent.status` (`SUCCESS`/…) |
| Status code | `snow.ai.observability.agent.status.code` (`200` = ok) |
| QoS flag | `snow.ai.observability.agent.status.description` (`SLOW` etc.) |
| User question | `ai.observability.record_root.input` |
| Agent answer | `ai.observability.record_root.output` |
| Agent version | `snow.ai.observability.object.version.name` (e.g. `VERSION$3`) |
| User (RESOURCE_ATTRIBUTES) | `snow.user.name`, `snow.session.role.primary.name` |

Discover any other key by flattening: `… , LATERAL FLATTEN(input => t.RECORD_ATTRIBUTES) f` and selecting `f.key`.

## §0 — Probe: span inventory & volume

Confirms there is traffic and reveals which tool/sub-agent spans exist (drives §6).

```sql
SELECT
    RECORD_ATTRIBUTES:"ai.observability.span_type"::string AS span_type,
    RECORD:name::string AS record_name,
    COUNT(*) AS n,
    MIN(TIMESTAMP) AS first_seen,
    MAX(TIMESTAMP) AS last_seen
FROM TABLE(SNOWFLAKE.LOCAL.GET_AI_OBSERVABILITY_EVENTS('{DB}','{SCHEMA}','{AGENT}','CORTEX AGENT'))
WHERE TIMESTAMP > DATEADD(day, -{DAYS}, CURRENT_TIMESTAMP())
GROUP BY 1,2 ORDER BY n DESC;
```

If zero `record_root` rows → no traffic in the window; tell the user and stop.

## §1 — Usage, adoption & reliability

```sql
SELECT
    COUNT(*) AS total_turns,
    COUNT(DISTINCT RECORD_ATTRIBUTES:"snow.ai.observability.agent.thread_id"::string) AS distinct_threads,
    COUNT(DISTINCT TO_DATE(TIMESTAMP)) AS active_days,
    MIN(TIMESTAMP) AS first_request,
    MAX(TIMESTAMP) AS last_request,
    ANY_VALUE(RECORD_ATTRIBUTES:"snow.ai.observability.object.version.name"::string) AS version,
    COUNT_IF(RECORD_ATTRIBUTES:"snow.ai.observability.agent.status"::string = 'SUCCESS') AS success_turns,
    COUNT_IF(RECORD_ATTRIBUTES:"snow.ai.observability.agent.status.code"::string != '200') AS error_turns,
    COUNT_IF(RECORD_ATTRIBUTES:"snow.ai.observability.agent.status.description"::string = 'SLOW') AS slow_turns
FROM TABLE(SNOWFLAKE.LOCAL.GET_AI_OBSERVABILITY_EVENTS('{DB}','{SCHEMA}','{AGENT}','CORTEX AGENT'))
WHERE TIMESTAMP > DATEADD(day, -{DAYS}, CURRENT_TIMESTAMP())
  AND RECORD_ATTRIBUTES:"ai.observability.span_type"::string = 'record_root';
```

Distinct users (RESOURCE_ATTRIBUTES — one distinct user usually means pre-production):

```sql
SELECT
    RESOURCE_ATTRIBUTES:"snow.user.name"::string AS user_name,
    RESOURCE_ATTRIBUTES:"snow.session.role.primary.name"::string AS primary_role,
    COUNT(*) AS events
FROM TABLE(SNOWFLAKE.LOCAL.GET_AI_OBSERVABILITY_EVENTS('{DB}','{SCHEMA}','{AGENT}','CORTEX AGENT'))
WHERE TIMESTAMP > DATEADD(day, -{DAYS}, CURRENT_TIMESTAMP())
  AND RECORD_ATTRIBUTES:"ai.observability.span_type"::string = 'record_root'
GROUP BY 1,2 ORDER BY events DESC;
```

## §2 — Daily trend

```sql
SELECT
    TO_DATE(TIMESTAMP) AS day,
    COUNT(*) AS turns,
    COUNT(DISTINCT RECORD_ATTRIBUTES:"snow.ai.observability.agent.thread_id"::string) AS threads,
    ROUND(AVG(RECORD_ATTRIBUTES:"snow.ai.observability.agent.duration"::float)/1000,1) AS avg_sec
FROM TABLE(SNOWFLAKE.LOCAL.GET_AI_OBSERVABILITY_EVENTS('{DB}','{SCHEMA}','{AGENT}','CORTEX AGENT'))
WHERE TIMESTAMP > DATEADD(day, -{DAYS}, CURRENT_TIMESTAMP())
  AND RECORD_ATTRIBUTES:"ai.observability.span_type"::string = 'record_root'
GROUP BY 1 ORDER BY 1;
```

## §3 — Latency percentiles (seconds)

```sql
SELECT
    ROUND(APPROX_PERCENTILE(RECORD_ATTRIBUTES:"snow.ai.observability.agent.duration"::float,0.50)/1000,1) AS p50_sec,
    ROUND(APPROX_PERCENTILE(RECORD_ATTRIBUTES:"snow.ai.observability.agent.duration"::float,0.90)/1000,1) AS p90_sec,
    ROUND(APPROX_PERCENTILE(RECORD_ATTRIBUTES:"snow.ai.observability.agent.duration"::float,0.95)/1000,1) AS p95_sec,
    ROUND(MIN(RECORD_ATTRIBUTES:"snow.ai.observability.agent.duration"::float)/1000,1) AS min_sec,
    ROUND(MAX(RECORD_ATTRIBUTES:"snow.ai.observability.agent.duration"::float)/1000,1) AS max_sec,
    ROUND(AVG(RECORD_ATTRIBUTES:"snow.ai.observability.agent.duration"::float)/1000,1) AS avg_sec
FROM TABLE(SNOWFLAKE.LOCAL.GET_AI_OBSERVABILITY_EVENTS('{DB}','{SCHEMA}','{AGENT}','CORTEX AGENT'))
WHERE TIMESTAMP > DATEADD(day, -{DAYS}, CURRENT_TIMESTAMP())
  AND RECORD_ATTRIBUTES:"ai.observability.span_type"::string = 'record_root';
```

To attribute latency to a tool, re-run §6 grouping and compare, or inspect the slowest turns via `ai.observability.record_root.input` ordered by `agent.duration` DESC.

## §4 — Conversation depth (turns per thread)

```sql
WITH per_thread AS (
    SELECT RECORD_ATTRIBUTES:"snow.ai.observability.agent.thread_id"::string AS thread_id, COUNT(*) AS turns
    FROM TABLE(SNOWFLAKE.LOCAL.GET_AI_OBSERVABILITY_EVENTS('{DB}','{SCHEMA}','{AGENT}','CORTEX AGENT'))
    WHERE TIMESTAMP > DATEADD(day, -{DAYS}, CURRENT_TIMESTAMP())
      AND RECORD_ATTRIBUTES:"ai.observability.span_type"::string = 'record_root'
    GROUP BY 1
)
SELECT COUNT(*) AS threads, ROUND(AVG(turns),2) AS avg_turns_per_thread, MAX(turns) AS max_turns,
       COUNT_IF(turns = 1) AS single_turn_threads, COUNT_IF(turns > 1) AS multi_turn_threads
FROM per_thread;
```

## §5 — (reserved) slowest turns drill-down (optional)

```sql
SELECT
    RECORD_ATTRIBUTES:"ai.observability.record_root.input"::string AS question,
    ROUND(RECORD_ATTRIBUTES:"snow.ai.observability.agent.duration"::float/1000,1) AS sec,
    RECORD_ATTRIBUTES:"snow.ai.observability.agent.status.description"::string AS qos,
    TIMESTAMP
FROM TABLE(SNOWFLAKE.LOCAL.GET_AI_OBSERVABILITY_EVENTS('{DB}','{SCHEMA}','{AGENT}','CORTEX AGENT'))
WHERE TIMESTAMP > DATEADD(day, -{DAYS}, CURRENT_TIMESTAMP())
  AND RECORD_ATTRIBUTES:"ai.observability.span_type"::string = 'record_root'
ORDER BY sec DESC LIMIT 5;
```

## §6 — Tool & sub-agent execution

Sub-agent routes appear as `AgentRouterTool_<SUBAGENT>`; tools as `SystemExecuteSQLTool_*`, `ServerSkillTool_*`, `CodeExecutionTool_*`, `SemanticContextTool_*`, `ToolCall-*`, etc.

```sql
SELECT
    RECORD:name::string AS tool_span,
    COUNT(*) AS invocations,
    MIN(TIMESTAMP) AS first_used,
    MAX(TIMESTAMP) AS last_used
FROM TABLE(SNOWFLAKE.LOCAL.GET_AI_OBSERVABILITY_EVENTS('{DB}','{SCHEMA}','{AGENT}','CORTEX AGENT'))
WHERE TIMESTAMP > DATEADD(day, -{DAYS}, CURRENT_TIMESTAMP())
  AND (RECORD:name::string ILIKE '%Tool%' OR RECORD:name::string ILIKE '%Router%'
       OR RECORD:name::string ILIKE '%SqlExecution%' OR RECORD:name::string ILIKE '%Skill%'
       OR RECORD:name::string ILIKE '%CodeExecution%')
GROUP BY 1 ORDER BY invocations DESC;
```

## §7 — User feedback

```sql
SELECT COUNT(*) AS feedback_events
FROM TABLE(SNOWFLAKE.LOCAL.GET_AI_OBSERVABILITY_EVENTS('{DB}','{SCHEMA}','{AGENT}','CORTEX AGENT'))
WHERE TIMESTAMP > DATEADD(day, -{DAYS}, CURRENT_TIMESTAMP())
  AND RECORD:name::string = 'CORTEX_AGENT_FEEDBACK';
```

If > 0, pull detail (positive/negative + text) by selecting the feedback rows and flattening `RECORD_ATTRIBUTES`. Note: CoCo thumbs-up/down is product feedback to Snowflake, not stored here.

## §8 — Token & credit economics (usage views, NOT observability)

Observability spans have no token totals. Query **both** views; use whichever has rows for the agent. CoWork / Snowflake Intelligence traffic → `SNOWFLAKE_COWORK_USAGE_HISTORY`; API / Teams / external → `CORTEX_AGENT_USAGE_HISTORY`.

```sql
-- API / Teams / external
SELECT COUNT(DISTINCT REQUEST_ID) AS requests, SUM(TOKENS) AS total_tokens, SUM(TOKEN_CREDITS) AS token_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AGENT_USAGE_HISTORY
WHERE AGENT_NAME = '{AGENT}' AND START_TIME > DATEADD(day, -{DAYS}, CURRENT_TIMESTAMP());

-- CoWork / Snowflake Intelligence
SELECT COUNT(DISTINCT REQUEST_ID) AS requests, SUM(TOKENS) AS total_tokens, SUM(TOKEN_CREDITS) AS token_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COWORK_USAGE_HISTORY
WHERE AGENT_NAME = '{AGENT}' AND START_TIME > DATEADD(day, -{DAYS}, CURRENT_TIMESTAMP());
```

**Reconcile:** the view's `requests` should equal §1 `total_turns`. If they differ, note telemetry drop or ACCOUNT_USAGE lag (~1h). To scope by exact agent object (if names collide across schemas), filter also on `AGENT_DATABASE_NAME = '{DB}'` and `AGENT_SCHEMA_NAME = '{SCHEMA}'`.
