# UOR-07 — Content Opportunity Radar — Build Log

## Status as of 2026-09-06, ~23:10 UTC: COMPLETE.

**Environment:** n8n Community Edition, self-hosted via Docker, localhost:5678.

**Standing rules honored:** UOR-01 through UOR-06 not touched. Reads UOR-01's storage and UOR-06's demand-clustering output (to reuse its already-computed `content_creation_automation`/`social_media_management` tag membership rather than re-deriving that classification). Writes only to a new, separate file — never touches UOR-01–06.

**Standing lesson applied from UOR-06:** every execution claim, including my own, is verified against `n8n_executions` before being accepted — not just at the Section 15 test step.

---

## Step 1 — Live verification, and an honest structural finding

**Record count / adapter roster:** confirmed live, unchanged since UOR-06 — **45 records** (`TEST-001`: 3, `hn-`: 22, `se-`: 20). Schema unchanged.

**Category placeholder:** confirmed still dead for 42/45 real records — content signals can only come from `raw_signal` free text.

### Neither live adapter is a content/creator platform — verified empirically, not assumed
Swept all 45 records for content/creator-platform keywords and **manually inspected every hit's actual context** (not just counted matches). Result: **all 4 keyword hits in the dataset are false positives** — "streaming" (API response streaming, not video), "tiktok/content/creator" (a passing mention inside an unrelated B2B SaaS team-hiring post), "youtube/video" (a demo-video link in an infra startup's launch post), "content" ("Docker Content Trust," a security feature name). **Zero of 45 records contain genuine content-opportunity signal.**

This is a structural mismatch, not a keyword-tuning problem: Hacker News is a tech-community discussion site, Stack Exchange Software Recommendations is a tool-request site — neither surfaces search demand, topic/format growth, or audience interest in actual content. HN points / SE scores measure interest in a text discussion, not audience engagement with a piece of content; treating them as an attention-potential proxy would violate Section 11's own "don't assume views equal profitability" principle in spirit (it isn't even views of content).

**Decision (confirmed with Damian):** build UOR-07 to honestly report this near-null result rather than manufacture signal that isn't there — same posture as UOR-04's `NO_SCORABLE_OBSERVATIONS` and UOR-06's `SINGLE_OBSERVATION` majority.

---

## Step 1 (continued) — Synthetic branch test, before touching n8n

Per Damian's instruction, the three classification branches were verified positively with hand-constructed synthetic strings run through the exact standalone logic — **not added to `uor-01-opportunities.jsonl`**, pure code-level check, nothing touched live storage:

| Input | Expected | Actual |
|---|---|---|
| *"I want to start a YouTube Shorts channel about home cooking tips for beginners."* | `GENUINE_CONTENT_SIGNAL` | `GENUINE_CONTENT_SIGNAL`, platforms `[youtube]`, formats `[shorts]` ✅ |
| *"How do we build a team? We got traction from organic content on TikTok and one meta ad..."* (mirrors real `hn-49556698`) | `WEAK_TANGENTIAL_MENTION` | `WEAK_TANGENTIAL_MENTION`, platforms `[tiktok]`, formats `[]` ✅ |
| *"Ask HN: Why don't LLM APIs have a first-class test mode?..."* | `NO_CONTENT_SIGNAL_DETECTED` | `NO_CONTENT_SIGNAL_DETECTED`, both empty ✅ |

All three branches confirmed correct before any production build work.

**Keyword-table correction applied before building:** the draft `" x "` (Twitter/X) keyword was dropped entirely — it matched on algebra-notation "emotion **x** prevails..." in an unrelated essay during the real-data dry run, confirming a single letter surrounded by spaces is too permissive to ever be safe. `twitter` alone remains as the keyword for that platform.

---

## Step 2 — Design (approved as proposed)

**Reads:** `uor-01-opportunities.jsonl` (raw text + `scoring_factors.automation_potential`, reused not recomputed) and `uor-06-demand-clustering.jsonl` (tag membership, reused not recomputed). Never writes to either.

**Computed deterministically (v1, no new AI call):**
- `content_platform_mentioned` / `content_format_mentioned`: keyword-matched, word-boundary/stem regex discipline consistent with UOR-05/06.
- `uor06_content_tag_membership`: boolean, reused from UOR-06's `content_creation_automation`/`social_media_management` cluster members.
- `content_opportunity_status`: `GENUINE_CONTENT_SIGNAL` (platform **and** format match) / `WEAK_TANGENTIAL_MENTION` (platform **or** format **or** UOR-06 tag, but not both) / `NO_CONTENT_SIGNAL_DETECTED`.
- `attention_potential` / `monetization_potential`: **always `"NOT_DETERMINED"`** for this dataset — no view, search-volume, or engagement-on-content evidence exists anywhere in this pipeline to support either number (Section 11).
- `automation_potential_pct`: reused directly from UOR-01, not recomputed.
- `copyright_platform_risk_flag`: deterministic — true if a platform is mentioned alongside a scraping/bot/automation term, or if UOR-01's own `compliance_review_required` is true. Genuinely computable, not fabricated.

**Explicitly not built:** search demand, topic/trend growth, format growth, audience interest, advertiser value, saturation, longevity-vs-virality. Not a Section 12/20 cost-benefit question about spending more on AI — there is no underlying data (search volume, trends, content-engagement metrics) anywhere in this pipeline to extract from, deterministically or via LLM. Requires a new, genuinely content-oriented data source (Section 5 decision) before these dimensions can ever be real.

**Formula version:** `uor07-v1-content-signal-2026-09-06`.

**Locked prediction against the real 45-record file:**
- `NO_CONTENT_SIGNAL_DETECTED`: 42
- `WEAK_TANGENTIAL_MENTION`: 3 (`hn-49560168` substack-as-example, `hn-49556698` TikTok-mentioned-in-passing, `hn-49552616` YouTube-demo-link)
- `GENUINE_CONTENT_SIGNAL`: 0

---

## Step 3 — Build (via `n8n-mcp` API, each node validated before saving)

All 4 distinct new node configs validated individually via `validate_node` — 0 errors on every node (standard suggestion only, same as every module this session). Read/Extract/Convert configs reuse patterns already proven identical in UOR-02–06.

**Workflow created:** `UOR-07-Content-Opportunity-Radar`, ID `e9SX7nJXj0jhMvEm`, inactive.

Full `n8n_validate_workflow` result: `valid: true`, 8/8 nodes enabled, 7/7 connections valid, 0 errors, 0 warnings.

Chain as built:
```
When clicking 'Execute workflow' → Read UOR-01 Storage → Extract UOR-01 Text →
Read UOR-06 Storage → Extract UOR-06 Text → Classify Content Signal →
Convert to File → Write UOR-07 Output
```

**Blocked on human execution** (same constraint as every prior module): Manual Trigger workflow, `n8n-mcp` cannot drive it. Not yet run.

### First live run, 2026-09-06 ~22:53 UTC — verified via `n8n_executions` before trusting anything

Execution `137` confirmed genuine via `n8n_executions`: `status: "success"`, `finished: true`, 8/8 nodes executed. Raw output verified against the locked prediction: **45 lines, 0 parse failures, status breakdown 42/3/0 — exact match**, all 3 `WEAK_TANGENTIAL_MENTION` IDs exactly as predicted, all 45 `attention_potential`/`monetization_potential` values `"NOT_DETERMINED"` with zero exceptions.

### A real bug, found during post-build verification (not the pre-build dry run) — disclosed honestly

Spot-checking `copyright_platform_risk_flag` on the two `true`-flagged rows (`hn-49556698`, `hn-49552616`) surfaced a genuine false positive: **`hn-49556698`'s flag was driven by the `'bot'` risk-term keyword stem-matching inside "we **both** just graduated college"** — not any real bot/automation reference. `keywordRegex('bot', false)` → `\bbot\w*` matched "both" (b-o-t-h). Same class of issue as UOR-06's "real estate" and the `" x "` keyword already dropped before this module was built — a term too short/generic to be safe as a stem match. `hn-49552616`'s `true` flag was separately confirmed correct (driven by UOR-01's own genuine `compliance_review_required: true`, matched "credit"/"insurance"). Two other `true`-flagged rows surfaced during the full-list review (`hn-49555592`, `hn-49555089`) were also checked and confirmed correctly compliance-driven ("privacy", "health") — not further false positives.

**Worth noting explicitly:** unlike UOR-06's two corrections (both caught in the *pre-build dry run*, before anything touched n8n), this one was only caught in *post-build verification*, after a live execution had already produced output. The dry run's synthetic branch test (Step 1) validated the three `content_opportunity_status` branches correctly but never exercised `copyright_platform_risk_flag` against real data — a real gap in that dry run's coverage, not just luck.

**Fix applied:** `'bot'` changed from stem-matching to exact word-boundary matching (`\bbot\b`), plus `'bots'` added as its own explicit exact keyword to still catch the plural. Verified via a standalone re-run against all 45 real records **before touching n8n**:
- `content_opportunity_status` breakdown (42/3/0): **completely unaffected**, re-verified explicitly rather than assumed.
- `hn-49556698`: `true → false` (fixed).
- `hn-49552616`: `true → true` (unaffected, correctly still compliance-driven).
- **Exactly 1 row changed** across all 45 — no other side effects introduced.

Applied to the live "Classify Content Signal" node via `n8n-mcp`, read back and confirmed exact, `n8n_validate_workflow` re-confirmed `valid: true`, 0 errors.

**Second live run, 2026-09-06 ~22:59 UTC — verified via `n8n_executions` first, every time, no exceptions:** execution `138` confirmed genuine (new ID, later timestamp than `137`) before any file was touched. Output re-verified: 45 lines, 0 parse failures, status breakdown **still 42/3/0**, same 3 `WEAK_TANGENTIAL_MENTION` IDs, `hn-49556698` flag now `false`, `hn-49552616` flag still `true`, 0 `NOT_DETERMINED` violations.

**Upstream files confirmed byte-for-byte unchanged** after both runs: UOR-01 (687,452 bytes), UOR-03 (16,282 bytes), UOR-04 (20,931 bytes), UOR-05 (19,507 bytes), UOR-06 (3,367 bytes) — all match their last independently-verified values exactly.

Happy path CONFIRMED, including a real bug found and fixed along the way.

## Step 4 — Section 15 controlled-failure test, 2026-09-06 ~23:01 UTC

**Break condition (via `n8n-mcp` API):** "Read UOR-01 Storage"'s `fileSelector` changed to a nonexistent path — read back and confirmed exact before proceeding.

**Human execution, verified via `n8n_executions` before trusting the report:** execution `139` confirmed genuine (new ID, `status: "error"`, `finished: false`). `executionPath` shows exactly 2 nodes (Manual Trigger success, Read UOR-01 Storage error) — the other 6 never appear. Corroborates the reported failure shape exactly: `"No file(s) found"`.

**Zero-corruption check:** `uor-07-content-opportunity-radar.jsonl` re-verified at exactly 45 lines / 19,499 bytes, mtime `2026-09-06 22:59:05 UTC` — matching execution `138`'s timestamp exactly, not `139`'s. The failed read produced no write whatsoever.

**Revert:** URL restored to the exact original value via API, read back and confirmed byte-for-byte identical.

**Result: PASSED.**

**Recovery run, 2026-09-06 ~23:05 UTC — verified via `n8n_executions` first:** execution `140` confirmed genuine (new ID, `status: "success"`, `finished: true`, later than the failure). Output re-verified: 45 lines, 0 parse failures, status breakdown still 42/3/0, `hn-49556698` flag still correctly `false`, `hn-49552616` flag still correctly `true`, 0 `NOT_DETERMINED` violations. mtime updated to `2026-09-06 23:05:32 UTC` (matching execution `140` exactly) while content stayed identical — correct full-overwrite behavior, no duplication.

**Upstream files reconfirmed byte-for-byte unchanged** after the entire failure/recovery cycle: UOR-01 (687,452 bytes), UOR-03 (16,282 bytes), UOR-04 (20,931 bytes), UOR-05 (19,507 bytes), UOR-06 (3,367 bytes) — all identical to every prior check.

## UOR-07 Adapter: COMPLETE

### What was built
`UOR-07-Content-Opportunity-Radar` (ID `e9SX7nJXj0jhMvEm`), inactive, 8 nodes: Manual Trigger → Read UOR-01 Storage → Extract UOR-01 Text → Read UOR-06 Storage → Extract UOR-06 Text → Classify Content Signal → Convert to File → Write UOR-07 Output. Reads UOR-01 (raw text + `automation_potential`, reused) and UOR-06 (content-tag membership, reused, not recomputed). Writes only to a new, separate file, `uor-07-content-opportunity-radar.jsonl`. Never touches UOR-01 through UOR-06.

### The central, honest finding — a data-source gap, not a UOR-07 defect
Verified empirically, not assumed: **zero of the 45 records in storage contain genuine content-opportunity signal.** Every keyword hit found during design and dry-run work traced to a false positive on manual inspection — an API-streaming mention, a passing TikTok aside in an unrelated startup post, a demo-video link, a security-feature name containing "content." Neither Hacker News nor Stack Exchange is a content/creator platform; Section 11's actual signals (search demand, topic/format growth, audience interest) require data this pipeline doesn't have. This is a Section 5 data-source gap, not something fixable by refining UOR-07's own logic.

### Two real bugs found and fixed, at two different stages — documented honestly, not smoothed over
1. **Pre-build:** the draft `" x "` (Twitter/X) keyword matched algebra notation ("emotion **x** prevails...") during the real-data dry run. Dropped before any n8n build work; `twitter` alone remains as the platform keyword.
2. **Post-build, not caught by the pre-build dry run:** `'bot'` risk-term stem-matching flagged `hn-49556698`'s `copyright_platform_risk_flag` true on **"we **both** just graduated college."** The synthetic branch test validated `content_opportunity_status`'s three branches correctly but never exercised the risk-flag logic against real data — a genuine coverage gap in that test, not just bad luck. Fixed (`'bot'` → exact-boundary, `'bots'` added explicitly), re-verified via a standalone re-run before touching n8n (exactly 1 row changed, `content_opportunity_status` completely unaffected), then re-verified live in n8n after a genuine re-execution.

### Standing lesson applied throughout, not just at Section 15
Every "it worked" / "all nodes green" report in this build — the original happy path, the fix re-run, the failure test, and the recovery run — was checked against `n8n_executions` for a genuine new execution ID *before* any file was inspected. No exceptions.

### Test results summary
Happy path: exact match to a locked, pre-execution prediction (42/3/0), a real bug found and fixed mid-verification, re-verified exact after the fix. Failure path: visible failure (`"No file(s) found"`), `executionPath` confirms only 2/8 nodes ran, zero corruption, clean revert, clean recovery — every execution independently confirmed via the audit trail.

### Deferred / non-blocking notes, and one priority recommendation
- **Priority recommendation:** the next Source Ingestion (UOR-02) work should target a genuinely content-oriented source — Reddit content-focused subreddits, the YouTube Data API, or Google Trends — since without one, UOR-07 will stay structurally near-dormant (42+/45 `NO_CONTENT_SIGNAL_DETECTED`) regardless of how the module itself is refined. This is the same conclusion UOR-06 reached about its own tag table, generalized: better keyword tables can't manufacture signal that isn't in the underlying data.
- The synthetic branch test should be extended to cover `copyright_platform_risk_flag` and `uor06_content_tag_membership`, not just `content_opportunity_status`, given this build's own bug slipped past exactly that gap.
- `attention_potential`/`monetization_potential` remain `NOT_DETERMINED` by design for the current adapter roster — will need a real evidence base (search volume, view/engagement data) before either can ever report a number, per Section 11's explicit prohibition on fabricating one.

---

## FIX — 2026-09-08: `attention_potential` was dead code, now a real deterministic formula

**Trigger:** the "priority recommendation" above got its evidence base — the YouTube adapter (Phase 3b) started populating `full_record.view_count`/`like_count`/`comment_count` on every YouTube record. But the Phase 4 regression pass found `attention_potential` was still a **hardcoded literal string `'NOT_DETERMINED'`** in the live code — it never read those fields at all, for any record. The evidence existed in storage; this module just never looked at it.

**Formula (deterministic, no LLM involvement, per Section 24), `attention_potential_formula_version: 'uor07-attention-v1-2026-09-08'`:**
- Requires `full_record.view_count` present and numeric to attempt scoring at all — view count is the one metric YouTube essentially never hides. Its absence (every HN/SE record) correctly falls through to `attention_potential: 'NOT_DETERMINED'`, `attention_potential_score: null`, same as before this fix.
- Three components, each bucketed 0–10 on fixed, named threshold constants (`VIEW_COUNT_THRESHOLDS`, `COMMENT_COUNT_THRESHOLDS`, `LIKE_COUNT_THRESHOLDS`), combined as a weighted average (view 0.5 / comment 0.3 / like 0.2) onto a 0–100 `attention_potential_score`, mapped to a status tier (`HIGH_ATTENTION` ≥70, `MODERATE_ATTENTION` ≥40, `LOW_ATTENTION` ≥10, `MINIMAL_ATTENTION` ≥0) via `ATTENTION_STATUS_THRESHOLDS`.
- **`like_count: 0` ambiguity, handled explicitly per the YouTube adapter build log's own flag:** YouTube's API returns the identical literal `0` both when a creator has hidden public like counts and when a video genuinely has zero likes (confirmed on `yt-EChXCPolIDQ` during the adapter build — `likeCount` was entirely absent from the raw API response, not a real zero). Per Section 24 (insufficient information over guessing), a `like_count` of exactly `0` is **excluded from the formula entirely** rather than scored as real negative evidence, and the weighted average renormalizes over whichever components are actually usable (view + comment only, in that case). The output records this explicitly via `like_signal_excluded_reason: 'zero_or_hidden_ambiguous'` and `attention_potential_inputs_used`, so a `MODERATE_ATTENTION` score arrived at via view+comment alone is distinguishable from one that includes a real like signal.

**Synthetic branch tests (6 cases), all passed:** a high-engagement YouTube-shaped record → `HIGH_ATTENTION`; an HN/SE-shaped record with no engagement fields at all → `NOT_DETERMINED`; a `like_count: 0` case → excluded from the like component, still scores via view+comment; a low-view record → `MINIMAL_ATTENTION`; a defensive check on a non-numeric `view_count` string → `NOT_DETERMINED` (type-checked, not just truthiness-checked); a missing `full_record` entirely → `NOT_DETERMINED`.

**Locked prediction (dry-run via `docker exec ... node` against the live 61-record corpus):** `status_breakdown` (content classification) unchanged — `NO_CONTENT_SIGNAL_DETECTED: 48` / `WEAK_TANGENTIAL_MENTION: 13` / `GENUINE_CONTENT_SIGNAL: 0`, confirming this fix touches only `attention_potential` and nothing else. New `attention_breakdown: {NOT_DETERMINED: 52, HIGH_ATTENTION: 7, MODERATE_ATTENTION: 2}`. All 9 YouTube records get real scores; both real `like_count: 0` records (`yt-EChXCPolIDQ`, `yt-TZFiGmF9B1I`) correctly land in `MODERATE_ATTENTION` via view+comment only, `like_signal_excluded_reason` set on both.

**Live verification:** applied via `n8n_update_partial_workflow`, `n8n_validate_workflow` → 0 errors. Run **headlessly** (`docker exec -e N8N_RUNNERS_BROKER_PORT=5691 n8n n8n execute --id e9SX7nJXj0jhMvEm`) — execution **176**, `mode: "cli"`, `status: "success"`, confirmed via `n8n_executions`. Output file: 61 lines, `status_breakdown` and `attention_breakdown` both **exact match** to the locked prediction, all 9 YouTube records' individual scores exact match (e.g. `yt--WSoI1zL_Z4`: `HIGH_ATTENTION`, score 91, inputs `[view, comment, like]`).

**`monetization_potential` remains `NOT_DETERMINED`, deliberately untouched** — no evidence base exists for it yet (no pricing/revenue signal anywhere in the pipeline); out of scope for this fix.

## Note: headless execution now available for this workflow (and others)

Verified 2026-09-08: `docker exec -e N8N_RUNNERS_BROKER_PORT=<unused port> n8n n8n execute --id <workflowId>` runs a Manual Trigger workflow without a browser click. The bare command fails (exit 1, zero writes) because n8n's Task Broker defaults to port 5679, already bound by the main server process in this container — the port override avoids the collision. Execution registers normally in `n8n_executions` with `mode: "cli"` (vs `"manual"` for browser clicks), so the existing verification method (check `n8n_executions` for a genuine new execution, then check the file via `docker exec`) is unaffected. Confirmed working for credential-free Code-node pipelines (UOR-03 through UOR-08); not yet confirmed for nodes requiring credentials (UOR-01's Anthropic call, the YouTube adapter's Query Auth) — verify before relying on it for those.
