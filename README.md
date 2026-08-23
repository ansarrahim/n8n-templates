# n8n Automation Templates

Three self-contained n8n workflows built for small/local businesses — each one is a real automation a business would actually pay for, not a demo. All three were built and tested live against real APIs (Gemini, Resend, Airtable) before being exported here.

## 1. AI Lead Auto-Responder (`workflows/1-ai-lead-autoresponder.json`)

Webhook receives a lead (name, email, message) from a contact form → Gemini drafts a personalized reply → Resend sends it. Instant human-sounding response to every inbound lead, 24/7, no one has to be at a keyboard.

**Tested live:** real leads sent through, real Gemini-drafted replies, real emails delivered.

## 2. Review Sentiment Alert (`workflows/2-review-sentiment-alert.json`)

Webhook receives a new review (customer, source, rating, text) → Gemini classifies sentiment + summarizes it → if negative, immediately emails the business owner so they can respond before it festers publicly.

**Tested live:** both branches confirmed — negative review triggers a real alert email, positive/neutral doesn't.

## 3. New Order Sync (`workflows/3-new-order-sync.json`)

Webhook receives a new order → validates/enriches it → logs it to an Airtable base (CRM/inventory stand-in) → emails the customer a confirmation → if the order is above a configurable VIP threshold, also alerts the business owner.

**Tested live:** validation, VIP branching, Airtable record creation (including the VIP checkbox field), customer confirmation email, and VIP alert email all confirmed working end-to-end against a real Airtable base. The Airtable node is set to fail gracefully (`onError: continueRegularOutput`), so even if a buyer's Airtable credentials are misconfigured, the confirmation/alert emails still go out.

## Setup (per workflow, ~5 minutes)

1. Import the JSON into n8n (Workflows → Import from File).
2. Create/attach credentials the workflow expects:
   - **Gemini API** — Header Auth type "Query Auth", field name `key`, value = your Gemini API key.
   - **Resend API** — Header Auth type "Header Auth", field name `Authorization`, value = `Bearer <your Resend API key>`.
   - **Airtable API** (workflow 3 only) — Header Auth, field name `Authorization`, value = `Bearer <your Airtable PAT>`. The PAT needs `data.records:write` scope and access granted to the specific base you'll log orders to.
3. In workflow 3's "Log to Airtable" node, replace the base ID in the URL with your own Airtable base ID, and make sure that base has an `Orders` table with fields: Order ID (single line text), Customer (single line text), Email (email), Items (long text), Total (number), Currency (single line text), VIP (checkbox).
4. In workflow 3's "Validate & Enrich" code node, edit the two config lines at the top (`VIP_THRESHOLD`, `BUSINESS_ALERT_EMAIL`) for the buyer's business.
5. Update the `from` address in every Resend HTTP node to the buyer's verified sending domain (currently set to Resend's test sender for demo purposes).
6. Activate the workflow — each one exposes a webhook URL to point the buyer's existing form/system/website at.
