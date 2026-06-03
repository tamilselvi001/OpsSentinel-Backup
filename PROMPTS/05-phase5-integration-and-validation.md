# PHASE 5 — Workflow Integration & Validation

```
Owner:        All four members (coordinated)
Runs after:   Phases 1–4 (infra, MCP servers/memory, ADK agent, frontend all in place)
Enables:      Internal beta sign-off — this phase proves the Exit Criteria.
How to use:   Open Claude Code at the repo root and paste this whole file as your message.
              Assign the sub-sections to their owners; integrate; then run the validation suite.
```

---

This is the **integration and validation** phase. The pieces exist; now you connect the **human-in-
the-loop Slack approval gate end-to-end**, run a **simulated alert storm**, and prove every **Exit
Criterion** for OpsSentinel. The goal is one clean, observable path: *storm → single correlated
incident → retrieved runbook → self-evaluated autonomy → Slack brief → human approve → mocked
remediation → closure written back → full Arize trace → dashboard renders it under auth, with zero
client-side exceptions.*

Work in small, reviewable steps and pause at each checkpoint. Use the exact SHARED CONTRACTS from
`00-README-and-build-conventions.md`.

## A. Complete the Slack human-in-the-loop gate  *(Member 3, with Member 2)*
Upgrade the Phase-2 slack-bot stub into the real gate:
1. Replace `/notify` with a **Slack Block Kit** decision brief: title, root cause, correlated
   evidence, confidence, historical match, proposed fix steps, **risk level**, and the autonomy/
   caution note. Buttons: **Approve** (`action_id = approve_incident`, `value = incident_id`),
   **Reject** (`action_id = reject_incident`), **See full reasoning** (link to the dashboard
   incident page).
2. Implement `POST /slack/interactions` with **Slack signature verification** (`SLACK_SIGNING_SECRET`).
   On **Approve** → publish to `opssentinel-actions` (or call the agent's execute endpoint), set the
   incident `status`, and append to `audit_log`. On **Reject** → `status = rejected` + audit entry.
3. Confirm the agent's Phase-3 executor consumes the approval and runs the deterministic, idempotent
   mocked remediation, then writes the closure summary to Elastic and the outcome to Arize.
Document local Slack testing (signing secret + a tunnel for the interactions URL).

## B. Build the alert-storm validation harness  *(Member 3, with Member 1)*
Using the Phase-2 Alert Simulator, create a repeatable `scripts/run_storm.py` (and `make validate`)
that publishes **≥ 50 duplicate/related signals** sharing a `correlation_key` onto
`opssentinel-alerts`, then waits and asserts the system's response.

## C. Prove deduplication & throughput  *(Member 2, with Member 1)*
Confirm the agent's correlation node folds the entire storm into a **single incident** with **no
dropped events** (verify via the DLQ being empty and event counts reconciling). This demonstrates
the AP-tuned Pub/Sub layer's zero-alert-loss and the system's scalability under load.

## D. Prove the autonomous-evaluation path  *(Member 2, with Member 3)*
Confirm the agent: retrieves a **mathematically relevant runbook** via the Elastic MCP server,
queries its **own baseline accuracy** via the Arize MCP server, sets the correct `autonomy_tier`,
formats a remediation proposal from the synthesized data, and routes it to the designated Slack
channel — where a human **Approve** click triggers the deterministic execution path that updates the
**mocked infrastructure state** and resolves the ticket.

## E. Prove absolute observability  *(Member 2, with Member 3)*
Confirm **every phase** of the stress test is represented as an **OpenInference trace** in **Arize
Phoenix**, clearly delineating tool executions, latency metrics, and token consumption. The
incident's `trace_id` must deep-link correctly from the dashboard.

## F. Prove the secured, exception-free frontend  *(Member 4, with Member 1)*
Confirm the Next.js dashboard loads **behind Google authentication**, renders the resulting incident
state via **server-side data fetching** with **zero unhandled client-side exceptions**, shows the
audit timeline and metrics, and links to the Arize trace.

## G. End-to-end system test + demo script  *(All)*
Author an end-to-end test (and a runnable demo following the project document's 3-minute script):
inject the incident → watch correlation in seconds → Gemini's structured analysis → enriched
incident in Postgres → Slack brief → **Approve** → mocked fix + dashboard goes green → Arize trace
timeline → Elastic knowledge base updated with the new closure document → state the before/after
time comparison.

## Exit Criteria — all must pass
- A synthetic storm of **≥ 50 duplicate signals** is ingested via Pub/Sub, **deduplicated**, and
  synthesized into a **single incident** with **no dropped event**.
- The agent retrieves a relevant historical runbook (Elastic MCP), queries its own metrics (Arize
  MCP), and formats a correct remediation proposal, routed to **Slack**; a human **Approve** click
  triggers a deterministic execution path that **updates the mocked infra state and resolves the
  ticket**.
- **Every** step appears as an **OpenInference trace** in **Arize Phoenix** (tool calls, latency,
  tokens).
- The **Next.js** frontend loads behind **Google auth** and renders incident state via **SSR** with
  **zero unhandled client-side exceptions**.

## Validation checklist to confirm against the success metrics
- Correlated analysis completes within **~30 s (MTTD)**.
- The storm yields a large reduction in duplicate P1/P2 incidents (target **85%**).
- Policy gates held (no high-risk/schema action auto-executed).
- Autonomy tier matched the Arize accuracy/calibration inputs.
- The closure summary is **immediately retrievable** by `search_runbooks` for the next incident.

## Deliverables
- Real Slack approval gate (Block Kit + verified interactions webhook).
- `scripts/run_storm.py` + `make validate` and an end-to-end test that asserts the Exit Criteria.
- A short `docs/DEMO.md` runbook for the 3-minute walkthrough.
- A green end-to-end run captured (logs / screenshots / trace links).

## Guardrails — keep scope frozen
Do not add deferred features (multi-agent collaboration, digital-twin simulation, predictive
prevention, cost-aware remediation, cross-org federated learning, executive intelligence). The
remediation execution remains **mocked** — no real cluster mutations.
