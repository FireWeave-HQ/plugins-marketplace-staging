---
name: "fw-feedback"
description: "Report that FireWeave behaved wrong in this session. Captures the agent session, enriches it with local FireWeave state (CLI status, cache lockfile, manifest versions, MCP tool drift), redacts it locally, takes the user's own remarks, proposes a category, and uploads the bundle to FireWeave after explicit confirmation. Use when the user says \"this feels off\", \"FireWeave is broken\", \"something's wrong with the rollout skill\", \"give feedback\", \"report a problem\", \"troubleshoot FireWeave\", or invokes `/fireweave:feedback`."
---
> Interaction: there is no skill-callable prompt tool — gate each decision by STOP and ask the user to pick a labelled option (or move the gate into the bundled MCP server via Elicitation).

# Feedback (session capture → FireWeave)

Turns "FireWeave felt wrong" into a report someone can act on: the session, the
local state that explains most failures, and the user's own words.

**Nothing leaves the machine without the user seeing what it is first.**

## Step 0 — Soft precondition (degrade, never PARK)

Run `mcp_rollout-server_ensure_auth` with `{ cwd: <absolute open-workspace root> }`.

Unlike every other FireWeave skill, this one **does NOT PARK** on a bad result.
A repo that is unauthenticated, unbound, or org-mismatched is exactly the repo
whose owner needs to be heard. Record the failure **as evidence** — note the
`ok`, `repo_binding` and `create_permission` values in the session account
written in Step 1 — and continue.

Collection never needs auth. Only the upload does, and Step 6 handles a missing
profile by keeping the bundle and printing the command that resumes it.

## Steps

| Step                                    | Action                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **1 — Write the session account**       | Write a structured account of this session to a temp file: what the user set out to do, which FireWeave skills and tools ran and in what order, what each returned, and where you believe it went wrong. Be specific and factual; this is the part a transcript cannot supply because it is your read of the failure. It is also the whole report on hosts where no transcript file exists.                                                                                                      |
| **2 — Take the remarks**                | One `a stop-and-ask user prompt`: a **mandatory free-text** "what felt off?" plus severity (`low`/`medium`/`high`/`critical`). Do not paraphrase the answer — it is stored verbatim and is never redacted. Do NOT ask for a category here.                                                                                                                                                                                                                                                                  |
| **3 — Propose a category**              | Read `references/taxonomy.md`. Pick the category that best fits the session and the remarks, then confirm with one `a stop-and-ask user prompt` with your proposal listed first and the other plausible options beside it. If nothing fits, propose `uncategorised` — a wrong label is worse than none.                                                                                                                                                                                                     |
| **4 — Collect (nothing uploads)**       | Run `fw feedback --dry-run --json --remarks-file <remarks.txt> --agent-summary-file <account.md> --category <key> --severity <level>`, where `account.md` is the file you wrote in Step 1 and `remarks.txt` holds the user's words from Step 2. The CLI collects the transcript and metadata, redacts locally, writes the bundle under `.fireweave/.cache/feedback/`, and prints a digest. **Nothing has left the machine at this point.**                                                       |
| **5 — Preview + explicit confirmation** | Show the user the digest as prose: bundle size, transcript message count, redaction counts by class, and the session span. Offer to print the bundle path so they can inspect it themselves. Then one `a stop-and-ask user prompt`: send it, or not. **Do not upload without an explicit yes.** A "no" is a complete, successful outcome — say where the bundle is and stop.                                                                                                                                |
| **6 — Submit**                          | On a yes, run `fw feedback --submit <bundlePath> --yes --json`. Report the returned `feedbackId` to the user. Handle the exit codes: `3` → not authenticated; relay the printed `fw login && fw feedback --submit …` line rather than trying to log them in. `4` → the server refused; relay its message verbatim (ingest that is not switched on for the org reads as _not enabled for this organization yet_). `5` → upload failed; the bundle is still on disk and `--submit` can be retried. |

## Rules

- **Remarks are mandatory.** A bundle with no human sentence in it is telemetry,
  not feedback. If the user will not write one, stop — do not invent one.
- **Never edit the redaction output.** If the digest shows fewer redactions than
  you expect, say so in the preview; do not hand-scrub the bundle.
- **Never re-run collection to "get a cleaner transcript."** The session that
  went wrong is the evidence.
- Every clarification uses `a stop-and-ask user prompt`.

## Tool manifest

```json
{
  "SKILL_EXPECTED_TOOL_MANIFEST": [
    { "name": "ensure_auth", "server": "rollout-server" }
  ]
}
```
