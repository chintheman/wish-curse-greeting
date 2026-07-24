# Wish/Curse Greeting Site — Handoff Status Report

**Status:** Working prototype, content brainstorm complete. Not yet hardened, hosted, or tested outside one automated browser run.
**Last updated:** 2026-07-24
**Reference files (to be added to this repo):** `wish-content-brainstorm.md` (copy reference), `wish-app/index.html` (the app itself)

> This report was written from a handoff summary during repo setup. The two reference files above have not yet been committed to this repository — see "Open item" at the bottom.

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

**Verified:** ran an automated Playwright pass (headless Chromium) that opened the sender UI, selected the Curse tab, picked a tier-5 item, generated a link, navigated to it, and confirmed the loading sequence resolved to the correct message, correct badge, and a valid random skin class. Confirmed the encode/decode round-trip works and that unrelated runs produce different skins (randomness is working, not hardcoded).

**Not yet verified — flag these before wider use:**

- Real mobile browsers (iOS Safari / Android Chrome) — only tested in desktop headless Chromium so far. Font stacks, tap targets, and viewport behavior on an actual phone haven't been eyeballed.
- Actual SMS/iMessage link-sharing behavior — long base64 payloads make for long, ugly URLs. Some messaging apps may wrap, truncate previews oddly, or generate a link-preview card that shows part of the raw payload before it's clicked. Worth a real test send.
- Clipboard copy button uses the deprecated `document.execCommand('copy')`, which still works in most browsers but should be swapped for `navigator.clipboard.writeText()` for reliability, especially on iOS Safari where the old API is flaky.
- Cross-browser visual QA of all 8 skins — only spot-checked one (Neon Arcade) via the automated test. The rest haven't been visually confirmed to render as intended.
- No accessibility pass (contrast ratios on skins like Neon or Starlight, screen reader behavior, reduced-motion preference for the pulsing heart / progress animation).

## 5. Known limitations / things to decide on purpose, not by accident

- The payload is not secret. It's just base64 + JSON, readable by anyone who pastes the `?w=` value into a decoder or opens dev tools. This is fine for the current friend-group use case (nobody's going to reverse-engineer a text from a friend), but if this ever gets shared more widely, someone will eventually write it up as "here's how to peek at your wish before opening it." Worth deciding now whether that matters, versus treating it as acceptable since the social trick (you don't expect your friend to spoil their own prank) is really what's carrying this, not technical secrecy.
- No persistence, no analytics. There's no record of who sent what to whom, whether a link was opened, or how many times. If you want open-rate tracking or a "your friend saw this" receipt, that requires an actual backend — a meaningful architecture change from the current fully-static approach.
- No expiry. Links work forever since all state lives in the URL. If you want one-time-use or time-limited links, that also requires server-side state.
- No moderation on custom messages. The free-text option has no filter at all beyond HTML-escaping for safety (no script injection). Anything the sender types goes out verbatim. Fine for a closed friend group; a real gap if this becomes multi-user/public.
- Category badge appears on custom messages too, tagged with whatever tab was active when the sender wrote it — there's no way currently to send a fully "uncategorized" custom message.

## 6. Suggested next steps (not yet started)

1. Real-device test pass — send an actual link via text to a phone, click through, screenshot each skin.
2. Swap clipboard copy to `navigator.clipboard.writeText()`.
3. Decide on hosting: this is a static file, so GitHub Pages, Netlify, Vercel, or even a personal S3 bucket all work with zero backend changes. Needs a domain decision (something in the make-everything-ok.com naming family, e.g. a "send-a-wish"-style name) — not yet chosen.
4. Decide whether to add the "sender chooses the skin too" option that was discussed and deferred, or keep skins fully random as-is.
5. Decide whether custom messages should support opting out of a category badge entirely.
6. If this expands past friends: revisit the savage-tier content boundary, add basic reporting/blocking, and consider whether payload obscurity is still acceptable or needs real server-side storage.

## 7. How to run it locally

No build step. Just open `wish-app/index.html` directly in a browser, or serve it (recommended for clipboard API reliability):

```
cd wish-app && python3 -m http.server 8000
```

then visit `http://localhost:8000/`. The sender view loads by default; append `?w=<encoded>` to preview a specific recipient state, or use the in-app "Preview as Recipient" button after generating a link.

---

## Open item

This repository (`chintheman/wish-curse-greeting`) was just created to host this project and currently contains only this report — `wish-app/index.html` and `wish-content-brainstorm.md` themselves have not been added yet. Add them (e.g. via a follow-up commit or PR) so the code referenced above actually lives here.
