# Customer Lead Processing System

An n8n workflow that receives customer leads via webhook, validates and normalizes the data, scores lead priority (HIGH / MEDIUM / LOW), routes each branch accordingly, and stores the result in a database.

- **Workflow name:** Customer Lead Processing System
- **Workflow ID:** `VMKBvj7hpL0jvHXS`
- **Status:** Inactive (pending credential configuration — see below)

---

## 1. Architecture

```
Webhook (POST)
  → Validate & Normalize Lead (Code)
    → Is Valid Lead? (IF)
        ├─ false → Respond - Invalid Lead (HTTP 400)
        └─ true  → Determine Lead Priority (Code)
                     → Route by Priority (Switch)
                         ├─ HIGH   → Prepare HIGH Priority Alert → Send Slack Notification ─┐
                         ├─ MEDIUM → Prepare MEDIUM Follow-up ───────────────────────────────┼─→ Store Lead in Database → Prepare Success Response → Respond - Lead Processed (HTTP 200)
                         └─ LOW    → Prepare LOW Follow-up ─────────────────────────────────┘
```

## 2. Prerequisites

- A running n8n instance (self-hosted or n8n Cloud) with API access enabled.
- A Postgres database (or update the `Store Lead in Database` node to point at whatever database you use).
- A Slack workspace with a bot token authorized to post to your target channel.

## 3. Required credentials (must be configured before the workflow can be activated)

n8n refuses to activate a workflow if any node is missing a required credential. Two nodes currently need one:

| Node | Credential type | What to do |
|---|---|---|
| `Send Slack Notification` | Slack API (`slackApi`) | In n8n: **Credentials → New → Slack API**, paste your bot token, then open the node and select it. Also update the placeholder channel (`sales-leads`) to your real channel name or ID. |
| `Store Lead in Database` | Postgres (`postgres`) | In n8n: **Credentials → New → Postgres**, fill in host/port/db/user/password, then open the node and select it. |

See `.env.example` for the values you'll need to have on hand when creating these credentials (n8n stores credentials in its own encrypted store, not in a `.env` file — the `.env.example` here is just a convenient place to stage the values before you paste them into the n8n UI, and to configure the database itself).

## 4. Database setup

The `Store Lead in Database` node runs a plain `INSERT` against a `leads` table. Create it first:

```sql
CREATE TABLE leads (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT NOT NULL,
  phone TEXT,
  message TEXT,
  source TEXT,
  priority TEXT,
  status TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

> Note: the current query uses inline string interpolation for simplicity as a placeholder. Before production use, switch it to parameterized values (n8n's `queryReplacement` option) to protect against SQL injection.

## 5. Activating the workflow

1. Add both credentials as described above.
2. In n8n, open the workflow and toggle it **Active** (top right), or run:
   - via API: `activate` on workflow ID `VMKBvj7hpL0jvHXS`.
3. Once active, the **production webhook URL** becomes live at:

   ```
   POST https://<your-n8n-host>/webhook/customer-lead-processing
   ```

   While the workflow is inactive, you can still test it by opening the workflow in the n8n editor, clicking **"Listen for test event"** on the Webhook node, and sending requests to the **test URL** instead:

   ```
   POST https://<your-n8n-host>/webhook-test/customer-lead-processing
   ```

   (Test URLs only work for one request after you click "Listen," and only while the editor is open.)

## 6. Sending a test request

```bash
curl -X POST https://<your-n8n-host>/webhook/customer-lead-processing \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Sarah Chen",
    "email": "sarah.chen@example.com",
    "phone": "+1-555-0148",
    "message": "Hi, I am interested in pricing for your enterprise plan — need this today if possible.",
    "source": "website_contact_form"
  }'
```

Expected response (HTTP 200):

```json
{
  "success": true,
  "leadName": "Sarah Chen",
  "priority": "HIGH",
  "processingStatus": "processed"
}
```

Other payloads to try:
- **MEDIUM:** `"message": "I have a question about your product."`
- **LOW:** `"message": "Just browsing, no rush."`
- **Invalid (triggers HTTP 400):** omit `email`, or send `"email": "not-an-email"`.

## 7. Priority scoring logic

- **HIGH** — message contains any of: `urgent`, `buy`, `pricing`, `today`, `asap`, `purchase`, `quote`
- **MEDIUM** — message contains any of: `interested`, `question`, `inquiry`, `demo`, `trial`, `info`, `information`, `details`, `schedule`, `meeting`
- **LOW** — everything else (also the fallback branch in the Switch node)

Edit the keyword lists in the `Determine Lead Priority` Code node to tune this.

## 8. Error handling

- Missing/invalid `name` or `email` → short-circuits to `Respond - Invalid Lead` (HTTP 400) before any downstream processing.
- The Slack and Postgres nodes both have `onError: continueRegularOutput` set, so a missing credential or a temporary outage in either service will not crash the workflow — the lead still gets a success response back, it just won't be notified/stored until those integrations are fixed.

## 9. Files in this package

- `README.md` — this file
- `.env.example` — reference values for setting up the Postgres and Slack credentials in n8n, and for locating your n8n instance
