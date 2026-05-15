---
name: reprise-html-capture
description: "Reprise HTML tour capture workflow — recording NEW tours via the Reprise Builder HTML Chrome extension. MUST be invoked before any `html_capture` call. Triggers on `html_capture`, `pairing_token`, `deep_link_url`, `capture_now`, a brand-new `draft_id`, or the Reprise Builder HTML extension. Usually invoked by the `reprise-mcp` router; can also auto-fire on direct keyword matches. Body has the 7-step capture flow, the 3-tier extension-pairing escalation, and the page-settle rules. NOT for editing existing replays — use `reprise-html-edit` for that."
version: 0.1.0
---

# Reprise HTML Tour Capture

An HTML replay is a linear product tour built from captured HTML snapshots. Capture happens in the user's own Chrome via the **Reprise Builder: HTML** extension — the agent doesn't snapshot pages itself; it orchestrates: create the draft, mint a pairing token, drive `capture_now` at each step.

## The capture flow (7 steps)

```
1. html(action='create', title='...')
                            → {draft_id, preview_url}
2. html_capture(action='start', draft_id=...)
                            → {pairing_token, deep_link_url, ...}
3. Pair the extension (see below — this is the most common failure point).
4. html_capture(action='status', pairing_token=...)
                            → confirm extension_last_seen_at fresh (<10s).
5. html_capture(action='ensure_toolbar', pairing_token=..., target_url='https://...')
                            → allow-list the host, mount the toolbar.
6. Per-screen loop:
     a. browser MCP: navigate / click / scroll
     b. wait for the page to settle (see below)
     c. html_capture(action='capture_now', pairing_token=...,
                     title='Step N', target_url='https://...')
7. html_capture(action='stop', pairing_token=...)
   html_lifecycle(action='publish', draft_id=...)
```

**Always pass `target_url` to `ensure_toolbar` and `capture_now`** — without it, the SW falls back to "active tab in the last-focused Chrome window" and silently picks the wrong tab in multi-window or background-agent setups.

## Pairing the extension — 3-tier escalation

This is the highest-failure point in the product. The `deep_link_url` looks like `chrome-extension://<id>/pair.html?token=...`. Browser-automation MCPs cannot `navigate` to `chrome-extension://` URLs — CDP rejects them as browser-internal. Try these tiers in order; escalate only when the prior tier fails.

### Tier 1 — JS-assignment from a real-page tab (fully automated)

1. Ensure the automation tab is on a real `https://` page (the target app is fine). JS execution is blocked on `chrome://newtab` and other internal pages.
2. **The JS assignment must be its own tool call — not chained in the same batch as the preceding navigate.** The navigate step's load signal can race the JS-eval; CDP rejects with "Can't interact with browser internal pages" even though the tab eventually settles. Two separate calls.
3. From that tab, evaluate `window.location.href = 'chrome-extension://<id>/pair.html?token=<token>'`. The assignment returns normally. Any *subsequent* read/exec on the same tab errors with "Can't interact with browser internal pages" — that error is the success signal.
4. The tab is now stranded on a chrome-extension page. Close it or navigate back. Use a different tab for `ensure_toolbar` / `capture_now`.
5. Poll `html_capture(action='status', pairing_token=...)` — `extension_last_seen_at` non-null and within ~10s ⇒ paired.

### Tier 2 — User pastes the deep link

When Tier 1 fails (no browser-automation MCP, automation Chrome lacks the extension, CSP intercepted), surface `deep_link_url` as a fenced code block and ask the user to paste it into Chrome's address bar.

### Tier 3 — User pastes the token

Last resort: ask the user to open the Reprise Builder: HTML popup and paste `pairing_token` into the "Pair with agent" field.

Never send the `chrome-extension://` URL as a clickable `<a href>` — Chrome blocks those clicks.

## Confirming pairing

`html_capture(action='status', pairing_token=...)` is the source of truth. `extension_last_seen_at` non-null and within ~10s of now ⇒ paired. Stays null past ~30s ⇒ re-share the deep link or have the user paste the token. If still no, ask them to check `chrome://extensions/` and confirm the extension is installed and the popup doesn't show a stale token (Unpair first).

Don't proceed past `ensure_toolbar` until status confirms — commands to an unpaired session silently no-op.

## Page-settle before each capture

A capture of an unsettled page produces an unstyled / half-rendered screen with **no error signal**. JS-heavy pages (Webflow, Framer, SaaS dashboards) load stylesheets after `DOMContentLoaded`.

Between navigation and `capture_now`:

- `page.waitForLoadState('networkidle')` — most reliable.
- `browser_wait_for({text: '<sentinel hero text>'})` if you know an above-the-fold element by content.
- 800–1500ms blanket sleep — works on marketing pages, brittle on slow networks.

Re-apply after every cross-document navigation.

## Verify the page is what you intended

`capture_now` happily snapshots a 404 page, a soft-redirect, or a "accept cookies" interstitial — none fail your call. Before capturing:

- Confirm the post-navigation URL matches expectation.
- Confirm page title isn't a 404 / "Page not found" / similar.
- Confirm a sentinel element from the intended content is present.

When picking URLs by convention ("this site probably has /about and /projects"), browse from the front page to discover real URLs first; don't guess.

### CTA → unexpected overlay → dismiss before capture

Clicks that advance a funnel (booking, signup, pricing) often pop a date picker, autocomplete, signup wall, or cookie banner over the destination. URL and title check out, but `capture_now` serializes whatever is on top, not the screen the customer reaches. After any such click, screenshot to verify; if an overlay is sitting on top, dismiss it (`Escape`, outside-click, close-button selector) before firing `capture_now`.

## `capture_now` response branching

Synchronous by default. Returns `{ok, observed_capture, timed_out, screen_count_before, screen_count_after, flush_complete, latest_snapshot_id, ...}`.

- `timed_out: true, observed_capture: false` → extension didn't pick up the command. Re-check `extension_last_seen_at`.
- `observed_capture: false, timed_out: false`, hint `"no capture in flight"` → nothing was queued. Safe to proceed.

## Preview before publish

`flush_complete=true` means every screen is processed. **Always wait for `flush_complete=true` before opening `preview_url`** — otherwise the preview shows placeholders for screens still being processed.

## After every session

`session_recap` + `report_friction` per distinct issue — see the `reprise-session-close` skill.

Theming guides, re-skinning to a new vertical, composing replays from screens of other replays, and the full `--rguide-*` CSS token table all live in `docs(slug='html')`. For editing existing replays (not capturing new ones), use the `reprise-html-edit` skill.
