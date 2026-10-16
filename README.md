# Multi-Agent Daily Digest (n8n)

Week 5 PM/TPM assignment. A scheduled multi-agent system that reads **Slack, Gmail and Google Calendar**. A specialist AI agent summarizes each source in parallel. A **Synthesizer** agent then dedupes, prioritizes and flags which sources were unavailable, and posts a single digest to Slack every morning at 8:00 AM IST.

![Workflow canvas](screenshots/canvas.png)

## Files

| File | Purpose |
|---|---|
| `workflows/multi-agent-daily-digest.json` | Main workflow (28 nodes) |
| `workflows/multi-agent-digest-error-handler.json` | Error workflow (Error Trigger -> Slack alert) |
| `screenshots/canvas.png` | Canvas of the main workflow |
| `screenshots/digest-output.png` | Example digest posted in Slack |
| `screenshots/error-alert.png` | Example error alert in Slack |

## Architecture: 3 layers

```
Schedule (8 AM IST)
   -> Orchestrator -> Guardrails -> State Management          [Layer 1: Control]
   -> Slack | Gmail | Calendar  (IF enabled -> fetch -> prep -> agent -> package)
   -> Merge -> Build Synthesis Input -> Synthesizer Agent      [Layer 2: Execution]
   -> Validate & Save State -> Send Digest to Slack
Any failure -> Error Workflow -> #project-october-error        [Layer 3: Reliability]
```

### Layer 1: Control

| Node | Role |
|---|---|
| **Daily 8 AM Trigger** | Schedule trigger that runs daily at 08:00, workflow timezone Asia/Kolkata |
| **Orchestrator** (Set) | One central config: user, role, team, VIP contacts, priority keywords, per-source enable flags, `slackSearchQuery` (`in:#project_updates`), `gmailQuery` (`-category:promotions -category:social`), `gmailMaxResults` (25), `lookbackHours` (24) |
| **Guardrails** (Code) | Pre-flight checks that run before any API or LLM call. Misconfigured sources are skipped with a warning; bad numeric values are reset to safe defaults. If **all** sources are disabled, it throws, which fires the error workflow. |
| **State Management** (Code) | Creates a `sessionId` and run timestamp, then loads `previousDigest` and `lastRunAt` from workflow static data for delta tracking. |

### Layer 2: Execution

| Node(s) | Role |
|---|---|
| **Slack / Gmail / Calendar Enabled?** (IF) | Routing per source. A disabled source skips straight to its Package node, marked as unavailable. |
| **Fetch Slack Messages / Fetch Recent Emails / Fetch Today Events** | Pull data from the last N hours. These nodes continue on error, so a failed fetch does not stop the run. |
| **Prep \* Data** (Code) | Normalize and trim the input. Slack prep also drops bot messages and earlier digests, which prevents a feedback loop. |
| **Slack / Gmail / Calendar Agent** (AI Agent, `gpt-5.4-mini`) | Specialist agents that each turn one source into URGENT / ACTION ITEMS / FYI bullets |
| **Package \* Result** (Code) | Common envelope: `{source, status: ok/unavailable/disabled, summary, note}`. A failed fetch or agent becomes `unavailable` instead of a crash. |
| **Merge Agent Outputs** | Fan-in of the 3 parallel branches |
| **Build Synthesis Input** (Code) | Splits sources into available and unavailable, and attaches the previous digest |
| **Synthesizer Agent** (AI Agent, `gpt-5.4`) | Dedupes across sources and cites them, e.g. (Slack+Gmail). Prioritizes by urgency and VIPs, flags conflicts, tags each bullet [New]/[Recurring], and fills a *Source Status* section. Never invents content for a missing source. |
| **Validate & Save State** (Code) | Output validation: checks for empty output, missing sections and length over 3,500 chars, with a fallback to raw agent outputs. Adds a header and a footer with session, source status and validation issues. Saves the digest as state for the next run. |
| **Send Digest to Slack** | Posts the digest to `#project_updates` |

### Layer 3: Reliability

- **Error workflow** "Multi-Agent Digest - Error Handler": an Error Trigger posts to `#project-october-error` with the **failed node name**, **error message**, time (IST), mode, execution ID and execution link. It is set as the main workflow's `errorWorkflow`.
- **Graceful degradation:**
  - A disabled or misconfigured source is skipped.
  - A failed fetch or agent becomes an `unavailable` status instead of crashing the run.
  - If the synthesizer fails, the raw agent outputs are still delivered.
  - Unavailable sources always appear under *Source Status*.

## Stretch goals implemented

- **Schedule**: daily at 8 AM IST
- **Delta tracking / state**: [New] vs [Recurring], using `$getWorkflowStaticData`. This persists only on scheduled (production) runs, not on manual test runs.
- **Output validation**: required sections, length cap, empty-output fallback
- **Multi-model**: `gpt-5.4-mini` for the specialist agents (cheap, fast) and `gpt-5.4` for synthesis (reasoning-heavy)
- **Guardrails** and a **feedback-loop guard** (bot digests are excluded from the Slack input)

## Production additions (documented, not built)

| Concern | Production approach |
|---|---|
| **Queues** | Run n8n in queue mode (Redis + workers) so source agents run on separate workers. Add retry with backoff for API rate limits. |
| **Isolation** | Move each specialist agent into a sub-workflow (Execute Workflow) with its own credentials, timeouts and error handling. Use separate dev and prod environments. |
| **Memory / RAG** | Store each digest in a vector store (e.g. Pinecone) for semantic dedupe across days. Use RAG over project docs to enrich context. |
| **Observability** | Log the session ID, per-source status, token usage and latency to a data table or dashboard. Alert when a source fails N days in a row. |
| **Security** | Least-privilege OAuth scopes, credential rotation, and PII filtering before the LLM call. |

## Testing evidence

- Live run: Slack 3 messages, Gmail 24 emails, Calendar 0 events. All nodes executed and the digest was posted to `#project_updates`.
- Error path: a simulated failure posted an alert with node name and error to `#project-october-error`.
- Feedback-loop guard: a re-run of Prep Slack Data on real data dropped the bot digest and kept the 2 human messages.

## Setup

1. Import both JSON files into n8n (credentials are not included).
2. Connect credentials: Slack OAuth2, Gmail OAuth2, Google Calendar OAuth2 and OpenAI.
3. Edit the **Orchestrator** node: your name, VIPs, keywords and queries. Pick your calendar and Slack channels.
4. In the main workflow's Settings, set the Error Workflow to "Multi-Agent Digest - Error Handler".
5. Publish both workflows.
