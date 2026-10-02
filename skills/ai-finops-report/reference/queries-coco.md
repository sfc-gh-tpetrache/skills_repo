# CoCo-specific queries

Run these only when scope is **CoCo** or **Both**. Base view: `SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COCO_USAGE_HISTORY` (time column `USAGE_TIME`). `{WINDOW_DAYS}` default 30. All read-only.

> CoCo has **no verified-query concept** and **no prompt clustering** (that is CoWork-only). CoCo sections are: interface split, conversations (length/duration + most-expensive sessions), model mix — plus the common blocks (KPIs, top users, cost distribution, reasoning steps, tools, trend) run with the CoCo substitution.

---

## CoCo1. Credits & requests by interface

```sql
SELECT
    LOWER(INTERFACE) AS interface,
    ROUND(SUM(COALESCE(TOKEN_CREDITS,0)), 4) AS credits,
    SUM(TOKENS)                              AS tokens,
    COUNT(DISTINCT REQUEST_ID)               AS requests
FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COCO_USAGE_HISTORY
WHERE USAGE_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
  AND USAGE_TIME < CURRENT_TIMESTAMP()
GROUP BY 1
ORDER BY credits DESC;
```

---

## CoCo2. Cache economics — RETIRED

Replaced by conversation stats (CoCo4/CoCo5). Do not render; the report no longer includes a cache-economics section (aggregate cache totals were descriptive, not actionable). Kept below for reference only.

<details><summary>retired query</summary>

`CREDITS_GRANULAR` / `TOKENS_GRANULAR` are objects keyed by model; each value has `input`, `output`, `cache_read_input`, `cache_write_input`. This splits spend and tokens into those components and derives the cache-read hit % and cache-write share.

```sql
WITH c AS (
    SELECT
        SUM(COALESCE(f.value:input::FLOAT,0))             AS input_cr,
        SUM(COALESCE(f.value:output::FLOAT,0))            AS output_cr,
        SUM(COALESCE(f.value:cache_read_input::FLOAT,0))  AS cache_read_cr,
        SUM(COALESCE(f.value:cache_write_input::FLOAT,0)) AS cache_write_cr
    FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COCO_USAGE_HISTORY h,
         LATERAL FLATTEN(input => h.CREDITS_GRANULAR) f
    WHERE h.USAGE_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND h.USAGE_TIME < CURRENT_TIMESTAMP()
),
t AS (
    SELECT
        SUM(COALESCE(f.value:input::FLOAT,0))             AS input_tok,
        SUM(COALESCE(f.value:cache_read_input::FLOAT,0))  AS cache_read_tok,
        SUM(COALESCE(f.value:cache_write_input::FLOAT,0)) AS cache_write_tok
    FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COCO_USAGE_HISTORY h,
         LATERAL FLATTEN(input => h.TOKENS_GRANULAR) f
    WHERE h.USAGE_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND h.USAGE_TIME < CURRENT_TIMESTAMP()
)
SELECT
    ROUND(c.input_cr,4) AS input_credits,
    ROUND(c.output_cr,4) AS output_credits,
    ROUND(c.cache_read_cr,4) AS cache_read_credits,
    ROUND(c.cache_write_cr,4) AS cache_write_credits,
    ROUND(c.cache_write_cr / NULLIF(c.input_cr+c.output_cr+c.cache_read_cr+c.cache_write_cr,0) * 100, 1) AS cache_write_pct_of_credits,
    ROUND(t.cache_read_tok / NULLIF(t.input_tok+t.cache_read_tok+t.cache_write_tok,0) * 100, 1) AS cache_read_hit_pct
FROM c, t;
```

Per-interface cache split (optional detail — feeds a stacked bar):

```sql
SELECT
    LOWER(h.INTERFACE) AS interface,
    ROUND(SUM(COALESCE(f.value:input::FLOAT,0)),4)             AS input_credits,
    ROUND(SUM(COALESCE(f.value:output::FLOAT,0)),4)            AS output_credits,
    ROUND(SUM(COALESCE(f.value:cache_read_input::FLOAT,0)),4)  AS cache_read_credits,
    ROUND(SUM(COALESCE(f.value:cache_write_input::FLOAT,0)),4) AS cache_write_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COCO_USAGE_HISTORY h,
     LATERAL FLATTEN(input => h.CREDITS_GRANULAR) f
WHERE h.USAGE_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
  AND h.USAGE_TIME < CURRENT_TIMESTAMP()
GROUP BY 1 ORDER BY input_credits DESC;
```

</details>

---

## CoCo3. Model mix

`CREDITS_GRANULAR` is keyed by model name (flatten key = model):

```sql
SELECT
    f.key AS model_name,
    ROUND(SUM(
        COALESCE(f.value:input::FLOAT,0) + COALESCE(f.value:output::FLOAT,0) +
        COALESCE(f.value:cache_read_input::FLOAT,0) + COALESCE(f.value:cache_write_input::FLOAT,0)
    ), 4) AS total_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COCO_USAGE_HISTORY h,
     LATERAL FLATTEN(input => h.CREDITS_GRANULAR) f
WHERE h.USAGE_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
  AND h.USAGE_TIME < CURRENT_TIMESTAMP()
GROUP BY f.key
ORDER BY total_credits DESC;
```

---

## CoCo4. Conversation length & duration

A **conversation** = a CoCo session; a **turn** = one user prompt = one `CodingAgentRun` span (one per trace). Anchor on `CodingAgentRun` and group by `snow.ai.observability.agent.coding_agent.session_id` (the CoCo conversation id).

Trap: `thread_id` is NOT the CoCo conversation key — CoCo stamps `thread_id='0'` as a sentinel, so group on `session_id` here. Best-effort event table (counts can under-report). Window-clipped: a session that started before the window is counted only for its in-window turns. Remove the `TIMESTAMP >=` filter for the all-time view.

Account-wide summary (feeds the "turns per conversation (CoCo)" tiles):

```sql
WITH ev AS (
    SELECT RECORD_ATTRIBUTES:"snow.ai.observability.agent.coding_agent.session_id"::STRING AS conversation_id,
           TRACE:trace_id::STRING AS trace_id,
           TIMESTAMP
    FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
    WHERE RECORD_TYPE='SPAN' AND RECORD:name::STRING='CodingAgentRun'
      AND RECORD_ATTRIBUTES:"snow.ai.observability.agent.coding_agent.session_id" IS NOT NULL
      AND TIMESTAMP >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
),
conv AS (
    SELECT conversation_id,
           COUNT(DISTINCT trace_id) AS turns,
           DATEDIFF('second', MIN(TIMESTAMP), MAX(TIMESTAMP)) AS duration_s
    FROM ev GROUP BY 1
)
SELECT COUNT(*) AS conversations, SUM(turns) AS total_turns,
       ROUND(AVG(turns),2) AS avg_turns, MEDIAN(turns) AS median_turns, MAX(turns) AS max_turns,
       ROUND(AVG(duration_s)/60,1)    AS avg_dur_min,
       ROUND(MEDIAN(duration_s)/60,1) AS median_dur_min,
       ROUND(MAX(duration_s)/60,1)    AS max_dur_min
FROM conv;
```

Top-sessions detail: replace the final SELECT with
`SELECT conversation_id, turns, ROUND(duration_s/60,1) AS duration_min FROM conv ORDER BY turns DESC LIMIT 20;`

Duration = wall-clock span from first to last turn; large maxima reflect sessions resumed across hours/days (idle gaps included).

---

## CoCo5. Most-expensive conversations (detail)

Per-conversation **credits + tokens** join: map each turn's `request_id` (from any `CodingAgent%` span) to `SNOWFLAKE_COCO_USAGE_HISTORY`, grouped by `session_id`. Render **one top-10 table** ranked by `credits` (most expensive) with the enriched columns the report shows: **Session | Surface | Prompts | Input tok | Output tok | Cache tok | Cache hit | Credits**.

- **Surface** = `INTERFACE` (desktop / snowsight / cli) — the entry surface.
- **Tokens** come from `TOKENS_GRANULAR` (OBJECT keyed by model; `input` / `output` / `cache_read_input` / `cache_write_input`). CoCo **input is ~0** because almost all context is served from cache — render as `~0`.
- **Cache tok** = cache_read + cache_write. **Cache hit** = `cache_read / (cache_read + cache_write)` — how well Prompt Cache is performing.
- Format tokens with K / M suffixes per cell. (Duration/started are already in the summary tiles — not repeated here.)

```sql
WITH runs AS (
    SELECT TRACE:trace_id::STRING AS tid,
           RECORD_ATTRIBUTES:"snow.ai.observability.agent.coding_agent.session_id"::STRING AS sid
    FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
    WHERE RECORD_TYPE='SPAN' AND RECORD:name::STRING='CodingAgentRun'
      AND RECORD_ATTRIBUTES:"snow.ai.observability.agent.coding_agent.session_id" IS NOT NULL
      AND TIMESTAMP >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
),
reqmap AS (   -- all request_ids per trace (planning.request_id on steps, request_id on run)
    SELECT DISTINCT TRACE:trace_id::STRING AS tid,
           COALESCE(RECORD_ATTRIBUTES:"snow.ai.observability.agent.planning.request_id"::STRING,
                    RECORD_ATTRIBUTES:"snow.ai.observability.agent.request_id"::STRING) AS rid
    FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
    WHERE RECORD_TYPE='SPAN' AND RECORD:name::STRING LIKE 'CodingAgent%'
      AND TIMESTAMP >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND COALESCE(RECORD_ATTRIBUTES:"snow.ai.observability.agent.planning.request_id"::STRING,
                   RECORD_ATTRIBUTES:"snow.ai.observability.agent.request_id"::STRING) IS NOT NULL
),
utok AS (   -- per request: surface, credits, token components from TOKENS_GRANULAR
    SELECT h.REQUEST_ID, ANY_VALUE(h.INTERFACE) AS interface, ANY_VALUE(h.TOKEN_CREDITS) AS credits,
           SUM(f.value:input::INT) AS in_tok, SUM(f.value:output::INT) AS out_tok,
           SUM(f.value:cache_read_input::INT) AS cr_tok, SUM(f.value:cache_write_input::INT) AS cw_tok
    FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COCO_USAGE_HISTORY h, LATERAL FLATTEN(input => h.TOKENS_GRANULAR) f
    WHERE h.USAGE_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
    GROUP BY h.REQUEST_ID
),
sess AS (
    SELECT r.sid,
           COUNT(DISTINCT r.tid) AS turns,
           LISTAGG(DISTINCT u.interface, '/') AS surface,
           SUM(u.credits) AS credits,
           SUM(u.in_tok) AS in_tok, SUM(u.out_tok) AS out_tok,
           SUM(u.cr_tok) AS cr_tok, SUM(u.cw_tok) AS cw_tok
    FROM reqmap m JOIN runs r ON r.tid = m.tid JOIN utok u ON u.REQUEST_ID = m.rid
    GROUP BY r.sid
)
SELECT LEFT(sid,12) AS session, surface, turns AS prompts,
       in_tok, out_tok, (cr_tok+cw_tok) AS cache_tok,
       ROUND(100.0*cr_tok/NULLIF(cr_tok+cw_tok,0),1) AS cache_hit_pct,
       ROUND(credits,2) AS credits
FROM sess ORDER BY credits DESC NULLS LAST LIMIT 10;   -- most expensive
```
