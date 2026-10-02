# HTML report template & section map

The report is a single self-contained `.html` file. **Follow the `html-authoring` skill** for the broader rules (provenance block, `light-dark()` theming, responsive layout, scrollable table wrappers). Save it to the **current workspace** (durable), never `/tmp`.

## Rendering: inline CSS bars (no library)

**Render all charts as inline CSS bars — no JavaScript, no chart library, no `/libs/` dependency.** The charts here are simple bar comparisons, and a CSS-bar report renders identically in a local file, in CoCo Desktop's report viewer, and when published — avoiding the "blank charts when opened locally" failure mode of library-based rendering.

Pattern (one row per value; `width` = value / series-max as a percentage, computed when writing the file):

```html
<style>
  .bars { margin: 14px 0; }
  .bar-row { display: flex; align-items: center; gap: 10px; margin: 5px 0; }
  .bar-label { flex: 0 0 42%; max-width: 42%; font-size: 13px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
  .bar-track { flex: 1; background: light-dark(#eef2f7,#1f2937); border-radius: 4px; height: 16px; }
  .bar-fill { background: light-dark(#2563eb,#60a5fa); height: 16px; border-radius: 4px; min-width: 2px; }
  .bar-val { flex: 0 0 64px; text-align: right; font-variant-numeric: tabular-nums; font-size: 13px; }
</style>
<div class="bars">
  <div class="bar-row"><span class="bar-label">desktop</span><div class="bar-track"><div class="bar-fill" style="width:100%"></div></div><span class="bar-val">181.86</span></div>
  <div class="bar-row"><span class="bar-label">snowsight</span><div class="bar-track"><div class="bar-fill" style="width:9.8%"></div></div><span class="bar-val">17.77</span></div>
</div>
```

- Compute each bar's `width` percentage yourself from the query results (value ÷ max of that series). Always print the real value in `.bar-val`.
- Use a second fill color (e.g. a `.bar-fill.alt` class) to distinguish a second series (e.g. CoWork vs CoCo).
- Do NOT use chart.js / vega / `/libs/` scripts for these charts. (Only reach for a library if a future chart genuinely needs it — e.g. a time-series line — and even then prefer inline SVG.)
- Still inline every value directly in the HTML; the sandbox has no network access.
- Include the provenance `<script type="application/json" id="snowflake-report-metadata">` block, and record per-section the SQL used.
- Put the run context (account, scope, "last 30 days", generated-at) and the "credits = usage views (source of truth); traces best-effort" note in the header.

## Section map (render only what the scope includes)

| # | Section | Scope | Shape |
|---|---|---|---|
| 0 | Run context + caveats | all | text header |
| 1 | Executive KPIs (+ CoCo-vs-CoWork split) | all (split only in Both) | metric tiles + CSS bars |
| 2 | Requests by user — distribution (high / median / low cohorts) | all | metric tiles + table (C2) |
| 3 | Cost-per-prompt (p50/p95/max) | all | metric tiles |
| 4 | Credits by interface | CoCo / Both | CSS bars (CoCo1) |
| 5 | Credits by agent | CoWork / Both | CSS bars + table (CW1) |
| 6 | Reasoning steps / interaction (complexity) | all | tiles / CSS bars (C5) |
| 7 | Conversations: length, duration & most-expensive threads | all | tiles (CoCo4/CW6) + enriched most-expensive table per surface (CoCo5/CW7): Session·Thread / Surface·Agent / Prompts / Input·Output·Cache tok / Cache hit % / Credits |
| 8 | Model mix | all | CSS bars (CoCo3 / CW4) |
| 9 | Popular prompt clusters | CoWork / Both | table (clustering.md) |
| 10 | Verified-query coverage + VQR candidates (ad-hoc; skill-driven turns excluded) | CoWork / Both | tiles + table (CW2 + clustering.md) |
| 11 | Recommendations | all | text list, grounded in docs |
| 12 | Sources & best practices | all | doc links |

For **CoCo-only**: omit 5, 9, 10 (and the split in 1); section 7 uses CoCo4/CoCo5 only. For **CoWork-only**: omit 4 (interface is a CoCo concept); section 7 uses CW6/CW7 (scoped function, named-agents only). For **Both**: render all, lead section 1 with the comparison, and in section 7 show CoCo and CoWork blocks side by side.

## Terminology — disambiguate "turn" (use consistently in labels and notes)

The word "turn" is overloaded; the report must keep these distinct:
- **Reasoning step** = the agent's internal planning / tool / response span **within one user prompt**. This is section 6 (the complexity signal). Label the tiles **"Reasoning steps / interaction (CoCo/CoWork)"** — never "steps / turn".
- **User turn / interaction** = one user prompt = one `trace_id`. This is what section 7 counts (turns per conversation) and what "turns per user" means. Label these **"turns (user prompts)"**.
- Add a one-line glossary at first use (section 6): *"User turn = one prompt = one trace; reasoning steps = the agent's planning/tool/response spans within a turn."*
- CoCo and CoWork count different span families, so the section-6 averages are **not comparable** across surfaces (CoCo `CodingAgent.Step-N` = coarse coding-agent iterations; CoWork `ReasoningAgentStep*` = fine-grained planning/tool/response sub-spans).

## Per-section plain-language note (REQUIRED on every section)

After **every** section, add a short `.sub` note (1-2 sentences, plain language) with two parts: **what we compute** and **why it matters**. Keep it jargon-free so a non-expert can read the report top to bottom unaided. **Rendered notes MUST stay user-friendly: no span/record names, no `trace_id`/`thread_id`/`request_id`, no function names, no "per the docs" mechanics.** All of that implementation detail belongs in the query references (CoCo4/CoCo5, CW6/CW7) for the author, NOT in the report. Also keep the run-context header note plain (e.g. "Credit figures are the source of truth; prompts/turns/verified-query usage come from best-effort AI usage logs and can slightly under-count; figures cover the last 30 days" — do not name internal views/tables or rendering tech). Examples:
- Executive KPIs: "What: total AI credits and the CoCo-vs-CoWork split. Why: shows where spend concentrates so you know which surface to optimize first."
- Requests by user: "What: how AI requests and credits spread across users, grouped into high / median / low cohorts by request volume. Why: concentration tells you who to engage — power users for enablement and per-user quotas, occasional users for adoption." For a single-user account add: "This account has one active user, so the distribution collapses to that person and the percentiles are all equal." (Author note, do NOT render: cohorts are percentile-based — high ≥ p90, low ≤ p10, else median — from the account-wide union in C2.)
- Reasoning steps / interaction: "What: the average number of internal steps the agent takes to answer one prompt. Why: it reflects **task complexity** and affects speed — higher means the agent is working harder per answer. CoCo and CoWork measure steps differently, so compare each tool only against itself." (Author note, do NOT render: steps/turn is a real complexity signal for CoWork (~2-13) but ~1 for CoCo, which is why the complexity metric lives in this section and is not a column in the section-7 detail tables; the span-family detail stays out of the report.)
- Conversations: "What: how many user prompts (turns) each session/thread holds, how long it ran, and its credits. Why: long or expensive conversations are the biggest optimization targets (CoCo `/compact` for long sessions; CoWork verified queries + custom instructions)." For the CoWork block, add one plain caveat only: "Covers conversations handled by named agents; ad-hoc Default Snowflake Intelligence chats are not included." (Author note, do NOT render: the named-agents-only limit comes from using the scoped per-agent function instead of the raw event table — see CW6/CW7.) The most-expensive table is enriched (CoCo5/CW7): per conversation show prompts, entry surface (CoCo = `INTERFACE` desktop/snowsight/cli; CoWork = Snowflake Intelligence, stated in caption), tokens (input / output / cache), **cache-hit rate** = cache_read / (cache_read + cache_write), and credits. Plain-language note for cache hit: "a high cache-hit rate means Prompt Cache is working well, so cost scales with new work rather than re-sent context." CoCo input is ~0 (context served from cache) — render `~0`; format tokens with K/M suffixes.
- Verified-query coverage: "What: share of text-to-SQL answers backed by a verified query. Why: higher coverage means lower cost, lower latency, and more consistent answers."

## Recommendations (section 12) — derive from the data, grounded in docs

Split recommendations by product. Tie each to a number shown elsewhere in the report, and keep them grounded in the documented best practices below (do NOT invent generic advice).

**Scrutiny rule (REQUIRED before writing any recommendation):** verify every claim against the linked Snowflake docs. State only what the docs actually support. Do NOT dress up general-LLM intuition or account-derived observations as documented Snowflake best practices — if a point is an analytical judgment (e.g. a complexity proxy) or an observation from this account's data (e.g. "model X is ~90% of credits"), label it as such, not as a doc-backed guideline. Specific claims that are NOT in the docs and must be avoided (common overstatements): "Cortex Agents have a finite context window and lose earlier context"; verified queries are a "fast path" that "skips SQL generation / lowers latency and cost" (docs support accuracy/consistency, not latency/cost); "parallelize tool calls"; "model can't be set per tool — use RBAC to constrain tool models". When unsure whether a claim is documented, soften it or drop it.

**CoWork (grounded in CoWork / Cortex Agents docs):**
- **Verified queries** when coverage is low: add verified queries to the semantic views behind the agents for recurring questions — verified queries improve the **accuracy and consistency** of repeated text-to-SQL answers. Point at the VQR-candidate clusters (section 11). (Do not claim a latency/cost "fast path" — not documented.)
- **Instructions over threads:** put durable guidance (tone, tool-routing rules) in the agent's **response/orchestration instructions** so it applies to every conversation. (Docs: agent behavior is shaped by instructions. Do NOT justify this with a "finite context window" claim — not documented.)
- **Agent design (documented in Build agents best practices):** scope each agent narrowly to one high-value use case; attach only the tools it needs; write **purpose-driven tool descriptions** (what it does, which data, when to use / when NOT to use, expected inputs) — docs note vague descriptions cause "cascading failures"/hallucinations; set data-specific defaults in the semantic view (e.g. default date window, exclude internal accounts).
- **Orchestration budget:** set an orchestration **budget** (seconds/tokens, orchestration-only) to cap what a single run consumes. (Do not add "parallelize tool calls".)
- **Model:** use the `auto` orchestration model so Cortex picks the best available model; agents run through cross-region inference. (Do not add the RBAC-per-tool claim.)
- **Governance:** enforce **per-user quotas** to cap each user's AI credit use (and optionally block users at their quota), and use resource budgets to monitor/alert on agent spend. (Documented in Cortex Agents cost + per-user quotas.)

**CoCo (grounded in CoCo docs):**
- **Long / expensive sessions (from section 7):** point at the specific top sessions by credits. `/compact` summarizes context in a long session; CoCo already offloads large tool results and sends per-turn deltas automatically. Frame this around the named outlier sessions, not a blanket "use shorter sessions".
- **Model:** CoCo model is user-selected via `/model`. Checking a complexity proxy (steps / tokens per turn) before suggesting a cheaper model is a reasonable **analytical** step — present it as our analysis, not a Snowflake guideline. Account-specific shares (e.g. "model X is ~90% of credits") are observations, not best practices.
- **Governance:** **per-user quotas** and per-user daily credit limits (`CORTEX_CODE_*_DAILY_EST_CREDIT_LIMIT_PER_USER`) to cap individual users, plus budgets; `COCO_ADDITIONAL_QUERY_TAGS` for team attribution.

## Sources & best practices (section 13) — always include

End the report with a links section so recommendations are traceable. Render only the links relevant to the chosen scope:

- CoWork: [Build agents](https://docs.snowflake.com/en/user-guide/snowflake-cortex/snowflake-cowork/build-agents) · [Troubleshooting & performance](https://docs.snowflake.com/en/user-guide/snowflake-cortex/snowflake-cowork/troubleshooting) · [Reference (verified-query fast path, auto model)](https://docs.snowflake.com/en/user-guide/snowflake-cortex/snowflake-cowork/reference)
- CoCo: [Context management](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code-desktop/context-management) · [Cost controls](https://docs.snowflake.com/en/user-guide/cortex-code/cost-controls) · [Daily credit usage limits](https://docs.snowflake.com/en/user-guide/cortex-code/credit-usage-limit)
- Verified queries: [Best practices for semantic views](https://docs.snowflake.com/en/user-guide/views-semantic/best-practices)
- Observability & cost: [AI Observability](https://docs.snowflake.com/en/user-guide/snowflake-cortex/ai-observability) · [AI cost management and governance](https://docs.snowflake.com/en/user-guide/snowflake-cortex/governance-and-availability/ai-cost-management-and-governance)
