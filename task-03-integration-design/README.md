# Task 03 — Integration Design (Landing Page → HubSpot → WhatsApp → Google Ads)

## Architecture

On submit, the landing page fires a **direct API call to a thin serverless endpoint** (e.g. a single Cloudflare Worker or AWS Lambda function) I control — not a native HubSpot embed, and not Zapier/Make as the primary path. The form posts JSON (`name`, `phone`, `clinic_preference`) to this endpoint over HTTPS.

Why not the native HubSpot embed or HubSpot Forms API directly from the browser: it gives me no control point to run the phone-based dedup logic or to fan out to WhatsApp and Google Ads in one transaction, and ties me to HubSpot's default behavior. Why not Zapier/Make as the *primary* path: they add 30–90s of latency and a third-party point of failure sitting directly in the critical 2-minute WhatsApp SLA — fine for non-time-sensitive automations, wrong for this one.

The endpoint does three things in sequence: (1) calls the **HubSpot Contacts API** directly, searching by phone (not email) to decide create-vs-update, then writes Name, Phone, Clinic Preference, Source = "Google Ads - Consultation Landing Page", Lead Status = "New Enquiry"; (2) calls **Karix's WhatsApp Business API** to send the confirmation template message; (3) returns success to the browser, which then fires the `consultation_form_submitted` dataLayer event so GTM/GA4 forwards the Google Ads conversion. HubSpot is updated synchronously first since it's the source of truth; WhatsApp and the conversion ping can fire in parallel after.

## Biggest Failure Point

HubSpot's **default deduplication keys on email**, not phone — and this form collects no email. Out of the box, two submissions with the same phone but different names would create two separate contacts, fragmenting the patient's record and breaking attribution. The fix: don't rely on HubSpot's native dedup at all. My endpoint explicitly searches Contacts by the `phone` property before writing, and if a match exists, updates that contact (overwriting name only if the new submission is more recent, logging the conflict) rather than creating a new one. If two patients genuinely share a phone (a common real-world case — shared family numbers), I'd append a note/timeline entry rather than silently overwriting, so no enquiry is lost.

## Protecting the 2-Minute WhatsApp SLA

Risks: Karix API latency/downtime, HubSpot write blocking the chain, or template message approval issues. Mitigation: run the HubSpot write and WhatsApp send as independent, retried steps rather than one blocking chain, with a dead-letter queue and Slack alert if WhatsApp fails after 2 retries within 90 seconds, plus a daily dashboard tracking p95 send latency against the 2-minute SLA.
