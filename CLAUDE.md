# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Static, build-free Hebrew (RTL) landing page for **"לחזור לעצמך"** — a women's empowerment / coaching group run by Orit Asor. Hosted on GitHub Pages at `https://israel194.github.io/orit-landing/`. There is no bundler, no package manager, no test runner — every file in the repo root is what gets served.

Open `index.html` directly in a browser to preview locally; pushing to `main` is the deploy.

## Files that ship

- `index.html` — the public landing page (single-page, all sections inline).
- `style.css` — site styles. CSS variables for the palette/spacing live in the `:root {}` block at the top.
- `content.json` — **the only place to change copy/images/links without editing markup**. Loaded at runtime by `index.html` and applied to every element with a matching `data-edit="<key>"` attribute (see "Content flow" below).
- `takanon.html` — Hebrew privacy / terms page (linked from the cookie banner).
- `*.jpeg` (`1.jpeg`–`4.jpeg`, `bit.jpeg`) — hero/about photos. They live at the repo root because `content.json` references them as plain filenames (e.g. `"hero_image": "4.jpeg"`).
- `orit20266/index.html` and `daniel2030/index.html` — two separate admin panels (see "Admin panels" below). They are intentionally `<meta name="robots" content="noindex, nofollow">` and have obscure URL slugs.

## Content flow (this is the spine of the site)

`index.html` defines the layout and *default* Hebrew copy inline. On load, an inline IIFE near the bottom of `index.html` does:

1. `fetch('./content.json?v=' + Date.now())` (cache-buster).
2. For every element with `data-edit="<key>"`, replace `innerHTML` with `content[key]` if defined.
3. Apply `content.hero_image` / `content.about_image` to the `<img>` tags by id (`heroPhoto`, `aboutPhoto`).
4. Build the WhatsApp float `href` from `content.wa_number` + `content.wa_message`.
5. Stash `content.payment_url` on `window._paymentUrl` for the modal form's redirect.

**Practical consequences when editing:**
- To change visible text, edit `content.json` (preferred) — the inline default in `index.html` is just a fallback if the fetch fails.
- If you add a new editable string, you must (a) put a `data-edit="new_key"` attribute on the element in `index.html`, (b) add `new_key` to `content.json`, and (c) — if you want the admin panel to manage it — add a `data-key="new_key"` field in `orit20266/index.html` *and* extend the `defaults` object in its inline script (around line ~690).
- `pain_text`, `process_text*`, etc. allow `<br>` because they are written via `innerHTML`. Don't sanitize them away.

## The registration → SUMIT payment flow

The CTA buttons call `openModal()` (defined inline in `index.html`). The form collects name/phone/email + a session date.

The "next 4 Mondays" radio list is generated client-side at page load by `generateMondays()` — no backend involved. Submitting the form does **not** post to anything; `handleSubmit()` shows a brief "redirecting…" message and then sets `window.location.href = window._paymentUrl` (or a hardcoded SUMIT fallback URL). All actual lead capture happens at SUMIT.

If you need to record submissions before redirecting (analytics, CRM, email), wire it in `handleSubmit()` *before* the `setTimeout` redirect. Use `keepalive: true` on the fetch so the request survives the navigation.

## Admin panels — two parallel systems (don't confuse them)

**`/orit20266/` — the real one.** Publishes to the live site via the **GitHub Contents API**. The user pastes a GitHub Personal Access Token (with `repo` scope) into the password field; it's stored in `localStorage['orit_gh_token']` and used to:
- `GET /repos/israel194/orit-landing/contents/content.json` to load current values.
- `PUT` the same path with a base64-encoded body to commit a new `content.json` (the SHA fetch + include is required by the API).
- `PUT /contents/<filename>` to upload images chosen via the file picker (saved as `hero-photo.<ext>` / `about-photo.<ext>`).

The repo constant is hardcoded as `var REPO = 'israel194/orit-landing';`. After publishing, GitHub Pages takes ~30–60s to redeploy. The token never leaves the browser.

**`/daniel2030/` — the local-preview one.** Older/simpler panel that *only* writes to `localStorage['orit_site_content']`. It does **not** publish anywhere. The public site (`index.html`) does not read from this localStorage key at all, so changes here have **no** effect on the live site or even on what other browsers see. Treat this as a draft/preview tool only; if a non-technical user expects their edits to "go live", they need to be in `/orit20266/` instead.

## RTL, fonts, accessibility, cookies

- Document is `<html lang="he" dir="rtl">`. The accessibility widget panel and cookie banner both also set `direction: rtl` explicitly because they're absolutely positioned.
- Fonts: `Assistant` (body) and `Secular One` (display headings), loaded from Google Fonts at the top of `<head>`. The accessibility widget's "readable font" toggle (`body.acc-readable`) overrides them with Arial.
- The accessibility widget is a self-contained block at the bottom of `index.html`: a fixed circular button (`#acc-btn`) on the right edge that opens `#acc-panel`. Each toggle adds a class to `<body>` (`acc-high-contrast`, `acc-grayscale`, `acc-invert`, `acc-links`, `acc-readable`, `acc-no-anim`) and persists the state under `localStorage['acc_<class-name>']`. Font-size persists under `localStorage['acc_fontSize']` as an integer step (-3..+5, each = 8% of root font size). The whole widget can be hidden via `localStorage['acc_hidden']`.
- Cookie banner state is `localStorage['cookie_consent']` (`'accepted'` / `'declined'`). It only gates the banner UI — no actual cookies are conditionally set based on it.

## Deploy

GitHub Pages serves the repo root from `main` (no Actions workflow checked in — it's the classic Pages-from-branch setup). Anything you push to `main` is live in seconds to a minute. There is no staging environment.

When editing through `/orit20266/`, commits land directly on `main` with the message `Update site content via admin panel` — that's why the git log is mostly those.
