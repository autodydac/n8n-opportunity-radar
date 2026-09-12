# UOR-02 — Source Ingestion — Build Log

## Status as of 2026-09-04, ~05:58 UTC: UOR-02 Adapter #1 COMPLETE — dedup fix confirmed, Section 15 failure-handling test passed, no open gates remaining.

**Environment:** n8n Community Edition, self-hosted via Docker, localhost:5678.

---

## UOR-01 summary (complete, see full history below)
UOR-01 (Opportunity Analysis Core) — all 9 nodes built, tested, verified end-to-end on 2026-09-03. Two bugs found and fixed that day: filename typo (`json1` vs `jsonl`), missing newline delimiter in Build Storage Record. **Standing practice: verify every fix via raw `docker exec ... cat`/`wc -l`/`grep`, never trust n8n's UI success indicator alone.**

Storage: `/home/node/.n8n-files/uor-01-opportunities.jsonl` inside the `uor01_data` Docker volume. Top-level record ID field is **`opportunity_id`** (not `signal_id` — that exists only nested under `full_record.signal_id`).

Security: three live API keys (Anthropic, n8n JWT, ElevenLabs) were pasted into chat sessions by mistake during setup — none stored/echoed by Claude. Rotation advised, not yet confirmed done.

Only the terminal Claude Code session (with `n8n-mcp` configured against the real Docker container) has a network path to `localhost:5678`. The Cowork/cloud session cannot reach n8n directly.

---

## UOR-02 Adapter #1: Hacker News — full status

**Workflow:** `UOR-02-Source-Adapter-01-HackerNews` (ID `QUUq1uIRlTBTS1Zt`), inactive.

Chain (current, confirmed working): Manual Trigger → Read Existing Storage → Extract From File → Fetch HN Signals → Split Hits → Normalize + Dedupe → Send to UOR-01.

### Timeline
- **~03:39–03:41 UTC:** First end-to-end test. 20 real HN records fetched, normalized, ran the full UOR-01 pipeline, landed in storage alongside 3 pre-existing synthetic TEST-001 records (23 total). Spot checks confirmed correct scoring, including a correct Section 25/26 compliance-gate trigger on a fintech-adjacent post.
- **~04:15–04:18 UTC — BUG FOUND:** Repeat-run test exposed total dedup failure (23 → 43 lines, all 20 `hn-` IDs duplicated). Root cause: the "Normalize + Dedupe" Code node's `this.helpers.binaryToBuffer(...)` call threw against n8n's filesystem-mode binary storage, and the node's own try/catch silently swallowed the error, leaving `existingUrls` empty. A Section 15 violation in its own right (failure masked as success).
- **~04:30–04:35 UTC — FIX APPLIED** via `n8n-mcp` API, each step read back and verified:
  1. Inserted an **Extract From File** node (`n8n-nodes-base.extractFromFile` v1.1, operation `text`, `binaryPropertyName: "data"`), correctly spliced into the real existing connection **Read Existing Storage → Fetch HN Signals** (the terminal session caught that the originally-assumed splice point didn't exist as a direct connection, and self-corrected rather than guessing).
  2. Replaced the broken read block in "Normalize + Dedupe" with a version that reads plain text from the new node and throws a real error instead of silently swallowing one.
  3. Validated: `n8n_validate_workflow` → valid, 7 nodes (was 6), 6/6 connections, 0 errors/warnings.
- **File cleanup (~04:35 UTC):** Corrupted file (43 lines: 20 duplicated `hn-` + 3 legitimate distinct TEST-001 records) needed repair without destroying the 3 real TEST-001 entries. **Two near-misses caught before any destructive write:** (a) the first proposed cleanup script deduped on `signal_id`, a field that doesn't exist at the top level of these records — would have silently done nothing; corrected to the real field, `opportunity_id`; (b) a blanket dedupe would have collapsed the 3 distinct TEST-001 test records into 1, destroying real audit history — scoped the dedupe to only affect `hn-`-prefixed IDs. Backup taken before rewrite (`uor-01-opportunities.jsonl.bak-pre-hn-dedupe`, 763,718 bytes, confirmed identical to pre-dedupe file, kept on disk). Result independently re-verified via raw `grep | sort | uniq -c`: 43 → 23 lines, 20 unique `hn-` + 3 TEST-001.
- **~04:50 UTC — LIVE RE-TEST, CONFIRMED WORKING.** Workflow executed live in the n8n editor (real browser click, not `n8n-mcp`). "Normalize + Dedupe" passed through **1 item** (not 0) to "Send to UOR-01" — initially flagged as needing verification, since the HN "Ask HN" feed is live/time-sensitive and a genuinely new post could plausibly appear between test runs, which dedup should correctly NOT filter.
  - **Verification method (deterministic, independent of feed drift):** checked whether the new record's `opportunity_id` existed in the pre-cleanup backup file (which contains all 20 original IDs). Result: `hn-49560630` — **0 matches in the backup** — confirmed genuinely new, never seen before.
  - **Final state:** file now 24 lines. All 20 original `hn-` IDs still appear exactly once each (zero re-duplication). Plus the 1 new legitimate record. Plus 3 TEST-001 untouched. 20 + 1 + 3 = 24, exact match.
  - **This is the true success condition:** dedup correctly suppressed every previously-seen record AND correctly did not suppress a genuinely novel one. Bug is closed.

### Status: dedup fix RESOLVED and confirmed under a live run.

### Stage 4/5 — Controlled-failure test (Section 15), completed 2026-09-04 ~05:55–05:58 UTC
Since `n8n-mcp`'s `n8n_test_workflow` cannot trigger a Manual Trigger workflow (confirmed empirically — only webhook/form/chat triggers are supported externally), this test required a human to click "Execute workflow" in the browser at the right moment while the terminal session set up/reverted the failure condition via API. Procedure:

1. **Baseline:** storage file confirmed at 24 lines before touching anything.
2. **Break condition (via `n8n-mcp` API):** "Fetch HN Signals" node's `url` changed from `https://hn.algolia.com/api/v1/search_by_date?tags=story&query=Ask%20HN&hitsPerPage=20` to `https://hn.algolia.com/api/v1/search_by_date_BROKEN?...` (same host, deliberately invalid path) — read back and confirmed exact before proceeding. No other parameter touched.
3. **Human execution:** user clicked "Execute workflow" in the n8n editor. Result: "Fetch HN Signals" failed visibly with a clear 404 ("The resource you are requesting could not be found"). Upstream nodes (Read Existing Storage, Extract From File) succeeded normally. Downstream nodes (Split Hits, Normalize + Dedupe, Send to UOR-01) **did not execute** — no false-green success anywhere. Corroborated independently via `n8n_executions` (execution 104): `status: "error"`, `finished: false`, `executionPath` shows exactly 4 nodes ran (3 success, 1 error, 0 items), the other 3 nodes never appear in the path.
4. **Zero-corruption check:** storage file re-verified at exactly 24 lines, unchanged — a failed fetch produced no partial or garbage write.
5. **Revert (via `n8n-mcp` API):** URL restored to the exact original value, read back and confirmed byte-for-byte identical to the pre-test value.
6. **Recovery confirmation:** human clicked "Execute workflow" once more. All 6 downstream nodes went green through to "Send to UOR-01." Storage file grew 24 → 25 lines (1 new record, `hn-49561016`). Novelty deterministically confirmed (not assumed from live-feed timing) by checking the pre-cleanup backup file — 0 matches, genuinely never seen before. File integrity re-verified: 25/25 lines newline-terminated (`cat -A`, no `$` markers missing), 23 distinct `opportunity_id` values (22 unique `hn-` + 1 `TEST-001` at its expected 3x count) — no unwanted duplication introduced by the test.

**Result: PASSED.** UOR-02 fails visibly and safely on source-API failure (Section 15: API failure / source outage / timeout class), recovers cleanly once the fault clears, and never silently mis-reports success. Section 15 gate closed.

### UOR-02 Adapter #1: COMPLETE
Checked against master instructions Sections 5 (Data-Source Governance — official public Algolia API, no scraping), 6 (Evidence Before Conclusions — passes raw signal through UOR-01's existing evidence-preserving pipeline unmodified), 15 (Failure Handling — see above, passed), 20 (Development Rule — happy path and failure path both proven before further automation), 21 (does not disturb UOR-01's already-validated pipeline). No open gates remain for Adapter #1 itself. Per standing instruction, UOR-01 and UOR-02 are not to be further modified unless a new bug or requirement surfaces.

**Deferred housekeeping (non-blocking):**
- Backup file `uor-01-opportunities.jsonl.bak-pre-hn-dedupe` still present on disk — fine to delete once satisfied, not yet actioned.

### Recommended follow-up (not yet done)
For future repeat-testing of this or any adapter without depending on live feed drift, n8n's **pin data** feature (lock a node's output so re-runs replay identical saved data instead of live-fetching) would make dedup tests fully deterministic. Worth setting up before testing Adapter #2.

### Known deprecation flagged, not urgent
`mode: "each"` on the "Send to UOR-01" Execute Sub-workflow node is deprecated in the current n8n schema — still functional, eventual replacement is Loop Over Items + "Run once with all items." Tracked for the next maintenance pass.

### Deferred / non-blocking notes
- Adapter #2 candidate: Reddit, once a registered/approved OAuth "script" app exists.
- HN query is currently a static "Ask HN" search sorted by date, `hitsPerPage=20` — worth revisiting query strategy once there's a sense of signal quality at larger volume.
- Maintenance/health-check routine in `UOR-Maintenance-Check-Prompt.md` should be run periodically — must run from the terminal Claude Code session, not the Cowork/cloud session.
- Section 12 cost note: the original dedup-failure incident burned ~20 unnecessary Anthropic Haiku calls (small dollar impact) — worth a glance at the Anthropic console usage page if cumulative testing spend is a concern.
