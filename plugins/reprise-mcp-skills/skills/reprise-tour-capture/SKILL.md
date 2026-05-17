---
name: reprise-tour-capture
description: "Reprise Product Tour capture workflow — recording NEW tours via the Reprise Builder HTML Chrome extension. MUST be invoked before any `tour_capture` call. Triggers on `tour_capture`, `pairing_token`, `deep_link_url`, `capture_now`, a brand-new `draft_id`, or the Reprise Builder HTML extension. Usually invoked by the `reprise-mcp` router; can also auto-fire on direct keyword matches. Body has the 7-step flow at headline depth, the 3-tier extension-pairing escalation named, and the load-bearing page-settle / verify-the-page principles. Per-tier mechanics, `capture_now` envelope, `set_auto`, post-stop cleanup, install URLs, theming, re-skinning, and composing all live in `docs(slug='tour')`. NOT for editing existing tours — use `reprise-tour-edit`."
version: 0.3.0
---

# Reprise Product Tour Capture

A Product Tour is a linear walkthrough built from captured DOM snapshots. Capture happens in the user's own Chrome via the **Reprise Builder: HTML** extension — the agent doesn't snapshot pages itself; it orchestrates.

## The capture flow

1. `tour(action='create', title=...)` → `{draft_id, preview_url}`.
2. `tour_capture(action='start', draft_id=...)` → `{pairing_token, deep_link_url, ...}`.
3. **Pair the extension** (3-tier escalation, see below — highest failure point).
4. `tour_capture(action='status', pairing_token=...)` until `extension_last_seen_at` is fresh (<10s) ⇒ paired.
5. `tour_capture(action='ensure_toolbar', pairing_token=..., target_url='https://...')`.
6. Per-screen loop: browser MCP navigate / click → wait for settle → `tour_capture(action='capture_now', pairing_token=..., title='Step N', target_url='...')`.
7. `tour_capture(action='stop', pairing_token=...)`, then `tour_lifecycle(action='publish', draft_id=...)`.

**Always pass `target_url` to `ensure_toolbar` and `capture_now`.** Without it the SW falls back to "active tab in the last-focused Chrome window" and silently picks the wrong tab in multi-window or background-agent setups.

## Pairing the extension — 3-tier escalation

The `deep_link_url` is a `chrome-extension://...` URL. Browser-automation MCPs can't `navigate` to those (CDP rejects them as browser-internal). Try in order, escalate on failure:

- **Tier 1 — JS-assignment from a real-page tab** (fully automated). From a tab on a real `https://` page, evaluate `window.location.href = '<deep_link_url>'` as **its own tool call** (don't chain with a preceding navigate — they race). The assignment succeeds; any subsequent tool call on that tab errors with "Can't interact with browser internal pages" — that error IS the success signal.
- **Tier 2 — User pastes the deep link** into Chrome's address bar.
- **Tier 3 — User pastes the token** into the popup's "Pair with agent" field.

Never send the `chrome-extension://` URL as a clickable `<a href>` — Chrome blocks the click. Full per-tier mechanics, race-condition details, and stranded-tab recovery in `docs(slug='tour')`.

Don't proceed past `ensure_toolbar` until status confirms pairing — commands to an unpaired session silently no-op. If the user doesn't have the extension installed, send them to the Chrome Web Store URL in `docs(slug='tour')`.

## Two pre-capture principles that prevent the most common silent failures

1. **Page-settle before each `capture_now`.** A capture of an unsettled page produces an unstyled / half-rendered screen with **no error signal**. Between navigation and `capture_now`: `page.waitForLoadState('networkidle')`, or `browser_wait_for` on a sentinel hero-text element, or an 800–1500ms blanket sleep. Re-apply after every cross-document navigation.
2. **Verify the page is what you intended.** `capture_now` happily snapshots a 404, a soft-redirect, a "you must accept cookies" interstitial, or a CTA-triggered overlay — none fail your call. Before capturing: confirm URL, confirm title isn't a 404, confirm a sentinel element from the intended content is present. After CTA clicks (booking, signup, pricing), screenshot to verify; if an overlay landed on top, dismiss it before `capture_now`.

## Pattern search before custom JS or diagnosis

Before writing custom JS for a screen or diagnosing a broken tour, call `search_patterns(symptom='...', product='tour')`. Treat `score < ~0.5` as weak guesses — the tour pattern corpus is thinner than clone's.

## Preview before publish

`capture_now` returns `flush_complete=true` when every screen is processed. **Always wait for `flush_complete=true` before opening `preview_url`** — otherwise the preview shows placeholders for screens still being processed.

## After every session

`session_recap` + `report_friction` per distinct issue — see `reprise-session-close`.

`capture_now` envelope fields, async / fire-and-forget mode, `set_auto` continuous capture, post-stop toolbar cleanup, install URLs, theming via `--rguide-*`, re-skinning, composing tours from screens of other tours — all in `docs(slug='tour')`. For editing existing tours (not capturing new ones), use the `reprise-tour-edit` skill. If you have a `/launch/<slug>/` URL or hit a `wrong_id_kind` error, see `reprise-tour-id-model`.
