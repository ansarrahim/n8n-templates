# n8n Automation Templates

Six self-contained n8n workflows built for small/local businesses — each one is a real automation a business would actually pay for, not a demo. All six were built and tested live against real APIs (Gemini, Resend, Airtable, Firecrawl) before being exported here.

## 1. AI Lead Auto-Responder (`workflows/1-ai-lead-autoresponder.json`)

Webhook receives a lead (name, email, message) from a contact form → Gemini drafts a personalized reply → Resend sends it. Instant human-sounding response to every inbound lead, 24/7, no one has to be at a keyboard.

**Tested live:** real leads sent through, real Gemini-drafted replies, real emails delivered.

## 2. Review Sentiment Alert (`workflows/2-review-sentiment-alert.json`)

Webhook receives a new review (customer, source, rating, text) → Gemini classifies sentiment + summarizes it → if negative, immediately emails the business owner so they can respond before it festers publicly.

**Tested live:** both branches confirmed — negative review triggers a real alert email, positive/neutral doesn't.

## 3. New Order Sync (`workflows/3-new-order-sync.json`)

Webhook receives a new order → validates/enriches it → logs it to an Airtable base (CRM/inventory stand-in) → emails the customer a confirmation → if the order is above a configurable VIP threshold, also alerts the business owner.

**Tested live:** validation, VIP branching, Airtable record creation (including the VIP checkbox field), customer confirmation email, and VIP alert email all confirmed working end-to-end against a real Airtable base. The Airtable node is set to fail gracefully (`onError: continueRegularOutput`), so even if a buyer's Airtable credentials are misconfigured, the confirmation/alert emails still go out.

## 4. Missed-Call Text-Back (`workflows/4-missed-call-textback.json`)

Webhook receives a missed-call event (shaped like a real Twilio Voice status callback — `From`, `To`, `CallStatus`) → Gemini drafts a short, warm SMS-length reply → sent back to the caller. Solves the classic "phone rings, nobody answers, lead is gone" problem for contractors, salons, clinics.

**Honest note on this one specifically:** the buyer needs their own SMS provider account (Twilio, Vonage, etc. — roughly $1/mo for a number plus a fraction of a cent per message) to actually send real texts. This workflow was built and tested with the SMS-send step standing in as a Resend email (clearly labeled `[DEMO]` in the subject) instead, since a real trial SMS account wasn't obtainable at build time. Swapping the final HTTP Request node for a real Twilio/Vonage SMS call is a single-node change — the trigger shape, the AI drafting step, and the whole rest of the logic don't change at all.

**Tested live:** a real simulated missed-call payload produced a real, on-brand, 101-character drafted message and a real (demo) delivery.

## 5. Support Triage + FAQ Auto-Answer (`workflows/5-support-triage-faq-answer.json`)

Webhook receives a support message (name, email, message) → Gemini checks it against an inline FAQ knowledge base → if it's covered, drafts and sends a grounded reply straight to the customer; if it's not (billing disputes, complaints, anything ambiguous), skips answering entirely and emails a human a one-paragraph summary instead. The FAQ list lives in a Code node as a plain array — easy to edit inline, or swap for an Airtable/Notion lookup if a buyer wants to self-edit it without touching the workflow.

**Tested live:** both branches confirmed with real requests — a pricing question got answered correctly and grounded in the actual FAQ text; a billing/refund complaint correctly skipped auto-answering and escalated with an accurate summary instead.

## 6. Lead Qualification & Follow-Up (`workflows/6-lead-qualification-followup.json`)

A lead comes in through a webhook → if a company website was given, Firecrawl scrapes it for real switching-vendor/size-confirming signals → the lead is scored against a rules-based config block (service area, building type, sqft, frequency, urgency, decision authority, contact completeness, enrichment signal, referral source — 10 rules total, all editable) → logged to Airtable as one clean CRM record with a plain-English score explanation → routed to one of three outcomes: a human gets alerted (low-confidence, incomplete contact info, or flagged as likely spam/test data — never silently discarded), a real Cal.com booking link is emailed (high-confidence), or Gemini drafts a genuine low-pressure follow-up asking about whatever's missing (mid-confidence). A second webhook (`lead-qualification-booked`) receives Cal.com's real `BOOKING_CREATED` event and marks the CRM record booked — first trying the lead ID passed through the booking link's metadata, falling back to an email lookup if that's missing.

Built for commercial cleaning specifically (service area/sqft/building-type/frequency map cleanly onto real, checkable qualification signals for that vertical), but every threshold lives in the `Score Lead` node's config block at the top — swap the constants for your own business and vertical.

**Tested live:** all three routing branches (human review, auto-book, follow-up) confirmed end-to-end against a real n8n instance — real Firecrawl scrape (verified against a live URL, confirmed real page content came back), real deterministic scoring across multiple test scores (0, 55, 100), real Airtable records created for every branch, real Gemini-drafted follow-up email, real Resend sends, and the booking-confirmation webhook confirmed working via both the lead-id-in-metadata path and the email-lookup fallback path.

**Honest note on this one specifically:** the booking-confirmation webhook's payload parsing is shaped for Cal.com's `BOOKING_CREATED` event based on their documented format, tested here with a simulated payload (a real Cal.com account/booking was used to generate the booking link itself and confirm it's live, but the webhook payload shape should be double-checked against your own Cal.com account's actual webhook delivery before fully relying on it — Cal.com's Settings → Developer → Webhooks page lets you see real delivered payloads).

## Setup (per workflow, ~5 minutes)

1. Import the JSON into n8n (Workflows → Import from File).
2. Create/attach credentials the workflow expects:
   - **Gemini API** — Header Auth type "Query Auth", field name `key`, value = your Gemini API key.
   - **Resend API** — Header Auth type "Header Auth", field name `Authorization`, value = `Bearer <your Resend API key>`.
   - **Airtable API** (workflows 3 and 6) — Header Auth, field name `Authorization`, value = `Bearer <your Airtable PAT>`. The PAT needs `data.records:write` scope and access granted to the specific base(s) you'll log records to.
   - **Firecrawl API** (workflow 6 only) — Header Auth, field name `Authorization`, value = `Bearer <your Firecrawl API key>`.
3. In workflow 3's "Log to Airtable" node, replace the base ID in the URL with your own Airtable base ID, and make sure that base has an `Orders` table with fields: Order ID (single line text), Customer (single line text), Email (email), Items (long text), Total (number), Currency (single line text), VIP (checkbox).
4. In workflow 3's "Validate & Enrich" code node, edit the two config lines at the top (`VIP_THRESHOLD`, `BUSINESS_ALERT_EMAIL`) for the buyer's business.
5. In workflow 6's three Airtable HTTP Request nodes ("Write Lead to CRM", "Find Lead by Email", "Mark Booked in CRM"), replace the base ID in each URL with your own Airtable base ID, and make sure that base has a `Leads` table with fields: Name, Company, Service Area, Building Type, Frequency, Urgency, Decision Role, Source, Route Decision (all single line text), Email (email), Phone (phone number), Website (URL), Score Explanation (long text), Sqft / Score / Response Time Ms / Booked At Ms (number), Booked (checkbox).
6. In workflow 6's "Score Lead" code node, edit the config block at the top (`SERVICE_AREA_KEYWORDS`, `TARGET_BUILDING_TYPES`, `MIN_SQFT_QUALIFIED`, `AUTO_BOOK_THRESHOLD`, `HUMAN_REVIEW_MIN`/`MAX`, `SALES_EMAIL`, `BOOKING_LINK`) for the buyer's business, vertical, and real Cal.com/Calendly link.
7. Update the `from` address in every Resend HTTP node to the buyer's verified sending domain (currently set to Resend's test sender for demo purposes).
8. Activate the workflow — each one exposes a webhook URL to point the buyer's existing form/system/website at.
