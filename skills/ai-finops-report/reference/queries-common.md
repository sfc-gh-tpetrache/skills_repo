# Common queries (scope-agnostic)

These blocks power sections that exist for every scope. They are parameterized — substitute the tokens per the scope table below, then run. All blocks are **read-only**.

> Note: semantic **prompt clustering is NOT here** — it is CoWork-only and lives in `queries-cowork.md` + `clustering.md`. Do not run it for CoCo scope.

## Substitution tokens

| Token | CoCo | CoWork |
|---|---|---|
| `{USAGE_VIEW}` | `SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COCO_USAGE_HISTORY` | `SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COWORK_USAGE_HISTORY` |
| `{TIME_COL}` | `USAGE_TIME` | `START_TIME` |
| `{STEP_SPAN}` (event-table span prefix) | `CodingAgent.Step-` | `ReasoningAgentStep` |

`{WINDOW_DAYS}` = number of days (default **30** = "last month"). Window predicate used everywhere:

```
{TIME_COL} >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE()) AND {TIME_COL} < CURRENT_TIMESTAMP()
```

For **Both** scope, run each per-product block for CoCo and CoWork and `UNION ALL` with a `source` label (pattern shown in C1). Always report credits as **AI credits**, not USD.

---

## C1. Executive KPIs

Per product (substitute for the chosen product):

```sql
SELECT
    ROUND(SUM(COALESCE(TOKEN_CREDITS,0)), 4) AS total_credits,
    SUM(TOKENS)                              AS total_tokens,
    COUNT(DISTINCT REQUEST_ID)               AS requests,
    COUNT(DISTINCT COALESCE(USER_NAME, TO_VARCHAR(USER_ID))) AS users,
    ROUND(SUM(COALESCE(TOKEN_CREDITS,0)) / NULLIF(COUNT(DISTINCT REQUEST_ID),0), 4) AS avg_credits_per_request
FROM {USAGE_VIEW}
WHERE {TIME_COL} >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
  AND {TIME_COL} < CURRENT_TIMESTAMP();
```

**Both scope** — one row per product for the comparison tiles/split:

```sql
SELECT 'CoCo' AS source,
    ROUND(SUM(COALESCE(TOKEN_CREDITS,0)),4) AS credits,
    COUNT(DISTINCT REQUEST_ID) AS requests
FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COCO_USAGE_HISTORY
WHERE USAGE_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE()) AND USAGE_TIME < CURRENT_TIMESTAMP()
UNION ALL
SELECT 'CoWork',
    ROUND(SUM(COALESCE(TOKEN_CREDITS,0)),4),
    COUNT(DISTINCT REQUEST_ID)
FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COWORK_USAGE_HISTORY
WHERE START_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE()) AND START_TIME < CURRENT_TIMESTAMP();
```

---

## C2. Top users & request distribution (high / median / low)

Per-user **top list** (single surface — substitute `{USAGE_VIEW}` / `{TIME_COL}`):

```sql
SELECT
    COALESCE(USER_NAME, TO_VARCHAR(USER_ID)) AS user_name,
    ROUND(SUM(COALESCE(TOKEN_CREDITS,0)), 4) AS credits,
    COUNT(DISTINCT REQUEST_ID)               AS requests,
    SUM(TOKENS)                              AS tokens,
    ROUND(SUM(COALESCE(TOKEN_CREDITS,0)) / NULLIF(COUNT(DISTINCT REQUEST_ID),0), 4) AS avg_credits_per_request
FROM {USAGE_VIEW}
WHERE {TIME_COL} >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
  AND {TIME_COL} < CURRENT_TIMESTAMP()
GROUP BY 1
ORDER BY credits DESC
LIMIT 25;
```

**Requests-by-user distribution** (account-wide across both surfaces; renders the "Requests by user" section near the top). Assigns each user a **cohort** by request-volume percentile — `high` = at/above p90, `low` = at/below p10, else `median` — and surfaces the top-user share. For a **single-user account** every percentile equals that user; render the one row with cohort "sole user" and a note that the distribution collapses to one person.

```sql
WITH u AS (
  SELECT USER_NAME, REQUEST_ID, TOKEN_CREDITS
  FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COCO_USAGE_HISTORY
  WHERE USAGE_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
  UNION ALL
  SELECT USER_NAME, REQUEST_ID, TOKEN_CREDITS
  FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COWORK_USAGE_HISTORY
  WHERE START_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
  -- scope=CoCo only → keep the first SELECT; scope=CoWork only → keep the second
),
per_user AS (
  SELECT USER_NAME,
         COUNT(DISTINCT REQUEST_ID) AS requests,
         ROUND(SUM(TOKEN_CREDITS),2) AS credits
  FROM u GROUP BY USER_NAME
),
pct AS (
  SELECT PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY requests) AS p90,
         PERCENTILE_CONT(0.1) WITHIN GROUP (ORDER BY requests) AS p10,
         SUM(credits) AS total_credits
  FROM per_user
)
SELECT pu.USER_NAME,
       CASE WHEN (SELECT COUNT(*) FROM per_user)=1 THEN 'sole user'
            WHEN pu.requests >= p.p90 THEN 'high'
            WHEN pu.requests <= p.p10 THEN 'low'
            ELSE 'median' END AS cohort,
       pu.requests, pu.credits,
       ROUND(100.0*pu.credits/NULLIF(p.total_credits,0),1) AS pct_credits
FROM per_user pu CROSS JOIN pct p
ORDER BY pu.requests DESC LIMIT 25;
```

Summary tiles (active users, requests/user median·p90·min, top-user share) come from `per_user` / `pct` above:

```sql
WITH per_user AS ( /* as above */ )
SELECT COUNT(*) AS active_users,
       MEDIAN(requests) AS median_req,
       PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY requests) AS p90_req,
       MIN(requests) AS min_req,
       ROUND(100.0*MAX(credits)/NULLIF(SUM(credits),0),1) AS top_user_pct
FROM per_user;
```

---

## C3. Cost-per-prompt distribution

Percentile stats (request ~= prompt; `PARENT_REQUEST_ID` does not reliably chain turns):

```sql
SELECT
    COUNT(*)                                                                   AS requests,
    ROUND(AVG(COALESCE(TOKEN_CREDITS,0)), 4)                                    AS avg_credits,
    ROUND(MEDIAN(COALESCE(TOKEN_CREDITS,0)), 4)                                 AS p50_credits,
    ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY COALESCE(TOKEN_CREDITS,0)), 4) AS p95_credits,
    ROUND(MAX(COALESCE(TOKEN_CREDITS,0)), 4)                                    AS max_credits
FROM {USAGE_VIEW}
WHERE {TIME_COL} >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
  AND {TIME_COL} < CURRENT_TIMESTAMP();
```

Banded histogram (feeds a bar chart):

```sql
SELECT
    CASE
        WHEN COALESCE(TOKEN_CREDITS,0) < 0.1 THEN '1) < 0.1'
        WHEN COALESCE(TOKEN_CREDITS,0) < 0.5 THEN '2) 0.1-0.5'
        WHEN COALESCE(TOKEN_CREDITS,0) < 1   THEN '3) 0.5-1'
        WHEN COALESCE(TOKEN_CREDITS,0) < 3   THEN '4) 1-3'
        ELSE '5) >= 3'
    END AS band,
    COUNT(*) AS requests,
    ROUND(SUM(COALESCE(TOKEN_CREDITS,0)), 2) AS band_credits
FROM {USAGE_VIEW}
WHERE {TIME_COL} >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
  AND {TIME_COL} < CURRENT_TIMESTAMP()
GROUP BY band ORDER BY band;
```

---

## C4. Daily trend (with week-over-week)

Reads 7 extra days to compute the WoW lag; output filtered back to the window.

```sql
WITH daily AS (
    SELECT DATE({TIME_COL}) AS d,
           ROUND(SUM(COALESCE(TOKEN_CREDITS,0)),4) AS credits,
           COUNT(DISTINCT REQUEST_ID) AS requests
    FROM {USAGE_VIEW}
    WHERE {TIME_COL} >= DATEADD('day', -({WINDOW_DAYS}+7), CURRENT_DATE())
      AND {TIME_COL} < CURRENT_TIMESTAMP()
    GROUP BY 1
)
SELECT d, credits, requests,
       ROUND((credits - LAG(credits,7) OVER (ORDER BY d))
             / NULLIF(LAG(credits,7) OVER (ORDER BY d),0) * 100, 1) AS wow_pct
FROM daily
WHERE d >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
ORDER BY d;
```

---

## C5. Turns / agentic depth (event table)

Steps per turn, per `trace_id`. `{STEP_SPAN}` = `CodingAgent.Step-` (CoCo) or `ReasoningAgentStep` (CoWork).

```sql
WITH steps AS (
    SELECT TRACE:trace_id::VARCHAR AS trace_id, COUNT(*) AS step_count
    FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
    WHERE TIMESTAMP >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND RECORD_TYPE = 'SPAN'
      AND RECORD:name::VARCHAR LIKE '{STEP_SPAN}%'
    GROUP BY 1
)
SELECT step_count AS steps_in_turn, COUNT(*) AS turns
FROM steps GROUP BY 1 ORDER BY 1;
```

---

## C6. Tool & skill usage + latency (event table)

Tool spans across either product; durations can be negative on wrapper spans, so filter `> 0`.

```sql
SELECT
    RECORD:name::VARCHAR AS tool_span,
    COUNT(*) AS calls,
    ROUND(AVG(CASE WHEN DATEDIFF('millisecond', START_TIMESTAMP, TIMESTAMP) > 0
                   THEN DATEDIFF('millisecond', START_TIMESTAMP, TIMESTAMP) END), 0) AS avg_ms
FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
WHERE TIMESTAMP >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
  AND RECORD_TYPE = 'SPAN'
  AND (RECORD:name::VARCHAR ILIKE '%Tool%' OR RECORD:name::VARCHAR ILIKE 'ToolCall%')
GROUP BY 1
ORDER BY calls DESC
LIMIT 30;
```

> Scoping tool/turn queries to one product: the event table has no usage `INTERFACE` column, but span families are product-specific (`CodingAgent.*` = CoCo; `ReasoningAgentStep*` / `Agent` / `CORTEX_AGENT_REQUEST` = CoWork). For a strict single-product tool view, filter span names to that family.
