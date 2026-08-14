#n8n Workflows for Advanced IT Product Management

Two production-ready n8n automation workflows built for Product Owners, Scrum Masters, and Product Managers who want to reduce manual triage work and catch sprint risk before it becomes sprint failure.

## Contents

- Prerequisites
- Quick Start
- Workflow 1: AI VoC → Backlog Synthesizer
- Workflow 2: Sprint Scope-Creep & Spillover Predictor
- Troubleshooting
- Customization Notes
- Security Notes

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Prerequisites

Before importing either workflow, make sure you have:

- **n8n instance** - self-hosted (v1.30+) or n8n Cloud, with permission to create/edit workflows and credentials.
- **Jira Cloud** account with API access (project you want to write PBIs to, and a board for sprint data).
- **Slack workspace** with permission to install/authorize an app (or an existing Slack app with `chat:write` scope).
- **OpenAI API key** (Workflow 1 only) - any account with access to chat completion models.
- **SMTP credentials** (Workflow 2 only, if you want email digests in addition to/instead of Slack) - e.g. Gmail, Outlook, SendGrid, or your company mail relay.

No credentials are embedded in these JSON files. Every node that needs authentication is intentionally left **unconfigured** so you can cleanly attach your own.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------


## Quick Start

1. In n8n, go to **Workflows → Import from File** (or **Import from URL** if hosting this repo publicly) and select the `.json` file.
2. Open every node flagged with a `CONFIGURE:` note (visible as a small note icon on the node, or by opening the node itself) and:
   - Select or create the required credential.
   - Fill in the dropdown/field it asks for (Slack channel, Jira project, etc.).
3. Save the workflow.
4. Toggle the workflow **Active**.
5. Test it (see the "Test It" section under each workflow below) before relying on it in a live sprint or feedback pipeline.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Workflow 1: AI VoC → Backlog Synthesizer

**What it does:** Ingests raw customer feedback, uses an AI model to extract sentiment, urgency, and feature intent, alerts the team in Slack when something is urgent, and automatically drafts a formatted Jira Product Backlog Item for genuine feature requests — so nothing from customers gets lost in a spreadsheet or inbox.

### Flow at a glance

```
Webhook → Normalize Data → AI Analysis (OpenAI) → Parse AI Output
   → [If Urgent] → Slack Alert
   → [If Feature Request] → Create Jira PBI → Slack Confirmation
   → Respond to Sender (JSON ack)
```

### Nodes you must configure

| Node | What to set | Credential needed |
|---|---|---|
| `Feedback Intake Webhook` | Copy the **Production URL** and point your feedback source at it (Typeform, in-app widget, support form, etc.) | None |
| `AI: Extract Sentiment & Urgency` | Select/create your OpenAI credential. Model defaults to `gpt-4o-mini` | OpenAI API |
| `Alert: High-Urgency Feedback` | Select/create your Slack credential; choose the Channel ID for urgent review (e.g. `#product-urgent-review`) | Slack OAuth2 |
| `Create Backlog PBI` | Select/create your Jira credential; choose your **Project** and **Issue Type** (Story/PBI) | Jira Software Cloud |
| `Notify: Backlog Item Created` | Select/create your Slack credential; choose your backlog-grooming channel; **replace `YOUR-DOMAIN`** in the message text with your real Atlassian subdomain so the ticket link resolves | Slack OAuth2 |

### Expected webhook payload

The `Normalize Feedback Data` node expects a JSON body roughly like:

```json
{
  "customerName": "Jane Doe",
  "customerEmail": "jane@customer.com",
  "feedbackText": "We keep losing our filter settings every time we refresh the dashboard.",
  "source": "In-app widget",
  "rating": 2
}
```

If your feedback source sends a different shape (e.g. Typeform's nested `form_response.answers`), edit the expressions in `Normalize Feedback Data` to match — this is the only node you may need to touch beyond credentials.

### Test it

Send a test payload with curl or Postman to the webhook's Production URL:

```bash
curl -X POST https://<your-n8n-domain>/webhook/voc-feedback \
  -H "Content-Type: application/json" \
  -d '{"customerName":"Test User","customerEmail":"test@example.com","feedbackText":"The export button is broken and it is blocking our billing team.","source":"Manual Test","rating":1}'
```

You should see: an AI-tagged item flow through, a Slack alert (if flagged High/Critical), and a new Jira ticket (if flagged as a feature request).

---

## Workflow 2: Sprint Scope-Creep & Spillover Predictor

**What it does:** Runs on a schedule mid-sprint, pulls the active sprint from Jira with full changelog history, flags tickets that were **added after sprint planning** (scope creep) or have sat **"In Progress" past your threshold** (spillover risk), and sends a digest to Slack and/or email so the PO and Scrum Master can intervene before the sprint goal is compromised.

### Flow at a glance

```
Schedule Trigger → Configuration → Get Active Sprint → Extract Sprint Details
   → Get Sprint Issues + Changelog → Analyze Scope Creep & Stagnation
   → [If Risks Found] → Format Digest → Slack Digest + Email Digest
   → [If No Risks] → No-op (stays quiet)
```

### Nodes you must configure

| Node | What to set | Credential needed |
|---|---|---|
| `Mid-Sprint Schedule` | Default cron is `0 9,14 * * 1-5` (9am & 2pm, weekdays). Adjust to your team's cadence | None |
| `Configuration` | Set `jiraDomain` (your Atlassian subdomain, e.g. `acme` for `acme.atlassian.net`), `boardId` (numeric ID from your board's URL), and `inProgressThresholdHours` (default `48`) | None |
| `Get Active Sprint` | Select/create your Jira credential | Jira Software Cloud |
| `Get Sprint Issues + Changelog` | Select/create your Jira credential (same as above) | Jira Software Cloud |
| `Post Sprint Risk Digest` | Select/create your Slack credential; choose the PO/Scrum Master channel | Slack OAuth2 |
| `Email Sprint Risk Digest` | Select/create your SMTP credential; set `fromEmail` and `toEmail` | SMTP |

### How risk is detected

- **Scope creep:** any issue in the active sprint whose `created` date is *after* the sprint's `startDate`.
- **Spillover risk:** any issue currently `In Progress` whose most recent transition into that status (from the Jira changelog) is older than `inProgressThresholdHours`.

If your team uses a status name other than `"In Progress"` (e.g. `"Doing"`), update the string comparisons inside the `Analyze Scope Creep & Stagnation` Code node accordingly.

### Test it

- Temporarily lower `inProgressThresholdHours` to `1` and manually run the workflow (**Execute Workflow** button) against a real active sprint to confirm it flags at least one ticket and the Slack/email digest renders correctly.
- Set the threshold back to your real value afterward.


## Troubleshooting

**"The workflow you are trying to save contains credentials that are not shared with you"**
This happens when n8n tries to auto-link a node to an existing credential of the same name in your instance that you don't have access to. These workflow files ship with no credentials pre-linked, so if you still see this after import:
1. Open every node listed in the "Nodes you must configure" tables above.
2. Confirm the Credential dropdown is empty or set to a credential you personally own.
3. Save the workflow again.

**Jira 401/403 errors**
Your Jira credential likely needs an API token (Jira Cloud) rather than a password. Generate one from your Atlassian account settings and use your email + API token in the Jira credential.

**Slack "channel_not_found"**
Make sure the Slack app has been invited to the target channel (`/invite @YourAppName` in Slack) and that you selected the channel via the node's dropdown rather than typing the name manually.

**AI output fails to parse (Workflow 1)**
The `Parse AI JSON Output` Code node has a built-in fallback that returns a safe default (`Low` urgency, `Neutral` sentiment) if the model doesn't return valid JSON, so the workflow won't crash — but check your OpenAI model/prompt if this happens often.

---

## Customization Notes

- Both workflows are intentionally modular — Code nodes (`Parse AI JSON Output`, `Analyze Scope Creep & Stagnation`) contain the business logic and can be edited without touching the surrounding nodes.
- Workflow 1's AI prompt can be swapped for any other LLM node (e.g. Anthropic, Azure OpenAI) as long as the output JSON schema (`sentiment`, `urgency`, `isFeatureRequest`, etc.) is preserved, since downstream nodes depend on those exact keys.
- Workflow 2's digest can be routed to Microsoft Teams, a different Slack channel per severity, or a Jira comment instead of/alongside email — the `Format Digest Message` node is the single place to adjust formatting.

---

## Security Notes

- No API keys, tokens, or credential IDs are stored in these JSON files — you must attach your own credentials after import.
- The Workflow 1 webhook is public by default (standard n8n behavior). If your feedback source can't authenticate, consider adding a shared-secret header check at the top of the `Normalize Feedback Data` node, or enabling n8n's built-in webhook authentication.
- Review your organization's data handling policies before sending customer feedback text to a third-party AI provider (OpenAI).
