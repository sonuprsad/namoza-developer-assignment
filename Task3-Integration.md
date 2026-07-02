# Task 03 — Integration Design (Landing Page → HubSpot → WhatsApp → Google Ads)

## Architecture

I'd skip the native HubSpot embedded form entirely and use a **direct server-side call to the
HubSpot Forms API / Contacts API**, triggered from a small backend endpoint the landing page
posts to — not Zapier or Make.

On submit, the front-end JS does two things in parallel: pushes the GTM dataLayer event (fires
the Google Ads conversion, client-side, immediately) and POSTs the form data to our own
`/api/leads` endpoint. That endpoint — a small serverless function (Vercel/Cloud Function is fine,
doesn't need to be a full backend) — is where the real orchestration happens, in this order:

1. **Upsert the HubSpot contact** via the Contacts API (`POST /crm/v3/objects/contacts`, with
   `idProperty` — more on this below), setting Name, Phone, Clinic Preference, Source = "Google
   Ads - Consultation Landing Page", Lead Status = "New Enquiry."
2. **Trigger the WhatsApp confirmation** via Karix's API, server-side, right after the HubSpot
   write succeeds (or in parallel with it — see SLA section below).
3. Return a success response to the front-end so it can show the thank-you state.

Why direct API over Zapier/Make: this flow has three steps that all need to succeed or fail
together with retry logic on failure, and it's not a "wire two SaaS tools together and forget
it" flow — it needs a fallback (below), and Zapier's native error handling is thin unless you're
on their pricier tiers with custom retry logic, at which point you're basically writing code
inside Zapier's UI anyway, with worse debugging than just writing the code. A native HubSpot form
embed is out for a different reason: it POSTs straight to HubSpot and gives us no hook to also
fire the WhatsApp message or control exactly when the Ads conversion fires relative to a
confirmed CRM write. Direct API calls from our own endpoint is more work upfront but it's the
only option that gives us control over sequencing, retries, and logging all three systems in one
place.

## The dedup trap

HubSpot's default contact deduplication key is **email**, not phone. OrthoNow's form doesn't
collect email — deliberately, it's a 2-field form by design, and that's correct for conversion
rate. That means out of the box, HubSpot will happily create a brand-new contact for every
submission, even from the same phone number, because as far as HubSpot's default dedup logic is
concerned, two contacts with no email and different names aren't duplicates.

If two patients submit with the same phone number but different names in this setup as
configured out-of-the-box: **two separate contact records get created.** That's a real problem in
Indian healthcare lead gen specifically — shared family phones are common (one number, multiple
patients), and typos/nicknames on the name field are common too.

Fix: create a **custom unique property** on the Contact object (`phone_number_normalized`, phone
digits only, no country code, no formatting) and set it as an **alternate ID property** for the
upsert call, using `idProperty=phone_number_normalized` on the API request instead of relying on
HubSpot's default email-based dedup. Normalize the phone number server-side before the API call
(strip spaces/dashes/leading zeros, standardize to 10 digits) so `9876543210` and `+91
98765-43210` are treated as the same value. If a second submission comes in on the same number
with a different name, the safest default is: **update the phone/clinic/source fields, but don't
silently overwrite the name** — instead append the new name as a note/timeline entry on the
contact ("Alt. name provided: Priya Sharma, second enquiry") so the front-desk team sees both
names when they call, rather than losing one. That's a product decision as much as a technical
one, and I'd flag it to the client rather than assume — but I would not ship a version that just
overwrites silently, because a wrong name on a phone-only contact record actively causes trouble
for reception.

## Biggest failure point, and the fallback

The single biggest failure point is the **HubSpot API call itself failing or timing out** — rate
limits, a transient 5xx, network blip between our endpoint and HubSpot. If that write fails and
we don't handle it, the patient sees a thank-you message (or worse, an error) and their enquiry
just evaporates. For a healthcare lead that's real money and, worse, a patient who thinks they've
booked and didn't.

Fallback: the `/api/leads` endpoint writes to a lightweight queue/log **first** (a simple database
row or a queue like SQS — "lead received" with a status field) before attempting the HubSpot
call. If the HubSpot call fails, the row stays in a `pending` state and a retry job (exponential
backoff, 3–4 attempts over ~10 minutes) picks it back up. If it still fails after retries, it
escalates — a Slack/email alert to the ops team with the raw lead data, so a human can manually
enter it into HubSpot rather than the lead being silently lost. The front-end never needs to know
any of this happened; the patient still sees "request received" as long as our own endpoint
accepted the submission, which decouples the user experience from HubSpot's uptime.

## The 2-minute WhatsApp SLA

What could break it: Karix API downtime or rate limiting, WhatsApp template message approval
issues (Business API messages outside a 24-hour session window must use a pre-approved template —
if the template gets rejected or paused by Meta for policy reasons, sends silently fail), or our
own endpoint being slow because it's waiting on the HubSpot call to finish before even attempting
WhatsApp.

That last one is avoidable by design: **don't chain WhatsApp after HubSpot, fire them in
parallel** (`Promise.all` or equivalent) rather than sequentially. The WhatsApp confirmation
doesn't functionally depend on the CRM write succeeding — the patient needs their confirmation
regardless of whether the CRM write is still retrying in the background. Sequencing them one
after another needlessly eats into the 2-minute budget.

Monitoring: track WhatsApp send status (Karix returns delivery status callbacks — sent/delivered/
failed) and log `time_to_send` for every lead. Alert if `time_to_send` exceeds 90 seconds
(buffer before the 2-minute SLA is actually breached, not after) or if the failure rate crosses a
threshold (e.g., >5% failed sends in a rolling hour) — that's usually the signal of a template
getting rejected or Karix having an outage, not one-off failures. A simple dashboard (or even a
scheduled query against the lead log table) showing send latency distribution over the last 24
hours is enough to catch this before it becomes "why didn't three patients get their WhatsApp
confirmation yesterday."

*(Word count: ~390)*
