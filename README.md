# Skills Repo

A collection of useful CoCo skills. Each skill lives in its own folder under `skills/` and packages the instructions, workflow, and reference material needed to run a specialized task against Snowflake.

## Skills

### ai-finops-report

Generates an account-wide **AI FinOps** HTML report covering Snowflake CoCo and CoWork usage. It attributes credits per prompt, ranks the top users, breaks spend down by product / interface / agent, analyzes conversation length and the most-expensive sessions, surfaces model mix and tool usage, semantically clusters popular prompts, and reports verified-query coverage. Read-only — it only runs `SELECT` / `SHOW` / `DESCRIBE` and produces an HTML report.

## Usage

Point your CoCo skills directory at this repo, or copy a skill folder into your configured skills location. Each skill's `SKILL.md` describes when it is invoked and how it works.
