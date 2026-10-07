# Report Template — Agent Observability Report

Fixed layout so runs are comparable week-over-week. Default output is **in-chat markdown + `visualize_data` charts**. Offer an optional HTML export (follow the `html-authoring` skill) and `publish_report` only if the user wants a shareable artifact.

## 1. Headline KPI card (`visualize_data`, `metric_card`, max 4 tiles)

- **Total Requests** — §1 `total_turns` (subvalue: "across N threads").
- **Success Rate** — `success_turns / total_turns` (color green ≥99%, red otherwise; subvalue: error count). If 100% SUCCESS but high `slow_turns`, make the subvalue "X% flagged SLOW".
- **Latency p50 / p95** — §3 (color red if p95 is high; subvalue: "100% flagged SLOW" when applicable).
- **Token Credits** — §8 (subvalue: total tokens). Omit this tile if neither usage view has rows.

## 2. Sections (markdown, in this order)

1. **Header line** — agent FQN + version + window dates + source (`AI_OBSERVABILITY_EVENTS`).
2. **Usage & Adoption** — turns, threads, active days, distinct users, version. Flag single-user traffic as likely pre-production.
3. **Reliability** — success/error counts; call out the `SLOW` QoS flag explicitly (it is not an error).
4. **Latency** — p50/p90/p95/max/avg table; one sentence on what drives it (orchestration depth, tool fan-out).
5. **Conversation Depth** — avg/max turns per thread, single- vs multi-turn split.
6. **Tool & Sub-Agent Execution** — ranked list from §6; separate sub-agent routes (`AgentRouterTool_*`) from tools.
7. **Token Economics** — tokens + credits + avg tokens/turn; **state which usage view supplied the data** and that observability doesn't carry tokens.
8. **User Feedback** — count + positive/negative (or "0 events — no in-product signal this window").
9. **Recommendations** — prioritized, specific, tied to the findings (e.g. investigate tail latency if p95 high; confirm pre-production if single-user; enable feedback capture if 0 events). No time estimates.

## 3. Charts (one `visualize_data` call each, after the relevant section)

- **Daily volume** — `bar_chart`, xKey `day`, yKey `turns` (from §2).
- **Tool & sub-agent invocations** — `horizontal_bar_chart`, xKey `tool`, yKey `invocations` (from §6).
- Keep to 2–3 charts; do not over-visualize small datasets.

## 4. Close

End with the window + source line and an offer to: re-run for a different window, drill into the slowest turns (§5), compare across agent versions, or export/publish HTML.
