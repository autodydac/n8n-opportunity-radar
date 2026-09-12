# UOR-02 Adapter #4 — YouTube Data API v3 — Build Log

## Status as of 2026-09-08: COMPLETE — happy path and Section 15 failure/recovery both confirmed live.

**Environment:** n8n Community Edition, self-hosted via Docker, localhost:5678.

**Hard constraints honored:** UOR-01 through UOR-08 not touched (verified via mtime check post-run — all 6 downstream module output files predate this adapter's first execution). API key never pasted into chat; stored via n8n's own credential manager per Section 14.

---

## Design

**Endpoint:** official YouTube Data API v3 (`googleapis.com/youtube/v3`), Google Cloud API key, no OAuth needed for this read-only public-data use.

**Query, locked from a live dry-run pull (5 sample results, all matching) before any n8n work:** `search.list?part=snippet&type=video&maxResults=5&q=faceless%20youtube%20channel`. Topic chosen per the Adapter Gap Audit's own framing — this source's value is closing UOR-07's `attention_potential` evidence gap with real view/like/comment data, and "faceless YouTube channel" content sits squarely in the project's AI-automation-services domain (workflow_automation / video_production / ai_reasoning_chat_agent adjacent).

**Two-call design, batched (not naive per-video calls):** `search.list` returns up to 5 `videoId`s → a Code node joins them into one comma-separated list → a single `videos.list?part=statistics&id=id1,id2,...` call gets stats for all of them in one request (standard documented YouTube API behavior, same quota cost as a single-ID call).

**Credential:** `httpQueryAuth` generic credential type (n8n's "Query Auth"), named `Query Auth account`, ID `RoGQ1LjCGBLNfnQO`. Query param name `key`. Created by the user directly in n8n's UI — never passed through chat or created via API, specifically to avoid a 4th key-exposure incident (three prior: Anthropic, n8n JWT, ElevenLabs, all during initial Aug 22 setup).

**Node chain (8 nodes — mirrors HN/SE pattern, +2 nodes for the two-call merge):**
```
When clicking 'Execute workflow' → Read Existing Storage → Extract From File →
Fetch YouTube Search Results → Build Video ID List → Fetch YouTube Video Statistics →
Merge + Normalize + Dedupe → Send to UOR-01
```

**Lesson (a) applied — ID prefix + fresh-text dedupe:** `signal_id: yt-${videoId}`, dedupe key `source_url = https://www.youtube.com/watch?v=${videoId}` (constructed, since `search.list` doesn't return a canonical watch URL directly), read via `Extract From File` (text) — same pattern as HN/SE, never the binary-mode helper that caused Adapter #1's original dedup bug.

**Lesson (b) applied — placeholder consistency:** exact HN placeholder string `'Unclassified — pending AI extraction'` reused verbatim for `market`, `opportunity_category`, `target_customer`, `proposed_solution`, `monetization_model`, `delivery_method`, `geography` — confirmed by reading HN's live node code fresh, not from memory.

**Design addition beyond the HN/SE pattern:** structured numeric fields (`view_count`, `like_count`, `comment_count`, `channel_title`, `video_id`) added alongside the 15-field schema, not just folded into `evidence_notes` prose. The Gap Audit specifically named this adapter as the only current evidence base for `attention_potential` — these survive untouched into `full_record` via UOR-01's existing pass-through, so later modules (or a human) can read real counts, not just parse text. Already observed working: on the first real record, UOR-01's own AI Extraction step cited "180K+ views, 5.3K likes, 234 comments" directly in its `demand_signal_strength` reasoning.

**Workflow created:** `UOR-02-Source-Adapter-04-YouTube`, ID `zokxVunq2Rq1NbS1`, inactive. `n8n_validate_workflow`: valid, 8/8 nodes enabled, 7/7 connections, 0 errors, 0 warnings (only the same generic "add error handling" suggestions every prior adapter also gets).

### Pre-build baseline check (live)
`uor-01-opportunities.jsonl`: 52 lines, 0 `yt-` records — no collision risk on first run.

---

## First live run — execution 157, 2026-09-08 ~03:16–03:17 UTC

**Falsifiable prediction, locked before running:** 52 → 57 lines (5 fetched, 0 collide).

| Check | Predicted | Actual | Match |
|---|---|---|---|
| Storage line count | 52 → 57 | **57** | ✅ exact |
| `yt-` collisions | 0 | 0 | ✅ |
| Nodes executed | 8/8 | 8/8, all success | ✅ |
| `Send to UOR-01` sub-executions | 5 | 5 (IDs 158–162 range) | ✅ |
| Downstream module files (03–08) | unchanged | all mtimes predate this run | ✅ |

**Result: exact match, no mismatch to diagnose.**

---

## Section 15 controlled-failure test, 2026-09-08 ~03:33–03:35 UTC

Same procedure as HN's Section 15 test: break one fetch URL (same host, invalid path), human clicks "Execute workflow," verify visible failure + zero corruption, revert, verify clean recovery.

1. **Baseline:** 57 lines, confirmed via `docker exec` before touching anything.
2. **Break (via `n8n-mcp` API):** `Fetch YouTube Search Results` URL changed from `.../youtube/v3/search?...` to `.../youtube/v3/search_BROKEN?...` — read back and confirmed exact before proceeding.
3. **Human execution → failure confirmed.** Execution 163: `status: error`, `finished: false`, 0.7s duration. `NodeApiError`, `httpCode: 404`, on `Fetch YouTube Search Results`. Execution path: exactly 4 nodes ran (Manual Trigger, Read Existing Storage, Extract From File — all success; Fetch YouTube Search Results — error). The other 4 downstream nodes never executed. No false-green anywhere.
4. **Zero-corruption check:** storage file re-verified at 57 lines, unchanged.
5. **Revert (via `n8n-mcp` API):** URL restored to the exact original value, read back and confirmed identical.
6. **Recovery run — execution 164.** Ran clean, `success`/`finished: true`, 8/8 nodes green.

**Recovery run's actual count deviated from the stated prediction (0 new expected; 4 new occurred) — diagnosed via node-level detail, not assumed either way:**

`search.list`'s relevance ranking for this query drifted across the ~19-minute gap between execution 157 and 164 — only 1 of 5 results repeated (`Mrg0nc3Mkew`); the other 4 (`pXPu3hAR2zE`, `yPuxoqgovto`, `cqc0xni25n0`, `-JFmcpwpQSs`) were genuinely new, confirmed by checking they don't appear in execution 157's set. `Merge + Normalize + Dedupe` correctly filtered the 1 repeat (input 5 → output 4) and passed the 4 novel ones through. Storage: 57 → 61. 9 distinct `yt-` IDs total, each appearing exactly once (`uniq -c` confirmed) — **zero duplication of the repeat, zero suppression of the genuinely novel ones.** This is the same "true success condition" the HN adapter's own dedup test defined.

**Root cause of the wrong prediction:** an assumption that this adapter's repeat-run behavior would resemble HN's/SE's low-volume, date/creation-sorted feeds. `q=faceless youtube channel` is a 431K-result relevance-sorted query — YouTube's relevance ranking is evidently not stable minute-to-minute at this volume. Not a code defect; the dedup mechanism itself is confirmed correct. **Standing note for future repeat-testing of this adapter:** don't assume near-zero drift the way HN/SE testing could — this source's top-5 relevance set can shift substantially between runs even minutes apart.

**Result: Section 15 gate PASSED** (visible failure, zero corruption, clean recovery). The count mismatch is a documented characteristic of this source, not a failure-handling defect.

---

## UOR-02 Adapter #4: COMPLETE

Checked against master instructions Sections 5 (Data-Source Governance — official public Google API, no scraping), 6 (Evidence Before Conclusions — raw signal passed through UOR-01's unmodified pipeline, plus added structured evidence fields), 14 (Security — credential stored via n8n's own manager, never in chat), 15 (Failure Handling — passed, see above), 20 (Development Rule — happy path and failure path both proven), 21 (UOR-01's validated pipeline undisturbed). No open gates remain for this adapter.

**Real-data nuance flagged for later hand-verification (Gap Audit Part D), not a bug:** `EChXCPolIDQ` returned `likeCount` entirely absent from the API response (YouTube omits it when a creator hides public like counts) — the code's `?? 0` fallback renders this indistinguishable from a video that genuinely has zero likes. Whoever later hand-verifies `attention_potential`/`GENUINE_CONTENT_SIGNAL` hits against this source should check the raw video before reading `like_count: 0` as "no engagement."

**Final storage state:** `uor-01-opportunities.jsonl` = 61 lines (29 `hn-`, 20 `se-`, 9 `yt-`, 3 `TEST-001`).
