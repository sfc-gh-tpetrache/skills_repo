# Root-cause playbook

Attribute each slow turn to **one dominant root cause**, checking in this order. Stop at the first cause that explains the bulk of the turn's time budget. Then apply the mapped fixes.

## Decision order

```
R. Router / sub-agent attribution        — resolve this FIRST if the agent routes
0. Platform-side latency        (§1.3)  — rule this out next
1. Too many Analyst calls       (§1.4)
2. Slow SQL execution           (§1.5)
3. Slow reasoning / planning    (§1.6)
+  Multi-turn context accrual   (§3.7)  — overlay on any of the above
```

---

## R. Router / sub-agent attribution — resolve BEFORE §0–§3 for router agents

**Why first:** on a router/main agent, the parent's `ReasoningAgentStepPlanning` span *wraps the routed sub-agent's entire run*. On the parent record this looks like huge `planning.duration` / "reasoning" with SQL ≈ 0 — but it is mostly **wait-on-sub-agent**, not parent LLM reasoning. Skipping this step produces two classic false verdicts: tagging the turn "slow reasoning (§1.6)", and tagging it "platform (§0)" because the *parent's* step count is low (the guide's §1.3 rule "p95 spike + low/normal steps → platform" is **invalid for routers** — low parent step count means work was delegated, not that the platform was slow).

> **Generalize beyond sub-agents:** a long `planning.duration` wraps *any nested execution the step kicked off*, not just a routed sub-agent. The same inflation happens when a step invokes a **server skill / agentic swarm** — e.g. `ServerSkillTool_deep-research` fanning out `TaskTool_SWARM_WORKER` + `TaskTool_SWARM_AUDITOR` spans. A 160s `Planning-N` with `status=SUCCESS`, cached input, and tiny output is almost never a model stall — look inside the span's time window for nested `TaskTool_*` / `ServerSkillTool_*` / `SqlExecution_*` / sub-agent spans before blaming the LLM. Open-ended "dig into why / investigate" prompts commonly trigger the deep-research path, which is multi-worker and expensive by design.

**Detect nested work:** `AgentRouterTool_<SUBAGENT>` (routing), `planning.tool_selection.type = agent_router`, `ServerSkillTool_*`, `TaskTool_SWARM_*`, `CodeExecutionTool_*`, or `SqlExecution_*` spans whose timestamps fall inside a slow planning step's window. Also check the agent spec for an `agent_router` tool + `subagents`, and for attached skills.

**Attribute correctly:** for each slow turn, order all spans by `TIMESTAMP` and find what ran *inside* the slow planning step's window. If a concurrent sub-agent `CORTEX_AGENT_REQUEST` (its own `record_id`/`DATABASE.SCHEMA`) or a swarm of `TaskTool_*` spans accounts for the duration, the time is **nested execution**, not parent reasoning — pivot the diagnosis into whatever ran (re-run §0–§3 against a slow sub-agent; for a swarm/skill, treat it as a feature-path cost).

**Fix:** the real work is in the nested layer, not the router. Diagnose the slowest sub-agent first; for deep-research/swarm paths, scope or guard when they fire and set expectations that exploratory "why" prompts run a multi-agent path. Router-side levers (trim its instructions, `auto` model) barely move p95 when <5% of the turn is parent compute. Parallelize only if a turn fans out to multiple independent tools (§2.4).

> **Quantify it:** use **§6c-A** for `pct_delegated` (how much of total latency is inherited from sub-agents vs. the router's own compute), **§6c-B** for the fan-out distribution (how often one question hits several sub-agents), **§6c-C** for tool-calls-per-question, and **§6b** (now with per-span `p95_s`) to rank *which* sub-agent is slowest. A high `pct_delegated` + a slow `route -> X` row = pivot the diagnosis into sub-agent X.

---

## T. Tool-time attribution (document generation, code execution, MCP) — resolve BEFORE §1.6

**Why here:** like §R, these are turns where "reasoning" is a red herring — but the time is in a **tool**, not a nested agent. Run the §6 tool-usage taxonomy in Phase 2; if a non-SQL tool category carries real seconds, attribute there before considering §1.6 slow reasoning. The §3c `tool_execution_ms` + `unaccounted_ms` buckets quantify it per turn.

**Document / chart generation** (`ServerSkillTool_pdf_generation` / `pptx` / `docx`, `CortexChartToolImpl`): a doc-gen turn typically shows **one long planning step + a paired `CodeExecutionTool_*`**. Read it correctly:
- The long planning step is usually the model **emitting the file's markup/code** (large output decode) — that is *real* generation work, not nested wait, and not a stall. (Proven live: a 74s PDF turn = 41s planning emitting ~4 KB of inline HTML/CSS/Python + a 6.5s `CodeExecutionTool_bash` that renders it.)
- `server_skill.duration` is **~0** for pdf/pptx — do **not** conclude the generation was free. The render seconds are on the paired `CodeExecutionTool_*` (category 5), so read categories 4 + 5 together.
- **Fix:** reduce what's generated, not the reasoning — shrink/simplify the template or output size, cap chart/table volume, or move boilerplate into the skill instead of asking the model to emit it each turn. Trimming planning tokens or swapping the model does little when the cost is decode of a large artifact.

**Code execution / file manipulation** (`CodeExecutionTool_bash/read/write/edit`): time is on `tool.code_execution.duration` / `tool.execution.duration`. Long bash = the script itself (rendering, data munging), not the LLM. Fix the script/data path; this is not a model-latency problem.

**MCP (external tools)** (`ServerMCPTool_*`, e.g. Slack): time is on `tool.server_mcp.duration` and is an **external-dependency / network** cost. Do not treat it as model or SQL latency and do not reach for an account parameter — diagnose the MCP server/endpoint (its own latency, auth, rate limits). High variance here points at the third party, not the agent.

---

## 0. Platform-side latency (§1.3) — check first

**Signal:** the turn's time budget is dominated by waiting that is NOT agent reasoning/SQL — e.g. the slowest `query_id` shows large `queued_overload_time` in `QUERY_HISTORY`, long cold-start, or cross-region round-trips. Latency is erratic (high stddev) rather than consistently high.

> **For router agents, do §R first.** A low *parent* step count with high p95 is NOT a platform signal on a router — it usually means the turn delegated to a slow sub-agent. Only read parent step count as a platform signal on a non-routing agent.

**Fixes:**
- **FIRST verify cross-region is not already enabled** before recommending it — a no-op recommendation otherwise:
  `SHOW PARAMETERS LIKE 'CORTEX_ENABLED_CROSS_REGION' IN ACCOUNT;`
  If `value` is already `ANY_REGION` (or lists regions), **do not** recommend enabling it; the capacity lever is already pulled — move to streaming (§0.1) and a support case citing the slow `query_id`/timestamp.
- If disabled, enable **cross-region inference** so requests aren't blocked waiting for in-region model capacity:
  `ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';`
- Right-size / un-queue the warehouse backing SQL execution (dedicated warehouse, higher concurrency, or auto-scale) if `queued_overload_time` is significant.
- **Single stalled step** (one planning step 10–20× slower than its neighbors at similar token counts) with cross-region already on = a transient model-serving stall, not config-fixable: mitigate with the streaming API (§0.1) and a support case, not an account parameter.
- Re-measure: if erratic latency flattens after this, it was platform — stop here.

---

## 1. Too many Analyst / tool calls (§1.4)

**Signal:** `planning_steps` is high (many `ReasoningAgentStepPlanning-N`), multiple Analyst/tool invocations per turn, each planning step adding ~5–8 s. The agent is re-planning or calling tools serially.

**Fixes:**
- **Parallelize tool execution** (§2.4 / §4.1) so independent tool calls don't run back-to-back.
- **Trim and sharpen instructions** (§2.7 / §2.8 / §4.5) so the planner reaches a tool-call decision in fewer steps.
- Reduce the number of tools/sub-agents exposed if several are rarely the right choice.
- Run **`cortex reflect`** (§4.7) to surface planner warnings that cause extra steps.
- Re-measure: `planning_steps` and total planning time per turn.

---

## 2. Slow SQL execution (§1.5)

**Signal:** `sql_execution_ms` dominates the budget; the slowest `query_id` in `QUERY_HISTORY` shows low partition pruning (`partitions_scanned ≈ partitions_total`), large `bytes_scanned`, or long `compilation_time`. Low **VQR hit rate** (§3d) means Analyst is regenerating SQL instead of reusing verified queries.

**Fixes:**
- **Add Verified Query Repository (VQR) entries** for the common/slow questions so Analyst reuses vetted SQL instead of generating it.
- **Materialize base tables** (§2.5): pre-aggregate or build a narrow serving table so the generated SQL scans less.
- Add clustering / search optimization on the hot predicates of the slow query.
- Re-measure: `sql_execution_ms`, VQR hit %, and `partitions_scanned/total` on the slow query.

---

## 3. Slow reasoning / planning (§1.6)

**Signal:** `planning_ms` dominates even with few steps; **step-0 input tokens are bloated** (well above the ~2,500 floor — e.g. 6,000–15,000+), usually from long system/agent instructions or large tool schemas loaded into context.

**Fixes:**
- **Trim instructions** (§2.7 / §2.8 / §4.5): cut verbose/duplicated guidance; move rarely-needed detail out of the always-on prompt.
- Switch the planner to the **auto model** so Snowflake picks an appropriately fast model instead of a pinned heavier one.
- Enable **cross-region inference** (as in §0) if the pinned model is capacity-starved in-region.
- **Disable chart generation** (§4.8) when charts aren't needed — the experimental `DisableDataToChart: true` flag removes a chart-planning round-trip.
- Re-measure: step-0 input tokens and `planning_ms`.

---

## + Multi-turn context accumulation (§3.7) — overlay

**Signal:** follow-up turns (`first_message_in_thread = FALSE`) are materially slower than first messages (§4 query); `avg_turn_number` is high on the slow turns. Growing thread history inflates planning tokens on every subsequent turn.

**Fixes:**
- Keep threads short / start a new thread for a new task.
- Trim what the agent carries forward (instruction/context hygiene, §2.7/§2.8).
- Combine with the §3 token-trim fixes — context accrual amplifies step-0 token bloat.
- Re-measure: first-vs-follow-up `avg_duration_ms` gap.

---

## Fix index (guide section → action)

| Guide | Fix |
|---|---|
| §1.3 / §0 | `ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION='ANY_REGION'`; right-size warehouse |
| auto model | Let Snowflake pick the planning model instead of pinning a heavy one |
| VQR | Add verified queries for common/slow questions |
| §2.4 / §4.1 | Parallelize tool execution |
| §2.5 | Materialize / pre-aggregate base tables |
| §2.7 / §2.8 / §4.5 | Trim and sharpen agent instructions |
| §4.7 | Run `cortex reflect` to surface planner warnings |
| §4.8 | Disable chart generation (`DisableDataToChart: true`, experimental) |
| §3.7 | Shorten threads; reduce carried-forward context |

**Remember:** these are hypotheses derived from telemetry. Validate each on a live re-test (via `agent-studio/debug`'s write path) before treating it as applied, and compare against the Phase 1 baseline.
