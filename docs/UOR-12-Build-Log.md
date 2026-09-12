# UOR-12 — Human Approval & Execution Gateway (v1: Approval Tracking Only) — Build Log

**Date:** 2026-09-10
**Workflow ID:** `S2vbOMj2g2U2FMcX`
**Workflow name:** `UOR-12-Human-Approval-Gateway`
**Status:** Built, verified live (baseline + real-record join), Section 15 controlled-failure test passed.

---

## 0. Hard boundary — read this first, non-negotiable

**UOR-12 v1 is an APPROVAL TRACKING mechanism, not an execution mechanism.** It does not, under any circumstance, send anything, post anything, purchase anything, or otherwise perform any Section 13-listed consequential action — not even for a record marked `APPROVED`. An `APPROVED` status recorded by this module means a human (Damian) has authorized a *future, separately-built, separately-approved* execution workflow to act on this opportunity later. It is not itself a trigger.

This is enforced two ways, not just stated in prose:
1. **Structurally**: there is no node in this workflow, and no code path in `Build Approval Status`, that calls any external API, sends any message, or writes anywhere other than the local approval-status file. The workflow has exactly one write target.
2. **In the data itself**: every output record carries `execution_authorized: false`, hardcoded unconditionally in the Code node — this field cannot be set `true` by any input, including a real `APPROVED` decision (verified directly in §8 below). It exists so a future execution module has an explicit, unambiguous field to check, while making it structurally impossible for *this* module to ever claim to have authorized execution.

No part of this design resembles auto-executing on approval. If a future increment ever needs to, that is explicitly out of scope for v1 and would be a new, separately-approved module (see §11).

## 1. Purpose

Per Section 13 (Human Approval Gate — the clearest anchor for this module) and Section 4's module list, UOR-12 gives Damian a way to record a decision (approve / reject / defer) on each opportunity UOR-11 has queued for review, and feeds that decision back into the pipeline as a durable, queryable `approval_status`. Per Section 20, this is deliberately v1: **approval tracking only**, proven end-to-end on real queued data, before any execution capability is even discussed.

## 2. Section 3 evaluation — approval-capture mechanism

Two options were weighed before building anything:

| Option | Assessment |
|---|---|
| **A. n8n Form Trigger / webhook** | Technically available in the existing stack (no new subscription). But it requires the workflow to be **active and persistently listening** — a different operational mode from every other module built so far (all inactive, on-demand Manual Trigger workflows). At the current volume (14 queued records, one reviewer) there's no concurrency or remote-access need it would solve. Introduces a new failure surface (an always-on listener) for no demonstrated benefit yet. |
| **B. Manually-edited file** (chosen) | Zero new infrastructure, zero new operational mode, consistent with every file-based module already built. Damian appends one JSON line per decision to a plain file; the workflow reads it back. Malformed input is handled defensively (see §6), not trusted blindly. |

**A–G scoring:** (A) fills a real gap — yes, nothing currently records a human decision; (B) quality — a form reduces human input error but at 14 records the risk is negligible and is defended against in code anyway; (C) cost — both $0; (D) time — no meaningful difference at this volume; (E) reliability — a manual file has *fewer* moving parts than an active listener, not more; (F) revenue — n/a; (G) capacity — nowhere near the volume that would justify an active-workflow/webhook pattern.

**Conclusion: no new tool.** Plain JSONL file (`uor-12-approval-decisions.jsonl`, hand-edited by Damian) + a Manual Trigger workflow that reads it back, matching the exact pattern of every prior module.

## 3. Trigger

Manual trigger, run headlessly via `docker exec -e N8N_RUNNERS_BROKER_PORT=<unused port> n8n n8n execute --id S2vbOMj2g2U2FMcX`, intended to be re-run whenever Damian wants his latest decisions reflected in `uor-12-approval-status.jsonl`.

## 4. Inputs

- `/home/node/.n8n-files/uor-11-alerts-needs-review.jsonl` — UOR-11's review queue (the set of opportunities eligible for a decision).
- `/home/node/.n8n-files/uor-12-approval-decisions.jsonl` — **the file Damian edits by hand.** One JSON line per decision: `{opportunity_id, approval_status, decided_by, decided_at, notes}`. To change a decision, append a new line for the same `opportunity_id` with a later `decided_at` — the workflow always takes the latest by timestamp, so no in-place editing is required. Created empty as part of this build (no decisions exist yet — that is the true current state, not a placeholder to be treated as real data).

## 5. Nodes

1. **When clicking 'Execute workflow'** — manual trigger
2. **Read UOR-11 Storage** (`readWriteFile`, read) — `uor-11-alerts-needs-review.jsonl`
3. **Extract UOR-11 Text** (`extractFromFile`, text)
4. **Read UOR-12 Decisions** (`readWriteFile`, read) — `uor-12-approval-decisions.jsonl`
5. **Extract UOR-12 Decisions Text** (`extractFromFile`, text)
6. **Build Approval Status** (`Code`, runOnceForAllItems) — join logic (full source in §6)
7. **Convert Approval Status to File** (`convertToFile`, toText)
8. **Write Approval Status File** (`readWriteFile`, write, `append:false`) — `/home/node/.n8n-files/uor-12-approval-status.jsonl`

Linear chain, same pattern as UOR-11 — one write target, no branches.

## 6. Transformation logic (`Build Approval Status` Code node, verbatim, live as of this build)

```javascript
const uor11Text = $('Extract UOR-11 Text').first().json.data || '';
const decisionsText = $('Extract UOR-12 Decisions Text').first().json.data || '';

function parseLines(text) {
  const out = []; let failures = 0;
  text.split('\n').filter((l) => l.trim().length > 0).forEach((line) => {
    try { out.push(JSON.parse(line)); } catch (e) { failures++; }
  });
  return { records: out, failures };
}

const uor11 = parseLines(uor11Text);
const decisions = parseLines(decisionsText);

const ALLOWED_STATUSES = new Set(['APPROVED', 'REJECTED', 'DEFERRED']);

// Latest decision per opportunity_id (same Map+timestamp pattern used
// throughout this project). A malformed or unrecognized decision line is
// NOT applied -- the id simply falls back to PENDING_DECISION below rather
// than trusting bad input (Section 15: bad/low-confidence data).
const latestDecisionById = new Map();
let invalidDecisionCount = 0;
decisions.records.forEach((d) => {
  if (!d.opportunity_id || !ALLOWED_STATUSES.has(d.approval_status) || !d.decided_at) {
    invalidDecisionCount++;
    return;
  }
  const existing = latestDecisionById.get(d.opportunity_id);
  if (!existing || String(d.decided_at) > String(existing.decided_at)) {
    latestDecisionById.set(d.opportunity_id, d);
  }
});

// HARD BOUNDARY (Section 13, non-negotiable): this workflow only records
// what a human has decided. It never sends, posts, purchases, or triggers
// any external action -- not even for an APPROVED record. APPROVED here
// means a human has authorized a *future, separately-built, separately-
// approved* execution workflow to act on this opportunity later. It is not
// itself a trigger. execution_authorized is hardcoded false unconditionally
// -- this module has no code path that can ever set it true.
const results = uor11.records.map((r) => {
  const decision = latestDecisionById.get(r.opportunity_id);
  return {
    opportunity_id: r.opportunity_id,
    approval_status: decision ? decision.approval_status : 'PENDING_DECISION',
    decided_by: decision ? (decision.decided_by ?? null) : null,
    decided_at: decision ? (decision.decided_at ?? null) : null,
    notes: decision ? (decision.notes ?? null) : null,
    opportunity_title: r.opportunity_title,
    productization_tier: r.productization_tier,
    reconciled_opportunity_score: r.reconciled_opportunity_score,
    reconciled_evidence_confidence: r.reconciled_evidence_confidence,
    source_url: r.source_url,
    execution_authorized: false,
    approval_formula_version: 'uor12-v1-approval-tracking-2026-09-10',
    computed_at: new Date().toISOString(),
  };
});

const fileText = results.map((a) => JSON.stringify(a)).join('\n') + (results.length ? '\n' : '');

return [{
  json: {
    approval_status_count: results.length,
    decisions_parsed: latestDecisionById.size,
    decisions_invalid_or_ignored: invalidDecisionCount,
    file_text: fileText,
  },
}];
```

Key design decisions:
- **Absence of a decision defaults to `PENDING_DECISION`, never to `APPROVED`.** This is deliberate and load-bearing: silence must never be mistaken for authorization.
- **Full regeneration, not append** (`append:false`) — the output always reflects Damian's *current* decisions, matching UOR-11's convention. Re-running after a new decision simply updates that one record's status; it does not create a growing log of every past state (that role is explicitly left to UOR-10's history log, deferred — see §11).
- **Defensive parsing of the decisions file** — a line missing `opportunity_id`/`decided_at`, or carrying an `approval_status` outside `{APPROVED, REJECTED, DEFERRED}`, is not applied; the affected `opportunity_id` simply stays at `PENDING_DECISION` rather than trusting malformed input.

## 7. Credentials required

None. Pure file I/O and JS.

## 8. Expected output / live verification

`/home/node/.n8n-files/uor-12-approval-status.jsonl` — one JSON object per line, one line per opportunity currently in UOR-11's queue, carrying `approval_status`, decision metadata, key opportunity fields for context, and the hardcoded `execution_authorized: false`.

**Test 1 — baseline (empty decisions file, the true current state since Damian hasn't reviewed anything yet).** Locked prediction: 14 records, all `PENDING_DECISION`. Live run (execution **209**): exact match — 14/14 `PENDING_DECISION`, `execution_authorized: false` on all.

**Test 2 — join against real queued records.** To satisfy the requirement to test against real data without fabricating a real business decision on Damian's behalf, two clearly-marked *synthetic test* decisions were written against two real `opportunity_id`s from the queue (`se-95480` → `APPROVED`, `yt-cqc0xni25n0` → `REJECTED`), each with `decided_by: "TEST_HARNESS_VERIFICATION_ONLY"` and a `notes` field stating explicitly this is not a real decision and would be reverted immediately. Locked prediction: 14 total, exactly those 2 non-pending with the stated statuses, 12 `PENDING_DECISION`. Live run (execution **210**): exact match, including confirming `execution_authorized` stayed `false` on the `APPROVED` record — direct proof the hard boundary holds even for an approved item. **The decisions file was reverted to empty immediately after** (`: > uor-12-approval-decisions.jsonl`, confirmed 0 bytes) so the real decisions file never carries a fabricated decision. Recovery run (execution **211**) confirmed a clean return to the all-`PENDING_DECISION` baseline.

## 9. Failure handling / Section 15 controlled-failure test

**Baseline (post execution 211):** `uor-12-approval-status.jsonl` — 14 lines, mtime `2026-09-10T03:51:51.630Z`, md5 `f4fc76e...`.

**Break applied:** `patchNodeField` on `Read UOR-12 Decisions`, `parameters.fileSelector`, changed to a nonexistent path. Read back to confirm the exact change before running.

**Failing run (execution 212):** visible `NodeApiError`, `"No file matching the selector ... found"`, attributed to node `read-uor12-decisions`. Confirmed as a genuine new execution (`status: "error"`, `finished: false`, distinct timestamps).

**Corruption / swallow check:** `uor-12-approval-status.jsonl` after the failing run — mtime and md5 **byte-identical** to baseline. The failure occurred upstream of `Build Approval Status`/write, so no partial or corrupted approval state was ever written — **no decision was silently lost.**

**Revert:** restored `parameters.fileSelector`, read back to confirm exact revert.

**Recovery run (execution 213):** `status: "success"`, fresh mtime, 14 records, all `PENDING_DECISION` — identical to the pre-failure baseline, confirming clean recovery with no duplication.

## 10. Cost considerations

No LLM calls, no external API calls. Pure file I/O + JS. Negligible/zero marginal cost per run.

## 11. Explicitly deferred (not built, disclosed rather than silently skipped)

- **Feeding UOR-10's history log.** UOR-10's schema already carries a `subsequent_outcome` field per opportunity (confirmed live, Section 16-aligned) that an approval decision would naturally populate. This was not wired up in v1: doing it correctly requires either (a) appending a new immutable history-log *event* entry per decision — which needs its own dedup/idempotency logic so re-running UOR-12 without a new decision doesn't re-append a duplicate snapshot every time — or (b) mutating an existing historical snapshot's `subsequent_outcome`, which would violate the append-only/immutable nature of that log. Neither is a small addition, and Section 20 calls for one narrow, fully-proven increment first. Flagged as an open v1.1 item, not silently dropped.
- **Any execution/action module.** Per the hard boundary in §0 — deliberately, permanently out of scope for this build.

## 12. Dependencies

- Upstream: UOR-11 (`uor-11-alerts-needs-review.jsonl`) for the queue; the hand-edited `uor-12-approval-decisions.jsonl` for decisions.
- Downstream: none yet. Any future execution-capable module must be a **separate, separately-approved** workflow that reads `uor-12-approval-status.jsonl` — this module does not and must not call it directly.

## 13. Test procedure summary

1. Read master instructions fresh; anchored design on Section 13, Section 4, Section 20.
2. Read UOR-11's live queue (14 records) before designing anything.
3. Section 3 evaluation of the approval-capture mechanism → manually-edited file, no new tool.
4. Built 8-node workflow, applied via `n8n_create_workflow`, validated (0 errors).
5. Locked + verified baseline (empty decisions → all `PENDING_DECISION`, execution 209).
6. Locked + verified real-record join using clearly-marked, immediately-reverted synthetic test decisions against 2 real IDs (executions 210/211) — proving the mechanism works against real UOR-11 output without ever fabricating a real decision in Damian's name.
7. Section 15 controlled-failure test (executions 212/213) — visible failure, zero corruption, no swallowed decision, clean recovery.

## 14. Version / change history

| Date | Change |
|---|---|
| 2026-09-10 | Initial build. 8 nodes, v1 approval-tracking mechanism (`APPROVED`/`REJECTED`/`DEFERRED`/`PENDING_DECISION`), hardcoded `execution_authorized: false`. Verified live (executions 209–211). Section 15 test passed (212/213). No execution capability built or planned in this module. |
| 2026-09-10 (later) | Re-investigated the deferred history-log feed (§11) per an explicit "investigate before fixing" instruction. **Conclusion: still not needed, left deferred.** See §15 below for the investigation. |

## 15. Follow-up investigation — deferred history-log feed (2026-09-10)

**Question:** is there currently any consumer or use case that actually needs UOR-12 decisions fed into UOR-10's `subsequent_outcome`, or is a real decision currently unrecorded anywhere important because this doesn't exist?

**Investigation, done before building anything:**
1. Grepped n8n's raw workflow store for every reference to `subsequent_outcome` across all 17 workflows in this instance. Every match was UOR-10 itself writing the field (always `null` — no module currently populates it). **No node anywhere reads `subsequent_outcome` back.**
2. Grepped for every reference to `uor-10-history-log.jsonl`. Every match was a `write` operation (UOR-10's own append, including the deliberately-broken Section 15 test paths). **No node anywhere performs a `read` operation on the history log.** No consumer exists.
3. Checked whether any real decision is currently sitting unrecorded because of this gap: `uor-12-approval-decisions.jsonl` is empty (0 bytes) — Damian has not yet reviewed any queued opportunity, so there is no real decision anywhere in the system that this gap is failing to propagate. The only decisions ever written to that file were the synthetic test entries from the initial build, which were deliberately reverted (§8) precisely because they weren't real.

**Conclusion: NOT yet needed. Left deferred, as instructed when either outcome is acceptable.** There is no current consumer of `subsequent_outcome` or the history log, and no real approval decision exists yet for this gap to be losing. Building the idempotent-append logic now would be solving a problem that doesn't exist yet — Section 20 discipline (build in testable increments, don't automate ahead of a proven need) argues against it. **Revisit when either becomes true:** (a) a real decision is recorded in `uor-12-approval-decisions.jsonl` for the first time, or (b) some future module is designed to actually read `subsequent_outcome` (e.g., a later analysis that wants to correlate approval outcomes with original opportunity scoring). Neither condition holds today.
