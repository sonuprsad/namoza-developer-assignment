# PageSpeed Insights — how to verify, and what I can't do for you

Being straight about this: I built this file inside a sandboxed environment with no access to
pagespeed.web.dev or a real Chrome instance, so I cannot generate an actual PSI screenshot for
you. You need to run this yourself before submitting — the brief is explicit that the screenshot
is required, and "trust me it's optimised" isn't going to fly with an interviewer who said outright
that this is "the hard technical filter."

## What to do
1. Push `landing-page.html` to GitHub (or host it anywhere public — GitHub Pages is the easiest
   zero-config option: Settings → Pages → deploy from the branch, the file will be live at
   `https://<username>.github.io/<repo>/task-2-landing-page/landing-page.html`).
2. Run it through https://pagespeed.web.dev/ — Mobile tab.
3. Screenshot the score and drop it in this folder as `pagespeed-score.png`.

## Why this page should score 90+ without a fight
- Zero external network requests — no Google Fonts, no CDN JS, no images, no analytics script in
  this standalone file (GTM's own container script is the one thing you'd add back in for the
  live version, which does cost a request — see note below).
- System font stack only. No FOUT/FOIT, no font file to download, headline renders on first paint.
- No hero photo. The "trust element" and hero motif are inline SVG, so they're part of the HTML
  payload, not a separate image request or a Largest Contentful Paint risk.
- All CSS is inline in `<head>` — nothing render-blocking is fetched externally.
- JS is minimal, vanilla, deferred by nature of sitting at the bottom of `<body>`.
- No layout shift sources: form card has fixed padding, no images popping in late, no ad slots.

## One honest caveat
The moment you actually wire in the real GTM container snippet (`<script src="googletagmanager.com/gtm.js">`)
for production, that's a genuine third-party request and it will cost you a few points — that's
normal and every real GTM-tracked site has this trade-off. If PSI comes back below 90 after adding
GTM, the fix isn't to rip out the tracking, it's to make sure the GTM snippet is placed right after
the opening `<body>` tag (not blocking `<head>`) and that you're not stacking additional tags
(chat widgets, pixels) on top of it for this landing page specifically. Keep the page's tag
footprint to GTM only — no bolted-on Meta Pixel, Hotjar, etc. on this URL if you can avoid it.
