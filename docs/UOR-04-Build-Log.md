# UOR-04 — Opportunity Scoring — Build Log

## Status as of 2026-09-05, ~05:10 UTC: COMPLETE.

**Environment:** n8n Community Edition, self-hosted via Docker, localhost:5678.

---

## Step 1 — Live schema inspection (UOR-01 + UOR-03), not assumed

**UOR-01 storage** (`/home/node/.n8n-files/uor-01-opportunities.jsonl`), confirmed live via `docker exec`: **25 lines.** Top-level keys: `opportunity_id, date_discovered, date_stored, source, source_type, source_url, category, raw_signal, normalized_opportunity, customer_problem, evidence, evidence_confidence, evidence_confidence_status, scoring_factors, scoring_factor_coverage, scoring_formula_version, final_score, opportunity_score_status, tools_required, existing_capability_match, estimated_fulfillment_cost, estimated_revenue_potential_usd, duplicates_related_opportunities, demand_cluster, subsequent_outcome, action_recommendation, interpretation_thresholds, status, ready_for_storage, requires_human_approval_for_execution, validation_notes, compliance_review_required, compliance_keywords_detected, full_record`.

**UOR-01's existing Opportunity Score formula** (already confirmed live earlier this session via `n8n-mcp` inspection of "Initial Factor Scoring," re-cited here rather than re-fetched since the node hasn't changed): deterministic 10-factor average — 7 direct factors (`demand_signal_strength, revenue_potential, automation_potential, recurring_revenue_potential, scalability, compatibility_with_existing_stack, speed_to_first_revenue`) plus 3 inverted factors (`competition_level, fulfillment_difficulty, customer_acquisition_difficulty`, each scored as `10 - estimate`), averaged and ×10, stored as `final_score` / `opportunity_score_status` (`FULL_COVERAGE`/`PARTIAL_COVERAGE`/`INSUFFICIENT_DATA`), version `v1-equal-weight-2026-09-03`. Per-observation, single-record scope only.

**UOR-03 enrichment** (`/home/node/.n8n-files/uor-03-evidence-enrichment.jsonl`), confirmed live: **25 lines** (1:1 with UOR-01, confirmed by matching count on both current checks and every prior test run). Top-level keys: `opportunity_id, source_type, source_name, date_discovered, days_since_discovery, base_confidence, recency_score, diversity_score, primary_source_score, sample_size_carried, manipulation_flag, manipulation_reason, low_content_flag, corroboration_status, corroborating_opportunity_ids, evidence_confidence_v2, evidence_confidence_v2_status, evidence_confidence_formula_version, computed_at`.

**Verified assumption (checked against UOR-03's actual code, not assumed):** UOR-03 processes UOR-01's file via a single `.map()` with no filtering, sorting, or item-dropping — its output preserves UOR-01's line order exactly, 1:1. This means the two files can be joined **positionally** (line *i* of UOR-01 ↔ line *i* of UOR-03) without needing `opportunity_id` to be unique across lines — important because it isn't (TEST-001 appears 3×). UOR-04's code adds an explicit guard: if the two files' line counts ever diverge (stale enrichment), it does not silently misalign them — see below.

---

## Step 2 — Architecture design (Section 7 compliant)

### What Section 7 actually asks for vs. what UOR-01 already does
Section 7 requires a deterministic, code-computed, documented Opportunity Score — UOR-01's "Initial Factor Scoring" already does exactly this, for a **single observation in isolation**. Per Section 3/4 discipline (don't rebuild what already works), UOR-04 does **not** re-extract or re-weight UOR-01's 10 factors. Its job, like UOR-03's relationship to UOR-01's Evidence Confidence, is to add what a single-record view structurally cannot:

1. **Cross-observation reconciliation.** When the *same* `opportunity_id` has been scored more than once (currently: only the synthetic `TEST-001` group, submitted 3× with independent AI Extraction calls), UOR-01 has no way to know that — each submission is scored blind to the others. UOR-04 groups by exact `opportunity_id`, and for any group with >1 member, computes a `reconciled_opportunity_score` (mean of the component scores) and, genuinely new information neither UOR-01 nor UOR-03 currently surfaces, a `score_variance` (max − min across observations) — a direct, quantified measure of how stable/reproducible the scoring actually is for a given signal. (UOR-01's own build log noted TEST-001 scored 66/68/64 across three genuine runs, calling it evidence the AI step "is actually re-executing each time" — UOR-04 turns that into a formal metric.)
2. **Explicit Section 7 dimension-coverage accounting.** Section 7 lists 21 potential scoring dimensions. UOR-01 covers 10. UOR-04 declares this explicitly per record — `dimensions_scored_by_uor01` (the 10) and `dimensions_not_yet_computable` (the remaining 11: `market_saturation, ease_of_market_entry, fulfillment_time, startup_cost, marginal_fulfillment_cost, required_new_skills, required_new_tools, defensibility, platform_risk, longevity, commercial_intent`), each explicitly `null` — not fabricated, not silently dropped. This mirrors UOR-01's own precedent (`Build Storage Record`'s explicit `null` placeholders for fields owned by later modules).
3. **Passthrough join, not recomputation, of `evidence_confidence_v2`.** UOR-04 carries UOR-03's superseding evidence value through for convenience/joinability but does **not** compute any new interpretation from it — combining Opportunity Score with Evidence Confidence into a final interpretation is Section 26's matrix, already implemented in UOR-01's own "Validation" node (on UOR-01's *original* 3-factor evidence_confidence, not UOR-03's fuller one — a real gap, but reconciling Section 26's interpretation against the superseding evidence value is a distinct, separate piece of work from Section 7 scoring, and out of scope for UOR-04 specifically. Flagged here rather than silently absorbed into this module.)

### Known, honestly-scoped limitation (not faked)
With only one source adapter live and 22 real, singleton (non-repeated) opportunities, the reconciliation logic will report `SINGLE_OBSERVATION` (score unchanged, variance 0) for 21 of them, and `RECONCILED_MULTI_OBSERVATION` only for the synthetic `TEST-001` group. This is the **correct, expected** output given current data — not a bug — same honest-limitation pattern as UOR-03's `NO_COMPARABLE_RECORDS_FOUND` result. The 11 not-yet-computable Section 7 dimensions genuinely require either a new data source (e.g., market-saturation/competitive-landscape research) or additional AI extraction not currently authorized — flagged, not invented.

### Formula (v1, documented)
```
formula_version: "uor04-v1-reconciliation-2026-09-05"

For each unique opportunity_id, over its group of 1+ UOR-01 observations:
  component_scores = [final_score for each observation with a non-null final_score]
  if component_scores is empty:
    reconciled_opportunity_score = null, status = NO_SCORABLE_OBSERVATIONS
  elif count == 1:
    reconciled_opportunity_score = component_scores[0], score_variance = 0, status = SINGLE_OBSERVATION
  else:
    reconciled_opportunity_score = round(mean(component_scores))
    score_variance = max(component_scores) - min(component_scores)
    status = RECONCILED_MULTI_OBSERVATION
```
No new weighting scheme invented — this is pure reconciliation across repeat observations of UOR-01's existing deterministic score, per Section 7's own instruction not to finalize/invent new weightings prematurely and to collect real observations first.

### Failure-handling guard (Section 15)
If UOR-01's and UOR-03's line counts don't match (stale enrichment relative to storage), UOR-04 still computes the opportunity-score reconciliation (which only needs UOR-01 data) but sets every `evidence_confidence_v2_latest` to `null` and emits a top-level `uor03_alignment_warning` in the run summary rather than silently joining misaligned rows.

### Planned node chain
1. Manual Trigger
2. Read UOR-01 Storage → Extract UOR-01 Text (proven pattern from UOR-02/UOR-03)
3. Read UOR-03 Storage → Extract UOR-03 Text (second independent read branch)
4. **Reconcile & Score** (Code, Run Once for All Items) — pulls both text blobs via `$('Extract UOR-01 Text')` / `$('Extract UOR-03 Text')`, does the grouping/reconciliation above, outputs one item with `file_text`
5. Convert to File (`toText`)
6. Write UOR-04 Output (`/home/node/.n8n-files/uor-04-opportunity-scoring.jsonl`, append OFF — full recompute each run, same rationale as UOR-03)

## Step 3 — Build (via `n8n-mcp` API, each node validated before saving)

All 6 distinct node configs validated individually via `validate_node` before creation — 0 errors on every node (only the standard "consider error handling" best-practice suggestion, same as every Code node built this session).

**Workflow created:** `UOR-04-Opportunity-Scoring`, ID `xXuDeXstVuD3bS3D`, inactive.

Full `n8n_validate_workflow` result: `valid: true`, 8/8 nodes enabled, 7/7 connections valid, 0 errors, 0 warnings.

Chain as built:
```
When clicking 'Execute workflow' → Read UOR-01 Storage → Extract UOR-01 Text →
Read UOR-03 Storage → Extract UOR-03 Text → Reconcile & Score →
Convert to File → Write UOR-04 Output
```

**Blocked on human execution** (same constraint as UOR-02/UOR-03, not a new blocker): Manual Trigger workflow, `n8n-mcp`'s `n8n_test_workflow` cannot drive it. Not yet run.

### First live run, 2026-09-05 ~05:02 UTC — independently verified via raw `docker exec`

| Check | Predicted | Actual | Match |
|---|---|---|---|
| Output row count | 23 | **23** | ✅ |
| Parse failures | 0 | **0** | ✅ |
| TEST-001 reconciliation | obs=3, score=66, variance=4, `RECONCILED_MULTI_OBSERVATION` | **obs=3, `component_scores: [66,68,64]`, score=66, variance=4, `RECONCILED_MULTI_OBSERVATION`, `evidence_confidence_v2_latest: 0`** | ✅ exact |
| All 22 `hn-` = `SINGLE_OBSERVATION`, variance 0 | 22× `SINGLE_OBSERVATION` | **13× `SINGLE_OBSERVATION` (variance 0) + 9× `NO_SCORABLE_OBSERVATIONS` (variance null)** | ❌ prediction wrong (see diagnosis) |
| UOR-01 / UOR-03 untouched | unchanged | UOR-01: 25 lines, 427,220 bytes, mtime `2026-09-04 05:58:10 UTC` (unchanged); UOR-03: 25 lines, 16,282 bytes, mtime `2026-09-04 06:53:11 UTC` (unchanged) | ✅ |

**Diagnosis of the 4th check's discrepancy:** checked UOR-01's live `final_score` field directly for all 22 `hn-` records. **9 of the 22 have `final_score: null`, `opportunity_score_status: "INSUFFICIENT_DATA"`, `scoring_factor_coverage: "0/10"`** — a genuine, pre-existing state of UOR-01's own stored data (the LLM returned "insufficient information" across every one of UOR-01's 10 scoring dimensions for these 9 records), unrelated to anything UOR-04 does. UOR-04's code produced exactly its documented behavior: `component_scores` empty → `score_reconciliation_status: "NO_SCORABLE_OBSERVATIONS"`, `score_variance: null` (per the Step 2 formula's own `if component_scores is empty` branch). **This is UOR-04 behaving correctly — the earlier prediction was incomplete, not the code.** No fix needed; documented here as a corrected expectation rather than a bug, per the standing practice of never rounding a mismatch up to a pass without diagnosis.

### Final resolution, confirmed 2026-09-05 ~05:20 UTC
Explicit sign-off received: `NO_SCORABLE_OBSERVATIONS` for the 9 zero-factor records is the correct, honest behavior — the original prediction was rejected, not the code. Root cause verified directly in UOR-01's own stored data (9 of 22 real `hn-` records have `final_score: null` / `scoring_factor_coverage: "0/10"` from UOR-01's "Initial Factor Scoring" node, independent of anything UOR-04 does). No code change made or needed.

## UOR-04 Adapter: COMPLETE
Built, tested (one run, one discrepancy from an incomplete prediction — not a code defect — diagnosed against live UOR-01 data and confirmed), independently verified via raw file inspection across all 5 checks, documented throughout. Does not modify or duplicate UOR-01's or UOR-03's logic; reads both, writes only to its own output file (`uor-04-opportunity-scoring.jsonl`); both source files confirmed untouched (line count + mtime) after execution.

### Deferred / non-blocking note (flagged here, not acted on — belongs to UOR-01, not UOR-04)
**9 of 22 real HN records (41%) came back with zero scorable factors from UOR-01's AI Extraction step.** Worth investigating eventually whether UOR-01's extraction prompt handles short/thin "Ask HN" posts poorly (versus richer "Show HN"/"Launch HN" posts, which tend to have more text for the LLM to extract factors from). This is a UOR-01 prompt-tuning question, not a UOR-04 concern — not fixed here, flagged for whenever UOR-01 is revisited.
