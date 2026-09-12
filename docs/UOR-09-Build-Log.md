# UOR-09 — Productization Engine — Build Log

## Status as of 2026-09-09: COMPLETE — design locked, built, one real bug found and fixed, happy path and Section 15 failure/recovery all verified live via headless execution.

**Environment:** n8n Community Edition, self-hosted via Docker, localhost:5678.

**Design phase** (full detail in the conversation preceding this build, not repeated here): Section 10's 18-field productization schema, Section 25/26's four-quadrant gate reusing UOR-01's own `HIGH_THRESHOLD = 60` convention applied to UOR-04's *reconciled* per-opportunity fields, a synthetic-test-exclusion rule (Section 12 cost governance — never spend a real LLM call productizing `TEST-001`), and an `automation_percentage` deterministic passthrough from UOR-05 rather than letting the LLM invent it. Design was locked and approved before any building, with a dry-run prediction against the live 59-opportunity corpus: **13 `LIGHT_VALIDATION_DRAFT` / 0 `FULL_PRODUCTIZATION_DRAFT` / 3 `COMPLIANCE_REVIEW_ONLY` / 43 `NO_DRAFT`**.

**Workflow built:** `UOR-09-Productization-Engine`, ID `XS3VNGf5cwE7wqnT`, inactive, 20 nodes:
```
Manual Trigger → (Read+Extract) × 5 [UOR-01, UOR-03, UOR-04, UOR-05, UOR-06]
→ Build Gate Decisions (Code, the locked gate/join logic, 59 items out)
→ Route: Needs LLM Draft? (If)
    → [true]  Build Productization Prompt → Generate Productization Draft (Anthropic) → Parse Productization Draft
    → [false] (straight through)
→ Merge (append) → Build Storage Record → Convert to File → Write UOR-09 Output (overwrite, matching UOR-03–08's full-regeneration convention, not UOR-01/02's append convention)
```
Model: `claude-sonnet-5` for `FULL_PRODUCTIZATION_DRAFT`, `claude-haiku-4-5-20251001` for `LIGHT_VALIDATION_DRAFT`, selected dynamically per item via an expression on the Anthropic node's `modelId` resource locator (`mode: "id"`) rather than two separate branches — confirmed via `validate_node` and a live run that the expression resolves correctly.

`n8n_validate_workflow`: valid, 20/20 nodes, 20/20 connections, 0 errors, 0 warnings (only the standard generic error-handling suggestions every module gets).

---

## Real bug found and fixed before calling this complete: Anthropic node does not pass through upstream fields

**First live run (execution 180)** completed with `status: "success"` at every level — but file verification (not the green checkmark) showed all 13 LLM-branch records had lost `opportunity_id`, `productization_tier`, and every other field set before the AI call, and `Build Storage Record` then silently misclassified all 13 as `draft_generation_status: "NOT_ATTEMPTED"` — **even though 13 real, paid Anthropic calls had actually been made and completed successfully.** This is a worse failure mode than a crash: the workflow reported success and quietly discarded real, paid-for output while reporting that nothing had been attempted.

**Root cause, confirmed via node-level execution detail:** the Anthropic node's own output item is `{content: [...]}` **only** — unlike a Code node (which requires an explicit `...spread`), it does not merge the upstream item's other fields into its output at all. `Parse Productization Draft` had assumed `$json` after the AI call would still carry `opportunity_id` etc. through, matching neither Code-node behavior nor UOR-01's own proven pattern (its `Structured Output` node explicitly re-fetches pre-AI-call data via `$('Intake Validation').all()`, precisely because it already learned this lesson once).

**The AI output itself was good** — confirmed by inspecting the raw response directly: well-reasoned, honest content that correctly used the corroboration/evidence context fed to it (e.g. one draft explicitly flagged *"corroboration status is NO_COMPARABLE_RECORDS_FOUND — unclear if this is a widespread pain point"*). The bug was purely in the plumbing after the AI call, not the prompt design.

**Fix:** `Parse Productization Draft` now sources everything except the raw AI text from `$('Build Productization Prompt').item.json` (matched via n8n's pairedItem lineage tracking), mirroring UOR-01's own established pattern exactly.

---

## Verified live (post-fix)

**First run — execution 180**, 20:11:38–20:13:50 UTC — completed `status: "success"` but carried the bug above (caught via file verification). **Second run — execution 181**, `mode: "cli"`, `status: "success"`, 20:15:54–20:18:09 UTC, run after the fix. Output: `uor-09-productization.jsonl`, **59 lines**, **0 records missing `opportunity_id`**.

| Check | Predicted | Actual | Match |
|---|---|---|---|
| Total records | 59 | 59 | ✅ |
| `FULL_PRODUCTIZATION_DRAFT` | 0 | 0 | ✅ |
| `LIGHT_VALIDATION_DRAFT` | 13 | 13 | ✅ |
| `COMPLIANCE_REVIEW_ONLY` | 3 | 3 | ✅ |
| `NO_DRAFT` | 43 | 43 | ✅ |
| `draft_generation_status: GENERATED` | 13 | 13 | ✅ |
| LLM calls made (cost check) | 13 | 13 | ✅ |
| `NO_DRAFT` reason breakdown | 1 synthetic + 34 low/low + 8 not-yet-scored | exact match | ✅ |

**Content spot-checks, all passed:**
- A `LIGHT_VALIDATION_DRAFT` record's `productization_draft` has exactly the 7 designed fields (6 LLM-generated + `automation_percentage` attached deterministically afterward) — no pricing/sales fields present, as designed for this tier.
- All 3 `COMPLIANCE_REVIEW_ONLY` records contain **only** the static notice + `compliance_keywords_detected` (real matched keywords: `privacy`, `health`, `credit`+`insurance`) — zero proposed-solution/pricing/sales language, zero LLM call made for any of them.
- `NO_DRAFT` records: `productization_draft: null` as designed.

**First real test of Anthropic-credential resolution under headless execution, as flagged before building: confirmed working.** 13 real `claude-haiku-4-5-20251001` calls succeeded headlessly (`mode: "cli"`), producing genuinely good, context-aware content — not assumed to carry over from the earlier YouTube Query Auth result, verified independently.

**Build-time mechanics resolved, as flagged in the design:**
- The Anthropic node **does** auto-iterate per item without a `Loop Over Items` wrapper — 13 items in, 13 executions, confirmed via node-level execution detail.
- `Merge` in `append` mode correctly recombined the two branches losslessly — confirmed via the final 59-record count and zero duplicate/missing IDs across two full runs.

---

## Section 15 controlled-failure test — 2026-09-09, ~20:19 UTC

1. **Baseline:** 59 lines, confirmed via `docker exec` before touching anything.
2. **Break (via `n8n-mcp` API):** `Read UOR-04 Storage`'s `fileSelector` changed to a nonexistent path (`..._BROKEN.jsonl`) — read back and confirmed exact before running.
3. **Failure confirmed — execution 182:** `status: "error"`, `finished: false`, 0.5s. `NodeApiError: "No file(s) found"` on `Read UOR-04 Storage` (node 6 of 20 — fails long before `Build Gate Decisions` or any Anthropic call, so **zero LLM spend** on the failed run, confirmed via node-level detail).
4. **Zero-corruption check:** storage file re-verified at 59 lines, mtime unchanged.
5. **Revert (via `n8n-mcp` API):** `fileSelector` restored to the exact original value, read back and confirmed identical.
6. **Recovery run — execution 183:** ran clean end-to-end, `status: "success"`, 20 nodes, 59 records, 0 missing IDs, exact same 13/43/3 tier split and 13/46 status split as the first successful run. Full recovery confirmed, not assumed.

**Result: Section 15 gate PASSED** — visible failure, zero corruption, zero wasted spend on the failed run, clean recovery, matching the standard set by every prior module.

---

## UOR-09: COMPLETE

Checked against master instructions Sections 6 (evidence before conclusions — UOR-04's reconciled fields used, not invented), 7/24 (deterministic scoring/evidence discipline — the gate and `automation_percentage` are code-computed, never LLM-invented; the LLM only drafts genuinely creative fields), 10 (productization — all 18 fields implemented across two tiers), 12 (cost governance — synthetic exclusion, gate-before-spend, per-tier model selection), 13 (human approval gate — every record carries `requires_human_approval_for_execution: true`), 25/26 (compliance override proven absolute in both synthetic testing and live data — 3/3 flagged records got zero LLM exposure and zero sales language), 15 (failure handling — passed). No code in UOR-01 through UOR-08 was touched.

**Final storage state:** `uor-01-opportunities.jsonl` unchanged at 62 lines throughout this entire build (UOR-09 only reads). `uor-09-productization.jsonl` = 59 lines (13 `LIGHT_VALIDATION_DRAFT`, 3 `COMPLIANCE_REVIEW_ONLY`, 43 `NO_DRAFT`, 0 `FULL_PRODUCTIZATION_DRAFT`).

**Deferred / non-blocking notes:**
- `FULL_PRODUCTIZATION_DRAFT`'s `claude-sonnet-5` path is correctly configured but has never actually fired against real data (0 candidates currently exist, per the already-documented UOR-03 category-gate evidence ceiling) — worth a synthetic live-fire test if that model path needs independent confirmation before it matters for real.
- Total LLM spend for this build: 3 full runs × 13 Haiku calls = 39 calls, same cost class as UOR-01's own per-record extraction calls.

---

## Version / change history

- **2026-09-09:** Initial build. 20 nodes, two-tier gate (Section 25/26 matrix on UOR-04's reconciled fields), Anthropic field-passthrough bug found and fixed, Section 15 test passed. Final state: 13 `LIGHT_VALIDATION_DRAFT` / 0 `FULL_PRODUCTIZATION_DRAFT` / 3 `COMPLIANCE_REVIEW_ONLY` / 43 `NO_DRAFT`.
- **2026-09-10:** `LIGHT_VALIDATION_DRAFT`'s `maxTokens` raised **1024 → 2048** on the `Generate Productization Draft` node (one-line `patchNodeField` change), fixing the known `PARSE_FAILED` issue on `hn-49598051` (its verbose response was being truncated mid-JSON-string at the old 1024-token ceiling). `FULL_PRODUCTIZATION_DRAFT`'s `maxTokens` (3072) was left untouched. **Locked prediction before running:** the exact set of 14 `llm_call_made=true` IDs (unchanged — the tier gate is computed upstream in `Build Gate Decisions` and is unaffected by this change) and the 45 `NO_DRAFT` records (gate-excluded, never reach this node). **Live result (execution 214):** exact match — `hn-49598051` now `GENERATED` with a complete, valid, non-null draft (`customer_problem`, `target_customer`, `proposed_solution`, `risks`, `proof_of_concept_requirements`, `next_validation_experiment` all populated); the other 13 previously-`GENERATED` records remained `GENERATED`; all 45 `NO_DRAFT` records and the full 14-ID `LIGHT_VALIDATION_DRAFT` set were byte-for-byte identical to the locked prediction (tier distribution unchanged at 14/45). **Checked for downstream staleness, not assumed clean:** UOR-10's join reads `estimated_cost`/`pricing_hypothesis` from `productization_draft`, which went from `null` to populated for this record — but `LIGHT_VALIDATION_DRAFT`'s field set deliberately excludes pricing/cost fields by design (only `FULL_PRODUCTIZATION_DRAFT` drafts ever populate them), so UOR-10's existing `null` values for this record remain correct; no downstream re-run was needed. New final state: `uor-09-productization.jsonl` = 59 lines (**14** `GENERATED`, 0 `PARSE_FAILED`, 45 `NOT_ATTEMPTED`).
