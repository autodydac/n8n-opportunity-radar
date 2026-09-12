# UOR-02 Adapter #1 Extension — "Who is Hiring?" Monthly Thread — Build Log

## Status as of 2026-09-08, ~00:30 UTC: COMPLETE.

**Environment:** n8n Community Edition, self-hosted via Docker, localhost:5678.

**Framing:** documented as an extension of Adapter #1 (Hacker News), not a new module number, per instruction. Uses the same already-vetted, already-rate-limit-confirmed Algolia HN API — zero new credentials, zero new governance work.

---

## Design findings, all verified live before building (three real deviations from the original task framing)

1. **"Ask HN: Freelancer? Seeking freelancer?" is discontinued** — the most recent thread is from October 2016, confirmed via a live Algolia search. Not included. Only "Who is hiring?" and "Who wants to be hired?" are actively recurring (both confirmed live for September 2026).

2. **A real direction-of-intent problem, caught before building, not after — "Who wants to be hired?" was dropped entirely.** Damian raised the concern that self-ads ("available for hire, $80/hr") are supply-side, not demand-side, mirroring UOR-08's already-fixed Show-HN-pricing issue. Verified empirically against all 488 available postings in that thread using UOR-08's *exact* deployed keyword logic: **42 of 488 (8.6%) would trigger `GENUINE_SERVICE_SEEKING_SIGNAL`**, and every one of the 42, on inspection, is direction-reversed (e.g. *"I'm available for **hire**!"*, *"**hire**@arseniyshestakov.com"*, *"As a **consultant**... I've..."*). This is a real, substantial fraction, not a rare edge case. Decision: ingest only "Who is hiring?" — the unambiguous demand-side thread. Building a direction-classifier for a thread not being used would be unjustified scope per Section 12.

3. **The Algolia `tags=comment,story_X` filter returns the entire nested reply tree, not just top-level postings.** Verified live: on "Who is hiring?", only 7 of 20 fetched comments (35%) are true top-level job postings (`parent_id === story_id`); the other 13 are replies/discussion. Fixed with a client-side filter, not an Algolia-side parameter (none was found to exist for this).

## Design

**Story ID is discovered dynamically every run, never hardcoded** — it changes monthly. Two-stage fetch: (1) pull the `author_whoishiring` account's 5 most recent stories, find the one titled "Who is hiring?" by regex (hitsPerPage=5, not 1, to be robust against same-timestamp ordering between the month's two threads); (2) build and fetch a comment query scoped to that story's ID.

**Cost governance (Section 12):** at full pull depth this thread alone returns dozens of genuine postings per call. Using `hitsPerPage=20` (same convention as every other adapter) rather than a larger pull, for predictable per-run Anthropic Extraction cost.

**Node chain:**
```
When clicking 'Execute workflow' → Read Existing Storage → Extract From File →
Discover Current Thread → Build Comment Query → Fetch Thread Comments →
Split Comments → Normalize + Dedupe → Send to UOR-01
```

`Normalize + Dedupe` mirrors the base HN adapter's exact pattern (dedupe by `source_url`, same placeholder reuse, same `$()` text-read pattern reading from `Extract From File`) with one addition: filters to `parent_id === story_id` before normalizing, to exclude nested-reply noise. ID prefix stays `hn-<objectID>` — HN's object-ID space is global across stories and comments, zero collision risk, and this correctly stays recognized as HN adapter data.

**Locked prediction (live, `hitsPerPage=20`):** 20 fetched, **7 genuine top-level postings** (13 nested-reply noise filtered out), **4 of 7 contain a literal stated dollar figure**. All 7 `objectID`s are new to storage — dedup should pass all 7 through on the first run.

## Build (via `n8n-mcp` API, each node validated before saving)

All 5 distinct new node configs validated individually via `validate_node` — 0 errors on each. Reused Read/Extract/Send-to-UOR-01 configs already proven identical to the base adapter.

**Workflow created:** `UOR-02-Source-Adapter-01b-HackerNews-WhoIsHiring`, ID `qJNTlBW31T89sioW`, inactive.

Full `n8n_validate_workflow` result: `valid: true`, 9/9 nodes enabled, 8/8 connections valid, 0 errors, 0 warnings, 1 expression validated (the dynamic `Fetch Thread Comments` URL).

**Blocked on human execution** (same constraint as every prior module): Manual Trigger workflow, `n8n-mcp` cannot drive it. Not yet run.

### First live run, 2026-09-08 ~00:10 UTC — verified via `n8n_executions` before trusting anything

Execution `147` confirmed genuine: `status: "success"`, `finished: true`, 9/9 nodes executed, 67s duration (consistent with sub-executions into UOR-01). `Split Comments` output 20 items; `Normalize + Dedupe` output exactly 7 — visible in the execution preview alone, matching the locked prediction before the raw file was even opened.

**Raw verification: exact match.** UOR-01 grew **45 → 52 (+7 exactly)**. `hn-` count 22 → 29 (+7), `se-` and `TEST-001` unchanged. All 7 new records confirmed genuine: real company job postings (SaaS Startup, BOSS-IQ, Lucia, Valkyrie Aero, PermitFlow (YC W22), Supero), **4 of 7 contain a literal stated compensation figure** (e.g. `$180,000 - $250,000`) — exactly the explicit demand-side signal UOR-08 previously found zero of.

**Downstream files confirmed byte-for-byte unchanged:** UOR-03 (29,550 bytes), UOR-04 (38,991 bytes), UOR-05 (36,040 bytes), UOR-06 (3,367 bytes), UOR-07 (19,499 bytes), UOR-08 (18,727 bytes) — all match their last independently-verified values exactly.

Happy path CONFIRMED.

## Section 15 controlled-failure test, 2026-09-08 ~00:18 UTC

**Break condition (via `n8n-mcp` API):** "Read Existing Storage"'s `fileSelector` changed to a nonexistent path — read back and confirmed exact before proceeding.

**Human execution, verified via `n8n_executions` before trusting the report:** execution `155` confirmed genuine (new ID, `status: "error"`, `finished: false`). `executionPath` shows exactly 2 nodes (Manual Trigger success, Read Existing Storage error) — the other 7 never appear. Corroborates the reported failure exactly: `"No file(s) found"`.

**Zero-corruption check:** `uor-01-opportunities.jsonl` re-verified at exactly 52 lines / 834,734 bytes, mtime `2026-09-08 00:11:55 UTC` — matching execution `147`'s completion time exactly, not `155`'s. The failed read produced no write whatsoever.

**Revert:** path restored to the exact original value via API, read back and confirmed byte-for-byte identical.

**Result: PASSED.**

**Recovery run, 2026-09-08 ~00:26 UTC — verified via `n8n_executions` first:** execution `156` confirmed genuine (new ID, `status: "success"`, `finished: true`, 2.1s duration — much shorter than the first run's 67s). Output re-verified: **still 52 lines** (`hn-`: 29, `se-`: 20, `TEST-001`: 3, unchanged), **zero unexpected duplicates**. No new postings had appeared in the ~15-minute window since the first run — a legitimate, correctly-handled outcome, not a failure.

**One thing double-checked before accepting it:** `uor-01-opportunities.jsonl`'s mtime was unchanged from execution `147`'s completion, not updated by `156`. Rather than assume this was stale, pulled `156`'s node-level execution detail directly: `Normalize + Dedupe` output **0** items (all 20 fetched comments were already-seen duplicates), so **`Send to UOR-01` correctly never executed at all** — only 8 of 9 nodes ran. This fully explains the unchanged mtime as correct behavior (nothing to write), not a bug.

## UOR-02 Adapter #1 Extension: COMPLETE

### What was built
`UOR-02-Source-Adapter-01b-HackerNews-WhoIsHiring` (ID `qJNTlBW31T89sioW`), inactive, 9 nodes, extending Adapter #1 (Hacker News) to the "Who is hiring?" monthly thread via a dynamic two-stage fetch (story ID discovered fresh every run, never hardcoded). Reuses the base adapter's Algolia API, dedupe pattern, and placeholder conventions exactly.

### Real findings that changed the plan from how it was originally framed
1. **"Freelancer? Seeking freelancer?" is discontinued** (last thread: October 2016) — dropped, not included.
2. **"Who wants to be hired?" was dropped after empirical verification confirmed a real direction-of-intent risk**, not a hypothetical one: 42 of 488 real postings (8.6%) would have triggered UOR-08's `GENUINE_SERVICE_SEEKING_SIGNAL` on self-ad language ("available for **hire**", "**hire**@email.com," "As a **consultant**...") — every one direction-reversed, mirroring UOR-08's already-fixed Show-HN-pricing issue. Confirmed against the full real dataset, not a sample, before deciding.
3. **The Algolia `tags=comment,story_X` filter returns the entire nested reply tree, not just top-level postings** — a client-side `parent_id === story_id` filter was required and verified live (65% of raw hits on this thread are nested noise, not genuine listings).

### Test results summary
Happy path: exact match to a locked, pre-execution prediction (7 genuine postings, 4 with real stated compensation figures) on the first live run. Failure path: visible failure, `executionPath` confirms only 2/9 nodes ran, zero corruption, clean revert. Recovery: correctly found zero new postings in a short time window, and the resulting "unchanged mtime" was verified via node-level execution detail rather than assumed stale — confirming `Send to UOR-01` legitimately never fired rather than silently accepting an unexplained non-write.

### Deferred / non-blocking notes
- This closes UOR-08's demand-side service-seeking gap with real data (4 postings with real compensation figures now in storage) — a full regression check of UOR-08 against this enlarged corpus is Phase 4 of the broader adapter-gap-audit plan, not yet run.
- "Who wants to be hired?" remains a real, large (488+ postings/month) data source that stays unused this round. If a future need arises for supply-side freelancer-availability signal specifically, it would need its own direction-aware classification (option (b) from the original design discussion), not a reuse of UOR-08's current demand-only logic.
