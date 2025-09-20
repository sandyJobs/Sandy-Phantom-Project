### Import and Testing Guide (Workflows 1–10)

- Ensure env vars are set in n8n (.env or UI):
  - N8N_BASE_URL, RATE_LIMIT_MINUTES, VA_API_BASE_URL, VA_API_KEY, SLACK_ESCALATIONS_CHANNEL, SLACK_CONTENT_REVIEW_CHANNEL, SLACK_INTERNAL_TEAM_CHANNEL, EXTERNAL_PARTY_WEBHOOK_URL, ACK_TIMEOUT_HOURS, OPERATOR_EMAIL_API_URL, OPERATOR_SLACK_FALLBACK_CHANNEL, DUMMY_API_BASE
  - Note: Safe testing defaults are wired to https://httpbin.org/anything for external calls if you do not set these. For real logging back into this same n8n, set N8N_BASE_URL to your public n8n URL.

- Import JSONs:
  - Only `workflows.json` (contains Workflows 1–10 + utilities).

- Enable the workflow and confirm the following webhook endpoints exist (under nodes):
  - Review decision: `/webhook/review`
  - Incoming data: `/webhook/incoming`
  - Broadcaster: `/webhook/w5-broadcast`
  - Meta Logger: `/webhook/w8-meta-log`
  - Draft Rewriter: `/webhook/w9-rewrite`
  - Operator Assigned: `/webhook/operator-assigned`
  - Pathway ACK: `/webhook/w10-ack` (also supports test mode: `?test=1&record_id=<id>`)

- Test flow end-to-end (dummy payloads):
  1) Trigger draft creation (Workflow 1) via Incoming Data webhook:
     ```bash
     curl -X POST "$N8N_BASE_URL/webhook/incoming" \
       -H "Content-Type: application/json" \
       -d '{"record_id": "12345", "inquiry": "Lorem ipsum"}'
     ```
  2) Confirm task created in VA Dashboard (/tasks) with status under_review.
  3) Review links (Workflow 2) → use browser or curl to simulate decisions, e.g. escalate:
     ```bash
     curl "$N8N_BASE_URL/webhook/review?record_id=12345&decision=escalate"
     ```
  4) Escalation (Workflow 3) → verify dummy pool append + (abstract) Slack node executed.
  5) Completion (Workflow 4) → when the task is marked completed in the VA Dashboard (PATCH /tasks/:id), the cron checks every minute, notifies the internal team, logs task_completed, then calls the Broadcaster.
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

  10b) Pathway + ACK (Workflow 10) → after intake, a pathway is generated and dispatched to `EXTERNAL_PARTY_WEBHOOK_URL`. To simulate ACK without a token during testing:
     ```bash
     curl "$N8N_BASE_URL/webhook/w10-ack?test=1&record_id=12345"
     ```
     If no ACK is received within `ACK_TIMEOUT_HOURS`, an escalation alert is sent to Slack and logged.

- Safeguards and Utilities:
  - Rate limiting: `RATE_LIMIT_MINUTES` prevents duplicate incoming records within the window (see `Rate Limit / Dedupe`).
  - Run tracking: internal per-workflow global store (`Run Tracker`) logs each run start (for debugging).
  - Env vars: dummy endpoints use `DUMMY_API_BASE` (default https://httpbin.org/anything). Set `N8N_BASE_URL` to your public n8n if you want meta logs to loop back into this workflow.


## Screenshots

Screenshots for all workflows are stored under `Screenshots/*`. 


## Sandy Phantom Project – VA Dashboard Integration

This n8n workflow set replaces Google Sheets with the Staff/VA Dashboard API as the system of record.

### Environment Variables
Add these in n8n (Settings → Variables) or a local .env for reference:

- N8N_BASE_URL: Public base URL of your n8n instance (set this for meta logs to come back to n8n). If not set, meta logs go to https://httpbin.org/anything for testing.
- VA_API_BASE_URL: Base URL of VA Dashboard backend API (if not set, falls back to https://httpbin.org/anything/api for testing)
- VA_API_KEY: Bearer token for VA API
- RATE_LIMIT_MINUTES: Duplicate-drop window (default 5)
- SLACK_ESCALATIONS_CHANNEL: Slack channel ID for escalations
- SLACK_CONTENT_REVIEW_CHANNEL: Slack channel ID for content review
- SLACK_INTERNAL_TEAM_CHANNEL: Slack channel ID for internal team notifications
- EXTERNAL_PARTY_WEBHOOK_URL: Where Workflow 10 dispatches pathway payloads (defaults to https://httpbin.org/anything/external)
- ACK_TIMEOUT_HOURS: Hours to wait for an ACK before escalation (default 2)
- OPERATOR_EMAIL_API_URL: Email endpoint for operator assignment (defaults using DUMMY_API_BASE)
- OPERATOR_SLACK_FALLBACK_CHANNEL: Slack channel ID fallback for operator notification
- DUMMY_API_BASE: Base for all dummy external calls (default https://httpbin.org/anything)

Example: see .env.example in this folder.

### New Operator + Pathway + Escalation Flows

1) Operator Notification Flow
- Trigger: `POST /webhook/operator-assigned` with `{ record_id, operator: { email?, slack_user?, name? }, notify_channel: 'slack'|'email' }`
- Normalizer chooses Slack vs Email and sends message; logs `operator_notified` to Meta Logger

2) Draft Pathway Builder
- After intake task is created, a prompt is generated to produce `{ pathway: { timeline: [{ step, eta_days }], parameters: { priority, owner } } }`
- The shaped payload includes a one-time `ack_url` pointing at `/webhook/w10-ack?token=...`
- The payload is dispatched to `EXTERNAL_PARTY_WEBHOOK_URL`; then the flow waits `ACK_TIMEOUT_HOURS` and escalates if missing

Test helpers:
- You can simulate an ACK without a token using: `GET /webhook/w10-ack?test=1&record_id=<id>` (for Postman screenshots).

3) Escalation Logic
- Missing ACK after timeout → Slack alert to `SLACK_INTERNAL_TEAM_CHANNEL` and `ack_missing_escalated` audit log
- Invalid ACK token → Slack alert and `invalid_ack_escalated` audit log

### Webhook Entry Points
- /webhook/incoming → Intake (record_id, inquiry)
- /webhook/review → Review actions (approve|edit|reject|escalate)
- /webhook/operator-assigned → Operator assignment notifications (Slack or Email)
- /webhook/w5-broadcast → Broadcast events logging/forwarding
- /webhook/w8-meta-log → Unified audit/meta logging
- /webhook/w9-rewrite → Draft restyling
- /webhook/w10-ack → Accepts ACK token; supports `?test=1&record_id=<id>` for testing

All webhooks write to the VA Dashboard via HTTP nodes using Authorization: Bearer VA_API_KEY.

### VA API Endpoints Used
- POST ${VA_API_BASE_URL}/tasks → Create task/intake and escalations
- PATCH ${VA_API_BASE_URL}/tasks/:id → Update task status
- GET ${VA_API_BASE_URL}/tasks → Retrieve tasks (used for under_review and escalations fetch)
- POST ${VA_API_BASE_URL}/notifications → Record listener responses and nudges
- POST ${VA_API_BASE_URL}/audit → Unified audit log entries (meta logger, broadcast logs, etc.)
- POST ${VA_API_BASE_URL}/drafts → Save AI restyled alternate drafts (priority supported)

### Workflow Mapping (n8n → VA API)
1) Intake & Draft Generation
- Trigger: POST /webhook/incoming with { record_id, inquiry }
- Creates task: POST /tasks { record_id, inquiry, ai_draft?, status: under_review }
- Review notification: Slack (content-review channel) with action links

2) Review & Decision
- Trigger: GET /webhook/review?record_id=...&decision=approve|edit|reject|escalate
- Updates: PATCH /tasks/:id { status }
- Logs decision: POST /webhook/w8-meta-log → POST /api/audit

3) Escalation Routing
- On decision=escalate, creates/reroutes: POST /tasks { record_id, status: escalated }
- Notifies escalation channel on Slack

4) Completion Cycle
- Mark task completed in VA Dashboard (or via PATCH /tasks/:id)
- Workflow posts audit and optionally calls broadcaster

5) Broadcaster
- Trigger: POST /webhook/w5-broadcast { record_id, broadcast_type }
- Logs broadcast to /api/audit

6) Listener (Responses)
- Poll-based dummy inbox is supported for testing; normalized responses are saved to /api/notifications
- Hidden hook logs response_tier to /api/audit via /webhook/w8-meta-log

7) Smart Responder
- Daily cron can generate a nudge payload; send via dummy endpoint (testing) and log token_supported via /webhook/w8-meta-log → /api/audit

8) Meta Logger
- /webhook/w8-meta-log computes latency_score and appends a unified audit entry at /api/audit with action naming (task_created, task_reviewed, task_escalated, task_completed, broadcast_sent, listener_saved, nudge_sent, draft_alt_saved)

9) Draft Rewriter
- /webhook/w9-rewrite builds a restyle prompt, calls AI, shapes priority draft, and saves via POST /api/drafts { task_id, content, priority }

### Safeguards
- Rate limiting: duplicates within RATE_LIMIT_MINUTES are dropped by a dedupe code node
- Error workflow: configured; Slack alert and error log are emitted on failures
- Payload consistency: standardized on record_id, inquiry, ai_draft (lowercase) across nodes
- Payload consistency: standardized on record_id (lowercase) across nodes
- Hidden hooks to audit: broadcast_type, response_tier, token_supported, latency_score, priority flag

### End-to-End Testing (curl)
Export or set:
- export N8N_BASE_URL="https://your-n8n.example.com"
  - If you skip setting this in n8n, meta logs will go to https://httpbin.org/anything (200 OK but won’t appear in this workflow’s meta log).

1) Draft Intake → /tasks
```bash
curl -X POST "$N8N_BASE_URL/webhook/incoming" \
  -H "Content-Type: application/json" \
  -d '{"record_id":"12345","inquiry":"Need help automating financial reports"}'
```
Verify in VA Dashboard (GET /api/tasks) that task 12345 exists with status under_review.

2) Review Decision → /review (approve)
```bash
curl "$N8N_BASE_URL/webhook/review?record_id=12345&decision=approve"
```
Verify task status updated to approved in /api/tasks.

3) Escalation Flow → /review (escalate)
```bash
curl "$N8N_BASE_URL/webhook/review?record_id=12345&decision=escalate"
```
Verify escalation visible in /api/tasks and Slack escalation notification.

4) Completion Cycle
Mark the task completed in the Staff Dashboard UI or via VA API directly:
```bash
curl -X PATCH "${VA_API_BASE_URL}/tasks/12345" \
  -H "Authorization: Bearer $VA_API_KEY" -H "Content-Type: application/json" \
  -d '{"status":"completed"}'
```
Confirm entry appears in /api/audit (via Meta Logger calls).

5) Broadcaster
```bash
curl -X POST "$N8N_BASE_URL/webhook/w5-broadcast" \
  -H "Content-Type: application/json" \
  -d '{"record_id":"12345","broadcast_type":"dummy_email"}'
```
Verify broadcast logged in /api/audit.

6) Listener (Response)
If using the dummy inbox, run the "Poll Every Minute" workflow or execute it manually in n8n to fetch a response; it will store to /api/notifications. Alternatively, POST directly to VA API to simulate a response:
```bash
curl -X POST "${VA_API_BASE_URL}/notifications" \
  -H "Authorization: Bearer $VA_API_KEY" -H "Content-Type: application/json" \
  -d '{"task_id":"12345","type":"response","message":"Got it, thanks.","response_tier":"standard","status":"received"}'
```

7) Smart Responder (Cron)
Manually run the daily cron in n8n to emit a nudge; confirm it appears in /api/notifications and audit includes token_supported.

8) Meta Logger
```bash
curl -X POST "$N8N_BASE_URL/webhook/w8-meta-log" \
  -H "Content-Type: application/json" \
  -d '{"record_id":"12345","action":"task_created","timestamp":"2025-09-13T12:00:00Z","latency_score":2}'
```
Verify entry in /api/audit.

9) Draft Rewriter
```bash
curl -X POST "$N8N_BASE_URL/webhook/w9-rewrite" \
  -H "Content-Type: application/json" \
  -d '{"record_id":"12345","draft":"This is a sample draft response."}'
```
Verify alt draft saved in /api/drafts with priority=true.

10) Pathway + ACK
```bash
# Dispatch happens automatically after intake; to simulate an ACK without a token:
curl "$N8N_BASE_URL/webhook/w10-ack?test=1&record_id=12345"
```
If no ACK arrives within ACK_TIMEOUT_HOURS, a Slack alert is sent and an `ack_missing_escalated` log is written.

11) Operator Assignment (Utility)
```bash
curl -X POST "$N8N_BASE_URL/webhook/operator-assigned" \
  -H "Content-Type: application/json" \
  -d '{
    "record_id": "12345",
    "operator": { "email": "agent@example.com", "slack_user": "U00000000", "name": "Agent Smith" },
    "notify_channel": "slack"
  }'
```
Expect a Slack message (or email if `notify_channel: "email"`) and an `operator_notified` meta log.

### Notes
- All outgoing VA API HTTP requests include Authorization: Bearer VA_API_KEY and use JSON bodies.
- Dates are ISO 8601 UTC strings.
- Use the VA Dashboard UI or API to verify /tasks, /notifications, /audit, and /drafts results.


## Postman Collection

- Included `postman_collection.json` has ready-to-run requests:
  - Operator assignment, Intake → Pathway → Dispatch, Review decision links, ACK valid/invalid, Missing-ACK escalation meta.
- Set variables:
  - `N8N_BASE_URL` to your n8n base (e.g., https://your-n8n.example.com)
  - `ACK_TOKEN` only if you want to demo a real token; otherwise use the `?test=1&record_id=` helper

## VA Dashboard Integration Task Sheet

### Env Vars
```bash
N8N_BASE_URL=https://dummy-n8n.example.com
VA_API_BASE_URL=https://dummy-va-api.example.com/api
VA_API_KEY=xxxx
RATE_LIMIT_MINUTES=5
SLACK_ESCALATIONS_CHANNEL=xxx
SLACK_CONTENT_REVIEW_CHANNEL=xxx
SLACK_INTERNAL_TEAM_CHANNEL=xxx
```

### Webhooks
- /webhook/incoming → Task intake + AI draft
- /webhook/review → Decision: approve | edit | reject | escalate
- /webhook/operator-assigned → Operator notification (Slack or Email)
- /webhook/w5-broadcast → Broadcast logs
- /webhook/w8-meta-log → Unified audit logging
- /webhook/w9-rewrite → AI draft restyle
- /webhook/w10-ack → ACK token receiver (supports `?test=1&record_id=<id>`)

All webhook nodes call VA Dashboard API with Authorization: Bearer VA_API_KEY.

### VA API Mapping
- POST /tasks → Intake {record_id, inquiry, ai_draft?, status}
- PATCH /tasks/:id → Status updates (approved, escalated, completed)
- GET /tasks → Fetch under_review / escalated
- POST /notifications → Listener responses & nudges
- POST /audit → Audit logs (task_created, reviewed, escalated, completed, broadcast_sent, listener_saved, nudge_sent, draft_alt_saved)
- POST /drafts → Save AI restyled drafts {task_id, content, priority}

### Workflow Coverage
- Intake → Draft → POST /tasks (status under_review)
- Review → Decision → PATCH /tasks/:id + POST /audit
- Escalation → PATCH /tasks/:id (status escalated) + Slack escalation alert
- Completion Cycle → PATCH /tasks/:id (completed) → POST /audit + Broadcast
- Broadcaster → Logs to /audit
- Listener → Saves responses to /notifications (+ audit response_tier)
- Smart Responder → Daily nudge → /notifications (+ audit token_supported)
- Meta Logger → Central audit (latency_score, transitions)
- Draft Rewriter → AI restyle → POST /drafts (priority flag)
10) Pathway + ACK
- Build pathway JSON via AI after intake, dispatch to EXTERNAL_PARTY_WEBHOOK_URL, wait for ACK via /webhook/w10-ack, escalate on timeout or invalid token; logs via /webhook/w8-meta-log → /api/audit

### Safeguards
- Rate-limit duplicates within RATE_LIMIT_MINUTES
- Error workflow → confirm Slack/email alerts
- Standardize payload keys → always lowercase: record_id, inquiry, ai_draft
- Hidden hooks audited: broadcast_type, response_tier, token_supported, latency_score, priority

### Confirmations (Before merge)
- Error Handling: Ensure the error workflow actually routes to Slack/email (not just logs silently). If not, wire a Slack node or a simple email node.
- Auth & Security: VA_API_KEY must be stored securely (n8n credentials, not plaintext). Optional: enable basic rate-limit middleware on webhooks to prevent spam.
- Testing Proof: Provide screenshots of all 10 workflows firing end-to-end against the VA Dashboard (not Google Sheets), plus the operator assignment utility. Store them under `Screenshots/`.
