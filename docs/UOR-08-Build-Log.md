# UOR-08 — Service Opportunity Radar — Build Log

## Status as of 2026-09-06, ~23:30 UTC: COMPLETE.

**Environment:** n8n Community Edition, self-hosted via Docker, localhost:5678.

**Standing rules honored:** UOR-01 through UOR-07 not touched. Reads UOR-01 (raw text + `automation_potential`, reused) and UOR-06 (three relevant tags' membership, reused, not recomputed). Writes only to a new, separate file. Every execution claim, including my own, verified against `n8n_executions` before any file is inspected.

---

## Step 1 — Live verification and honest finding

**Record count / adapter roster:** confirmed live, unchanged since UOR-07 — 45 records (`TEST-001`: 3, `hn-`: 22, `se-`: 20). Schema unchanged.

**Service-seeking/willingness-to-pay signal swept and hand-verified — zero genuine, harder wall than UOR-07:**
- Only 2 keyword hits total (both "budget"), both false positives on inspection: `hn-49556909` ("budget cap on real API load tests" — API cost limit) and `hn-49552616` ("burn its whole budget halfway through a batch" — AI agent compute allowance). Neither is a hiring/service budget.
- The stronger keywords (`freelancer`, `contractor`, `consultant`, `will pay`, `hire`, `outsource`, `need someone`, `/hr`, `looking for a developer`) matched **nothing at all** across all 45 records — not even a false positive.
- **Real dollar figures exist in 2 records, but the wrong direction:** `hn-49554207` ($5) and `hn-49552616` ($19/$99) are both the *poster's own product pricing* (Show HN / Launch HN posts announcing what they charge), not demand-side willingness to pay for a service. Extractable fact, deliberately excluded from elevating `service_opportunity_status`.
- **UOR-06 tag reuse:** only `customer_communication_automation` has any members (`hn-49556909`, `hn-49556698`); `freelance_gig_service_demand` and `lead_gen_qualification` have zero members currently.

---

## Step 2 — Design (approved as proposed)

**Output fields:** `opportunity_id`, `service_seeking_keywords_matched`, `stated_dollar_amounts`, `uor06_tag_membership` (`{freelance_gig_service_demand, lead_gen_qualification, customer_communication_automation}`, reused), `service_opportunity_status`, `automation_potential_pct` (reused from UOR-01), `formula_version`, `computed_at`.

**Status logic:** explicit service-seeking keyword match → `GENUINE_SERVICE_SEEKING_SIGNAL`; UOR-06 tag membership alone (no keyword) → `WEAK_TANGENTIAL_MENTION`; neither → `NO_SERVICE_SIGNAL_DETECTED`. A bare dollar figure never elevates status by itself — deliberate, verified design decision (see synthetic test below).

**Formula version:** `uor08-v1-service-signal-2026-09-06`.

### Regex bug found and fixed pre-build (unlike UOR-07's bug, caught before touching n8n)
The first draft dollar-extraction regex (`/\$\s?[0-9][0-9,.]*.../`) captured trailing sentence punctuation — `"$500."` (period) and `"$5,"` (comma) instead of clean `"$500"` / `"$5"`. Fixed by requiring a comma only as a proper thousands-separator (`[0-9]{1,3}(?:,[0-9]{3})*`) and a period only as a proper decimal (`(?:\.[0-9]+)?`). Re-verified against all synthetic and real cases, including a multi-comma case (`"$1,250,000"`), before building anything.

### Synthetic branch test — every output field positively exercised (closing UOR-07's exact coverage gap)
Run standalone, not touching any storage:

| Case | Result |
|---|---|
| Explicit keyword + dollar figure | `GENUINE_SERVICE_SEEKING_SIGNAL`, keywords `["freelancer"]`, dollar `["$500"]` ✅ |
| UOR-06 `customer_communication_automation` membership only | `WEAK_TANGENTIAL_MENTION`, membership correctly reflected ✅ |
| No keyword, no dollar, no tag | `NO_SERVICE_SIGNAL_DETECTED` ✅ |
| Dollar figure present, no keyword (mirrors real `hn-49554207`/`hn-49552616` pattern) | `NO_SERVICE_SIGNAL_DETECTED`, dollar amounts still captured — design intent confirmed ✅ |
| `freelance_gig_service_demand` membership alone | `WEAK_TANGENTIAL_MENTION` ✅ |
| `lead_gen_qualification` membership alone | `WEAK_TANGENTIAL_MENTION` ✅ |

All 6 branches, all fields, confirmed correct before any n8n build work.

### Locked prediction against real 45-record data
**43 `NO_SERVICE_SIGNAL_DETECTED`, 2 `WEAK_TANGENTIAL_MENTION` (`hn-49556909`, `hn-49556698`, both via `customer_communication_automation` reuse), 0 `GENUINE_SERVICE_SEEKING_SIGNAL`.** Dollar amounts `$5` / `$19`, `$99` correctly surfaced on two `NO_SERVICE_SIGNAL_DETECTED` rows.

**Planned node chain:**
```
When clicking 'Execute workflow' → Read UOR-01 Storage → Extract UOR-01 Text →
Read UOR-06 Storage → Extract UOR-06 Text → Classify Service Signal →
Convert to File → Write UOR-08 Output
```

Not yet built. Continuing to Step 3.

## Step 3 — Build (via `n8n-mcp` API, each node validated before saving)

Both distinct new node configs validated individually via `validate_node` — 0 errors on each (standard suggestion only, same as every Code node this session). Read/Extract/Convert configs reuse patterns already proven identical in UOR-02–07.

**Workflow created:** `UOR-08-Service-Opportunity-Radar`, ID `AwhxEYLHe4iV76pt`, inactive.

Full `n8n_validate_workflow` result: `valid: true`, 8/8 nodes enabled, 7/7 connections valid, 0 errors, 0 warnings.

**Blocked on human execution** (same constraint as every prior module): Manual Trigger workflow, `n8n-mcp` cannot drive it. Not yet run.

### First live run, 2026-09-06 ~23:22 UTC — verified via `n8n_executions` before trusting anything

Execution `141` confirmed genuine: `status: "success"`, `finished: true`, 8/8 nodes executed. `status_breakdown`'s schema in the execution preview already showed only two keys (`NO_SERVICE_SIGNAL_DETECTED`, `WEAK_TANGENTIAL_MENTION`) with no `GENUINE_SERVICE_SEEKING_SIGNAL` key at all — consistent with the zero-genuine prediction before the raw file was even opened.

Raw output verified: **45 lines, 0 parse failures, status breakdown 43/2/0 — exact match.** Both `WEAK_TANGENTIAL_MENTION` rows (`hn-49556909`, `hn-49556698`) exactly as predicted, both correctly driven by `customer_communication_automation` tag membership alone (no keywords). Dollar amounts `$5` (`hn-49554207`) and `$19`/`$99` (`hn-49552616`) extracted clean — no trailing punctuation — confirming the pre-build regex fix held in the live environment, and both correctly stayed `NO_SERVICE_SIGNAL_DETECTED`.

**Upstream files confirmed byte-for-byte unchanged:** UOR-01 (687,452 bytes), UOR-03 (16,282 bytes), UOR-04 (20,931 bytes), UOR-05 (19,507 bytes), UOR-06 (3,367 bytes), UOR-07 (19,499 bytes) — all match their last independently-verified values exactly.

Happy path CONFIRMED.

## Step 4 — Section 15 controlled-failure test, 2026-09-06 ~23:26 UTC

**Break condition (via `n8n-mcp` API):** "Read UOR-01 Storage"'s `fileSelector` changed to a nonexistent path — read back and confirmed exact before proceeding.

**Human execution, verified via `n8n_executions` before trusting the report:** execution `142` confirmed genuine (new ID, `status: "error"`, `finished: false`). `executionPath` shows exactly 2 nodes (Manual Trigger success, Read UOR-01 Storage error) — the other 6 never appear. Corroborates the reported failure exactly: `"No file(s) found"`.

**Zero-corruption check:** `uor-08-service-opportunity-radar.jsonl` re-verified at exactly 45 lines / 18,727 bytes, mtime `2026-09-06 23:22:07 UTC` — matching execution `141`'s timestamp exactly, not `142`'s. The failed read produced no write whatsoever.

**Revert:** URL restored to the exact original value via API, read back and confirmed byte-for-byte identical.

**Result: PASSED.**

**Recovery run, 2026-09-06 ~23:27 UTC — verified via `n8n_executions` first:** execution `143` confirmed genuine (new ID, `status: "success"`, `finished: true`, later than the failure). Output re-verified: 45 lines, 0 parse failures, status breakdown still 43/2/0. mtime updated to `2026-09-06 23:27:58 UTC` (matching execution `143` exactly) while content stayed identical — correct full-overwrite behavior, no duplication.

**Upstream files reconfirmed byte-for-byte unchanged** after the entire cycle: UOR-01 (687,452 bytes), UOR-03 (16,282 bytes), UOR-04 (20,931 bytes), UOR-05 (19,507 bytes), UOR-06 (3,367 bytes), UOR-07 (19,499 bytes) — all identical to every prior check.

## UOR-08 Adapter: COMPLETE

### What was built
`UOR-08-Service-Opportunity-Radar` (ID `AwhxEYLHe4iV76pt`), inactive, 8 nodes: Manual Trigger → Read UOR-01 Storage → Extract UOR-01 Text → Read UOR-06 Storage → Extract UOR-06 Text → Classify Service Signal → Convert to File → Write UOR-08 Output. Reads UOR-01 (raw text + `automation_potential`, reused) and UOR-06 (three relevant tags' membership, reused, not recomputed). Writes only to `uor-08-service-opportunity-radar.jsonl`. Never touches UOR-01 through UOR-07.

### Bug found and fixed pre-build, unlike UOR-07's
The draft dollar-extraction regex captured trailing sentence punctuation (`"$500."`, `"$5,"`). Fixed to require proper thousands-separator/decimal structure before it ever touched n8n. The extended synthetic branch test — covering every output field, not just the primary classification, per the standing lesson from UOR-07's `'bot'`/"both" bug — confirmed all 6 branches correct, including the specific case (dollar figure present, no service-seeking keyword) that mirrors the real data's actual pattern.

### Test results summary
Happy path: exact match to a locked, pre-execution prediction (43/2/0) on the first live run — no bug surfaced post-build this time. Failure path: visible failure, `executionPath` confirms only 2/8 nodes ran, zero corruption, clean revert, clean recovery — every single execution across the whole build (happy path, failure, recovery) independently confirmed via `n8n_executions` before any file was inspected, no exceptions.

## Headline finding: this is now a pattern, not a one-off

**UOR-08 is the second consecutive "radar" module — after UOR-07 — to correctly confirm zero genuine signal for its target dimension in the current HN/Stack Exchange data.** UOR-07 found 0/45 genuine content-opportunity signal. UOR-08 found 0/45 genuine service-seeking/willingness-to-pay signal — an even harder wall, since not even a single false-positive keyword hit occurred on the strongest terms (`freelancer`, `hire`, `contractor`, etc.), unlike UOR-07 which at least had incidental mentions to inspect.

**Both modules are correctly built and correctly verified.** Neither is a defect. The pattern is structural: **Hacker News and Stack Exchange are not marketplace, hiring, or content-creator platforms** — they're a tech-discussion site and a tool-recommendation site, respectively. No amount of keyword refinement in either module can manufacture signal that isn't in the underlying data.

### Priority recommendation
**Before building further radar-style modules (UOR-09 onward) against this same two-adapter pair, adding at least one adapter genuinely suited to the missing signal types should take priority.** Concretely:
- For service/hiring signal (UOR-08's gap): a source like Reddit's r/forhire or r/slavelabour, or a freelance-platform API with permitted access, if one exists.
- For content-opportunity signal (UOR-07's gap, already flagged there): Reddit content-focused subreddits, the YouTube Data API, or Google Trends.

Without at least one of these, each new radar-style module built against only HN + Stack Exchange risks repeating the same outcome: correctly built, structurally starved of data. This isn't a call to stop building — UOR-07 and UOR-08 are both legitimate, working, honestly-reported modules — but the marginal value of building UOR-09+ the same way, against the same two adapters, is now demonstrably low until Source Ingestion (UOR-02) is expanded.

### Deferred / non-blocking notes
- `stated_dollar_amounts` is populated and correct, but currently only ever fires on supply-side pricing disclosures (Show HN/Launch HN posts), never on demand-side hiring budgets — worth re-checking once a marketplace-style adapter exists to see whether the extraction logic itself needs adjustment for that different context.
- The synthetic branch test's expanded field-coverage discipline (introduced this module) should become the standing template for all future radar-style modules, not just this one.
