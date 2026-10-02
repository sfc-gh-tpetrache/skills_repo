# Workflow

Execute in order. **Step 0 (permissions) runs before everything else** — do not ask inputs, run analysis, or generate anything until access is confirmed.

## Step 0 — Permissions preflight (MANDATORY, FIRST)

The report reads `SNOWFLAKE.ACCOUNT_USAGE` usage views and the `SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS` event table. Confirm the current role can read them before doing anything else.

1. Capture context: `SELECT CURRENT_USER() AS usr, CURRENT_ROLE() AS role;`
2. Probe each required source with a zero-scan authorization check (`WHERE 1=0` prunes all partitions, so it is cheap and tests only the SELECT privilege):

```sql
SELECT COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COCO_USAGE_HISTORY   WHERE 1=0;  -- CoCo / Both
SELECT COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWFLAKE_COWORK_USAGE_HISTORY WHERE 1=0;  -- CoWork / Both
SELECT COUNT(*) FROM SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS                WHERE 1=0;  -- any scope
```

Probe only the sources the chosen scope needs — but note scope is asked in Step 1, so either (a) probe the event table + both usage views up front, or (b) ask scope first, then probe. Prefer probing all three up front so the access message is complete.

3. If any probe fails (error contains `does not exist or not authorized` / `Insufficient privileges`), **stop and show this message** — do not partially continue:

> **Can't run the AI FinOps report — missing access.**
> Running as role `<CURRENT_ROLE>` (user `<CURRENT_USER>`). The following are not readable:
> - `<failed source(s)>`
>
> Ask an `ACCOUNTADMIN` to grant your role:
> ```sql
> -- Account Usage AI views (credits/tokens)
> GRANT DATABASE ROLE SNOWFLAKE.USAGE_VIEWER TO ROLE <your_role>;
> GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE <your_role>;
> -- AI Observability event table (prompts/traces/verified_query_used)
> GRANT APPLICATION ROLE SNOWFLAKE.AI_OBSERVABILITY_READER TO ROLE <your_role>;
> ```
> Then re-run. (Note: content in the event table is redacted unless the role also holds the `READ UNREDACTED AI OBSERVABILITY EVENTS TABLE` account privilege — metadata and credits still work without it.)

If only the event table is missing but a usage view is readable, you may offer a **reduced report** (credits/users/trend/cost-distribution from the usage view only — no prompts, turns, tools, or verified-query coverage) and let the user decide.

Only proceed to Step 1 when the sources for the chosen scope are confirmed readable.

## Step 1 — Input (scope only)

Ask with `ask_user_question` for the **scope** — the only input:
- **Scope** — **CoCo**, **CoWork**, or **Both** (default Both).

**Period is fixed at last month (`{WINDOW_DAYS}=30`) — do NOT ask the user for a period.**

Confirm the active connection is the account to analyze.

## Step 2 — Run queries (scope if/else)

```
IF scope == CoCo:
    run queries-common.md with CoCo substitution (C1-C6)
    run queries-coco.md (CoCo1, CoCo3, CoCo4, CoCo5)   # CoCo2 cache RETIRED
    # no clustering, no verified-query coverage
ELSE IF scope == CoWork:
    run queries-common.md with CoWork substitution (C1-C6)
    run queries-cowork.md (CW1-CW4, CW6, CW7)          # CW5 cache RETIRED
    run clustering.md  (sections 10 & 11)
ELSE (Both):
    run both substitutions of queries-common.md
    run queries-coco.md AND queries-cowork.md
    run clustering.md
    build the CoCo-vs-CoWork comparison (C1 Both variant)
```

**Conversations (section 7):** C5 (queries-common) is the **reasoning-steps-per-interaction** complexity metric (section 6), NOT conversations. Section 7 conversation stats come from **CoCo4/CoCo5** (CoCo) and **CW6/CW7** (CoWork). CoWork conversation queries use the **scoped table function** `GET_AI_OBSERVABILITY_EVENTS(...,'CORTEX AGENT')` per the docs — enumerate agents from `SNOWFLAKE_COWORK_USAGE_HISTORY` (CW6 Step 1) and `UNION ALL` per agent. This is **named-agents only**: the default SI object (`AGENT_NAME IS NULL`) is not reachable via the function, and the caller needs `MONITOR` on each agent. Do NOT substitute the raw `AI_OBSERVABILITY_EVENTS` table for CoWork conversations — CoCo's shared `thread_id` space contaminates it.

Reconcile headline credits against a direct `SUM(TOKEN_CREDITS)` on each usage view. Never present event-table numbers as spend.

## Step 3 — Cluster prompts (CoWork / Both only)

Run `clustering.md` (collapse -> cap top-N -> embed once -> self-join). Produces popular clusters (section 10) and VQR candidates (`verified_rate < 0.5`, section 11).

## Step 4 — Assemble the report

Build the HTML per `report-template.md` + the `html-authoring` skill. Render only the sections the scope includes. Save to the current workspace. Open it with `coco <path>` if helpful.

## Step 5 — Summarize & offer scheduling

Print a short text summary (headline credits, top user, CoCo-vs-CoWork split, biggest VQR candidate) and the recommendations. Offer to schedule recurring runs via `cortex automation`.
