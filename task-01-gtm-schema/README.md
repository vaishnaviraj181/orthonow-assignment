# Task 01 — GTM Event Schema (OrthoNow)

## 1. Full Event Schema

| Event Name | Trigger Type | Key Parameters | Feeds Into (GA4) |
|---|---|---|---|
| `booking_step_complete` | Custom Event (dataLayer push, fired by front-end on each step completion) | `step_number`, `step_name`, `clinic_location`, `specialty` (step 1 only) | Funnel Exploration report, "Booking Funnel" audience, drop-off analysis |
| `booking_confirmed` | Custom Event (dataLayer push on step 3 success) | `clinic_location`, `specialty`, `preferred_date`, `booking_id` | Conversions report, "Confirmed Bookings" audience, Google Ads import |
| `call_now_click` | Click — Just Links / Click Trigger filtered on `.call-now-btn` (or `tel:` href) | `page_location`, `clinic_location` (if on a location page), `cta_position` (header/footer/page-body) | Engagement → "Phone Intent" audience |
| `whatsapp_chat_open` | Click Trigger filtered on the floating WhatsApp widget element (`#whatsapp-widget`) | `page_location`, `clinic_location`, `device_category` | Engagement → "WhatsApp Intent" audience |
| `patient_guide_download` | Form Submission Trigger on the gated PDF-download form | `form_id`, `lead_phone_provided` (boolean, not the raw number), `page_location` | Conversions report, lead-magnet audience |
| `clinic_page_view` | History Change / Page View Trigger filtered on URL path `\/clinics\/*` | `clinic_name`, `clinic_city`, `page_location` | Engagement → location-level traffic in Explorations, used to weight local campaigns |
| `blog_scroll_depth` | Scroll Depth Trigger at 25/50/75/90% thresholds, filtered to `\/blog\/*` | `percent_scrolled`, `article_title`, `time_on_page` | Engagement report, content-quality audience for retargeting |

Notes:
- `call_now_click` and `whatsapp_chat_open` use **native GTM triggers** (Click/Scroll) — no developer work needed beyond adding a stable class/ID to the elements.
- `booking_step_complete`, `booking_confirmed`, and `patient_guide_download` parameters that look sensitive (phone, name) are **never** pushed into the dataLayer as raw PII — only booleans/flags or hashed values, to stay clean for GA4/Google Ads server-side use later.

## 2. Booking Funnel — Step-Level Drop-off Tracking

**Important constraint:** GTM cannot natively detect "step 2 of a multi-step form" the way it can detect a click or a page view, because steps 2 and 3 typically render without a URL change or a full page load (it's a JS-driven state change inside one page). GTM has no visibility into front-end application state on its own.

This means **step completion must be pushed to the dataLayer by the front-end developer**, at the exact moment each step's "Continue" action succeeds client-side. GTM's job is then just to listen for that custom event and fire tags off it — GTM is a listener, not a detector, for this kind of interaction.

**What GTM is configured to do:**
- A single **Custom Event trigger** matching `event equals booking_step_complete` fires a GA4 event tag.
- The GA4 event tag maps `step_number`, `step_name`, `clinic_location`, `specialty` from the dataLayer into GA4 event parameters (registered as custom dimensions in GA4 Admin so they're queryable later).

**dataLayer push the front-end dev implements, per step:**

Step 1 — location + specialty selected:
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Indiranagar - Bengaluru",
  "specialty": "Knee & Joint Care"
}
```

Step 2 — contact details entered:
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "Indiranagar - Bengaluru",
  "preferred_date": "2026-07-04"
}
```

Step 3 — booking confirmed (separate, dedicated conversion event, not just step 3 of the funnel):
```json
{
  "event": "booking_confirmed",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Indiranagar - Bengaluru",
  "booking_id": "ON-2026-04821"
}
```

**Briefing the front-end dev (what I'd actually say to them):**
"GTM can't see your form's internal state — it only sees the DOM and the dataLayer. Every time a user successfully completes a step (i.e. validation passes and you move them to the next step's UI), fire `window.dataLayer.push({...})` with the exact shape above, before or as you render the next step. Step 1 and 2 are `booking_step_complete` with an incrementing `step_number`; the final confirmation is a distinct `booking_confirmed` event so we can set it as the hard conversion separately from the funnel steps. Don't fire it on render of step 1 (that's a step *view*, not a *completion* — we may add `booking_step_view` later for entry-vs-completion comparison, but that's a separate event)."

**Surfacing drop-off in GA4 Funnel Exploration:**
1. New Exploration → **Funnel Exploration**.
2. Steps configured as: Step 1 = `booking_step_complete` where `step_number = 1`; Step 2 = `booking_step_complete` where `step_number = 2`; Step 3 = `booking_confirmed`.
3. Set "Open funnel" off (closed funnel) so users must hit steps in order — gives a true step-to-step conversion rate, not just users who touched each event in any order.
4. Breakdown dimension: `clinic_location`, so we can see if drop-off is concentrated at specific clinics (e.g. one clinic's date picker is broken) rather than a global UX issue.
5. The visualization directly shows % drop-off between Step 1→2 and 2→3 — that's the number the performance team uses to decide whether the form itself needs UX work before scaling ad spend.

## 3. Conversion Action to Import into Google Ads

**Import `booking_confirmed`**, not `booking_step_complete`, `call_now_click`, or `patient_guide_download`.

**Why:** Google Ads' bidding algorithms (Smart Bidding / tCPA / tROAS) optimize toward whatever conversion action they're given, and they'll do it efficiently regardless of whether that action is actually valuable to the business. `call_now_click` and `patient_guide_download` are *intent* signals, not commercial outcomes — optimizing spend toward "got a PDF download" would fill the funnel with people who never book. `booking_confirmed` is the one event that maps directly to "OrthoNow got an actual appointment," which is the real unit of value for a 9-clinic healthcare business. It's also the cleanest, least noisy event (one per completed booking, with a `booking_id`), so Google Ads' attribution and value-based bidding has a reliable, deduplicatable signal instead of a fuzzy proxy.

A secondary recommendation: once volume is sufficient, set `call_now_click` as a **secondary, non-bid-affecting conversion** to track total demand generated, but keep the algorithm's bidding strategy anchored to `booking_confirmed` only.
