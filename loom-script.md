# Loom script (personal notes — not part of the graded repo)

Target: under 8 minutes. Screen share only, no slides. Rough timing in brackets.

## 1. GTM schema (≈2 min)
- Open `Task1-GTM-Schema.md`, scroll past the table quickly — don't read it row by row, they can
  read.
- Land on the booking funnel section. Say directly: "GTM can't see inside a JS-driven multi-step
  form on its own — this needs a dataLayer push from the front-end at each step, GTM is just
  listening." Show the four JSON blocks.
- Point out the conversion choice: step-2 completion, not final booking, as the Ads import — say
  the volume/Smart Bidding reasoning in one sentence, don't over-explain, save depth for their
  follow-up question.

## 2. Live demo — landing page (≈3 min)
- Open `landing-page.html` in browser, resize to mobile width (or use dev tools device toolbar).
- Show the page load fast, no flash of unstyled content.
- Open console, type `window.dataLayer` before submitting — show it's empty/just GTM boilerplate.
- Fill form with a bad phone number first — show inline validation error, no page reload.
- Fix it, submit — show thank-you state swap (no reload), then immediately show the console:
  `window.dataLayer` now has the `consultation_form_submitted` push with params visible.
- **This is the part they said they'd specifically check — don't rush it, let it actually render
  in the console before moving on.**

## 3. Integration architecture (≈3 min)
- Open `Task3-Integration.md`.
- Walk the flow in order: form submit → own endpoint → HubSpot upsert (parallel) → Karix WhatsApp
  (parallel, not chained) → Ads conversion already fired client-side.
- Say the dedup issue out loud even if they don't ask: "HubSpot dedupes on email by default, this
  form doesn't collect email, so I'm using a normalized phone number as a custom alternate ID
  property instead." This is the trap question — answer it before they ask it.
- One sentence on the fallback queue, one sentence on WhatsApp SLA monitoring. Don't read the doc
  verbatim, they've already read it.

## Anticipated first follow-up (per the brief's own interviewer notes)
"Who writes the dataLayer push — you or the front-end dev?" → Front-end dev writes it, I brief
them exactly what event name, what parameters, and at what point in the step transition to fire
it — GTM is the listener/router, not the source of the event.
