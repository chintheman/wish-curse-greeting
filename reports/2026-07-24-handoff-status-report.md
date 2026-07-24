# Wish/Curse Greeting Site — Handoff Status Report

**Status:** Hardened and deployed. Clipboard, reduced-motion, and keyboard/screen-reader basics fixed; all 8 skins visually QA'd; live on GitHub Pages.
**Last updated:** 2026-07-24
**Reference files:** `wish-content-brainstorm.md` (copy reference), `wish-app/index.html` (the app itself)

> This report was written from a handoff summary during repo setup, then updated in place as follow-up work landed the same day. See "Update — 2026-07-24 (build-out pass)" near the bottom for what changed after the initial handoff.

---

## 1. The concept, in one paragraph

This is the same genre as make-everything-ok.com — single-purpose, emotionally resonant, zero-friction — but structured as a trojan horse greeting card. A sender picks a message ranging from genuinely wholesome to fully savage, and generates a link. Whoever opens that link always sees the same generic "something wholesome is loading" sequence first, with no way to tell in advance what's actually inside. Then it reveals — randomly wrapped in one of several distinct visual "skins" each time, so the experience doesn't get stale or learnable, and a friend can be tricked with the same mechanism repeatedly. Audience for now is personal/friend-group use; no guardrails have been built in beyond the content-tier judgment calls below, on the understanding that broader guardrails get revisited if this ever goes wider than friends (per the user's direction in the original conversation).

## 2. Product spec

### Sender side (fixed UI, same every time)

- Category tabs across 7 buckets: Wholesome, Genie Wish, Backhanded Compliment, Curse, Roast, Prophecy, Chaos.
- Clicking a tab shows a scrollable list of preset messages for that category, each with a 🌶️1-5 spice indicator so the sender can judge intensity before picking.
- A "write my own message" checkbox switches to free text entry, with a manual spice slider (1-5) since there's no way to infer tier from arbitrary text.
- "Generate Link" produces a URL the sender can copy or immediately preview as if they were the recipient.

### Recipient side (deliberately generic first, then randomized)

- Landing on the link always shows the identical loading sequence regardless of what's inside: a pulsing heart, "Sending you something good...", a progress bar, and five rotating status lines ("Gathering good vibes...", etc.). Duration is randomized (~2.6–3.2s) so it doesn't feel like a scripted animation.
- At 100%, one of 8 pre-built visual skins is picked at random — Confetti Pop, Starlight, Floral Bloom, Retro Postcard, Balloon Drop, Zen Garden, Neon Arcade, Paper Lantern — completely decoupled from message category. This is what keeps the trick fresh: the visual wrapper carries no signal about what's about to appear.
- The reveal shows a small category badge (💛/🧞/🙃/🔮/🔥/✨/🌀 + label) and the message text. The badge appears after the twist is already known, so it's flavor, not a spoiler.
- A "Send someone their own →" link at the bottom loops back to the sender page (strips the query param).

### Content tiers

Full text lives in `wish-content-brainstorm.md`. 7 categories × spice 1-5. Savage tier is deliberately kept at exaggerated, impersonal insult-comedy (roast-battle style) rather than anything targeting real insecurities, appearance, or mental health — a judgment call made because these links can go to anyone a sender picks, with no visibility into that person's state of mind. Worth revisiting explicitly if a future collaborator wants to push further in a specific direction.

## 3. Technical architecture

No backend. No database. No auth. It's a single static HTML file (`wish-app/index.html`, ~560 lines, inline CSS + JS, no external dependencies or CDN calls).

**How state travels:** the entire message payload is encoded into the URL itself.

```
{ m: "<message text>", c: "<category key>", t: <spice tier 1-5> }
→ btoa(encodeURIComponent(JSON.stringify(payload)))
→ index.html?w=<encoded>
```

- `encodeURIComponent` before `btoa` handles non-ASCII/emoji safely; decode path is the exact inverse (`JSON.parse(decodeURIComponent(atob(encoded)))`), with a try/catch fallback that defaults to a wholesome message if decoding ever fails (e.g., a mangled link).
- Routing is a single `if (params.get('w'))` check at the bottom of the script: presence of `?w=` means "render recipient," absence means "render sender." Both views live in the same file and same `#app` div.

**Key functions/data (for anyone picking this up):**

- `MESSAGE_BANK` — object keyed by category (`wholesome`, `genie`, `backhanded`, `curse`, `roast`, `prophecy`, `chaos`), each with `label`, `emoji`, and an `items` array of `[text, tier]` pairs. Add new messages/categories here.
- `SKINS` — array of 8 skin name strings, matched to `.skin-<name>` CSS classes further up in the stylesheet. Add a new skin by adding a CSS block + pushing the name into this array.
- `BADGES` — category key → display badge string.
- `STATUS_LINES` — the 5 rotating loading-screen phrases (shared across all sends, do not vary by category).
- `renderSender()` — builds and wires up the sender UI (tabs, list, custom-message toggle, link generation, copy/preview buttons).
- `renderRecipient(encoded)` → `decodePayload()` → progress-bar interval → `revealMessage(data)` → `decorate(skin)` — the recipient pipeline. `decorate()` scatters skin-appropriate glyphs (confetti emoji, stars, florals, balloons, lantern, etc.) as absolutely-positioned decorative spans.
- `escapeHtml()` — used when injecting the message text into the DOM, to avoid any HTML injection if a sender's custom message contains `<`/`>`/etc.

## 4. What's been tested vs. not

**Verified:**

- Full sender → link → loading → reveal flow via Playwright (headless Chromium): Curse tab, tier-5 item, link generation, and correct message/badge/skin on reveal. Encode/decode round-trip confirmed correct, and skin randomness confirmed (not hardcoded).
- All 8 skins rendered and screenshotted with `Math.random` deterministically forced per-skin — confetti, starlight, floral, retro, balloon, zen, neon, and lantern all show correct background/text/badge/button contrast and legible text at a 900px desktop viewport. No skin-specific layout breakage found.
- Mobile viewport emulation (iPhone 13, Pixel 5 via Playwright device profiles) for both the sender view and a full tap-driven reveal flow: no horizontal overflow on either device, cat-tab and msg-item tap targets both ≥32px tall.
- `prefers-reduced-motion: reduce` respected — heart pulse and reveal fade-in animations are disabled via a media query when the user has that OS-level preference set.
- Keyboard navigation confirmed: category tabs and message list items are focusable (`tabindex="0"`) and selectable with Enter/Space, with `role="tab"`/`role="option"` and `aria-selected` wired up for screen readers. The loading progress bar has `role="progressbar"` with live `aria-valuenow`, and the status line is `aria-live="polite"`.
- Clipboard copy now uses `navigator.clipboard.writeText()` (verified programmatically against `navigator.clipboard.readText()`), falling back to the old `execCommand('copy')` only if the modern API throws. Button shows "Copied!" for 1.5s as user feedback.
- Deployed to GitHub Pages via `.github/workflows/deploy-pages.yml` — see the Update section below for details and the live URL.

**Not yet verified — flag these before wider use:**

- Real physical mobile devices (iOS Safari / Android Chrome) — only tested via Playwright's device emulation (Chromium with spoofed viewport/UA), not an actual phone. Font rendering, real touch behavior, and Safari-specific quirks are still unconfirmed.
- Actual SMS/iMessage link-sharing behavior — long base64 payloads make for long, ugly URLs. Some messaging apps may wrap, truncate previews oddly, or generate a link-preview card that shows part of the raw payload before it's clicked. Worth a real test send now that there's a stable hosted URL to send.
- Real screen-reader testing (VoiceOver/NVDA/TalkBack) — the ARIA roles and live regions added in this pass are structurally correct but haven't been listened to with an actual screen reader.
- Formal contrast-ratio audit (e.g. WCAG AA numbers per skin) — the skins were checked by eye via screenshots and all read as clearly legible, but no automated contrast-checker pass has been run.

## 5. Known limitations / things to decide on purpose, not by accident

- The payload is not secret. It's just base64 + JSON, readable by anyone who pastes the `?w=` value into a decoder or opens dev tools. This is fine for the current friend-group use case (nobody's going to reverse-engineer a text from a friend), but if this ever gets shared more widely, someone will eventually write it up as "here's how to peek at your wish before opening it." Worth deciding now whether that matters, versus treating it as acceptable since the social trick (you don't expect your friend to spoil their own prank) is really what's carrying this, not technical secrecy.
- No persistence, no analytics. There's no record of who sent what to whom, whether a link was opened, or how many times. If you want open-rate tracking or a "your friend saw this" receipt, that requires an actual backend — a meaningful architecture change from the current fully-static approach.
- No expiry. Links work forever since all state lives in the URL. If you want one-time-use or time-limited links, that also requires server-side state.
- No moderation on custom messages. The free-text option has no filter at all beyond HTML-escaping for safety (no script injection). Anything the sender types goes out verbatim. Fine for a closed friend group; a real gap if this becomes multi-user/public.
- Category badge appears on custom messages too, tagged with whatever tab was active when the sender wrote it — there's no way currently to send a fully "uncategorized" custom message.

## 6. Suggested next steps

**Done:**

1. ~~Swap clipboard copy to `navigator.clipboard.writeText()`.~~ Done — see Update section below.
2. ~~Decide on hosting.~~ Done — deployed to GitHub Pages (free, zero-backend, matches the static-file architecture). No custom domain chosen yet; still using the default `chintheman.github.io/wish-curse-greeting/` URL.

**Still open:**

1. Real-device test pass — send an actual link via text to a phone, click through, screenshot each skin. Emulated-device testing (Playwright) is done; a real phone still isn't.
2. Custom domain decision (something in the make-everything-ok.com naming family, e.g. a "send-a-wish"-style name) — not yet chosen. GitHub Pages supports a custom domain via a `CNAME` file whenever one is picked.
3. Decide whether to add the "sender chooses the skin too" option that was discussed and deferred, or keep skins fully random as-is.
4. Decide whether custom messages should support opting out of a category badge entirely.
5. If this expands past friends: revisit the savage-tier content boundary, add basic reporting/blocking, and consider whether payload obscurity is still acceptable or needs real server-side storage.

## 7. How to run it locally

No build step. Just open `wish-app/index.html` directly in a browser, or serve it (recommended for clipboard API reliability):

```
cd wish-app && python3 -m http.server 8000
```

then visit `http://localhost:8000/`. The sender view loads by default; append `?w=<encoded>` to preview a specific recipient state, or use the in-app "Preview as Recipient" button after generating a link.

---

## Update — 2026-07-24 (build-out pass)

Same day as the initial handoff, a follow-up pass closed out most of the "not yet started" items:

- **Clipboard:** `copyBtn` now calls `navigator.clipboard.writeText()` with a fallback to the old `document.execCommand('copy')` if the modern API throws (e.g. insecure context). Button label flips to "Copied!" for 1.5s as feedback.
- **Reduced motion:** added `@media (prefers-reduced-motion: reduce)` disabling the heart-pulse animation, the reveal fade-in, and the progress-bar width transition.
- **Accessibility basics:** category tabs and message-list items are now keyboard-focusable and selectable (Enter/Space), with `role="tab"`/`role="option"`/`aria-selected` wired up; the loading bar is `role="progressbar"` with a live `aria-valuenow`; the status line is `aria-live="polite"`.
- **Visual QA:** all 8 skins screenshotted at a 900px desktop viewport (forcing `Math.random` deterministically per skin to bypass the random assignment) and reviewed by eye — all render with legible text, working badges, and no layout breakage.
- **Mobile QA:** Playwright device emulation (iPhone 13, Pixel 5) covering both the sender view and a full tap-through reveal flow — no horizontal overflow, tap targets ≥32px.
- **Hosting:** deployed via a new `.github/workflows/deploy-pages.yml` GitHub Actions workflow (`actions/configure-pages` + `actions/upload-pages-artifact` + `actions/deploy-pages`, serving the `wish-app/` directory as the Pages root). This requires no manual "enable Pages" step in repo settings — the workflow's own `pages: write` permission is sufficient to provision the Pages site on first run. **Live at:** https://chintheman.github.io/wish-curse-greeting/

None of this changed the message content, the payload encoding scheme, or the skins' visual design — it's hardening and shipping, not a rewrite. The content-tier judgment calls in Section 2, and the known limitations in Section 5 (payload not secret, no persistence/expiry, no moderation on custom text), are all still exactly as they were and still worth a deliberate decision before this goes past a friend group.

## Open item

~~This repository (`chintheman/wish-curse-greeting`) was just created to host this project and currently contains only this report — `wish-app/index.html` and `wish-content-brainstorm.md` themselves have not been added yet.~~ **Resolved:** both files were added in a follow-up commit and now live at [`wish-app/index.html`](../wish-app/index.html) and [`wish-content-brainstorm.md`](../wish-content-brainstorm.md).

No other open items from repo setup remain. See "Still open" under Section 6 for the substantive product/content decisions that remain.
