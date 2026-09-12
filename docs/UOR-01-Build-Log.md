# UOR-01 — Opportunity Analysis Core — Build Log

## Status as of 2026-09-03 (end of day): UOR-01 COMPLETE — all 9 nodes built, tested, and verified end-to-end

**Environment:** n8n Community Edition, self-hosted via Docker, localhost:5678.
**Workflow:** UOR-01-Opportunity Analysis Core.

Per Section 21 of the master project instructions, UOR-01's full pipeline is: Manual Trigger → Opportunity Intake → AI Extraction/Analysis → Structured Output → Initial Factor Scoring → Validation → Storage. All stages are now built and confirmed working, with confirmation based on raw on-disk file verification (`docker exec ... cat`), not merely n8n's UI execution indicator — see "Key lesson" below for why that distinction mattered.

### Final pipeline (all 9 nodes, all green, all independently verified)
1. **Manual Trigger** — controlled dev/test entry point.
2. **Edit Fields** (synthetic test record) — `signal_id = TEST-001`: AI lead-response automation service for small businesses, explicitly marked synthetic.
3. **Intake Validation** (Code node) — 15 required fields incl. `opportunity_category`, deterministic validation/normalization. CONFIRMED (pass + fail paths).
4. **AI Extraction** (Anthropic node, "Message a Model") — Model `claude-haiku-4-5-20251001` (Section 12 cost governance), temp 0.2, max tokens 1024. System prompt forbids a final score (Sections 7/24), requires "insufficient information" over guessing. CONFIRMED — non-deterministic across runs as expected (scores 66/68/64 across three genuine end-to-end runs today, proving the AI step is actually re-executing each time, not replaying cached output).
5. **Structured Output** (Code node) — parses/strips markdown fences, re-merges with intake record, sets validity/parse-error/missing-key flags. CONFIRMED, pass and fail paths.
6. **Initial Factor Scoring** (Code node) — deterministic Opportunity Score (0-100) and Evidence Confidence (0-100, hard-zero for synthetic records). `scoring_formula_version: "v1-equal-weight-2026-09-03"`. CONFIRMED.
7. **Validation** (Code node) — applies Section 26's interpretation matrix (threshold: high ≥ 60/100) and Section 25/26 compliance override. Every record carries `requires_human_approval_for_execution: true`. CONFIRMED — positive path, compliance-override path, and structural-block path all independently tested.
8. **Build Storage Record** (Code node) — maps onto Section 16's historical-intelligence schema, preserves full upstream record under `full_record` for audit (Section 6), serializes to a `storage_line` string. **Fixed 2026-09-03 (this session):** `storage_line` originally had no trailing newline (`JSON.stringify(storageRecord)`), which meant multiple appended records concatenated into one unparseable blob. Now reads `JSON.stringify(storageRecord) + "\n"` — each record lands on its own line, valid JSONL. CONFIRMED via raw byte inspection (`cat -A`, checking for `$` line-ending markers).
9. **Convert to File** → **Read/Write Files from Disk** (Write File to Disk, Append mode ON) — writes to `/home/node/.n8n-files/uor-01-opportunities.jsonl` inside the `uor01_data` Docker volume (see prior architecture notes below for why this path/volume was chosen). CONFIRMED end-to-end: three separate full-workflow executions today each appended one well-formed, newline-terminated JSON record, verified directly on disk.

### Key lesson from today's session: n8n's "successful execution" UI is not proof of a correct disk write
This is worth preserving prominently for future modules, because it cost significant time to diagnose. A green checkmark and "Workflow executed successfully" in n8n's UI only means the node ran without throwing an error — it does **not** mean the file it wrote actually contains what you expect, or is even the file you think it is. Two distinct real bugs hid behind apparently-successful executions today:

1. **Filename typo, `json1` vs `jsonl` (digit one vs. letter L).** The Read/Write Files from Disk node's "File Path and Name" field contained `/home/node/.n8n-files/uor-01-opportunities.json1` instead of `...jsonl`. Every "successful" execution was actually writing to a different, wrongly-named file. n8n never surfaced this as an error because the write itself succeeded — just to the wrong target. This typo appears to have predated this session (likely inherited from earlier work, not introduced by manual typing today) and had gone undetected because prior verification had trusted the UI's success indicator rather than reading the file directly. **Root fix:** rather than retyping the field by hand in the browser (which is how confusion compounded — L and 1 are visually near-identical, especially over screenshots), the field was corrected via direct API write through an MCP server connected to the terminal session (see the tooling note below), with the before/after value confirmed unambiguously via Python's `repr()` on both sides of the change. This is the recommended method for any future single-field fix that's error-prone to type: read the current value via API + `repr()`, write the corrected value via API, read it back + `repr()` again to confirm.
2. **Missing newline delimiter.** Described above under Node 8 — fixed by appending `"\n"` in the Build Storage Record Code node.
3. **A related, non-bug gotcha worth documenting:** clicking "Execute step" on a single node (rather than "Execute workflow" from the Manual Trigger) can replay cached upstream data instead of genuinely re-running the pipeline. This produced three byte-identical duplicate records early in today's session (same AI score, same timestamp, concatenated with no separators) that looked like a data-corruption bug but was actually just repeated cached-data replay compounding the missing-newline bug. **Standing practice going forward:** always use the "Execute workflow" button (runs from Manual Trigger through every node) to validate a genuinely fresh end-to-end run — reserve "Execute step" on an individual node for quick iteration when upstream data doesn't need to change.

**Verified final state (2026-09-03, 21:51):** `/home/node/.n8n-files/uor-01-opportunities.jsonl` contains exactly 3 well-formed JSONL records (TEST-001, scores 66/68/64 across independent AI calls), each newline-terminated, confirmed via `docker exec n8n cat -A`. No `.json1` file remains — the typo'd file and the correctly-named file were consolidated into one.

### Credentials & security incidents
An early credential-hygiene incident occurred during initial setup (a live API key was briefly pasted into chat) and was resolved; the affected key was rotated.

### Tooling note: MCP-based direct workflow edits
The build process used an MCP server connected to the terminal session with direct API read/write access to this n8n instance's workflows. Used deliberately and safely to fix the `json1`/`jsonl` filename typo directly via API, with `repr()`-verified before/after reads — a precise, low-risk approach for exactly this kind of fix when browser text entry is error-prone, as long as changes are deliberate, scoped to one field, and verified by reading back after writing. One general caution from this build: any unexplained canvas/config change should be checked against the possibility of independent API-level activity from another session before assuming a bug elsewhere.

### Storage architecture (established earlier this session, still current)
Two Docker volumes: `n8n_data` at `/home/node/.n8n` (n8n's own workflows/credentials/encryption key — untouched, permanently blocked to the Read/Write Files from Disk node by `N8N_RESTRICT_FILE_ACCESS_TO` with no override, by design, to protect n8n's own database) and `uor01_data` at `/home/node/.n8n-files` (UOR-01's business data — the only path this node type is allowed to write to by default). Container recreated with `--restart unless-stopped`. Storage mechanism: Build Storage Record (Code node, builds one JSON-Lines record) → Convert to File (Convert to Text File) → Read/Write Files from Disk (Write, Append ON) → `/home/node/.n8n-files/uor-01-opportunities.jsonl`.

Sources consulted: n8n Read/Write Files from Disk docs, n8n Convert to File docs, n8n community — "Access to the file is not allowed".

### Deferred / non-blocking notes
- Consider externalizing `requiredFields` / `regulatedKeywords` / `expectedFactorKeys` out of hardcoded arrays once real ingestion volume (UOR-02) starts.
- Word-boundary regex matching for compliance keywords is optional polish, low priority at current scale.
- Track actual per-call Anthropic API cost once several real (non-test) records run through, per Section 12.
- Evidence Confidence formula (Node 6) currently only uses 3 of the Section 24 evidence factors — the rest only become meaningful once multiple records can be cross-referenced, which is UOR-03 territory.
- Node 7's HIGH threshold (60/100) is a first-pass convention, not calibrated against real outcomes yet.
- Node 9's flat-file JSONL store is a deliberate v1 shortcut, not the final Section 16 database — flagged for migration to a real DB (UOR-10) once ingestion volume exists.
- Node 8's storage schema carries several intentional `null` placeholders (`tools_required`, `existing_capability_match`, `estimated_fulfillment_cost`, `estimated_revenue_potential_usd`, `duplicates_related_opportunities`, `demand_cluster`, `subsequent_outcome`) that later modules (UOR-05, UOR-06, UOR-09) or post-hoc review are expected to fill in — not a bug, a forward-compatible schema decision.

### Next step
UOR-01 (Opportunity Analysis Core) is complete and validated per Section 21's directive to prove the analytical pipeline before layering in automated ingestion. Ready to begin **UOR-02 — Source Ingestion** per Section 5's data-source governance rules (classify each candidate source's access mechanism — official API, RSS, etc. — before building any ingestion, and explicitly rule out unauthorized scraping).

**Note (top-level record ID field, confirmed 2026-09-04):** the storage schema's top-level record identifier is `opportunity_id`, not `signal_id` — `signal_id` exists only nested under `full_record.signal_id`. This distinction matters for any future code that needs to identify/dedupe records against this file.
