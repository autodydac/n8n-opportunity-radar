# UOR-02 Adapter #3 — Stack Exchange (Software Recommendations) — Build Log

## Status as of 2026-09-06: IN PROGRESS — Steps 1–3 done (classification confirmed live), building next.

**Environment:** n8n Community Edition, self-hosted via Docker, localhost:5678.

**Hard constraints honored:** UOR-01 through UOR-05 not touched. Adapter #1 (HN, `QUUq1uIRlTBTS1Zt`) not modified — its exact placeholder string and dedupe pattern were read fresh from its live code (not memory) to mirror faithfully. Reddit (Adapter #2) deferred, not deleted — see `UOR-02-Adapter-02-Reddit-Build-Log.md`, updated separately.

---

## Step 1 (of this task) — Adapter numbering
This is **Adapter #3**, not a renumbered #2. Reddit keeps its slot in the history as deferred/paused.

## Step 2 — Section 5 classification, verified live (not assumed)

**Endpoint:** `https://api.stackexchange.com/2.3/questions` — official public Stack Exchange API, v2.3.

**Tested live** via `curl` against the exact proposed query before writing anything into n8n:
```
GET https://api.stackexchange.com/2.3/questions?site=softwarerecs&order=desc&sort=creation&pagesize=20
```
Result: `HTTP/2 200`, 20 items returned (matches `pagesize=20`), real current questions from `softwarerecs.stackexchange.com`.

**Rate limit confirmed from the live response body itself** (not assumed from any prior claim): `quota_max: 300`, `quota_remaining: 299` after this one call — exactly matches the expected "300 queries/24hr per IP, no key/registration required" for this endpoint. `backoff` field absent (would appear if throttled). **No authentication needed** for this read-only, low-frequency use — no new credential required, no cost, well within Section 12 governance for a manual/occasional poll.

**No custom User-Agent requirement found** for the Stack Exchange API (unlike Reddit, which mandates one) — HN's adapter also sends no custom headers (`sendHeaders: false`); Stack Exchange's own API docs impose no such requirement, so the same minimal-header pattern is used here for consistency, not because it was skipped.

**Target site confirmed:** `softwarerecs.stackexchange.com` (Software Recommendations) — per the task's own framing, "does a tool exist for X" questions are a direct, structurally clean opportunity signal (arguably cleaner than HN's general-purpose "Ask HN" firehose).

## Step 3 — Design (mirrors Adapter #1's proven pattern exactly)

**Exact HN placeholder string, confirmed from live code** (not retyped from memory) via fresh `n8n-mcp` inspection of the HN adapter's "Normalize + Dedupe" node: `'Unclassified — pending AI extraction'` (em dash, exact capitalization). Reused verbatim.

**Node chain** (7 nodes — the user's abbreviated 6-step description omits "Split Items," but the *actual* HN adapter has a `Split Hits` node between fetch and normalize, structurally required because the API response wraps results in an array; included here to match the real pattern, not the shorthand):
```
Manual Trigger → Read Existing Storage → Extract From File →
Fetch Stack Exchange Signals → Split Items → Normalize + Dedupe → Send to UOR-01
```

**Lesson (a) applied — ID prefix + fresh-text dedupe:** `signal_id: se-${question_id}`, dedupe key is `source_url` (the SE question's `link` field), read via `Extract From File` (text) exactly like Adapter #1 — never the binary-mode helper that caused Adapter #1's original dedup bug. Dedupes against the **full existing storage file**, which currently has 25 lines (22 `hn-` + 3 `TEST-001`) — confirmed live, and confirmed **zero** existing `se-` records, so no ID or URL collision risk on the first run.

**Lesson (b) applied — placeholder consistency:** `market`, `opportunity_category`, `target_customer`, `proposed_solution`, `monetization_model`, `delivery_method`, `geography` all set to the exact HN placeholder string above, not a new one.

**Content mapping:** `opportunity_title` = question title; `problem`/`demand_signal` = title + tags (SE questions don't have a separate body field in the list endpoint, only title + tags — tags are a genuine, real signal, e.g. "database, offline, user-interface" seen in the live test pull); `signal_date` = `creation_date` (Unix epoch seconds from the API) converted to ISO string, since SE returns epoch seconds where HN's Algolia already returns ISO strings — a necessary format difference, not a deviation from the pattern; `evidence_notes` includes question ID, score, answer count, view count, mirroring HN's points/comments convention.

### Pre-build baseline check (live, not assumed)
`uor-01-opportunities.jsonl`: **25 lines**, **0** `se-` records. First run is predicted to add **up to 20 new records** (the live test pull returned exactly 20 items, and none can collide with existing storage since no `se-` IDs or `stackexchange.com` URLs exist there yet) — expected result: **25 → 45**, all 20 genuinely new. A second immediate re-run should dedupe all 20 and add only whatever genuinely new questions appear in the meantime (mirroring Adapter #1's proven Stage 2/3 behavior exactly).

Continuing to Step 4 — build.

## Step 4 — Build (via `n8n-mcp` API, each node validated before saving)

All 6 distinct node configs validated individually via `validate_node` — 0 errors on every node (HTTP node's "consider authentication" warning is expected/intentional, matching Adapter #1's own pattern for this kind of public API; Code node's standard error-handling suggestion, same as every Code node built this session).

**Workflow created:** `UOR-02-Source-Adapter-03-StackExchange`, ID `O0fvA0c4LeKG827e`, inactive.

Full `n8n_validate_workflow` result: `valid: true`, 7/7 nodes enabled, 6/6 connections valid, 0 errors, 0 warnings.

Chain as built:
```
When clicking 'Execute workflow' → Read Existing Storage → Extract From File →
Fetch Stack Exchange Signals → Split Items → Normalize + Dedupe → Send to UOR-01
```

**Falsifiable predictions, locked in before the first run:**
- Storage file: **25 → 45 lines** (all 20 fetched questions are genuinely new; zero `se-` records or `stackexchange.com` URLs exist in storage yet).
- A second immediate re-run should dedupe all 20 just-added records and add only whatever genuinely new questions appear on softwarerecs.stackexchange.com in the interim (expected: 0, possibly 1–2 given the site's low posting volume — will check the actual site activity rate if this needs distinguishing from a bug).

**Blocked on human execution** (same constraint as every prior module): Manual Trigger workflow, `n8n-mcp` cannot drive it.

### First live run, 2026-09-06 ~20:37 UTC — independently verified via raw `docker exec`

**Execution confirmed via `n8n_executions`:** execution `111`, `status: "success"`, `finished: true`, **7/7 nodes executed**. `Normalize + Dedupe` passed all 20 items through untouched (0 deduped, as predicted). `Send to UOR-01` shows 20 sub-executions into UOR-01.

| Check | Predicted | Actual | Match |
|---|---|---|---|
| Storage line count | 25 → 45 (exactly +20) | **45** | ✅ exact |
| Parse failures | 0 | **0** | ✅ |
| `se-` record count / collisions | 20 new, 0 collisions | **20 unique, 0 collisions with the existing 25** | ✅ |
| Spot-check fields | `source_name: "Stack Exchange (Software Recommendations)"`, exact HN placeholder category, `softwarerecs.stackexchange.com` URLs | **All 3 spot-checked records match exactly** | ✅ |
| UOR-03 output untouched | unchanged | 16,282 bytes, mtime `2026-09-04 06:53:11 UTC` — unchanged | ✅ |
| UOR-04 output untouched | unchanged | 20,931 bytes, mtime `2026-09-05 05:02:58 UTC` — unchanged | ✅ |
| HN adapter workflow untouched | unchanged | `updatedAt: 2026-09-04T05:57:09Z` — unchanged since its own last edit | ✅ |
| `UOR-02-Build-Log.md` untouched | unchanged | File mtime `2026-09-04` — not modified this session | ✅ |

**Result: every prediction matched exactly. No mismatch to diagnose. Happy path CONFIRMED.**

Continuing to the Section 15 controlled-failure test before calling this adapter complete.
