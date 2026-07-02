# Namoza Developer Assignment — OrthoNow

Submission for the Developer (Client Web + Martech) role. Three tasks, each in its own folder.

## Structure

```
.
├── README.md
├── task-1-gtm-schema/
│   └── Task1-GTM-Schema.md       # event schema, dataLayer JSON, funnel + conversion reasoning
├── task-2-landing-page/
│   ├── landing-page.html         # single-file, no build step, open directly in a browser
│   └── PAGESPEED-NOTES.md        # how I verified perf + what to screenshot before submitting
└── task-3-integration/
    └── Task3-Integration.md      # HubSpot / Karix / Ads architecture, written answer
```

## How to review

- **Task 1** — open `task-1-gtm-schema/Task1-GTM-Schema.md`. The table is the schema; the
  section below it is the actual answer to "how do you track a 3-step form," which is the part
  worth reading closely rather than skimming.
- **Task 2** — open `task-2-landing-page/landing-page.html` directly in a browser (double-click
  it, no server needed). Open dev tools console, submit the form, and you'll see the
  `consultation_form_submitted` dataLayer push logged. I still need to run this through PageSpeed
  Insights myself and drop the screenshot in before this goes to Namoza — see
  `PAGESPEED-NOTES.md` for why I'm confident it'll clear 90 and what to check if it doesn't.
- **Task 3** — `task-3-integration/Task3-Integration.md`, ~390 words as scoped.

## A few decisions I want to be upfront about

- The booking form's multi-step tracking assumes it's a client-side (no page reload) multi-step
  form. If OrthoNow's actual dev team built it as three separate pages, half of Task 1's funnel
  section is unnecessary complexity — I've stated that assumption directly in the doc rather than
  hiding it.
- The landing page has zero images and no webfont on purpose. That's a performance decision, not
  a "didn't have time for design" shortcut — it's explained in the HTML comments and in
  PAGESPEED-NOTES.md.
- Task 3's HubSpot dedup section is the part I'd want to walk through live — it's the one place
  in this assignment where getting it wrong doesn't just look bad in a report, it actually loses
  patient data in production.

## Still to do before this ships to Namoza
- [ ] Run the landing page through PageSpeed Insights (Mobile) and add the screenshot
- [ ] Record the Loom (script drafted in `loom-script.md` — not part of the graded repo, just my
      own notes so I don't ramble past 8 minutes)
- [ ] Push to a public repo / share with himanshu@namoza.com
- [ ] Email repo + Loom links to naman@namoza.com, subject: `Developer Assignment - [Your Name]`
