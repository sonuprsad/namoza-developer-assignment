# Suggested commit history

Not a requirement of the brief, but a repo with one giant "add all files" commit is a small tell.
Here's a realistic sequence if you're initializing fresh — squash/reorder as fits how you actually
worked.

```
git init
git add README.md
git commit -m "Initial repo structure and README"

git add task-1-gtm-schema/Task1-GTM-Schema.md
git commit -m "Task 1: GTM event schema and booking funnel design"

git add task-2-landing-page/landing-page.html
git commit -m "Task 2: landing page structure and copy (HTML + CSS)"

git add task-2-landing-page/landing-page.html
git commit -m "Task 2: form validation and dataLayer push on submit"

git add task-2-landing-page/PAGESPEED-NOTES.md
git commit -m "Task 2: performance notes ahead of PSI verification"

git add task-3-integration/Task3-Integration.md
git commit -m "Task 3: HubSpot/Karix/Ads integration architecture writeup"

# after you actually run PageSpeed Insights:
git add task-2-landing-page/pagespeed-score.png
git commit -m "Task 2: add PageSpeed Insights mobile score screenshot"
```

If you used this as a starting point and then actually edited the HTML/CSS yourself (recommended —
know every line before the Loom), your real commit history will naturally look more granular than
this, which is a good thing.
