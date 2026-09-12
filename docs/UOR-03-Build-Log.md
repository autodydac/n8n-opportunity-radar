# UOR-03 — Evidence & Verification — Build Log

## Status as of 2026-09-04, ~06:10 UTC: IN PROGRESS — architecture designed, not yet built.

**Environment:** n8n Community Edition, self-hosted via Docker, localhost:5678.

---

## Step 1 — Live UOR-01 schema/formula inspection (not assumed from build log)

Inspected via `n8n-mcp` (`n8n_get_workflow`, mode `filtered`, nodes `Initial Factor Scoring` + `Build Storage Record` + `Validation`) on workflow `0wrSuhTsnMMHWScF`.

**Confirmed exact live Evidence Confidence formula** (`scoring_formula_version: "v1-equal-weight-2026-09-03"`, in "Initial Factor Scoring"):
- If `is_synthetic === true`: `evidence_confidence = 0`, status `SYNTHETIC_TEST_ONLY`. Hard zero, no exceptions.
- Else, three sub-scores (each 0–10) averaged and ×10:
  1. `directnessScore` — bucketed from `ai_extracted_factors.evidence_directness.estimate` (LLM-classified text): 1 for hypothesis/speculation language, 5 for request/inquiry, 9 for transaction/budget/purchase, 3 default/unclear.
  2. `sampleSizeScore = min(ai_extracted_factors.evidence_sample_size.estimate, 10)`.
  3. `sourceMissingPenalty` = 0 if `evidence_source_missing` else 10 (binary, from deterministic intake check).
- `evidence_confidence = round(avg(1,2,3) × 10)`, status `INITIAL_ESTIMATE`.

This confirms the build log's deferred note exactly: **3 single-record factors only** — no cross-referencing against other stored records anywhere in UOR-01.

**Confirmed exact live storage schema** (`Build Storage Record`): top-level fields include `opportunity_id`, `date_discovered`, `date_stored`, `source`, `source_type`, `source_url`, `category`, `raw_signal.{opportunity_title, problem, demand_signal, market, geography}`, `normalized_opportunity`, `evidence.{evidence_notes, evidence_status, is_synthetic, evidence_source_missing}`, `evidence_confidence`, `evidence_confidence_status`, `scoring_factors` (= `ai_extracted_factors`), `final_score`, several `null` placeholders reserved for later modules, and `full_record` (complete upstream audit trail — includes `full_record.ai_extracted_factors.evidence_sample_size`, etc.). No drift from what's been observed empirically in the file all session.

---

## Step 2 — Architecture design (Section 6 / Section 24 compliant)

### Key design decision: separate enrichment file, not in-place mutation
UOR-03 will **not** rewrite `uor-01-opportunities.jsonl` in place. It reads it, computes enrichment, and writes a **new, separate file**: `/home/node/.n8n-files/uor-03-evidence-enrichment.jsonl`, one line per input record, keyed by `opportunity_id` (joinable, not duplicative of storage).

Reasons (Section 4 modularity, Section 6 evidence preservation, Section 15 failure isolation):
- Never risks corrupting or losing UOR-01's original audit trail.
- UOR-01 is append-only; a module that needs to *recompute across the whole corpus every run* (which UOR-03 must, since corroboration depends on the current full dataset) doing an in-place full-file rewrite would create a real write-write race against any concurrent UOR-01 append. A separate file sidesteps this entirely.
- UOR-03 is independently re-runnable/testable without touching UOR-01's data at all.

**This means each UOR-03 run OVERWRITES `uor-03-evidence-enrichment.jsonl` fresh** (not append) — it's a full batch recompute over the current corpus, not an incremental ingest. Documented here so this doesn't get mistaken for a bug later.

### Avoiding duplicated scoring logic (explicit instruction)
UOR-03 does **not** reimplement UOR-01's directness-bucketing, sample-size, or source-missing logic. It takes UOR-01's existing `evidence_confidence` (renamed `base_confidence` in UOR-03's output) as one **input factor**, and adds only the factors that genuinely require seeing multiple records at once — which UOR-01 structurally cannot do from a single record in isolation.

### Factors computed (Section 24), all deterministic/code-computed, no LLM in the loop
| Factor | How computed | Section 24 mapping |
|---|---|---|
| `recency_score` (0–10) | Deterministic decay from `date_discovered`: ≤7d=10, ≤30d=7, ≤90d=4, ≤365d=2, older/unknown=0 | recency |
| `diversity_score` (0–10) | Group records by exact `opportunity_id` first (catches literal repeats, e.g. TEST-001 ×3); if the group has ≥2 distinct `source_name` → 10 (genuine multi-source corroboration), if same source repeated → 0. For ID-singletons, a secondary pass looks for a different-ID record with matching `category` + Jaccard word-overlap ≥0.25 on title/normalized-opportunity text → 6 if different source, 2 if same source, 0 if no match | source diversity, corroboration/contradiction |
| `corroboration_status` | Categorical, one of `MULTI_SOURCE_CORROBORATED` / `SINGLE_SOURCE_REPEAT_ONLY` / `RELATED_SIGNAL_FOUND_DIFFERENT_SOURCE` / `RELATED_SIGNAL_FOUND_SAME_SOURCE` / `NO_COMPARABLE_RECORDS_FOUND`, plus `corroborating_opportunity_ids` for audit | corroboration/contradiction |
| `sample_size_carried` | Passthrough of `full_record.ai_extracted_factors.evidence_sample_size.estimate` — not recomputed, just re-surfaced for transparency | sample size |
| `primary_source_score` (0/10) | 10 if `evidence_source_missing === false` AND `source_url` matches `^https?://`, else 0 | primary-source availability |
| `manipulation_flag` | True if the same non-empty `source_url` appears across ≥2 **different** `opportunity_id`s (a real, deterministic duplicate-source signal) | manipulation/spam signals |
| `low_content_flag` | True if combined title+normalized-opportunity text is under 15 characters | manipulation/spam signals (weak, informational only — does not affect the score) |

### Known, explicitly-scoped limitation (not faked)
True semantic corroboration/contradiction detection (recognizing that two differently-worded posts describe the *same underlying demand*) would benefit from a real similarity/embeddings capability, which is not in the current stack (Section 3: no new paid tool without a justified gap). The keyword/category-overlap heuristic above is a genuine, deterministic, zero-cost approximation, but it will only catch near-duplicate phrasing, not true semantic equivalence. With only one live source adapter (HN) and 22 unrelated real posts, this limitation is expected to show up as `NO_COMPARABLE_RECORDS_FOUND` for nearly all of them right now — that is the **honest, correct** output given current data, not a bug. Flagged prominently rather than papering over it with a fabricated confidence number (Section 6).

### Formula (v1, documented per Section 24's reproducibility requirement)
```
formula_version: "uor03-v1-cross-reference-2026-09-04"

raw = 0.5 × base_confidence
    + 0.2 × (recency_score × 10)
    + 0.2 × (diversity_score × 10)
    + 0.1 × (primary_source_score × 10)

evidence_confidence_v2 = round(raw), clamped [0, 100]

Overrides:
  if is_synthetic          → evidence_confidence_v2 = 0   (preserves UOR-01's synthetic hard-zero intent)
  else if manipulation_flag → evidence_confidence_v2 = min(evidence_confidence_v2, 30)
```
Weights: `base_confidence` carries 50% (it already encodes directness/sample-size/source-missing at the single-record level per Section 24); the three genuinely-new cross-record dimensions split the remaining 50%.

### Planned node chain
1. Manual Trigger
2. Read Existing Storage (Read/Write Files from Disk, read `uor-01-opportunities.jsonl`)
3. Extract From File (operation `text`) — same proven pattern as UOR-02's dedup fix
4. **Cross-Reference & Score** (Code, Run Once for All Items) — parses all lines, computes the above per record, outputs one item: `{ record_count, parse_failures, file_text }`
5. Convert to File (operation `toText`, `sourceProperty: "file_text"`)
6. Write Enrichment File (Read/Write Files from Disk, write, `fileName: /home/node/.n8n-files/uor-03-evidence-enrichment.jsonl`, append OFF — full overwrite each run)

## Step 3 — Build (via `n8n-mcp` API, each node validated before saving)

All 6 node configs validated individually via `validate_node` before creation — 0 errors on every node (only the standard "consider error handling" best-practice suggestion seen on every Code node built this session, not addressed here either, consistent with UOR-01/UOR-02).

**Workflow created:** `UOR-03-Evidence-Verification`, ID `IBJfYBeMFzHd4bBN`, inactive.

Full `n8n_validate_workflow` result: `valid: true`, 6/6 nodes enabled, 5/5 connections valid, 0 errors, 0 warnings.

Chain as built:
```
When clicking 'Execute workflow' → Read Existing Storage → Extract From File →
Cross-Reference & Score → Convert to File → Write Enrichment File
```

**Blocked on human execution** (same constraint as UOR-02, not a new blocker): `n8n-mcp`'s `n8n_test_workflow` only supports webhook/form/chat triggers, confirmed empirically earlier this session — it cannot drive a Manual Trigger workflow. Testing requires a human click of "Execute workflow" in the browser. Not yet run.

### First live run, 2026-09-04 ~06:39 UTC — independently verified via raw `docker exec`, BUG FOUND

| Check | Result |
|---|---|
| File exists / line count | ✅ 16,678 bytes, 25 lines (matches 25 input records) |
| All lines valid JSON | ✅ 0 parse failures / 25 |
| TEST-001 ×3 | ✅ `corroboration_status: "SINGLE_SOURCE_REPEAT_ONLY"`, `evidence_confidence_v2: 0`, `evidence_confidence_v2_status: "SYNTHETIC_TEST_ONLY"` on all 3 — exactly as expected |
| 22 real `hn-` records | ❌ **FAILED — 22/22, not a partial miss.** All showed `corroboration_status: "RELATED_SIGNAL_FOUND_SAME_SOURCE"`, `diversity_score: 2`, `evidence_confidence_v2: 54` instead of the expected `NO_COMPARABLE_RECORDS_FOUND` |
| UOR-01 storage file untouched | ✅ Still 25 lines; mtime `2026-09-04 05:58:10 UTC`, unchanged since the last legitimate UOR-02 write — confirms UOR-03 never wrote to it |

**Root cause, confirmed via `grep` on the live storage file:** all 22 real `hn-` records carry the *exact same literal placeholder* `category` value — `"Unclassified — pending AI extraction"` (UOR-02's static placeholder for un-AI-classified signals; only the 3 TEST-001 records have a real category, `"Lead Generation Automation"`). The "Cross-Reference & Score" node's similarity logic gated matching on `category === category` and included category text in the word-overlap bag — since the placeholder is identical across every real record, every one of them trivially "matched" another uncategorized record, and the placeholder's own words ("unclassified", "pending", "extraction") leaked into the topical-similarity score as if they were real content. A design flaw, not expected behavior — the predicted `NO_COMPARABLE_RECORDS_FOUND` outcome was wrong given this interaction, and independent verification caught it.

### Fix applied, 2026-09-04 ~06:52 UTC, via `n8n-mcp` `patchNodeField`
Added `PLACEHOLDER_CATEGORY = 'unclassified — pending ai extraction'` and a single-point normalization: any record whose category (case-insensitively) equals that literal placeholder now gets `category = ''` before it's used for *either* the equality gate *or* the word-similarity bag. An empty category correctly fails the `!other.category` gate (never matches another uncategorized record) and contributes nothing to the word bag. Read back and confirmed the change landed exactly as intended before re-validating (`n8n_validate_workflow`: valid, 0 errors, 0 warnings, same standard suggestion as before).

### Second live run, 2026-09-04 ~06:53 UTC — independently re-verified via raw `docker exec`, ALL 5 CHECKS PASS

| Check | Result |
|---|---|
| File exists / line count | ✅ 16,282 bytes, 25 lines |
| All lines valid JSON | ✅ 0 parse failures / 25 |
| TEST-001 ×3 | ✅ `SINGLE_SOURCE_REPEAT_ONLY`, `evidence_confidence_v2: 0`, `SYNTHETIC_TEST_ONLY` — unaffected by the fix, as expected (TEST-001 has a real category, never touched the placeholder path) |
| 22 real `hn-` records | ✅ **All 22/22** now `corroboration_status: "NO_COMPARABLE_RECORDS_FOUND"`, `diversity_score: 0` (was 22/22 `RELATED_SIGNAL_FOUND_SAME_SOURCE` before the fix). `evidence_confidence_v2` now correctly tracks each record's own `base_confidence` (21 records: base 40 → v2 50; 1 record `hn-49560630`: base 47 → v2 54) instead of the flat, spuriously-inflated 54 seen before the fix |
| UOR-01 storage file untouched | ✅ Still 25 lines, 427,220 bytes, mtime `2026-09-04 05:58:10 UTC` — byte-for-byte and timestamp-identical across both UOR-03 runs |

**Result: PASSED, genuinely, on independent re-verification — not rounded up.**

## UOR-03 Adapter: COMPLETE
Built, tested (twice — once exposing a real bug, once confirming the fix), independently verified via raw file inspection (never just the green checkmark), and documented throughout per Sections 6/24. Does not modify or duplicate UOR-01's scoring logic; writes only to its own separate output file; UOR-01's original storage confirmed untouched across both test runs.

**Known, deliberately-scoped limitation (not a defect):** with only one live source adapter (HN) and 22 unrelated real posts in storage, `NO_COMPARABLE_RECORDS_FOUND` is the correct, expected result for essentially all real records right now — there is nothing to corroborate against yet. The `MULTI_SOURCE_CORROBORATED` / `RELATED_SIGNAL_FOUND_*` paths remain untested against genuine cross-source data because no such data exists yet; they're exercised today only by the synthetic TEST-001 group (same-ID, same-source case). This will need re-validation once a second source adapter (e.g. Reddit, per UOR-02's deferred notes) produces a record that plausibly overlaps with an existing one.

**Deferred / non-blocking notes:**
- True semantic corroboration/contradiction (recognizing the *same underlying demand* described in different words) would benefit from an embeddings/similarity capability not currently in the stack — a Section 3 cost/tool decision, not undertaken here. The keyword/category-overlap heuristic is a real, deterministic, zero-cost approximation with known limits, not a substitute for genuine semantic matching.
- `manipulation_flag` (duplicate `source_url` across distinct `opportunity_id`s) and `low_content_flag` have not yet been exercised by any real positive case in the current 25-record dataset — logic is in place and validated, but unconfirmed against a true positive.
- Per Section 18, this file plus the inline code comments constitute UOR-03's build documentation; no separate node-level doc has been written yet.
