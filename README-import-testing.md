### Import and Testing Guide (All-in-one Workflow)

- Ensure env vars are set in n8n (.env or UI):
  - N8N_BASE_URL, DUMMY_API_BASE, DUMMY_DB_URI, RATE_LIMIT_MINUTES

- Import JSONs:
  - Only `workflows.json` (contains Workflows 1–9 + utilities).

- Enable the workflow and confirm the following webhook endpoints exist (under nodes):
  - Review decision: `/webhook/review`
  - Incoming data: `/webhook/0e4309c1-b600-45d5-a2ca-b12968875b25`
  - Broadcaster: `/webhook/w5-broadcast`
  - Meta Logger: `/webhook/w8-meta-log`
  - Draft Rewriter: `/webhook/w9-rewrite`

- Test flow end-to-end (dummy payloads):
  1) Trigger draft creation (Workflow 1) via Incoming Data webhook:
     ```bash
     curl -X POST "$N8N_BASE_URL/webhook/0e4309c1-b600-45d5-a2ca-b12968875b25" \
       -H "Content-Type: application/json" \
       -d '{"Record Id": "12345", "Inquiry": "Lorem ipsum"}'
     ```
  2) Confirm AI Draft row appended (Status: Under review) in placeholder sheet.
  3) Review links (Workflow 2) → use browser or curl to simulate decisions, e.g. escalate:
     ```bash
     curl "$N8N_BASE_URL/webhook/review?record_id=12345&decision=escalate"
     ```
  4) Escalation (Workflow 3) → verify dummy pool append + (abstract) Slack node executed.
  5) Completion (Workflow 4) → when the “Escalated task” sheet row becomes completed, the Completed trigger fires. After 3 minutes, internal notification runs, then `sent` → logs “completed” to Meta Logger and calls Broadcaster.
  6) Broadcaster (Workflow 5) → can be invoked directly for testing:
     ```bash
     curl -X POST "$N8N_BASE_URL/webhook/w5-broadcast" -H "Content-Type: application/json" \
       -d '{"record_id":"12345","broadcast_type":"dummy_email"}'
     ```
     Hidden hook posts to Meta Logger (`broadcast_sent`).
  7) Listener (Workflow 6) → runs every minute. To avoid waiting, manually execute the node chain by running from `Fetch Dummy Inbox` with sample:
     ```json
     {"items":[{"record_id":"12345","response_text":"Got it, thanks.","response_tier":"standard"}]}
     ```
     Confirm holding table append and a `listener_saved` log.
  8) Smart Responder (Workflow 7) → runs daily. Manually execute from `Build Nudge with Token` to send a dummy nudge and write `nudge_sent` log.
  9) Meta Logger (Workflow 8) → receives all transition logs at `/webhook/w8-meta-log`; verify rows in unified_logs with `latency_score`.
  10) Draft Rewriter (Workflow 9) → test restyle endpoint:
      ```bash
      curl -X POST "$N8N_BASE_URL/webhook/w9-rewrite" -H "Content-Type: application/json" \
        -d '{"record_id":"12345","draft":"This is a sample draft response."}'
      ```
      Confirm alt draft row and `priority=true` saved.

- Safeguards and Utilities:
  - Rate limiting: `RATE_LIMIT_MINUTES` prevents duplicate incoming records within the window (see `Rate Limit / Dedupe`).
  - Run tracking: internal per-workflow global store (`Run Tracker`) logs each run start (for debugging).
  - Env vars: all dummy endpoints respect `N8N_BASE_URL` and `DUMMY_API_BASE` so you can point at mocks.

- Screenshots checklist:
  - Draft creation runs (AI Draft row in sheet).
  - Notifications sent (content review/escalation Slack nodes executed).
  - Escalations logged (reFetch/in progress nodes run).
  - Completion cycle works (Completed trigger → internal team → sent → Meta Logger → Broadcaster).
  - Nudges fire after inactivity (Smart Responder log).
  - Responses saved by Listener (holding table rows).
  - Logger shows latency (unified_logs with `latency_score`).
  - Broadcaster and Rewriter produce dummy outputs (w5 and w9 runs).


