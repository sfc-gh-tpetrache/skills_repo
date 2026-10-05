# Semantic prompt clustering (CoWork only)

Groups near-duplicate CoWork prompts so the report shows "popular questions" rather than 100 one-off phrasings, and flags which recurring questions are **not** backed by a verified query. **Do not run for CoCo scope.**

## Efficiency contract (why this is cheap)

Cost of `AI_EMBED` scales with the number of *distinct* prompts embedded, and the similarity self-join is O(n^2). Both are bounded by doing the cheap work first:

1. **Collapse before embedding** — normalize + `GROUP BY` to distinct prompts (plain SQL, no AI).
2. **Cap before embedding** — keep only the top `{TOP_N}` distinct prompts by `request_count` then `credits`. Embedding runs `{TOP_N}` times at most; the self-join is at most `{TOP_N}^2/2` comparisons.
3. **Embed once** — `AI_EMBED` is computed in a CTE and reused on both sides of the join (never recomputed per comparison).

Defaults: `{TOP_N}` = 150, `{THRESHOLD}` = 0.75, model `snowflake-arctic-embed-l-v2.0`. Lower `{TOP_N}` if the account has very high prompt volume. This stays read-only (no Cortex Search service, no objects created).

## Query

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
verified AS (   -- per-trace verified-query flag (CW2)
    SELECT TRACE:trace_id::VARCHAR AS trace_id,
           MAX(IFF(RECORD_ATTRIBUTES:"snow.ai.observability.agent.tool.sql_execution.verified_query_used"::BOOLEAN,1,0)) AS used_vq
    FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS
    WHERE TIMESTAMP >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE())
      AND RECORD:name::VARCHAR = 'SystemExecuteSQLTool_system_execute_sql'
    GROUP BY 1
),
usage AS (
    SELECT REQUEST_ID, COALESCE(AGENT_NAME,'(no agent)') AS agent_name, SUM(COALESCE(TOKEN_CREDITS,0)) AS credits
    FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COWORK_USAGE_HISTORY
    WHERE START_TIME >= DATEADD('day', -{WINDOW_DAYS}, CURRENT_DATE()) AND START_TIME < CURRENT_TIMESTAMP()
    GROUP BY 1, 2
),
distinct_prompts AS (   -- collapse (cheap, no AI)
    SELECT LOWER(p.prompt) AS norm_prompt,
           MAX(p.prompt)   AS sample_prompt,
           MAX(u.agent_name) AS agent_name,
           COUNT(*) AS request_count,
           ROUND(SUM(u.credits),4) AS total_credits,
           SUM(IFF(v.trace_id IS NOT NULL, 1, 0)) AS sql_turns,          -- turns that ran text-to-SQL
           SUM(COALESCE(v.used_vq, 0))            AS verified_sql_turns  -- of those, how many used a verified query
    FROM prompts p
    JOIN agent_runs r ON r.trace_id = p.trace_id
    JOIN usage u      ON u.REQUEST_ID = r.request_id
    LEFT JOIN verified v ON v.trace_id = p.trace_id
    WHERE p.prompt IS NOT NULL AND LENGTH(p.prompt) > 0
    GROUP BY LOWER(p.prompt)
),
top_prompts AS (    -- cap before embedding
    SELECT * FROM distinct_prompts
    ORDER BY request_count DESC, total_credits DESC
    LIMIT {TOP_N}
),
emb AS (            -- embed once per distinct prompt
    SELECT norm_prompt, sample_prompt, agent_name, request_count, total_credits, sql_turns, verified_sql_turns,
           AI_EMBED('snowflake-arctic-embed-l-v2.0', sample_prompt) AS vec
    FROM top_prompts
),
assign AS (         -- assign each prompt to its most-frequent similar neighbor (greedy canonical)
    SELECT a.norm_prompt AS member,
           b.norm_prompt AS cluster_key,
           b.sample_prompt AS cluster_label,
           a.agent_name,
           a.request_count, a.total_credits, a.sql_turns, a.verified_sql_turns,
           ROW_NUMBER() OVER (PARTITION BY a.norm_prompt
                              ORDER BY b.request_count DESC, VECTOR_COSINE_SIMILARITY(a.vec,b.vec) DESC) AS rn
    FROM emb a
    JOIN emb b ON VECTOR_COSINE_SIMILARITY(a.vec, b.vec) >= {THRESHOLD}
)
SELECT
    MAX(cluster_label)        AS representative_prompt,
    MAX(agent_name)           AS agent_name,
    COUNT(*)                  AS variants,
    SUM(request_count)        AS requests,
    ROUND(SUM(total_credits),4) AS cluster_credits,
    SUM(sql_turns)            AS sql_turns,          -- text-to-SQL turns in this cluster
    -- verified_rate is NULL when the cluster never ran SQL (so it is excluded from VQR candidates)
    ROUND(SUM(verified_sql_turns) / NULLIF(SUM(sql_turns),0), 2) AS verified_rate
FROM assign
WHERE rn = 1
GROUP BY cluster_key
ORDER BY requests DESC, cluster_credits DESC;
```

## Outputs

- **Popular prompt clusters** (section 10): `representative_prompt`, `agent_name`, `variants`, `requests`, `cluster_credits` — over **all** prompts (SQL and non-SQL), ranked by volume/credits.
- **VQR-authoring candidates** (section 11): filter the same clusters to `sql_turns > 0 AND verified_rate < 0.5`, ordered by `requests` / `cluster_credits` desc. These are recurring **ad-hoc text-to-SQL** questions not backed by a verified query.

`verified_rate` is computed **only over ad-hoc text-to-SQL turns** and is `NULL` for clusters that never ran ad-hoc SQL. The per-trace flag `v` (from `queries-cowork.md` CW2) already **excludes skill-driven traces** (any trace with a `ServerSkillTool_%` span), so `sql_turns` here counts ad-hoc SQL turns only. This correctly keeps out both non-SQL questions (Cortex Search, news/web, chart-only) **and** skill-driven turns (packaged skills that run their own SQL — not ad-hoc verified-query candidates). Caption section 11 as "ad-hoc text-to-SQL turns only (skill-driven turns excluded)."
