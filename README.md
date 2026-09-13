# n8n Opportunity Radar

An AI-assisted market-intelligence pipeline that continuously ingests signals from public sources, verifies and scores them as evidence-backed commercial opportunities, and routes validated results downstream — built as a 12-module system on n8n, with Claude (Anthropic API) doing the extraction, scoring, and drafting work.

This isn't a single automation — it's a pipeline with the kind of engineering discipline that usually only shows up in production software: locked-prediction testing before deployment, deliberate failure-injection tests on every module, audited before/after correction logs, cost-governed model selection, and a hard human-approval gate before anything execution-capable runs.

## What it does

1. **Ingest** — pulls signals from public sources (Hacker News / Algolia API, Hacker News "Who's Hiring" threads, Stack Exchange API, YouTube Data API v3).
2. **Extract & analyze** — Claude (Haiku, for cost control) extracts structured opportunity data from raw signal text, explicitly instructed to say "insufficient information" rather than guess.
3. **Score** — a deterministic (code-calculated, not LLM-invented) Opportunity Score and a separate Evidence Confidence score, each 0–100.
4. **Verify** — cross-checks extracted claims against source evidence before anything is trusted downstream.
5. **Match & cluster** — matches opportunities against known capabilities and clusters related demand signals together.
6. **Route** — separate radars for content opportunities vs. service opportunities, feeding a productization engine that drafts a next-step recommendation (reserving a more expensive model for these higher-value drafts).
7. **Track & alert** — a historical-intelligence layer joins new signals against past runs, with alerting/reporting for anything above threshold.
8. **Gate** — every record is marked `requires_human_approval_for_execution: true`. Nothing acts on the world without a person saying so.

## Engineering practices worth noting

- **Verified, not assumed.** A green "success" indicator in the workflow UI isn't treated as proof of a correct result — every module is checked against raw output (disk/DB reads), not the UI's execution log alone.
- **Deliberate failure-injection testing.** Each module has a documented test for what happens when its inputs are wrong, missing, or malicious — not just a happy-path test.
- **Cost governance.** Cheap models handle high-volume extraction; a stronger model is reserved for the small number of high-value productization drafts. No workflow runs unmonitored spend.
- **Credential hygiene.** Workflow credentials are referenced by ID, never embedded as values, in every exported workflow.
- **Human-in-the-loop by design.** No module is allowed to take an external action (contacting anyone, publishing, spending) without explicit human sign-off.

## Tech stack

- **n8n** (Community Edition, self-hosted via Docker) — orchestration
- **Claude** via the Anthropic API — extraction, scoring input, and productization drafts
- **Public APIs** — Hacker News / Algolia, Stack Exchange, YouTube Data API v3
- **Docker** — isolated, restart-safe deployment

## What's not included / limitations

- Cross-source corroboration (independently confirming one signal across multiple sources before treating it as high-confidence) is a known current limitation, not yet built.
- This is a research/decision-support pipeline, not an execution engine — every output still requires a human decision before any downstream action is taken.

## Included in this repo

- `/workflows` — 16 exported n8n workflow JSON files covering all 12 pipeline modules (module 2, signal ingestion, ships as 4 separate source adapters: Hacker News, Hacker News "Who's Hiring," Stack Exchange, and YouTube), plus one standalone sample workflow (see below)
- `/samples` — sanitized example output records
- `/docs` — condensed build notes (public-safe version of the original engineering log)

---

## Second project: Sample — Service-Business Lead Auto-Responder

A smaller, standalone demo showing the same engineering pattern applied to a common small-business problem: a hosted intake form that catches a new lead and returns an instant, compliant auto-reply instead of leaving it for a human to get to. Built end-to-end — form integration, workflow logic, AI-assisted reply drafting, and testing.

**Note:** this is a self-authored demo built to illustrate the pattern, not a live production deployment — no real client data or credentials are involved.
