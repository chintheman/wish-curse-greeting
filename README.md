# Wish/Curse Greeting Site

A trojan-horse greeting card generator: the sender picks a message anywhere from genuinely wholesome to fully savage, generates a link, and the recipient always sees the same generic "something wholesome is loading" sequence before a randomized reveal.

**Status:** working prototype, content brainstorm complete, not yet hardened or hosted.

See [`reports/2026-07-24-handoff-status-report.md`](reports/2026-07-24-handoff-status-report.md) for the full handoff report — concept, product spec, technical architecture, test coverage, known limitations, and suggested next steps.

## Contents

- `reports/` — status reports and handoff docs
- `wish-app/` — the app itself (`index.html`)
- `wish-content-brainstorm.md` — message copy reference

## Run it locally

No build step. Open `wish-app/index.html` directly, or serve it (recommended for clipboard reliability):

```bash
cd wish-app && python3 -m http.server 8000
```

Then visit `http://localhost:8000/`. The sender view loads by default; append `?w=<encoded>` to preview a specific recipient state, or use the in-app "Preview as Recipient" button after generating a link.
