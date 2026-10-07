# Report template

Default output is in-chat markdown plus `visualize_data` charts. Offer an optional HTML export (follow the `html-authoring` skill) and `publish_report` for a shareable artifact. Keep it tight — this is a diagnosis, not a dashboard.

---

## 1. Header

> **Agent:** `DB.SCHEMA.AGENT` · **Window:** last N days · **Turns analyzed:** M

One-line verdict: the dominant root cause and the headline latency, e.g.
*"p95 = 153 s over 16 turns, 100% flagged SLOW. Dominant cause: latency inherited from routed sub-agents (11/12 planning turns delegate) + a deep-research swarm on exploratory prompts — not parent reasoning."*

## 2. Baseline latency profile (§1.2)

`metric_card` row: total requests, avg, p50, p90, p95, stddev (seconds). Note the SLOW/TAINTED share. **Label this the baseline** — every recommendation references it.

## 3. Aggregate diagnostics by root cause (§R, §1.3–§1.6, §3.7)

Work the decision order and report what each check found: router/nested-execution (§R — routed sub-agent & skill/swarm span counts), tool-time (§T — document generation, code execution, MCP), platform (§1.3 — daily p95 vs steps, cross-region setting), too-many-calls (§1.4 — step distribution + tool breakdown), slow SQL (§1.5 — time budget, VQR, slowest `query_id`), slow reasoning (§1.6 — per-step duration, step-0 vs step-1+ tokens), multi-turn (§3.7 — first vs follow-up). `metric_card` / small `table` per check; a `bar_chart` of daily p95 and a `pie_chart` of sub-agent routing where relevant.

For a **router** agent, lead §R with the quantified split: a `metric_card` of `router_own_s` vs `subagent_s` and `pct_delegated` (§6c-A), a small `table` ranking sub-agents by `p95_s` + volume (§6b `route -> ` rows), and the fan-out distribution (§6c-B) — plus tool-calls-per-question (§6c-C, avg/p95/max) for the §1.4 read. A high `pct_delegated` is the headline: the router is a pass-through and the latency is in the sub-agents.

## 3b. Tool-usage mix (§6, §T)

Run the §6 tool-usage taxonomy and report **what the agent actually does**. A `table` (or `pie_chart`) of the category rollup — routing / SQL & semantic / retrieval / **code execution & file manipulation** / **document & chart generation** / **MCP** / other server skills — with `calls`, `turns_used_in`, `total_s`, and `pct_of_tool_time`. Then call out, in one line each:
- **Document / chart generation** share — and the reminder that `pdf_generation`/`pptx` report ~0 duration while the real render time sits in the paired `code execution` category (read 4+5 together).
- **File manipulation / code execution** share — long bash is the script, not the model.
- **MCP** share — external-dependency time; variance points at the third-party server, not the agent.

This is what lets the reader see a document-generation or MCP-bound workload instead of mis-reading it as "slow reasoning."

## 4. Slowest-turn span attribution (§R)

For the 1–2 slowest turns, show the **span timeline ordered by `TIMESTAMP`** and attribute the big `planning.duration` to what ran inside it (routed sub-agent `CORTEX_AGENT_REQUEST`, `ServerSkillTool_*` / `TaskTool_SWARM_*` swarm, or `SqlExecution_*`). Call out the slowest `query_id` + its `QUERY_HISTORY` stats only where SQL actually dominates. This is the step that prevents mis-tagging inherited wait as parent reasoning.

## 5. Common patterns (§R, §1.4–§1.6, §3.7)

Bulleted findings aggregated across the window, e.g.:
- Dominant cause and the evidence for it (routed sub-agent / skill-swarm wrap, slow table, VQR misses, step-0 token bloat, follow-ups slower than first turns).
- For routers: how many turns delegate, `pct_delegated` (§6c-A), the fan-out profile (§6c-B — how often one question hits several sub-agents), and which sub-agents are slowest by p95 (§6b).
- First-vs-follow-up gap (§3.7) if present.

## 6. Prioritized recommendations

Ordered by expected impact. **Each item:** the fix, the guide section, the pattern it addresses, and **the metric to re-measure** against the baseline.

| # | Fix (guide §) | Addresses | Re-measure |
|---|---|---|---|
| 1 | Diagnose the nested sub-agent/skill (§R) | router inherits sub-agent / swarm latency | that sub-agent's own p95 |
| 2 | Trim instructions (§2.7/§2.8/§4.5) | step-0 token bloat | step-0 input tokens, planning_ms |
| 3 | Add VQRs | low VQR hit %, regenerated SQL | VQR hit %, sql_generation_ms |
| 4 | Materialize base tables (§2.5) | slow SQL / poor pruning | sql_execution_ms, partitions_scanned/total |
| 5 | Cross-region inference (§1.3) **— only if not already enabled** | erratic / capacity-bound latency | p95, stddev |
| 6 | Parallelize tools (§2.4/§4.1) | many serial tool calls | planning_steps, total planning time |
| 7 | Disable chart gen (§4.8) | chart-planning overhead | planning_ms |
| 8 | Shrink the generated artifact (§T) | document generation — large markup decode + render | planning_ms on the gen step, code-exec total_s |
| 9 | Diagnose the MCP server (§T) | MCP / external-tool latency | `tool.server_mcp.duration`, that server's own latency |

(Include only the rows that apply.)

## 7. Caveats & next step

State the caveats that apply (SLOW = QoS flag not error; latency in ms; QUERY_HISTORY lag; recommendations are hypotheses to validate on a live re-test).

**Always close with the per-turn debug guideline.** List the specific slowest `request_id`(s) from Phase 3 and recommend running the `agent-studio` → `debug` sub-skill (analysis-only) on each to confirm root cause before applying any fix — e.g.:
> To validate, run `agent-studio/debug` on the slowest turns: `<request_id_1>`, `<request_id_2>`. For a router agent, also run this whole diagnosis against the sub-agent the slow turn routed to (`<DB.SCHEMA.SUBAGENT>`), since that's where the time is actually spent.

Then offer to: re-run for a different window, run the diagnosis on a routed sub-agent, or export/publish the report.
