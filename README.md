# Skills Repo

A collection of useful CoCo skills. Each skill lives in its own folder under `skills/` and packages the instructions, workflow, and reference material needed to run a specialized task against Snowflake.

## Skills

### ai-finops-report

Generates an account-wide **AI FinOps** HTML report covering Snowflake CoCo and CoWork usage. It attributes credits per prompt, ranks the top users, breaks spend down by product / interface / agent, analyzes conversation length and the most-expensive sessions, surfaces model mix and tool usage, semantically clusters popular prompts, and reports verified-query coverage. Read-only — it only runs `SELECT` / `SHOW` / `DESCRIBE` and produces an HTML report.

### agent-observability-report

Produces a repeatable operational monitoring report for a single Cortex Agent over the last N days (default 30), driven by `SNOWFLAKE.LOCAL.AI_OBSERVABILITY_EVENTS`. It answers the questions an agent owner has to answer once an agent is live:

- **Adoption** — is anyone using it? how many users/threads/days? (In your case: 1 user, 16 turns — a signal it's still pre-production.)
- **Reliability** — success vs error rate, and quality-of-service flags like the 100% SLOW status I found.
- **Latency / SLA** — p50/p95/max to know whether UX is acceptable at scale (your p95 was 152s — a real finding).
- **Cost** — token/credit burn per turn, to forecast spend before a wider rollout.
- **Behavior** — which tools/sub-agents actually fire (your HR/Marketing/Sales routing), so you know the config is exercised and where latency is spent.
- **Satisfaction** — thumbs up/down feedback trend.

Read-only — it only runs `SELECT` / `SHOW` / table functions.

### agent-performance-diagnosis

Finds out **why a Cortex Agent is slow** and recommends what to fix first. It asks which agent to look at and over what window (7 / 14 / 30 days, default 30), establishes a latency baseline (p50/p90/p95), then works through the likely causes in order — platform, too many tool calls, slow SQL, slow reasoning, and multi-turn growth — so you get a ranked set of fixes instead of a guess.

Its main value is **not jumping to the wrong conclusion on agents that call other agents**. When an agent routes to sub-agents or runs agentic skills, its own "reasoning time" often just reflects *waiting* on that nested work — so a naive read blames the wrong layer. The skill traces where the time actually goes and points you at the component that's really slow, and it checks your account settings before suggesting a change so you're not handed a fix that's already in place.

Read-only: it only inspects observability data and never modifies the agent. Recommendations are starting points to validate on the specific slow requests it flags.

### Which agent report should I use?

All three read the same observability data but answer different questions:

- **`ai-finops-report`** — *account-wide* AI spend across CoCo + CoWork and **all** agents. Use it for a FinOps/cost overview, not a single agent.
- **`agent-observability-report`** — *one agent, "how is it doing?"* A broad health scorecard (adoption, reliability, latency, cost, feedback). Start here to monitor a live agent or spot a problem.
- **`agent-performance-diagnosis`** — *one agent, "why is it slow?"* A deep latency root-cause investigation with ranked fixes. Use it after observability-report flags a latency issue (e.g. high p95 / mostly-SLOW turns).

Typical flow: `agent-observability-report` to find the problem → `agent-performance-diagnosis` on the same agent to explain and fix it.

## Usage

Point your CoCo skills directory at this repo, or copy a skill folder into your configured skills location. Each skill's `SKILL.md` describes when it is invoked and how it works.
