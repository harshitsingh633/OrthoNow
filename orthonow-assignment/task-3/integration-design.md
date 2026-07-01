# Task 03 — Integration Design: Landing Page → HubSpot → WhatsApp → Google Ads

**Architecture:** On submit, client-side JS posts form data to a small serverless function
(Cloudflare Worker), not HubSpot's native embed or Zapier/Make. I chose a direct API call
because of the 2-minute WhatsApp SLA — native embeds can't branch into a WhatsApp send at
all, and Zapier/Make typically add 30–90 seconds of trigger latency before a workflow even
starts, eating too much of that budget. A direct call keeps logic in code I can log, retry,
and monitor precisely.

The function does three things: (1) searches HubSpot Contacts by phone
(`POST /crm/v3/objects/contacts/search`) before deciding to `PATCH` an existing record or
`POST` a new one. This is required because **HubSpot's default dedup key is email, not
phone**, and this form never collects email — skip the search and HubSpot creates a
duplicate contact instead of merging into the patient's existing history. (2) In parallel,
it calls Karix's WhatsApp API with a pre-approved template (required for business-initiated
messages). (3) It fires the Google Ads Enhanced Conversion, passing `booking_id` so it lines
up with the `booking_confirmed` event from the GTM schema.

**Duplicate phone numbers:** if two different people submit the same number (a shared
family phone), the phone search finds the existing contact and the submission becomes an
update — but the incoming name is compared against the stored one first. If they differ, I
don't overwrite silently; the record is updated but flagged `possible_duplicate_contact =
true` and routed to manual review, since guessing wrong means a coordinator calls the wrong
person.

**Biggest failure point:** the WhatsApp send via Karix — an external provider dependency,
plus Meta's template approval can silently block sends if it lapses. **Fallback:** every
submission is written to a queue with status `pending` *before* any of the three calls run.
A failed WhatsApp send retries up to 3 times over 90 seconds; if it still fails, a Slack
alert notifies clinic ops to call the patient manually, so the failure degrades to a phone
call rather than silence.

**Monitoring the SLA:** the function timestamps submission and each downstream call. A
scheduled check flags any record where `whatsapp_sent_at - submitted_at > 120s`. Likely
breach causes: Karix rate limits during traffic spikes, template approval lapses, or
HubSpot latency blocking the flow if built sequentially — which is why WhatsApp isn't made
dependent on HubSpot finishing first.

*(Word count: ~365)*