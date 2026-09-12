# UOR-02 Adapter #2 — Reddit — Build Log

## Status as of 2026-09-06: DEFERRED — blocked on Reddit's Data API access request/approval process (support.reddithelp.com ticket form), not a technical defect. Revisit if/when approved, or if Damian submits and clears that application.

*(Original blocker finding from 2026-09-05, preserved below for the record — not deleted, not treated as abandoned.)*

**Update, 2026-09-06:** confirmed via Reddit's own official documentation that Reddit's self-serve "script"-type app registration is no longer instant — Reddit's current Responsible Builder Policy requires a manual application submitted through a support ticket form (support.reddithelp.com) and an approval process, not a bug or misconfiguration on any client, account, or network tried. This is a policy change on Reddit's side, not something workaroundable. Adapter #2 (Reddit) is **paused, not closed out as failed** — it keeps its slot/number in the adapter history. Work redirected to Adapter #3 (Stack Exchange) per Damian's decision; see `UOR-02-Adapter-03-StackExchange-Build-Log.md`.

**Environment:** n8n Community Edition, self-hosted via Docker, localhost:5678.

**Hard constraint honored:** `UOR-02-Source-Adapter-01-HackerNews` (ID `QUUq1uIRlTBTS1Zt`) and `UOR-02-Build-Log.md` were not touched, read, or modified as part of this work — this is a separate, new adapter and a separate, new log file.

---

## Step 1 — Credential check, via `n8n-mcp`

`n8n_manage_credentials` (action: `list`) against the live n8n instance returns exactly **one** credential:

```
id: JdsEfr8k3k0UoUQO
name: "UOR - Anthropic API"
type: anthropicApi
```

**No Reddit-related credential of any kind exists.** Confirmed via the actual live credentials list, not assumed.

## STOPPED HERE — blocker requiring Damian

Per instruction, not attempting any workaround. What's needed, precisely:

- Reddit requires a registered **"script"-type app** at `reddit.com/prefs/apps` (create app → type "script" → any name → redirect URI can be `http://localhost`) to get a `client_id` and `client_secret`.
- Script-app auth also requires a Reddit account's **username/password** in the token request (Reddit's "password" OAuth grant) — recommend creating a separate bot/dev Reddit account for this rather than using a personal one, for least-privilege (Section 14).
- Whatever credentials result must go into **n8n's credential manager** (Credentials → Add new) — never pasted into any chat, prompt, or workflow field in plaintext.

**I need a Reddit `client_id`, `client_secret`, and the username/password of the Reddit account to authenticate with, entered directly into an n8n credential — let me know once that's set up and I'll continue.**

Steps 2–5 (access-mechanism classification, design, build, testing including the Section 15 failure test, and final documentation) are not started and will not begin until this credential exists.
