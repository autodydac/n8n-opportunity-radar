# UOR-10 — Database / Historical Intelligence — Build Log

## Status as of 2026-09-10: Phase 1 (storage layer) COMPLETE and verified live. Phase 2 (the full 8-module join) is designed but NOT built — per Section 20's build-in-increments rule, this phase proves the storage mechanism in isolation before any real data touches it.

**Environment:** n8n Community Edition, self-hosted via Docker, localhost:5678.

---

## Section 3 — existing-stack evaluation (done before designing anything)

**Confirmed live, not assumed:**
- Only the `n8n` container is running — no Postgres/MySQL/Mongo service exists in this environment.
- `DB_TYPE` / `DATABASE_*` env vars are unset — n8n runs on its own internal SQLite (`/home/node/.n8n/database.sqlite`, confirmed present with active WAL files). That's n8n's *own* state (workflows/credentials/executions), off-limits to business data by the same reasoning that already walls off the `n8n_data` volume from the Read/Write File node.
- No first-class SQLite node exists in n8n's core node catalog (confirmed via `search_nodes`); no community DB nodes are installed (`/home/node/.n8n/custom/` doesn't exist).
- `NODE_FUNCTION_ALLOW_EXTERNAL` is unset — Code nodes run in n8n's default sandbox and cannot `require()` external/native packages. An `sqlite3` package is bundled inside n8n's *own* installation, but that's for n8n's internal use, not exposed to user workflow code.

**Evaluated explicitly against Section 3 criteria A–G:**

| Criterion | Assessment |
|---|---|
| A. Fills a clearly identified gap | Marginal — real query/join capability would help *eventually*; every module already parses the full corpus into memory per run at zero cost at this volume (64 records) |
| B. Materially improves output quality | No |
| C. Materially reduces cost | **No — the opposite.** A real DB means a new container, new maintenance surface, new failure modes |
| D. Materially reduces fulfillment time | No — 64 records parse in milliseconds either way |
| E. Improves reliability/scalability | The one real point in favor — JSONL caused 2 real bugs early in this project (filename typo, missing newline), both fixed; the pattern has since been solid across 9 modules and dozens of runs |
| F. Unlocks revenue | No |
| G. Capacity justifies cost | No — not at 64 records |

**Conclusion: no new tool/service.** UOR-10 v1 uses the exact same proven mechanism (`Read/Write Files from Disk` + `Code` + `Convert to File`) already used by every other module. This doesn't contradict UOR-01's own build-log note calling JSONL "a deliberate v1 shortcut, not the final Section 16 database" — it concludes the stated graduation condition ("once ingestion volume exists") isn't met yet, and says so explicitly rather than deciding silently. Revisit if volume or query needs change.

---

## Purpose

Preserve structured historical intelligence per Section 16, consolidating UOR-01 through UOR-09's outputs into one canonical, queryable record per opportunity — and, distinctly, a genuine point-in-time history log so the system can actually observe how an opportunity's state changes over time (Section 16: "historical data should allow the system to improve decisions over time"). A current-state-only consolidated view wouldn't add anything beyond what the other 8 files already individually provide; the append-only log is what makes this a *historical* module, not just another join.

## Schema — reconciled against live field names before finalizing (not the spec alone)

Pulled the actual current top-level keys from all 8 live output files (UOR-01, 03, 04, 05, 06, 07, 08, 09 — UOR-02 has no own file, it writes into UOR-01). Two design decisions worth recording:

1. **Some Section 16 concepts are computed twice in this pipeline, at different reconciliation stages** — `final_score` exists as UOR-01's raw per-observation value *and* UOR-04's reconciled `reconciled_opportunity_score`; `action_recommendation` exists as UOR-01's raw-score interpretation *and* UOR-09's reconciled `gate_status`. UOR-10 surfaces **both**, clearly labeled (`initial` vs `reconciled`), rather than silently picking one — the same principle already applied when UOR-09's own gate was designed against UOR-04's reconciled fields instead of UOR-01's raw ones.
2. **UOR-07 and UOR-08 produce real signal with no literal Section 16 field name** (`content_opportunity_status`, `attention_potential`, `service_opportunity_status`). Included as clearly-labeled additions beyond the literal spec (`content_opportunity`, `service_opportunity`), not silently dropped.

| Section 16 field | Populated from (live-confirmed) |
|---|---|
| opportunity ID | `UOR-01.opportunity_id` (join key) |
| date discovered | `UOR-01.date_discovered` |
| source / source type | `UOR-01.source` / `source_type` |
| category | `UOR-01.category` (honestly still placeholder for all real records) |
| raw signal | `UOR-01.raw_signal` |
| normalized opportunity | `UOR-01.normalized_opportunity` |
| customer/problem | `UOR-01.customer_problem` |
| evidence | `UOR-01.evidence` (raw) + `UOR-03` (`corroboration_status`, `manipulation_flag`, etc.) |
| evidence confidence | `UOR-01.evidence_confidence` (initial) + `UOR-04.evidence_confidence_v2_latest` (reconciled, authoritative) |
| scoring factors | `UOR-01.scoring_factors` |
| final score | `UOR-01.final_score` (per-observation) + `UOR-04.reconciled_opportunity_score` (authoritative) |
| tools required | `UOR-01.tools_required` (declared, still null) + `UOR-05.matched_stack_tools` (actually populated) |
| existing capability match | `UOR-05.capability_match_status` / `matched_capability_domains` |
| estimated cost | `UOR-01.estimated_fulfillment_cost` (null) + `UOR-09.productization_draft.estimated_cost` when a draft exists |
| estimated revenue potential | `UOR-01.estimated_revenue_potential_usd` (null) + `UOR-09.productization_draft.pricing_hypothesis` when present |
| action recommendation | `UOR-01.action_recommendation` (initial) + `UOR-09.gate_status` / `productization_tier` (reconciled, authoritative) |
| status | `UOR-01.status`, `compliance_review_required`, `compliance_keywords_detected` |
| duplicates/related opportunities | `UOR-01.duplicates_related_opportunities` (null) + `UOR-03.corroborating_opportunity_ids` |
| demand cluster | `UOR-06` reverse-indexed tag membership |
| subsequent outcome | `UOR-01.subsequent_outcome` (honestly null — no module tracks real-world follow-up yet) |
| *(addition)* content/service opportunity signal | `UOR-07` / `UOR-08` |

## Trigger

Manual Trigger (matching every other module — headless-executable via `docker exec -e N8N_RUNNERS_BROKER_PORT=<port> n8n n8n execute --id <workflowId>`, confirmed working this session).

## Inputs

**Phase 1 (built):** none — a self-contained synthetic test record, isolated from all real data.
**Phase 2 (designed, not built):** UOR-01 through UOR-09's 8 live output files, read fresh each run.

## Nodes/components (Phase 1, as built)

```
Manual Trigger → Build Test Record
  → Build Current-State File Text → Convert to File → Write Current-State File (write, append: false)
  → Build History-Log Line       → Convert to File → Write History-Log File   (write, append: true)
```
8 nodes total. `Build Test Record` constructs one synthetic record (`opportunity_id: "UOR10-TEST-001"`, deliberately distinct from UOR-01's own `TEST-001` to avoid any collision or confusion) matching the full designed schema above, then two independent branches write it to each target file with format-appropriate timestamp fields (`computed_at` for the current-state view, `snapshot_taken_at` for the history log).

## Transformations

None beyond JSON construction/serialization in Phase 1 — no AI calls, no keyword matching, no scoring logic. Phase 2 will add the 8-way join (mirroring UOR-09's `Build Gate Decisions` join pattern, extended to 3 more source files).

## Prompts / models / APIs / credentials required

None in Phase 1. Phase 2 requires no new credentials either — it only reads existing flat files, same as UOR-03 through UOR-09.

## Expected output

- `uor-10-historical-intelligence.jsonl` — current-state view, one line per opportunity, full regeneration each run (`write`, `append: false`).
- `uor-10-history-log.jsonl` — append-only point-in-time log, one snapshot line per opportunity **per run** (`write`, `append: true`).

## Cost considerations

Zero — no AI/API calls, no new infrastructure. Storage grows linearly with runs × opportunities for the history log (bounded, monitorable, same class of consideration as every other JSONL file in this project).

## Failure handling — Section 15 controlled-failure test

1. **Baseline:** both files at 1 line each (from the first successful run), confirmed via `docker exec`.
2. **Break:** `Write History-Log File`'s path changed to a nonexistent directory, read back and confirmed exact.
3. **Failure confirmed — execution 200:** `status: error`, `NodeApiError`/`ENOENT` on `Write History-Log File`.
4. **A real, worth-documenting finding, not assumed away:** the two write branches are **not atomic**. Checking the actual files (not just the error node's reported ancestry) showed `Write Current-State File` — an independent sibling branch with no dependency on the failing node — **still executed successfully and rewrote the current-state file during the same failed execution**. Content was re-verified as fully valid JSON, not corrupted — this is a clean independent success, not a partial/garbage write — but it means a partial-failure state (one file updated, one not) is a real possible outcome of this design, not something to assume can't happen. Flagged for Phase 2 design awareness; not itself a defect requiring a fix, since neither file was ever left malformed.
5. **Zero-corruption check on the failing branch:** `uor-10-history-log.jsonl` confirmed completely unchanged (still exactly the pre-break content) — a failed write left no partial data.
6. **Revert:** exact original path restored, read back and confirmed, `n8n_validate_workflow` → 0 errors.
7. **Recovery run — execution 201:** `success`. Current-state file correctly still 1 line (overwrite semantics, freshly regenerated). History-log correctly grew to 2 lines — two distinct `snapshot_taken_at` timestamps for the same `opportunity_id`, exactly demonstrating the point-in-time tracking this file exists to provide.

**Result: PASSED**, with the non-atomicity characteristic documented rather than glossed over.

## Dependencies

Phase 2 depends on UOR-01 through UOR-09 all being in their current, already-verified state (all COMPLETE as of this session's compliance-fix regression). No UOR-01–09 code or data was touched by this build (confirmed via mtime check — all 8 real files' mtimes predate every UOR-10 test run in this session).

## Test procedure (for future re-verification)

1. Confirm `uor-10-historical-intelligence.jsonl` and `uor-10-history-log.jsonl` don't exist (or note their current state) before running.
2. Run headlessly, verify via `n8n_executions` for a genuine new execution ID and `docker exec` content checks — never the UI checkmark alone.
3. Confirm current-state file: exactly 1 line, valid JSON, `opportunity_id: "UOR10-TEST-001"`.
4. Confirm history-log file: append-only growth, one new line per run, each with a distinct `snapshot_taken_at`.
5. Confirm zero side effects on any UOR-01–09 real file (mtime check).

## Version / change history

- **2026-09-10:** Phase 1 (storage layer) designed and built. Section 3 evaluation against A–G criteria (no new tool). Schema reconciled against live field names across all 8 source files. Storage layer proven via a single isolated synthetic record — round-trip verified live (execution 199), Section 15 tested (200 fail / 201 recovery), zero side effects on real data confirmed.
- **2026-09-10 (Phase 2):** the full 8-way join, built and verified live against real data for the first time. See section below.

---

# PHASE 2 — 2026-09-10: full join against real UOR-01–09 data

**Workflow:** `UOR-10-Database-Historical-Intelligence-Phase2`, ID `eCZ5Gp45Lt7vwJcY`, inactive, 23 nodes:
```
Manual Trigger → (Read+Extract) × 8 [UOR-01,03,04,05,06,07,08,09]
→ Build Join (Code, the full schema mapping from Phase 1, one summary item out)
→ Convert to File → Write Current-State File (write, append: false)
→ Build History-Log Text → Convert to File → Write History-Log File (write, append: true)
```

## Step 1 — source data reconfirmed live, not assumed from the Phase 1 log

Pulled all 8 files fresh: field names identical to Phase 1's mapping (no drift). Live counts: UOR-01/03/05/07/08 = 64 raw lines each (UOR-01's unique id count grew to 59 since Phase 1, and `TEST-001` now appears 6× from this session's own headless-execution testing — not a data problem, just accumulated test runs); UOR-04/09 = 59 lines each (already unique); UOR-06 = 7 tag rows. Confirmed UOR-01's 59 unique ids exactly equal UOR-04's 59 ids (same set).

## Step 2 — the join

Built exactly as mapped in Phase 1's schema table. Real-opportunity universe = UOR-01's unique ids **excluding** `evidence.is_synthetic === true` (same exclusion discipline as UOR-09's own gate) — **59 unique − 1 synthetic (`TEST-001`) = 58 real records**, locked as the predicted row count before touching n8n.

## Step 3 — non-atomicity resolved

Phase 1's Section 15 test found the two write branches could complete independently even when one failed. **Resolved: the two writes are now a single linear chain, not parallel siblings** — `Build History-Log Text` sits *after* `Write Current-State File` in the graph, so it only ever runs once that write has actually succeeded. **Current-state is authoritative; the log only records confirmed successful state writes** — an audit-log entry claiming a snapshot was taken when the actual state file was never updated would be actively misleading, worse than the log simply lagging. This also means a failure now correctly halts the whole run before either write is reached (if it occurs upstream) or before the second write (if the first fails) — verified directly in the Section 15 test below, not just designed in theory.

## Step 4 — locked prediction (dry-run via `docker exec … node`, before touching n8n)

**Row count: 58.** Hand-verification target: `hn-49552616` (chosen because its exact values across all 8 modules were already independently known in detail from this session's compliance-fix work, making it a strong cross-check). Full predicted record generated and compared field-by-field against everything already verified this session — exact match on every field, including a genuine pre-existing data artifact the join's own honesty surfaces (not a join bug): `status.pipeline_status` still reads `"COMPLIANCE_REVIEW_REQUIRED"` (UOR-01's own `status` field, deliberately never touched by the narrowly-scoped compliance corrections) sitting next to `compliance_review_required: false`. Flagged for awareness, not fixed here — fixing it would be exactly the kind of undisclosed scope expansion this project has consistently avoided.

## Step 5 — verified live

**Execution 202**, `success`. **Exact match**: current-state file = 58 lines; `hn-49552616`'s full record matched the locked prediction field-for-field (`final_score`, `evidence_confidence`, `action_recommendation`, `status`, `content_opportunity`, `existing_capability_match` all spot-checked directly from the live file).

## Step 6 — Section 15 controlled-failure test

1. **Baseline:** 58/60 lines (60 = Phase 1's 2 synthetic history-log lines + this run's 58 new real ones), confirmed via `docker exec`.
2. **Break:** `Read UOR-05 Storage`'s path changed to a nonexistent file, read back and confirmed.
3. **Failure confirmed — execution 203:** `status: error`, 0.44s, `NodeApiError: "No file(s) found"`. Fails at node 8 of 23 — **before `Build Join` or either write is ever reached**, directly demonstrating the sequential-dependency fix from Step 3 working, not just designed.
4. **Zero-corruption check:** both output files completely unchanged — same line counts, same mtimes as the pre-break state.
5. **Revert:** exact original path restored, confirmed, `n8n_validate_workflow` → 0 errors.

## Step 7 — recovery + idempotency (combined, since no source data changed)

**Execution 204**, `success`. Doubled as the idempotency test required: current-state file regenerated at exactly 58 lines again, same field values for `hn-49552616` (`final_score`, `status`, etc. identical) — confirming deterministic regeneration, not drift. History-log correctly grew to **118** lines (60 + 58) — `hn-49552616` specifically now has **exactly 2** snapshot entries, one per real join run, each with a distinct `snapshot_taken_at` — exactly the point-in-time tracking behavior this file exists to provide, now demonstrated across multiple real records, not just the single isolated Phase 1 case.

## Step 8 — zero side effects on UOR-01–09

Confirmed via `docker exec ls -la` across all four Phase 2 executions: every one of UOR-01, 03, 04, 05, 06, 07, 08, 09's real files' mtimes predate every Phase 2 test run.

## Phase 2: COMPLETE

Final state: `uor-10-historical-intelligence.jsonl` = 58 lines (one per real opportunity, full regeneration each run). `uor-10-history-log.jsonl` = 118 lines (2 Phase 1 synthetic + 58×2 real, growing by 58 per future run). No UOR-01–09 code or data touched.

---

## Follow-up investigation — missing `source_url` field (2026-09-10)

**Question:** UOR-11's build log flagged that UOR-10's schema doesn't carry `source_url`, worked around by having UOR-11 read UOR-01 directly. Is this an actual defect (something currently broken/wrong because of the gap) or an acknowledged incompleteness with an adequate working workaround?

**Investigation, done before touching anything:**
1. Grepped n8n's raw workflow store (`/home/node/.n8n/database.sqlite`) for every reference to `uor-10-historical-intelligence.jsonl` across all 17 workflows in this instance. Found exactly **one** `"operation":"read"` node referencing it — UOR-11's own `Read UOR-10 Storage`. Every other match was either this build's own `write` operations or stored execution-history snapshots. **No other current or planned workflow reads UOR-10's current-state file.**
2. The one actual consumer (UOR-11) already has a verified, working substitute: it reads UOR-01 directly for a latest-observation-per-ID `source_url` lookup. This was already confirmed live and correct for all 14 queued records (UOR-11 Build Log §8) — every alert record carries the correct, real `source_url`.
3. No wrong, missing, or broken output currently exists anywhere in the live pipeline as a result of this gap. It has a verified workaround, not a symptom.

**Conclusion: NOT a defect. Left alone, as instructed when either outcome is acceptable.** This remains an acknowledged incompleteness in UOR-10's own schema (Section 16 lists no `source_url` field explicitly, but it's an obviously useful addition for a "historical intelligence" record) — worth adding in a future schema revision for its own sake, but nothing is currently broken that requires it now. If a second consumer of UOR-10 is ever built that needs `source_url` directly (rather than via UOR-11's workaround), that would be the point to revisit this, not before.
