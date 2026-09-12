# UOR-06 — Demand Clustering — Build Log

## Status as of 2026-09-06, ~22:30 UTC: COMPLETE.

**Environment:** n8n Community Edition, self-hosted via Docker, localhost:5678.

**Standing rules honored:** UOR-01 through UOR-05 not touched. Reads only UOR-01's storage file — UOR-03/UOR-04's outputs are scoring/evidence layers, no field describing what an opportunity *is about*, so they're not joined here. Never writes to `uor-01-opportunities.jsonl` or its `demand_cluster` field — output goes only to a new, separate file.

---

## Step 1 — Live verification (done before any design decision was trusted)

The design was drafted without live n8n access, so every assumption was checked against the real system before proceeding:

- **Schema:** confirmed unchanged — all expected top-level keys present, including `demand_cluster` (still `null`, confirmed as the field this module must never write to).
- **Category placeholder:** confirmed still dead for 100% of real records — `category`/`normalized_opportunity` are the exact literal placeholder for 22/22 `hn-` and 20/20 `se-` records.
- **Record count — differed materially from the design's assumption, flagged before proceeding:** the design assumed "~25 records, single adapter." Live count was **45** (22 `hn-` + 20 `se-` + 3 `TEST-001`) — Adapter #2/3 (Stack Exchange) had gone live since the design was drafted. This meant the design's stated limitation *"source_diversity_count will be capped at 1 until Adapter #2 exists"* was already outdated before this module was ever built.

Both discrepancies were reported and confirmed with Damian before any further work — see conversation record. Corrected basis: use 45 as the real dataset, and treat cross-source hits as genuinely possible from the first run.

---

## Step 2 — Two-stage dry run (honest record, not silently corrected)

### Stage 1: original 16-tag table, as first drafted
Run standalone via `docker exec ... node` against the real 45-record file, before any n8n build work. Result: **5 of 16 tags matched anything** (`scheduling_appointment_automation`: 3, `customer_communication_automation`: 2, `social_media_management`: 1, `saas_app_dev_demand`: 1, `real_estate_local_business_ops`: 1). No tag reached `RECURRING_PATTERN`. No tag had `source_diversity_count == 2`.

**Two evidence-grounded problems surfaced from this real data, not speculation:**
1. **Zero of the 20 real Stack Exchange records matched any tag at all.** The site's entire premise — "does a tool exist for X" — wasn't represented by any of the 16 tags as originally scoped (`saas_app_dev_demand` was scoped to *building* an app, not *finding* an existing one).
2. **A genuine false positive:** `real_estate_local_business_ops` matched `hn-49555089` — a long personal essay about guilt and rest with zero commercial content — on the phrase *"more real estate"* used colloquially (personal wealth/possessions), not the industry.

### Stage 2: two narrow, evidence-backed revisions, then re-run
Per Damian's explicit decision — both corrections grounded directly in Stage 1's real output, not a broader re-tuning pass:

1. **Added `tool_recommendation_seeking`** (new 17th tag, not compliance-sensitive): `alternative to`, `alternatives to`, `recommend a tool`, `recommend software`, `what tool`, `what software`, `best tool for`, `looking for a tool`, `web-based tool`, `app that supports`, `software that supports` — grounded directly in the three SE sample titles from Stage 1. Also added `payroll` to `financial_admin_automation`'s keyword list, grounded in the visible "Zoho Payroll" title.
2. **Narrowed `real_estate_local_business_ops`:** removed the bare `real estate` keyword entirely; replaced with `realtor`, `real estate agent`, `real estate investor`, `property management`, `real estate market`. `local business` and `small business ops` left untouched (not flagged for revision).

**Re-run result: 6 of 17 tags matched.** `real_estate_local_business_ops` now has **zero hits** (the false positive is gone, and nothing else in the dataset triggers the narrowed phrases). The other 4 original tags (`scheduling_appointment_automation`, `customer_communication_automation`, `social_media_management`, `saas_app_dev_demand`) are **byte-identical** to Stage 1 — confirming the two edits were narrowly scoped and didn't ripple elsewhere.

**`tool_recommendation_seeking` now fires — with one disclosed nuance, not a clean pass:**
- `se-95540` ("Alternative to Docker Content Trust (and cosign)?") — clean, unambiguous tool-seeking question.
- `hn-49559724` ("Ask HN: How do you market Open Source Software?... an alternative to Zendesk, Jira, and linear.") — a real match on the literal phrase, but the poster is describing **their own product** as an alternative, not seeking a recommendation. The *direction* of intent is inverted from the tag's name. Reported honestly rather than presented as a clean cross-source confirmation — same class of accepted imprecision as UOR-05's "agent" keyword (Section 9's known keyword-table limitation), not reverted, because the SE-side contribution is genuinely clean and the coverage gap it fixes is real.
- `financial_admin_automation`'s new `payroll` hit (`se-95546`, Zoho Payroll alternatives) is unambiguous.

### Final locked v1 table (the one actually deployed)
`formula_version: "uor06-v1-tag-cluster-2026-09-06"`. 17 tags total — the 16 as specified, minus `real_estate` narrowed, plus `tool_recommendation_seeking` added and `payroll` added to `financial_admin_automation`. Full keyword table matches the deployed Code node exactly (see workflow `kT5XggpeeLMHdHJE`, node "Tag & Cluster").

**Locked prediction for the live n8n run** (against the same 45-record file, assuming no new ingestion runs in between): **6 rows output** —
| tag | frequency_count | source_diversity_count | cluster_status |
|---|---|---|---|
| scheduling_appointment_automation | 3 | 1 | EMERGING_PATTERN_CANDIDATE |
| customer_communication_automation | 2 | 1 | EMERGING_PATTERN_CANDIDATE |
| tool_recommendation_seeking | 2 | 2 | EMERGING_PATTERN_CANDIDATE |
| social_media_management | 1 | 1 | SINGLE_OBSERVATION |
| saas_app_dev_demand | 1 | 1 | SINGLE_OBSERVATION |
| financial_admin_automation | 1 | 1 | SINGLE_OBSERVATION |

---

## Step 3 — Build (via `n8n-mcp` API, each node validated before saving)

All 6 distinct node configs validated individually via `validate_node` — 0 errors on every node (standard suggestions only, same as every module this session).

**Workflow created:** `UOR-06-Demand-Clustering`, ID `kT5XggpeeLMHdHJE`, inactive.

Full `n8n_validate_workflow` result: `valid: true`, 6/6 nodes enabled, 5/5 connections valid, 0 errors, 0 warnings.

Chain as built:
```
When clicking 'Execute workflow' → Read UOR-01 Storage → Extract UOR-01 Text →
Tag & Cluster → Convert to File → Write UOR-06 Output
```

**Blocked on human execution** (same constraint as every prior module): Manual Trigger workflow, `n8n-mcp` cannot drive it. Not yet run.

### First live run, 2026-09-06 ~21:51 UTC — independently verified via raw `docker exec`, EXACT MATCH

| Row | Predicted | Actual | Match |
|---|---|---|---|
| Line count / parse failures | 6 rows, 0 failures | **6 rows, 0 failures** | ✅ |
| scheduling_appointment_automation | freq 3, diversity 1, EMERGING_PATTERN_CANDIDATE | **freq 3, diversity 1, EMERGING_PATTERN_CANDIDATE, members [TEST-001 ×3]** | ✅ |
| customer_communication_automation | freq 2, diversity 1, EMERGING_PATTERN_CANDIDATE | **freq 2, diversity 1, EMERGING_PATTERN_CANDIDATE, members [hn-49556909, hn-49556698]** | ✅ |
| tool_recommendation_seeking | freq 2, diversity **2**, EMERGING_PATTERN_CANDIDATE | **freq 2, diversity 2, EMERGING_PATTERN_CANDIDATE, members [hn-49559724, se-95540]** | ✅ exact, cross-source hit confirmed as predicted |
| social_media_management | freq 1, SINGLE_OBSERVATION | **freq 1, SINGLE_OBSERVATION, [hn-49556698]** | ✅ |
| saas_app_dev_demand | freq 1, SINGLE_OBSERVATION | **freq 1, SINGLE_OBSERVATION, [hn-49556698]** | ✅ |
| financial_admin_automation | freq 1, SINGLE_OBSERVATION | **freq 1, SINGLE_OBSERVATION, [se-95546]** | ✅ |
| RECURRING_PATTERN anywhere | none expected | **none present** | ✅ |

**Result: every field of every row matched the locked prediction exactly. No mismatch to diagnose.**

**Upstream files confirmed byte-for-byte unchanged**, each checked against its own last legitimate write:
- UOR-01 storage: 45 lines, mtime `2026-09-06 20:39:57 UTC` — matches Adapter #3's own successful run exactly; UOR-06 only read it.
- UOR-03 enrichment: 16,282 bytes, mtime `2026-09-04 06:53:11 UTC` — unchanged.
- UOR-04 scoring: 20,931 bytes, mtime `2026-09-05 05:02:58 UTC` — unchanged.
- UOR-05 capability matching: 19,507 bytes, mtime `2026-09-05 06:06:00 UTC` — unchanged.

Happy path CONFIRMED. Continuing to the Section 15 controlled-failure test before calling this module complete.

## Step 4 — Section 15 controlled-failure test, 2026-09-06 ~21:59–22:26 UTC

**Break condition (via `n8n-mcp` API):** "Read UOR-01 Storage"'s `fileSelector` changed from `/home/node/.n8n-files/uor-01-opportunities.jsonl` to a nonexistent path — read back and confirmed exact before proceeding.

**Human execution:** "Read UOR-01 Storage" failed visibly with `"No file(s) found"`. Corroborated via `n8n_executions` (execution `135`): `status: "error"`, `finished: false`, `executionPath` shows **exactly 2 nodes** (Manual Trigger success, Read UOR-01 Storage error) — the other 4 nodes never appear in the path at all. No false-green anywhere.

**Zero-corruption check:** `uor-06-demand-clustering.jsonl` re-verified at exactly 6 lines / 3,367 bytes, mtime unchanged from the original happy-path run — the failed read produced no write whatsoever.

**Revert:** URL restored to the exact original value via API, read back and confirmed byte-for-byte identical.

**Recovery verification — caught and corrected a real discrepancy before accepting it:** the first "recovery confirmed" report did not correspond to any actual new execution — `n8n_executions` showed only the original success (`134`) and the failure (`135`), with no third execution despite the workflow's own `saveManualExecutions: true` setting confirming a real run would have been recorded. The output file's `computed_at` field was still `134`'s timestamp, not a fresh one — proving the file on disk was the *original* happy-path output, not new recovery output. This was surfaced plainly rather than accepted, a second genuine execution (`136`) was then run and independently confirmed via `n8n_executions` before any verification proceeded.

**Genuine recovery run (execution `136`, confirmed via `n8n_executions`):** `status: "success"`, `finished: true`. Output re-verified: all 6 rows **field-for-field identical** to the original happy-path output (same tags, frequencies, diversity counts, statuses, member IDs, representative titles — including the `tool_recommendation_seeking` cross-source hit) — only `computed_at` differs, as expected for a full-overwrite recompute. mtime updated to the new execution's timestamp (`2026-09-06 22:26:31 UTC`) while content stayed identical — exactly the expected append-OFF, full-overwrite behavior, no duplication.

**Result: PASSED**, after correctly catching and resolving a false "recovery confirmed" report along the way rather than taking it at face value.

## UOR-06 Adapter: COMPLETE

### What was built
`UOR-06-Demand-Clustering` (ID `kT5XggpeeLMHdHJE`), inactive, 6 nodes: Manual Trigger → Read UOR-01 Storage → Extract UOR-01 Text → Tag & Cluster → Convert to File → Write UOR-06 Output. Reads only UOR-01's storage file (UOR-03/UOR-04's scoring/evidence outputs deliberately not joined — no field there describes what an opportunity is *about*). Writes only to a new, separate file, `uor-06-demand-clustering.jsonl` — never touches `uor-01-opportunities.jsonl` or its `demand_cluster` field. Full overwrite each run (append OFF), same rationale as UOR-03/04/05.

### The two-stage tag table, documented honestly (not presented as right on the first pass)
**Stage 1 (as originally drafted, 16 tags):** dry-run against the real 45-record dataset found only 5/16 tags with any hit, and surfaced two real, evidence-grounded problems: (1) zero of the 20 real Stack Exchange records matched anything — the site's core "does a tool exist for X" pattern wasn't represented by any tag; (2) a genuine false positive — `real_estate_local_business_ops` matched a personal essay on the colloquial phrase "more real estate" (personal wealth), not the industry.

**Stage 2 (two narrow, evidence-backed corrections):** added a 17th tag, `tool_recommendation_seeking` (11 keywords grounded directly in the Stage-1 SE sample titles), added `payroll` to `financial_admin_automation`; narrowed `real_estate_local_business_ops` by dropping the bare "real estate" keyword and replacing it with five more specific phrases. Re-run confirmed: the false positive is gone, the 4 unaffected original tags are byte-identical to Stage 1, and `tool_recommendation_seeking` now catches real SE content — including one genuine cross-source hit.

**Final deployed table:** `formula_version: "uor06-v1-tag-cluster-2026-09-06"`, 17 tags, exactly as shown in the deployed "Tag & Cluster" node.

### Disclosed nuance, not presented as a clean result
`tool_recommendation_seeking`'s cross-source hit (`hn-49559724` + `se-95540`, `source_diversity_count: 2`) was scrutinized rather than taken as automatic corroboration: `se-95540` is a clean, unambiguous tool-seeking question; `hn-49559724` matched on "alternative to" but the poster is describing *their own product* as an alternative, not seeking a recommendation — a real match on the literal phrase with an inverted sense of intent. Reported honestly, not reverted, since the SE-side contribution and the coverage gap it fixes are both genuinely real. Same class of accepted v1 keyword-table imprecision as UOR-05's "agent."

### Test results summary
Happy path: exact match to a dry-run-locked, pre-execution prediction — all 6 rows verified field-by-field via raw `docker exec`, 0 parse failures, UOR-01/03/04/05 all confirmed byte-for-byte unchanged. Failure path: visible failure, zero corruption, clean revert, and — notably — a false "recovery confirmed" report was caught via the execution audit trail (not the file content or the UI) before being accepted, then a genuine recovery run was independently confirmed.

### Deferred / non-blocking notes
- **UOR-02's own build log has a documented gap** (flagged, not fixed here): it records Adapter #1 (HN) as complete and names Reddit as the Adapter #2 candidate, but never documents Stack Exchange going live as Adapter #2/3. The Section 5 governance question for Stack Exchange *was* answered in practice (verified live via `curl` before that adapter was built — see `UOR-02-Adapter-03-StackExchange-Build-Log.md`) but never recorded in UOR-02's primary log. Should get its own entry there at some point.
- **The tag table is a v1, single-snapshot artifact.** With 45 records and one dominant adapter pairing, most tags sit at `SINGLE_OBSERVATION` or low `EMERGING_PATTERN_CANDIDATE` — expected, not a defect. Worth revisiting the table (and possibly the frequency thresholds) once more adapters and higher ingestion volume exist, both to catch tag-coverage gaps the current ~45-record sample can't reveal and to see whether `RECURRING_PATTERN` ever fires meaningfully.
- Growth rate / trend velocity remains uncomputable from a single time-window snapshot, as originally noted — needs repeated timestamped ingestion history, unchanged from the original design.
- `source_diversity_count` is no longer structurally capped at 1 (Adapter #2/3 went live before this build), but with only two adapters live, most tags still land at diversity 1 by chance of content, not by architectural limitation — worth re-checking once a third adapter exists.

---

## FIX — 2026-09-08: `hr_recruiting_automation` co-occurrence requirement

**Trigger:** Phase 4 regression pass (against the 61-record corpus, post YouTube adapter) hand-verified that all 4 real hits on this tag (`hn-49598051`, `hn-49595476`, `hn-49578811`, `hn-49576039`) were false positives — every one was an HN "Who is Hiring" job posting matching on literal "hiring"/"onboarding," not a genuine demand signal for HR/recruiting-automation tooling. Diagnosed as systemic, not incidental: virtually any post in that thread will contain hiring-adjacent vocabulary.

**V1 attempt, tried and rejected before touching n8n:** required an automation-signal keyword (`automate`, `tool`, `ai`, `system`, `app`) to co-occur *anywhere in the record* alongside the hiring/recruiting keyword. Dry-run against the live corpus showed **all 4 original false positives still fired** — long HN posts routinely mention "AI" or "system" somewhere unrelated to the hiring context (e.g. `hn-49578811`: *"I do not use **AI** to screen your applications"* — negated, invisible to a keyword match — and *"an airborne **system**"*, a drone-defense system, nothing to do with recruiting). Anywhere-in-text co-occurrence is too loose for long, multi-topic posts. This was caught by dry-running against real data before applying, not assumed to work from the synthetic tests alone (which were single-sentence and didn't expose the gap).

**V2 (shipped):** the automation-signal keyword must appear in the **same sentence/paragraph** as the hiring/recruiting keyword — text is split on `<p>` (HN's own paragraph marker) and sentence-ending punctuation, and the tag only fires if one single chunk contains both. Verified via dry-run to eliminate all 4 real false positives while still firing on a genuine single-sentence automation-demand case.

**Keyword set deliberately narrower than the original ask:** `software` was excluded from the automation-signal list — it collides with ubiquitous job titles ("Software Engineer") in exactly this corpus and would just recreate the bug under a different word. `system` and `app` are matched as **exact whole words only**: exact `\bsystem\b` does not match "systems" (as in "Systems Engineer"), and exact `\bapp\b` does not match "apply"/"application"/"applicant" (every job posting's own call-to-action). `ai` is also exact, to avoid the non-exact prefix form matching "aid"/"aim"/"air"/etc.

**Synthetic branch tests (5 cases), all passed before the live dry-run:**
1. Real automation-demand case ("a tool to automate our hiring and resume screening... an AI system") → fires.
2. Plain job posting ("Hiring a Software Engineer... Apply now!... Systems Engineer") → does not fire.
3. Reconstructed `hn-49598051` text → does not fire.
4. Reconstructed `hn-49578811` text → does not fire.
5. Adversarial mixed-sentence case (hiring in one sentence, an unrelated app mention in the next) → does not fire under V2's same-sentence rule — documented as an accepted, more-conservative outcome, not a defect.

**Locked prediction (dry-run via `docker exec ... node` against the live 61-record corpus, before touching n8n):** `tags_with_hits` 8 → 7; `hr_recruiting_automation` drops out entirely (0 hits); all other 7 tags unchanged, identical members.

**Live verification:** applied via `n8n_update_partial_workflow`, `n8n_validate_workflow` → 0 errors. Run **headlessly** (`docker exec -e N8N_RUNNERS_BROKER_PORT=5692 n8n n8n execute --id kT5XggpeeLMHdHJE`) — execution **177**, `mode: "cli"`, `status: "success"`, confirmed via `n8n_executions`. Output file: exactly **7 lines**, `hr_recruiting_automation` absent (`grep -c` → 0), all 7 other tags' members byte-identical to the pre-fix state. **Exact match to the locked prediction.**

**Scope discipline:** only the `hr_recruiting_automation` tag's matching logic changed. `data_entry_extraction`'s own known false positive (`hn-49598051`, self-description of the poster's own product feature) is untouched — a separate tag with separate keyword logic, out of scope for this fix.

This corpus currently has **zero genuine automation-demand signal for HR/recruiting** — an honest finding, not a shortfall of the fix.
