---
name: reprise-tour-capture
description: "Reprise Product Tour capture workflow — recording NEW tours via the Reprise Builder HTML Chrome extension. MUST be invoked before any `tour_capture` call. Triggers on `tour_capture`, `pairing_token`, `deep_link_url`, `capture_now`, a brand-new `draft_id`, or the Reprise Builder HTML extension. Usually invoked by the `reprise-mcp` router; can also auto-fire on direct keyword matches. Body has the 7-step flow at headline depth, the 3-tier extension-pairing escalation, the load-bearing page-settle / verify-the-page principles, and operational gotchas around browser-automation MCPs and proxy timeouts. Per-tier mechanics, `capture_now` envelope, `set_auto`, post-stop cleanup, install URLs, theming, re-skinning, and composing all live in `docs(slug='tour')`. NOT for editing existing tours — use `reprise-tour-edit`."
version: 0.3.2
---

# Reprise Product Tour Capture

A Product Tour is a linear walkthrough built from captured DOM snapshots. Capture happens in the user's own Chrome via the **Reprise Builder: HTML** extension — the agent doesn't snapshot pages itself; it orchestrates.

## Preload tool schemas before starting

If your client uses deferred tool schemas (Claude Cowork, etc.), preload everything you'll need in **one** `ToolSearch` call before step 1 — discovering tools mid-flow forces back-to-back schema loads that fragment the workflow. The full set for a capture session:

- Reprise MCP: `tour`, `tour_capture`, `tour_guide`, `tour_lifecycle`, `tour_screen`, `docs` (for install-URL fallback), `whoami` (only if you have multiple Reprise MCPs connected and need to pick by host)
- Browser-automation MCP (Claude in Chrome, Playwright, etc.): `list_connected_browsers`, `select_browser`, `tabs_context_mcp`, `navigate`, `javascript_tool` (and your tool inspector / `get_page_text` equivalent for verifying iframe presence)
- Task tracking: `TaskCreate`, `TaskUpdate` (if your client has them and the flow is ≥3 steps)

## The capture flow

1. `tour(action='create', title=...)` → `{draft_id, preview_url}`.
2. `tour_capture(action='start', draft_id=...)` → `{pairing_token, deep_link_url, ...}`.
3. **Pair the extension** (3-tier escalation, see below — highest failure point).
4. `tour_capture(action='status', pairing_token=...)` until `extension_last_seen_at` is fresh (<10s) ⇒ paired.
5. `tour_capture(action='ensure_toolbar', pairing_token=..., target_url='https://...')`. **On a browser-automation MCP, follow this with an explicit `navigate(tabId, target_url)` and confirm `#reprise_iframe` is in the DOM before continuing — see "Browser-automation MCPs" below.**
6. Per-screen loop: browser MCP navigate / click → wait for settle → `tour_capture(action='capture_now', pairing_token=..., title='Step N', target_url='...', wait=False)` → poll `tour_capture(action='status', pairing_token=...)` and **verify `screen_count` actually incremented**. `capture_now` returns `ok=true` even when no tab matches `target_url` — the SW accepted the command, but couldn't dispatch it; nothing was captured. `screen_count` unchanged + `extension_last_seen_at` still fresh = re-issue `ensure_toolbar` + an explicit `navigate(tabId, target_url)` to re-mount the toolbar, then retry. **Default to `wait=False`.** The default `wait=True` sync window exceeds the ~25-30s cap on the Anthropic MCP proxy and Claude Cowork — the underlying capture completes but the response gets killed mid-flight, which looks like a failure when it isn't.
7. `tour_capture(action='stop', pairing_token=...)`, then `tour_lifecycle(action='publish', draft_id=...)`.

**Always pass `target_url` to `ensure_toolbar` and `capture_now`.** Without it the SW falls back to "active tab in the last-focused Chrome window" and silently picks the wrong tab in multi-window or background-agent setups.

## Pairing the extension — 3-tier escalation

The `deep_link_url` is a `chrome-extension://...` URL. Browser-automation MCPs can't `navigate` to those (CDP rejects them as browser-internal). Try in order, escalate on failure:

- **Tier 1 — JS-assignment from a real-page tab** (fully automated). From a tab on a real `https://` page, evaluate `window.location.href = '<deep_link_url>'` as **its own tool call** (don't chain with a preceding navigate — they race). If your browser-MCP nudges you toward `browser_batch` to combine calls, ignore it for the pairing flow — the JS-assign must be its own call. **Authoritative pairing signal:** poll `tour_capture(action='status', pairing_token=...)` for ~10s after the assignment; `extension_last_seen_at` non-null and fresh ⇒ paired. The JS step itself returns nothing informative — a stranded-tab error after the assign is a positive secondary signal *when it appears*, but absence ≠ failure. **Fast diagnostic for extension-not-installed:** if your browser MCP's tab inspector shows the JS-assigned tab URL as `chrome-extension://invalid/`, the extension isn't installed — skip the retry loop and go straight to `AskUserQuestion` + the Chrome Web Store install URL from `docs(slug='tour')`. Otherwise, if status stays null past ~10s, ask the user whether the extension is installed before retrying.
- **Tier 2 — User pastes the deep link** into Chrome's address bar.
- **Tier 3 — User pastes the token** into the popup's "Pair with agent" field.

Never send the `chrome-extension://` URL as a clickable `<a href>` — Chrome blocks the click. Full per-tier mechanics, race-condition details, and stranded-tab recovery in `docs(slug='tour')`.

Don't proceed past `ensure_toolbar` until status confirms pairing — commands to an unpaired session silently no-op. If the user doesn't have the extension installed, send them to the Chrome Web Store URL in `docs(slug='tour')`.

## Browser-automation MCPs (Claude in Chrome, Playwright, etc.)

The extension's `ensure_toolbar` step allow-lists the host and reloads the tab so the content script injects. **The reload step does not reach tabs in Chrome's MCP tab group** — those tabs live in a separate group ID from the user's normal tabs, and the extension's tab-matcher misses them. On a browser-automation MCP, after `ensure_toolbar` returns `ok:true`:

1. Issue an explicit `navigate(tabId, target_url)` on the same tab. Same URL is fine — what matters is the navigation event.
2. After settle, probe the DOM for `#reprise_iframe`. Its presence is the *only* signal that the content script loaded.
3. If absent, the toolbar didn't mount — retry the navigate, or confirm `tour_capture(action='status')` still shows the extension as fresh.

This only matters for the first `capture_now` in the session. Subsequent same-origin navigations keep the iframe mounted.

## Two pre-capture principles that prevent the most common silent failures

1. **Page-settle before each `capture_now`.** A capture of an unsettled page produces an unstyled / half-rendered screen with **no error signal**. Between navigation and `capture_now`: `page.waitForLoadState('networkidle')`, or `browser_wait_for` on a sentinel hero-text element, or an 800–1500ms blanket sleep. Re-apply after every cross-document navigation.
2. **Verify the page is what you intended.** `capture_now` happily snapshots a 404, a soft-redirect, a "you must accept cookies" interstitial, or a CTA-triggered overlay — none fail your call. Before capturing: confirm URL, confirm title isn't a 404, confirm a sentinel element from the intended content is present. After CTA clicks (booking, signup, pricing), screenshot to verify; if an overlay landed on top, dismiss it before `capture_now`.

## Pattern search before custom JS or diagnosis

Before writing custom JS for a screen or diagnosing a broken tour, call `search_patterns(symptom='...', product='tour')`. Treat `score < ~0.5` as weak guesses — the tour pattern corpus is thinner than clone's.

## Preview before publish

`capture_now` returns `flush_complete=true` when every screen is processed. **Always wait for `flush_complete=true` before opening `preview_url`** — otherwise the preview shows placeholders for screens still being processed.

## Adding guides during capture

When the user asks for guides on the captured screens, **default to tethered (pointing) guides, not floating**. A floating guide is a card hovering in the middle of the screen with no anchor — it works for intro/outro/banner messages but it's the wrong default for "explain this feature." Pointing guides attach to a real DOM node and visually call out what they describe.

Per-screen flow:

1. `tour_guide(action='list_nodes', snapshot_id=<from tour_screen(action='list')>)` — returns up to 200 anchorable candidates (interactive elements, headings, aria-labeled nodes) with `tag` / `text` / `role` / `aria_label`. Pick the node whose `text` or `aria_label` matches the feature the guide describes.
2. `tour_guide(action='create', draft_id=..., snapshot_id=..., guide_type='tethered', target_node_id=<from step 1>, text='<h2>Title</h2><p>Body</p>', buttons='[{"label":"Next","action":"flow_next_screen"}]')`. The `position` field auto-defaults sensibly (the server walks ancestors for header/nav and places the popup below if found, above otherwise) — only override when the default clips off-viewport.

Only fall back to `guide_type='floating'` when there's genuinely no good anchor: a welcome screen, an outro CTA on an empty/transition screen, or a callout that's about the whole screen rather than one element. In that case, pass only the floating-compatible fields — `position` / `target_node_id` / `element_click` / `caret_enabled=True` / `pulse_enabled=True` are rejected at the MCP boundary with `floating_with_<field>` errors.

End-of-tour CTAs: the last guide on the last screen should use `restart_demo` or `goto_url` rather than `flow_next` / `flow_next_screen` (those silently no-op when there's nothing to advance to).

## Transient errors

Reprise MCP calls occasionally return `Session terminated` from the transport layer (not the server). **Retry once before assuming the session is dead** — most return cleanly on the second attempt.

## After every session

`session_recap` + `report_friction` per distinct issue — see `reprise-session-close`.

`capture_now` envelope fields, async / fire-and-forget mode, `set_auto` continuous capture, post-stop toolbar cleanup, install URLs, theming via `--rguide-*`, re-skinning, composing tours from screens of other tours — all in `docs(slug='tour')`. For editing existing tours (not capturing new ones), use the `reprise-tour-edit` skill. If you have a `/launch/<slug>/` URL or hit a `wrong_id_kind` error, see `reprise-tour-id-model`.
