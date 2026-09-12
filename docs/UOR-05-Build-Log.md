# UOR-05 — Capability Matching — Build Log

## Status as of 2026-09-05, ~06:15 UTC: COMPLETE.

**Environment:** n8n Community Edition, self-hosted via Docker, localhost:5678.

---

## Step 1 — Live schema/data inspection (UOR-01, UOR-03, UOR-04), not assumed

**`tools_required` / `existing_capability_match` — confirmed null on all 25 records** (0/25 non-null for either field), via direct `docker exec` inspection of `uor-01-opportunities.jsonl`. Matches the build log's claim exactly — no module has populated either field yet.

**`category` / `normalized_opportunity` — confirmed dead placeholders for real data.** `category === "Unclassified — pending AI extraction"` for 22/25 records; `normalized_opportunity` (UOR-01's `Build Storage Record` maps this from `record.proposed_solution`) is the identical placeholder for the same 22/25. Only the 3 `TEST-001` records have a real value (`category: "Lead Generation Automation"`). Distinct category values across the whole file: exactly two — the placeholder, and that one real synthetic-test value. **Neither field is usable as a matching signal for any of the 22 real records currently in storage** — the same underlying condition UOR-03 hit and fixed (UOR-02's static placeholder, never overwritten by anything downstream — UOR-01's AI Extraction step only produces `ai_extracted_factors`, it does not re-classify `opportunity_category` or `proposed_solution`).

**UOR-03 and UOR-04 schemas re-confirmed** (from this session's own prior live inspection, unchanged since): UOR-03's enrichment (`evidence_confidence_v2`, `corroboration_status`, `recency_score`, etc.) and UOR-04's reconciliation (`reconciled_opportunity_score`, `score_variance`, `dimensions_scored_by_uor01`/`dimensions_not_yet_computable`) — **neither contains any field describing what an opportunity actually *needs*** to be fulfilled. Both are scoring/evidence layers, not requirements layers. **Conclusion: UOR-05 reads only UOR-01's storage file** — UOR-03/UOR-04's outputs have no bearing on capability matching, so joining them would add complexity with no functional benefit (Section 4: avoid unnecessary complexity). Flagged explicitly here rather than silently skipped.

### Design decision (Section 9), stated explicitly per instruction
**No structured "what does this opportunity need" field exists anywhere in the pipeline.** The only usable free text for real records is `raw_signal.opportunity_title`, `raw_signal.problem`, and `raw_signal.demand_signal` (all three are the real HN post title/body — `category` and `normalized_opportunity` are proven dead for 22/25 records, per above).

**Decision: UOR-05 v1 uses a deterministic keyword-to-stack rule table against that raw free text — not a new AI-extraction step.** Reasoning:
- Section 12 (cost governance) and Section 20 (prove the deterministic/cheap approach before adding automation) both favor exhausting a zero-cost, code-only approach first.
- Section 9 explicitly says "do not purchase tools automatically" — the same discipline extends here: don't reach for a new paid AI-extraction call by default when a documented, reproducible keyword rule table can answer the same coarse question ("does anything in our existing stack plausibly address this?") for free.
- This mirrors UOR-03/UOR-04's established pattern: deterministic, documented, versioned formula; no LLM in the loop.

**Explicit, honest limitation:** a keyword rule table is coarse. It will produce false negatives (real fits it doesn't recognize) and some false positives (e.g., "agent" matches "AI agent" but also "real estate agent" or "travel agent" — a known, accepted imprecision of a v1 keyword approach, not silently hidden). A genuine "what does this opportunity actually require" characterization would need a dedicated AI-extraction step — not built here, flagged as future work, same pattern as UOR-03's semantic-similarity gap.

---

## Step 2 — Architecture design

### Capability domains → Section 3 stack mapping (deterministic keyword table, v1)
| Domain | Section 3 stack items | Example keywords |
|---|---|---|
| `ai_reasoning_chat_agent` | Claude, Claude Code, Anthropic API, MCP, ChatGPT, Grok | ai, chatbot, agent, assistant, automat*, gpt, llm, prompt, claude, copilot |
| `workflow_automation` | n8n, MCP | workflow, automat*, integrat*, zapier, pipeline, n8n, no-code, low-code |
| `voice_audio_generation` | ElevenLabs | voice, audio, podcast, speech, narrat*, tts, text-to-speech |
| `video_production` | CapCut | video, editing, shorts, reel, youtube, tiktok, clip |
| `graphic_design_image` | Canva | design, graphic, thumbnail, banner, logo, poster, flyer, template |
| `app_software_development` | Xcode, MacBook | ios app, mobile app, swift, app store, software, saas, web app, website |
| `ai_media_generation` | KIE.ai | image generation, ai image, ai video, media generation, generative |

Keywords use word-boundary regex (`\bterm\b` for short/ambiguous acronyms like "ai"/"llm"/"gpt"/"tts"/"saas"/"n8n", `\bterm\w*` stem-matching for unambiguous English roots like "automat-"/"video"/"design"). This specifically avoids the substring false-positive that a naive `.includes('ai')` would produce (e.g. matching "maintain," "certain," "explain").

### Per-record output logic
- `signal_text_available`: false if title+problem+demand_signal are all empty or placeholder-only → `INSUFFICIENT_SIGNAL_TEXT`.
- Else, run all 7 domains' keyword sets against the combined text.
- Any domain hit → `MATCHED_EXISTING_STACK`, `fulfillment_feasible_now: true`, union of matched domains' stack tools listed, no cost (nothing new needed).
- No domain hit → `NO_STACK_MATCH_FOUND`, `fulfillment_feasible_now: false`, `estimated_new_tool_cost: "NOT_DETERMINED — requires manual review"` (Section 9: never invent a dollar figure for a gap this rule table can't characterize).
- **`automation_potential_pct`: reused, not recomputed**, from UOR-01's own `scoring_factors.automation_potential.estimate` (0–10 → ×10 for a percentage) — this directly answers Section 9's "what percentage can be automated?" without duplicating UOR-01's extraction logic.
- `category`/`normalized_opportunity` explicitly excluded from matching, with the reason recorded on every output row (`category_excluded_reason`).

### Formula version
`uor05-v1-keyword-stack-2026-09-05`

### Failure-handling / output
Reads only `uor-01-opportunities.jsonl` (never modifies it). Writes a **separate** new file, `/home/node/.n8n-files/uor-05-capability-matching.jsonl`, one row per UOR-01 line (25 rows expected, 1:1 — no reconciliation/deduplication here, unlike UOR-04, since capability matching is about the opportunity's *content*, not its scoring history; each submission is matched independently). Full overwrite each run (append OFF), same rationale as UOR-03/UOR-04.

### Planned node chain
1. Manual Trigger
2. Read UOR-01 Storage → Extract UOR-01 Text
3. **Match Capabilities** (Code, Run Once for All Items) — the logic above
4. Convert to File (`toText`)
5. Write UOR-05 Output (`/home/node/.n8n-files/uor-05-capability-matching.jsonl`, append OFF)

### Pre-build dry run against real data (lesson applied from UOR-04's mispredicted check)
Before writing anything into n8n, the exact matching algorithm above was run standalone via `docker exec ... node` directly against the live 25-record `uor-01-opportunities.jsonl` — not a guess, the literal code that will be deployed. Results, used as the falsifiable predictions for Step 3's verification:

- **Total: 25 rows** (1:1 with UOR-01 — no reconciliation/dedup at this stage, unlike UOR-04).
- **Status breakdown: 16 `MATCHED_EXISTING_STACK`, 9 `NO_STACK_MATCH_FOUND`, 0 `INSUFFICIENT_SIGNAL_TEXT`** (every record has at least some real title/problem/demand_signal text — none are entirely empty).
- `TEST-001` (all 3 rows, identical): `MATCHED_EXISTING_STACK`, domains `["ai_reasoning_chat_agent","workflow_automation"]`, stack tools `["Claude","Claude Code","Anthropic API","MCP","ChatGPT","Grok","n8n"]`, `automation_potential_pct: 80`.
- Full per-record predicted output captured and will be diffed against the actual n8n run in Step 3.

Continuing to Step 3 — build.

## Step 3 — Build (via `n8n-mcp` API, each node validated before saving)

Both distinct new node configs (`Write UOR-05 Output`, `Match Capabilities`) validated individually via `validate_node` — 0 errors on each (only the standard "consider error handling" best-practice suggestion, same as every Code node built this session). The read/extract/convert nodes reuse configs already proven identical in UOR-02/03/04.

**Workflow created:** `UOR-05-Capability-Matching`, ID `VodEowtpg3ypOuGu`, inactive.

Full `n8n_validate_workflow` result: `valid: true`, 6/6 nodes enabled, 5/5 connections valid, 0 errors, 0 warnings.

Chain as built:
```
When clicking 'Execute workflow' → Read UOR-01 Storage → Extract UOR-01 Text →
Match Capabilities → Convert to File → Write UOR-05 Output
```

**Blocked on human execution** (same constraint as every prior module, not a new blocker): Manual Trigger workflow, `n8n-mcp` cannot drive it. Not yet run.

### First live run, 2026-09-05 ~06:06 UTC — independently verified via raw `docker exec`, ALL CHECKS PASS

**Execution confirmed via `n8n_executions`:** execution `110`, `status: "success"`, `finished: true`, 6/6 nodes executed (Manual Trigger → Read UOR-01 Storage → Extract UOR-01 Text → Match Capabilities → Convert to File → Write UOR-05 Output), 167ms.

| Check | Predicted | Actual | Match |
|---|---|---|---|
| Output line count | 25 | **25** | ✅ |
| Parse failures | 0 | **0** | ✅ |
| Status breakdown | 16 `MATCHED_EXISTING_STACK`, 9 `NO_STACK_MATCH_FOUND`, 0 `INSUFFICIENT_SIGNAL_TEXT` | **16 / 9 / 0** | ✅ exact |
| TEST-001 (×3) | domains `[ai_reasoning_chat_agent, workflow_automation]`, tools `[Claude, Claude Code, Anthropic API, MCP, ChatGPT, Grok, n8n]`, `automation_potential_pct: 80` | **Identical on all 3 rows** | ✅ exact |
| UOR-01 storage untouched | unchanged | 25 lines, 427,220 bytes, mtime `2026-09-04 05:58:10 UTC` — unchanged | ✅ |
| UOR-03 / UOR-04 outputs | not read by design | 16,282 bytes / `06:53:11 UTC` and 20,931 bytes / `05:02:58 UTC` respectively — both unchanged from last known state; confirmed `Match Capabilities`' actual deployed code never references either file | ✅ |
| Append-OFF behavior | overwrite, not grow | Confirmed via node configuration (`options.append: false`) — **not yet empirically re-verified by a second run**; would need one more manual execution to observe directly rather than trust config alone | ⚠️ configuration-confirmed only |

**Result: every numeric/structural prediction matched exactly. No mismatch to diagnose.**

### Step 4 — cross-check finding (not a pass/fail condition, recorded as requested)
Compared UOR-01's 9 zero-scorable-factor IDs (`final_score: null`, `scoring_factor_coverage: "0/10"`) against UOR-05's 9 `NO_STACK_MATCH_FOUND` IDs.

**They are NOT the same set — only partial overlap.**
- Zero-scorable-factor (UOR-01): `hn-49553531, hn-49554580, hn-49555089, hn-49557192, hn-49557834, hn-49558794, hn-49559036, hn-49559320, hn-49560630`
- No-stack-match (UOR-05): `hn-49553531, hn-49554207, hn-49555089, hn-49555592, hn-49557834, hn-49557875, hn-49558794, hn-49559630, hn-49561016`
- **Overlap: 4 of 9** (`hn-49553531, hn-49555089, hn-49557834, hn-49558794`). 5 IDs appear only in one set, 5 only in the other.

**Interpretation:** partial correlation, not a single shared root cause. 4 records are thin/ambiguous enough to defeat both UOR-01's AI factor-extraction *and* UOR-05's keyword regex — plausibly genuinely low-content posts. But 5+5 fail only one mechanism each, meaning UOR-01's "insufficient information" LLM classification and UOR-05's deterministic keyword miss are measuring different things and only loosely correlated. Worth investigating both independently later (this ties back to the UOR-04 deferred note on UOR-01's extraction-prompt handling of thin posts) — not conflating them into one fix.

## UOR-05 Adapter: COMPLETE
Built, tested against real data with predictions locked in *before* the run (from an actual dry-run of the deployed algorithm, not a guess), independently verified field-by-field via raw file inspection — every structural/numeric prediction matched exactly, no rounding-up needed. Does not modify or duplicate UOR-01/UOR-03/UOR-04's logic; reads only UOR-01 (by explicit, documented design decision); writes only to its own output file. Both potentially-affected upstream files (UOR-01 storage, plus UOR-03/UOR-04 outputs checked for good measure) confirmed untouched.

### Deferred / non-blocking notes
- Append-OFF (overwrite) behavior confirmed by configuration, not yet empirically observed via a second run — low-risk given it's the identical, already-proven pattern from UOR-03/UOR-04, but flagged for whenever a second UOR-05 run happens anyway.
- Step 4's partial (not full) overlap between UOR-01's zero-factor records and UOR-05's no-match records is worth a closer look eventually, alongside UOR-04's already-deferred note on UOR-01's thin-post extraction handling — two related but distinct threads, not one bug.
- The keyword rule table has known, accepted imprecision (e.g., "agent" matches AI-agent posts and human-agent posts alike) — coarse v1, not a substitute for genuine requirement extraction.
