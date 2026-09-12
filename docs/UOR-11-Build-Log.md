# UOR-11 — Alerts / Reporting — Build Log

**Date:** 2026-09-10
**Workflow ID:** `vOZxCcgeUcHZPbYo`
**Workflow name:** `UOR-11-Alerts-Reporting`
**Status:** Built, verified live, Section 15 controlled-failure test passed.

---

## 1. Purpose

UOR-11 has no dedicated master-instructions section (unlike UOR-01/09/16), so this build was designed directly against Section 4 (module list — "Alerts / Reporting"), Section 1 (mission: surface real, human-actionable opportunities), Section 22 (routing concept — downstream consumers act on flagged items), and Section 20 (build in testable increments).

Per Section 20, this is deliberately **v1: one narrow, well-understood alert condition**, not exhaustive alert coverage. The condition chosen: surface every opportunity that UOR-09's Productization Engine has already judged worth a human review draft (`LIGHT_VALIDATION_DRAFT` or `FULL_PRODUCTIZATION_DRAFT` tier), by reading UOR-10's reconciled historical-intelligence view — the single source of truth across all upstream modules — rather than re-deriving tier logic independently. This makes UOR-11 a thin, honest **review queue**, not a duplicate scoring engine.

## 2. Section 3 delivery-mechanism evaluation (A–G)

Before building, the delivery mechanism was evaluated against Section 3's seven criteria rather than defaulting to an external notification service:

| Criterion | Assessment |
|---|---|
| A. Fills a clearly identified gap | Yes — no existing module surfaces "needs human review" as a discrete, queryable list. |
| B. Materially improves output quality | A file-based queue is sufficient; no evidence a push/email service improves quality at this volume (14 records). |
| C. Materially reduces cost | External notification service = new cost with no demonstrated need. File output = $0. |
| D. Materially reduces fulfillment time | Not demonstrated — nothing in the project currently polls in real time; a file is read on demand. |
| E. Improves reliability/scalability | A local JSONL file, full-regeneration on every run, is simpler and more reliable than an external dependency at current scale. |
| F. Unlocks revenue | No. |
| G. Capacity justifies cost | No — corpus is currently 58 UOR-10 records / 14 alert-tier records. Nowhere near the volume that would justify a paid service. |

**Conclusion:** no new tool/service. Output is a plain JSONL file (`uor-11-alerts-needs-review.jsonl`), matching the project's existing file-based convention (same as UOR-01–10's outputs). This can be revisited under Section 3 if/when volume or a real downstream consumer (e.g., UOR-12's approval gateway) demands push delivery.

## 3. Trigger

Manual trigger (`When clicking 'Execute workflow'`), matching every other module's current build pattern. Intended to run on the same cadence as UOR-10 (i.e., after each full pipeline pass), invoked headlessly via `docker exec -e N8N_RUNNERS_BROKER_PORT=<unused port> n8n n8n execute --id vOZxCcgeUcHZPbYo`.

## 4. Inputs

- `/home/node/.n8n-files/uor-10-historical-intelligence.jsonl` (UOR-10's reconciled current-state view — one row per opportunity, full-regeneration file)
- `/home/node/.n8n-files/uor-01-opportunities.jsonl` (UOR-01's raw intake log — read only to look up `source_url`, a field UOR-10's schema does not currently carry; **known schema gap, not fixed here**, flagged in §9 below rather than reopening an already-verified module mid-task)

## 5. Nodes

1. **When clicking 'Execute workflow'** — manual trigger
2. **Read UOR-10 Storage** (`readWriteFile`, read) — `uor-10-historical-intelligence.jsonl`
3. **Extract UOR-10 Text** (`extractFromFile`, text)
4. **Read UOR-01 Storage** (`readWriteFile`, read) — `uor-01-opportunities.jsonl`
5. **Extract UOR-01 Text** (`extractFromFile`, text)
6. **Build Alerts** (`Code`, runOnceForAllItems) — the alert logic (full source in §6)
7. **Convert Alerts to File** (`convertToFile`, toText)
8. **Write Alerts File** (`readWriteFile`, write, `append:false`) — `/home/node/.n8n-files/uor-11-alerts-needs-review.jsonl`

Linear chain, no branches — matches the "sequential, not parallel-sibling" write pattern established as the fix in UOR-10 Phase 2 (avoids the non-atomicity class of bug found there; here there is only one write node, so the class doesn't even apply, but the linear read order was kept deliberately simple for the same reason).

## 6. Transformation logic (`Build Alerts` Code node, verbatim, live as of this build)

```javascript
const uor10Text = $('Extract UOR-10 Text').first().json.data || '';
const uor01Text = $('Extract UOR-01 Text').first().json.data || '';

function parseLines(text) {
  const out = []; let failures = 0;
  text.split('\n').filter((l) => l.trim().length > 0).forEach((line) => {
    try { out.push(JSON.parse(line)); } catch (e) { failures++; }
  });
  return { records: out, failures };
}

const uor10 = parseLines(uor10Text);
const uor01 = parseLines(uor01Text);

// UOR-10 doesn't currently carry source_url (a known schema gap, not fixed
// here -- flagged in the build log rather than reopening an already-verified
// module mid-task). Look it up directly from UOR-01's latest observation per id.
const sourceUrlById = new Map();
uor01.records.forEach((r) => {
  const existing = sourceUrlById.get(r.opportunity_id);
  if (!existing || String(r.date_stored) > String(existing.date_stored)) {
    sourceUrlById.set(r.opportunity_id, { source_url: r.source_url, date_stored: r.date_stored });
  }
});

// v1 alert condition (Section 20: one narrow condition, proven end-to-end,
// before generalizing): a record currently sitting in a productization tier
// that means UOR-09 judged it worth a human review draft -- LIGHT_VALIDATION_DRAFT
// or FULL_PRODUCTIZATION_DRAFT. This is a REVIEW QUEUE, not a one-shot "just
// happened" notification: it's a full regeneration every run (not an append
// log), so it always reflects the current true set of items needing review --
// no duplicate-fire risk by construction, and a failed run simply leaves the
// last known-good queue in place rather than silently losing anything.
const ALERT_TIERS = new Set(['LIGHT_VALIDATION_DRAFT', 'FULL_PRODUCTIZATION_DRAFT']);

const alerts = uor10.records
  .filter((r) => ALERT_TIERS.has(r.action_recommendation?.productization_tier))
  .map((r) => {
    const srcUrl = sourceUrlById.get(r.opportunity_id)?.source_url ?? null;
    return {
      opportunity_id: r.opportunity_id,
      alert_type: 'PRODUCTIZATION_READY_FOR_REVIEW',
      productization_tier: r.action_recommendation.productization_tier,
      reconciled_opportunity_score: r.final_score?.reconciled ?? null,
      reconciled_evidence_confidence: r.evidence_confidence?.reconciled ?? null,
      opportunity_title: r.raw_signal?.opportunity_title ?? null,
      source: r.source,
      source_url: srcUrl,
      demand_cluster_tags: (r.demand_cluster || []).map((t) => t.tag),
      requires_human_approval_for_execution: true,
      alert_formula_version: 'uor11-v1-review-queue-2026-09-10',
      computed_at: new Date().toISOString(),
    };
  });

const fileText = alerts.map((a) => JSON.stringify(a)).join('\n') + (alerts.length ? '\n' : '');

return [{
  json: {
    alert_count: alerts.length,
    uor10_record_count: uor10.records.length,
    file_text: fileText,
  },
}];
```

Key design decisions:
- **Full regeneration, not append** (`append:false`): the output always reflects the *current* true set of items needing review. This structurally prevents duplicate-fire — re-running never adds a second copy of an already-queued item, it just regenerates the same (or updated) queue.
- **`requires_human_approval_for_execution: true`** on every alert — a hard-coded reminder that this module surfaces candidates only; it does not authorize action (consistent with Section 13's decision-support framing, carried over from UOR-01's `Validation` node language, and anticipating UOR-12's approval gateway).
- **Both raw and reconciled fields surfaced where relevant** — `reconciled_opportunity_score` / `reconciled_evidence_confidence` are explicitly the *reconciled* UOR-10 fields (not a re-derivation), consistent with the reconciliation-surfacing pattern established in UOR-09/UOR-10.
- **`alert_formula_version`** stamped on every record, matching the versioning convention used elsewhere in the pipeline (e.g., UOR-06's tag-formula versioning) so future changes to the alert condition are traceable per-record.

## 7. Credentials required

None. Pure file I/O and JS — no external API calls.

## 8. Expected output

`/home/node/.n8n-files/uor-11-alerts-needs-review.jsonl` — one JSON object per line, one line per opportunity currently in `LIGHT_VALIDATION_DRAFT` or `FULL_PRODUCTIZATION_DRAFT` tier per UOR-10's latest reconciled view.

**Locked prediction (made before touching n8n):** using a standalone Python read of the then-current `uor-10-historical-intelligence.jsonl`, filtered to the two alert tiers, the predicted output was exactly 14 records with `opportunity_id`s:
`hn-49552616, hn-49555417, hn-49556698, hn-49556909, hn-49573833, hn-49576039, hn-49598051, hn-49601673, se-95480, yt--JFmcpwpQSs, yt-EChXCPolIDQ, yt-Mrg0nc3Mkew, yt-cqc0xni25n0, yt-pXPu3hAR2zE`

**Live result (execution 206):** `finished: true`, `status: "success"`. File verification: `wc -l` = 14, and the sorted `opportunity_id` list read back from the live file was an **exact match**, in the same 14 IDs, to the locked prediction. Spot-checked `hn-49552616`'s full record: all fields (tier, reconciled score/confidence, title, source, source_url via the UOR-01 lookup, demand cluster tags) matched expectations.

## 9. Known limitation (flagged, not fixed this build)

UOR-10's schema does not carry `source_url` — a genuine gap noted during UOR-10's build. Rather than reopening UOR-10 mid-task, UOR-11 works around it locally by reading UOR-01 directly for a "latest observation per ID" `source_url` lookup. This is scoped, narrow, and documented — not a silent workaround. Whether to add `source_url` to UOR-10's own schema is left as an open item (see checkpoint).

## 10. Failure handling / Section 15 controlled-failure test

**Baseline (post execution 206):** `uor-11-alerts-needs-review.jsonl` — 14 lines, mtime `2026-09-10T03:38:50.795Z`.

**Break applied:** `patchNodeField` on `Read UOR-01 Storage`, `parameters.fileSelector`, changed from `/home/node/.n8n-files/uor-01-opportunities.jsonl` → `/home/node/.n8n-files/uor-01-opportunities-NONEXISTENT.jsonl`. Read back via `n8n_get_workflow` (filtered) to confirm the exact change before running.

**Failing run (execution 207):** `docker exec -e N8N_RUNNERS_BROKER_PORT=5722 n8n n8n execute --id vOZxCcgeUcHZPbYo`. Result: visible failure — `NodeApiError`, `"No file matching the selector ... found"`, attributed to node `read-uor01`. `n8n_executions` confirmed a genuine new execution (id `207`, `status: "error"`, `finished: false`, distinct timestamps from 206).

**Corruption / swallow / duplicate-fire check:** `uor-11-alerts-needs-review.jsonl` after the failing run — still 14 lines, mtime **unchanged** (`2026-09-10T03:38:50.795Z`, byte-identical to baseline, confirmed via `md5sum`). The failure occurred upstream of `Build Alerts`/`Convert Alerts to File`/`Write Alerts File`, so the write step never ran — the last-known-good 14-record queue was left fully intact. **No alert was silently swallowed** (the existing queue remained visible and complete) and **nothing duplicate-fired** (no write occurred at all on the failing run).

**Revert:** `patchNodeField` restored `parameters.fileSelector` to `/home/node/.n8n-files/uor-01-opportunities.jsonl`. Read back to confirm exact revert.

**Recovery run (execution 208):** `docker exec -e N8N_RUNNERS_BROKER_PORT=5723 n8n n8n execute --id vOZxCcgeUcHZPbYo`. Result: `status: "success"`. File re-verified: 14 lines, **new** mtime (`2026-09-10T03:43:47.519Z`, proving a genuine fresh write, not a stale file), and the sorted `opportunity_id` list was identical to both the original locked prediction and the pre-failure baseline — confirming clean recovery with **no duplication** (still exactly 14 records, not 28) and no data loss.

## 11. Cost considerations

No LLM calls, no external API calls. Pure file I/O + JS execution. Negligible/zero marginal cost per run.

## 12. Dependencies

- Upstream: UOR-10 (`uor-10-historical-intelligence.jsonl`, full-regeneration file — must be re-run before UOR-11 to reflect the latest pipeline state) and UOR-01 (`uor-01-opportunities.jsonl`, for `source_url` lookup only).
- Downstream: none yet. Anticipated consumer is UOR-12 (Human Approval & Execution Gateway, not yet begun) — `requires_human_approval_for_execution: true` on every record was included specifically to anticipate that boundary.

## 13. Test procedure summary

1. Design v1 alert condition against live UOR-09/UOR-10 tier semantics (read fresh, not assumed).
2. Section 3 A–G evaluation of delivery mechanism → plain file, no new service.
3. Lock a deterministic prediction (exact count + exact IDs) via standalone script against live UOR-10 data, before touching n8n.
4. Build 8-node workflow, apply via `n8n_update_partial_workflow`, validate (0 errors).
5. Run headlessly (execution 206) → exact match to locked prediction, field-level spot check passed.
6. Section 15 controlled-failure test (executions 207/208) → visible failure, zero corruption, no swallowed alert, no duplicate-fire, clean recovery — all confirmed above.

## 14. Version / change history

| Date | Change |
|---|---|
| 2026-09-10 | Initial build. 8 nodes, v1 review-queue alert condition (`LIGHT_VALIDATION_DRAFT` / `FULL_PRODUCTIZATION_DRAFT`). Verified live (execution 206, exact match to locked prediction). Section 15 test passed (executions 207/208). |
