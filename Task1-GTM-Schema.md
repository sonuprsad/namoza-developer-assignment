# Task 01 — GTM Event Schema (OrthoNow)

Before the table, one assumption I'm making explicit because it changes how half of this schema
gets built: the booking form is a **client-side multi-step form** (steps swap in/out via JS, no
full page reload between them). That's the only way "track drop-off between steps 1/2/3" is even
a real problem — if each step were its own URL, you'd just use pageview funnels and this task
would be trivial. Given the brief explicitly asks *how* to track step-level drop-off, I'm treating
it as an SPA-style form, which is also how basically every healthcare booking widget I've worked
with actually behaves.

## Naming convention

`{object}_{action}` in snake_case, lowercase, no spaces — matches GA4's own event naming rules
(GA4 will silently reject or mangle names with spaces/special chars, so I don't fight that).
Booking funnel events are prefixed `booking_` so they group together in GA4's event list instead
of being scattered alphabetically.

## Event Schema

| Event Name | Trigger Type (GTM) | Key Parameters | GA4 Report / Audience |
|---|---|---|---|
| `booking_form_start` | Custom Event — fires when the booking widget/modal mounts (front-end pushes this on "Book Appointment" click, before step 1 renders) | `form_id`, `entry_point` (homepage / clinic_page / landing_page), `page_path` | Funnel Exploration (step 0, baseline). Also seeds the "started booking, didn't finish" remarketing audience |
| `booking_step_complete` (step_number: 1) | Custom Event, filtered on `step_number = 1` via a Data Layer Variable | `step_number`, `step_name`, `clinic_location`, `specialty` | Funnel Exploration step 1. Feeds "location selected" audience for clinic-specific retargeting |
| `booking_step_complete` (step_number: 2) | Custom Event, filtered on `step_number = 2` | `step_number`, `step_name`, `clinic_location`, `preferred_date` | Funnel Exploration step 2. This is the point where we have a phone number — see conversion note below |
| `booking_confirmed` | Custom Event, fires on successful step-3 submit (after backend/API confirms slot) | `booking_id`, `clinic_location`, `specialty`, `booking_value` (optional, if you want to model LTV later) | Funnel Exploration step 3 / final. Marked as GA4 Key Event. Excludes converted users from remarketing audiences |
| `call_now_click` | Click trigger — Just Links, or All Elements with CSS selector `[data-cta="call-now"]` (don't rely on `tel:` href matching alone, WhatsApp share links also start with weird schemes and you'll get false positives) | `click_location` (homepage / clinic_page / landing_page), `clinic_name`, `phone_number` | Engagement report, filtered by `click_location`. Import as a Google Ads "phone call click" conversion — see note |
| `whatsapp_widget_click` | Click trigger on widget's fixed ID (`#wa-float-btn`), not a class, because classes get reused across the theme | `page_path`, `page_type`, `clinic_name` (if on a clinic page, else `not_set`) | Engagement report. Secondary/micro-conversion for Ads |
| `patient_guide_form_submit` | Form Submission trigger, scoped to the guide's form ID (not "All Forms" — the site has other forms and you don't want this polluting the booking funnel) | `form_id`, `page_path`, `lead_source` | Lead gen report. Builds "guide downloader" audience (mid-funnel, not yet booked) |
| `patient_guide_download` | Custom Event, fired by front-end *only after* the gated form submit succeeds — not a raw click trigger on the PDF link | `file_name`, `guide_topic`, `gated` (true) | File download report. See dedup note below — this deliberately does **not** rely on GA4's automatic `file_download` event |
| `clinic_page_view` | No new event — reuse GA4's automatic `page_view`, but push `clinic_name` and `clinic_city` into the dataLayer *before* the GTM container loads, then map them as event parameters on the GA4 Configuration tag | `clinic_name`, `clinic_city`, `page_type: clinic_location` | Standard GA4 pages report, sliced by the `clinic_name` custom dimension (register it as a custom dimension in GA4 Admin, or it just sits in BigQuery unused) |
| `blog_scroll_depth` | GTM's built-in **Scroll Depth** trigger type, vertical percentages `25,50,75,90`, restricted via trigger condition to Page Path matching `/blog/*` | `percent_scrolled`, `article_title`, `article_category` | Engagement report. 75%+ scrollers feed an "engaged reader" remarketing list — these convert better than cold traffic |

A couple of things I did on purpose that are worth calling out because they're easy to get wrong:

- **`call_now_click` isn't just "trigger on tel: links."** OrthoNow has call buttons on the
  homepage, 9 clinic pages, and the landing page. If you don't capture `click_location`, GA4 will
  show you "127 call clicks" with no way to know if the landing page's CTA is actually pulling its
  weight versus people just calling from a clinic's own page. The parameter *is* the insight here.
- **`patient_guide_download` is deliberately a custom event, not GA4's automatic file_download.**
  GA4's Enhanced Measurement already auto-fires `file_download` for any PDF link click. If you also
  wire up a GTM trigger on the same link, you get two events for one action and your download
  count in reports is inflated 2x. Either turn off automatic file/download tracking for this
  specific asset in GA4 Admin, or — what I'd actually do — leave Enhanced Measurement on for the
  rest of the site (fine for non-gated downloads) and only override behaviour for this one gated
  PDF, firing the custom event instead of letting the auto one double up. Worth flagging in the
  Loom because it's the kind of bug that ships silently and nobody notices until someone asks
  "why did downloads jump 100% in March" three months later.

## Booking funnel — step tracking in detail

This is the part that actually matters, so I'm not glossing over it.

**GTM cannot see inside a multi-step JS form on its own.** There's no DOM event for "user
completed step 2 of a form that never navigates or submits until the very end." A Form Submission
trigger only fires on an actual `<form>` submit event — and if this is a single form element with
steps hidden/shown via JS (the more common pattern), that submit event only fires once, at the
very end, which tells you nothing about where people dropped off. GTM's job here is just to
*listen* for something the front-end tells it happened. The front-end developer has to explicitly
push a dataLayer event at each step transition. This is a briefing item, not a GTM config item —
I'd tell the dev team exactly what to push and when, then build GTM triggers against that contract.

**dataLayer pushes** (fired client-side, by the front-end, at the moment each step is
successfully completed and validated):

```json
{
  "event": "booking_form_start",
  "form_id": "appointment_booking",
  "entry_point": "landing_page",
  "page_path": "/book-a-consultation"
}
```

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Indiranagar - Bengaluru",
  "specialty": "Knee & Joint Care"
}
```

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "Indiranagar - Bengaluru",
  "preferred_date": "2026-07-05"
}
```

```json
{
  "event": "booking_confirmed",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Indiranagar - Bengaluru",
  "specialty": "Knee & Joint Care",
  "booking_id": "ORTHO-20260705-0453"
}
```

Note I do **not** push the patient's name or phone number into the dataLayer. GTM/GA4 payloads
are not a safe place for PII — GA4 will actually flag and drop hits containing things that look
like phone numbers/emails in some fields, and even where it doesn't, you don't want raw contact
details sitting in Google's analytics pipeline. Name/phone go to HubSpot via the API call
directly (see Task 3), not through the dataLayer.

**GTM setup for each:**
- One Custom Event trigger per event name (`booking_form_start`, `booking_step_complete`,
  `booking_confirmed`).
- For `booking_step_complete`, use **two separate GA4 Event tags**, each firing on the same
  Custom Event trigger but with a trigger-level filter on a Data Layer Variable
  `{{DLV - step_number}}` equal to `1` or `2`. (You *could* do this with one tag and pass
  `step_number` straight through as an event parameter without splitting trigger conditions —
  either works. I'd split them only if step 1 and step 2 ever need genuinely different downstream
  logic, like a different remarketing audience per step. If they don't, one tag with
  `step_number` as a parameter is less config to maintain. I've listed them separately in the
  table above mainly so the funnel steps are legible.)

**Surfacing this in GA4 Funnel Exploration:**
Explore → Funnel exploration → build a **closed funnel** (strict, sequential) with steps:
`booking_form_start` → `booking_step_complete` (step_number = 1) → `booking_step_complete`
(step_number = 2) → `booking_confirmed`. Closed funnel matters here specifically because an open
funnel would count someone who jumps straight to `booking_confirmed` via a weird edge case (bug,
double-submit, back-button re-render) as having "completed" earlier steps they never actually
saw — that inflates your step 1→2 numbers artificially. Break down by `clinic_location` as a
secondary dimension so you can see if, say, one specific clinic's specialty list is confusing
people and causing step-1 drop-off that's clinic-specific rather than form-wide.

## Which conversion action to import into Google Ads, and why

I'd import **`booking_step_complete` at step 2 (contact details entered)** as the primary Google
Ads conversion — not `booking_confirmed`, even though that's the "real" outcome.

The obvious answer is "import the actual booking, that's the thing that matters." It's also the
wrong answer for a campaign that's just going live. Google Ads' automated bidding (Target CPA /
Maximize Conversions) needs a reasonable volume of conversion events to have enough signal to
optimise against — Google's own guidance is roughly 15–30 conversions in a 30-day window per
campaign before Smart Bidding has enough data to be reliable. A confirmed booking, at a 3-step
funnel with real drop-off between each step, is going to be a low-volume event, especially in the
first month. Bidding against a signal that thin means the algorithm is basically guessing, and it
tends to guess by throwing spend at whatever superficially correlates with the handful of
conversions it's seen.

Step 2 completion means we have a name, phone number, and preferred clinic — that's already a
usable lead the sales/reception team can call, even if the person never clicks "confirm" on step
3. It's a meaningfully higher-volume signal, it still represents real purchase intent (nobody
enters their phone number by accident), and it gives the bidding algorithm enough data to
actually learn what a good click looks like.

I'd still track `booking_confirmed` — just not as the primary Ads-optimised conversion. Once
there's enough booking volume (a month or two in), or once the sales team can mark step-2 leads
that convert into real bookings via a phone follow-up, I'd bring `booking_confirmed` in as an
**offline conversion import** with an actual booking value attached, so Ads can eventually
optimise toward booking quality rather than just lead volume. That's a phase-2 move, not a
launch-day one.

## Common mistake I'd flag if I were reviewing someone else's version of this

Picking `booking_confirmed` as the Ads conversion because it's literally the named goal in the
brief, without checking whether it'll have enough volume to feed Smart Bidding. It reads correct
on paper and quietly wrecks campaign performance for the first month.
