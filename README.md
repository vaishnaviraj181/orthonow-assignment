# OrthoNow — Namoza Developer Assignment

Submission for: Developer - Position 1 (Client Web + Martech)

## Repo structure

```
orthonow-assignment/
├── README.md                          ← this file
├── task-01-gtm-schema/
│   └── README.md                      ← event schema table + dataLayer JSON
├── task-02-landing-page/
│   ├── index.html                     ← single-file landing page
│   └── pagespeed-screenshot.png       ← add after running PageSpeed Insights
└── task-03-integration-design/
    └── README.md                      ← written integration answer
```

## How to run Task 02 locally
Open `task-02-landing-page/index.html` directly in a browser — no server, build step, or dependencies needed. To test the dataLayer push: open DevTools Console, fill in the name + a valid 10-digit Indian mobile number (starts with 6-9), submit, and run `window.dataLayer` in the console to see the `consultation_form_submitted` event object.
