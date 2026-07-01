# Task 01 — GTM Event Schema — OrthoNow

## 1. Full Event Schema

| Event Name | Trigger Type (GTM) | Key Parameters | Feeds Into (GA4) |
|---|---|---|---|
| `booking_step_complete` (step 1) | Custom Event — listens for `event: booking_step_complete` pushed by front-end on step 1 success | `step_number: 1`, `step_name: location_specialty_selected`, `clinic_location`, `specialty` | GA4 Funnel Exploration (booking funnel, step 1); Booking Funnel Drop-off report |
| `booking_step_complete` (step 2) | Same Custom Event trigger, filtered in the GA4 tag/variable logic by `step_number == 2` | `step_number: 2`, `step_name: contact_details_entered`, `clinic_location`, `preferred_date` | GA4 Funnel Exploration (step 2); remarketing audience "abandoned at contact details" |
| `booking_step_complete` (step 3 / `booking_confirmed`) | Custom Event on booking confirmation success (fired after backend confirms, not on button click) | `step_number: 3`, `step_name: booking_confirmed`, `clinic_location`, `specialty`, `booking_id` | GA4 conversion event; Google Ads conversion import (see §3); Funnel Exploration step 3 |
| `call_now_click` | Click trigger — Click Classes contains `call-now-btn`, all pages | `page_location`, `page_type` (`homepage` / `clinic_page` / `landing_page`), `clinic_location` (blank on homepage/LP) | GA4 conversion (soft), Engagement report, per-page CTA performance |
| `whatsapp_widget_open` | Click trigger — Click ID/Class `whatsapp-float-btn` | `page_location`, `page_type`, `time_on_page_seconds` | GA4 engagement report; remarketing audience "high-intent, no form fill" |
| `patient_guide_gate_submit` | Custom Event — front-end pushes on successful gated-form submit, before PDF unlocks | `form_location`, `guide_title`, `lead_source` | GA4 conversion (lead magnet); nurture audience for email/WhatsApp remarketing |
| `patient_guide_download` | Custom Event — fired only after gate passes and PDF actually begins downloading (separate from the gate submit, since some users submit but the file may fail to load) | `guide_title`, `form_location` | GA4 secondary engagement metric; QA signal (submit vs. actual download mismatch) |
| `clinic_page_view` | Trigger: standard GA4 Page View, filtered by URL path pattern `/clinics/*` (not a custom event — this is a normal pageview GTM/GA4 already handles) | `clinic_name`, `clinic_city`, `page_location` | GA4 standard Pages report, segmented by clinic; feeds location-based remarketing lists |
| `blog_scroll_depth` | Scroll Depth trigger — vertical, thresholds 25/50/75/90% | `scroll_threshold`, `article_title`, `article_category` | GA4 engagement/content report; audience "engaged blog readers" for retargeting |
| `landing_page_call_click` | Click trigger, same as `call_now_click` but isolated by `page_type == landing_page` for clean campaign attribution | `page_location`, `campaign_id` (from URL param), `clinic_location` | Google Ads conversion candidate (secondary), Landing Page CTA report |

---

## 2. Booking Form Funnel Tracking (3-step)

**The core fact this section is built on:** GTM cannot natively detect step transitions inside a JS-driven multi-step form. There's no DOM event for "user is now on step 2" unless the form does a full page reload or URL change between steps (unlikely, since that would hurt UX). GTM only reacts to things the browser actually exposes — clicks, page loads, native form `submit` events, history changes — so a JS-rendered step change is invisible to it by default.

**Because of that, tracking this funnel requires three actors working together:**

1. **Front-end developer** — pushes a `dataLayer.push()` call at the moment each step's validation succeeds and the UI is about to show the next step (or shows the confirmation, for step 3). This is app code, not GTM config.
2. **GTM** — a Custom Event trigger listens for `event == 'booking_step_complete'`, and a GA4 Event tag fires on that trigger, forwarding the pushed parameters as GA4 event parameters.
3. **GA4** — a Funnel Exploration report is built where each step of the funnel is defined by filtering the same event on `step_number = 1`, then `= 2`, then `= 3`. GA4 computes drop-off between steps automatically once the funnel is defined this way.

### dataLayer push — Step 1 (location + specialty selected)

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Indiranagar - Bengaluru",
  "specialty": "Knee & Joint Care"
}
```

### dataLayer push — Step 2 (contact details entered)

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "Indiranagar - Bengaluru",
  "preferred_date": "2026-07-08"
}
```

### dataLayer push — Step 3 (booking confirmed)

```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Indiranagar - Bengaluru",
  "specialty": "Knee & Joint Care",
  "booking_id": "ORN-20260701-00417"
}
```

**Important implementation detail:** step 3's push should fire after the backend confirms the booking succeeded — not on the button click itself. If it fires on click, a failed booking (e.g., slot taken, API error) still gets counted as a conversion, which corrupts the funnel data and, worse, gets imported into Google Ads as a false conversion signal.

### GTM setup to make this work

- **Trigger:** Custom Event, Event name = `booking_step_complete` (fires for all 3 steps; step is distinguished by the `step_number` parameter, not by separate triggers).
- **Variables:** three Data Layer Variables — `DLV - step_number`, `DLV - step_name`, `DLV - clinic_location` (and step-specific ones for `specialty`, `preferred_date`, `booking_id`).
- **Tag:** one GA4 Event tag, Event Name = `booking_step_complete`, mapping each Data Layer Variable to a matching GA4 event parameter.

### Surfacing drop-off in GA4 Funnel Exploration

1. Create a new Funnel Exploration.
2. Step 1 condition: `Event name = booking_step_complete` AND `step_number = 1`.
3. Step 2 condition: same event, `step_number = 2`.
4. Step 3 condition: same event, `step_number = 3`.
5. Enable "Show elapsed time" to catch steps where users stall before dropping (useful for identifying if step 2's form, specifically, is the friction point — long, then it's a UX issue, not a tracking issue).
6. Break down by `clinic_location` to see if drop-off is concentrated at specific clinics (could indicate slot availability issues, not funnel design issues).

---

## 3. Conversion Action to Import into Google Ads

**Chosen event: `booking_step_complete` where `step_number = 3` (i.e., `booking_confirmed`).**

**Why this one over the others:**

- It's the deepest, most commercially meaningful signal in the funnel — an actual confirmed appointment, not just intent (a call click or WhatsApp open could be a wrong number, a question about a bill, or someone just browsing).
- It has enough volume to let Smart Bidding learn (unlike, say, `patient_guide_download`, which is meaningful but too far upstream and too low-intent to reliably correlate with revenue).
- It's clean — because it only fires after backend confirmation (not on click), Google Ads won't optimize toward a signal contaminated by failed bookings.

`call_now_click` and `whatsapp_widget_open` are useful as **secondary/observation conversions** for reporting, but importing them as the primary optimization target would have Google Ads chasing top-of-funnel clicks rather than actual booked patients — the wrong incentive for a healthcare client trying to fill appointment slots.