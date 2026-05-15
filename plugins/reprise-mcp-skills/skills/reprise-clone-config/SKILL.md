---
name: reprise-clone-config
description: Reprise Clone (formerly Replicate) configuration workflow. MUST be invoked before any action on a Reprise clone — including writing or editing RB snippets, diagnosing white screens / blank panels / stale data on a clone preview, routing API endpoints, or walking a capture path. Triggers on `clone_id`, `RB snippet`, `clone_api`, `replay_backend`, any `clone_*` MCP tool, `/sw-setup/`, or any clone debug/preview URL. Usually invoked by the `reprise-mcp` router; can also auto-fire on direct keyword matches. Body has the phase workflow, the two forcing functions before any snippet, and the RB-snippet gotcha catalog.
version: 0.1.0
---

# Reprise Clone Configuration

A Reprise Clone is a captured, interactive replica of a web app, served by a service worker from cached network responses. Configuring a clone means making the captured content work correctly when served from the clone — fixing request matching, API routing, telemetry noise, stale dates, and other capture-time-vs-serve-time mismatches.

## Two forcing functions — do these BEFORE writing any snippet

Skip either and you will write broken snippets.

1. **`docs(slug='clone-api')`** — the authoritative method reference for the `clone_api` global (also exposed as `replay_backend`). A snippet that calls a non-existent method (e.g. `match_url()`, `match_prefix()`) throws at init and crashes the whole service worker, taking the clone down. Don't guess method names; read the reference.
2. **`search_patterns(product='clone', symptom='...')`** — phrase the symptom as the behavior you're seeing, not the fix you think you need. Returns top-k matching patterns by BM25. Treat hits with `score < ~0.5` as weak guesses.

## Workflow

### Phase 1 — Assess

1. **`clone(clone_id)`** — read notes, existing snippets, and capture sessions left by prior agents.
2. **`clone_request(action='analyze')`** — domain breakdown, telemetry count, API routing candidates, version drift.
3. **`clone_path(clone_id=..., session_uuid=..., include_health=True)`** — cross-references path steps with content health. Surfaces `has_content` / `empty` / `not_captured`.
4. **`clone_request(action='check_duplicates')`** — for any `empty` step, find a non-empty duplicate and plan a `match_id` snippet pointing at the `recommended_id`.

### Phase 1.5 — Offline test (critical, do not skip)

Before spending time on URL rewriting or domain interception, verify the SW actually serves cached assets. Ask the user to open the preview URL with devtools Network tab open and "Offline" throttling enabled:

- Page still renders offline → SW is serving from cache. Focus on API routing; skip URL rewriting.
- Page goes blank offline → call `clone_request(action='check_health')` for 0-byte or missing content.

A 404 in the Network panel does **not** mean the file wasn't served — the CDN plugin rewrites requests internally and the panel shows the original URL. Always verify with the offline check.

### Phase 2 — Baseline snippets

Create these before any browser testing:

- **Telemetry block** — `clone_api.match_domain('<vendor>', true).block()` for google-analytics, googletagmanager, segment, mixpanel, amplitude, heap, fullstory, sentry, datadog, newrelic, pendo, intercom, hotjar, optimizely, launchdarkly. Always needed; reduces error noise dramatically.
- **Original SW block** — handle `/sw.js` to return an empty JS blob so the captured site's own SW doesn't fight the Reprise one.
- **`match_id` snippets** for any duplicates found in Phase 1.
- **API routing snippets** for endpoints flagged by `clone_request(action='analyze')`.

### Phase 3 — Walk the path with the user

Always start a debug session even if the page renders fine — debug captures JS errors and failed requests invisible to the agent.

1. `clone_debug(action='start')` → debug URL + preview URL.
2. User opens debug URL (registers SW), then navigates to preview URL.
3. Walk each step in the captured path one at a time. After each step: `clone_debug(action='errors')`, then `clone_debug(action='errors', clear=True)` before the next step.
4. If a snippet route looks right but isn't firing: `clone_debug(action='routes')` shows every registered route + the last 50 SW fetch decisions (which route matched each URL).
5. When a step fails: diagnose, write a snippet, then re-register the SW. Every `clone_snippet(action='create'|'update')` response gives two apply paths:
   - **Fast path** — evaluate `navigator.serviceWorker.controller?.postMessage({type:'reprise:reload-sw'})` in the preview tab via your browser MCP. SW reloads its clients in place; no `/sw-setup/` round-trip.
   - **Fallback** — navigate to `apply_url` (`/sw-setup/`), optionally with `?next=<current-path>` so the redirect lands back on the page you were iterating on.

### Phase 3.5 — Verify data completeness

A rendered page is not a working page. Verify:

- Rendered row count matches expected (a "25 per page" table with only 12 rows means API responses are missing).
- No placeholder / skeleton cells, no `--` or red error indicators.
- Console error count is near-zero — a page that "looks fine" with 30+ console errors has hidden problems.
- Data-dependent interactions work (sort, filter, pagination all depend on API routing).

### Phase 4 — Record

- `clone_note(action='create', category='status')` — green/blocked steps, snippets created.
- `clone_note(action='create', category='fix')` — non-obvious fixes future agents won't re-derive.
- New reusable fix? `patterns(action='append', product='clone')`.

## Top 5 RB-snippet gotchas

1. **One broken snippet crashes ALL routes.** Any RB snippet that throws at init breaks every other snippet's routes. Errors only surface in the browser's SW devtools console, not in MCP debug output. If everything suddenly stops working, check the most recently created/updated snippet for a syntax or runtime error first.
2. **`handle()` always fires when the route matches — there is no cache-miss fallthrough.** Don't chain `.handle()` expecting it to stub only uncaptured operations; it will hijack every matching request. To target one operation, scope the route with a `request_filter` callback on the chain opener. To modify a cached response, use `post_process()`.
3. **Path routes are same-origin only.** `clone_api.get("/api/data")` only matches requests to the clone's own domain. For cross-origin APIs use `clone_api.match_domain("api.example.com")`.
4. **Route matching ignores query strings and bodies.** Chain openers decide by `url.host` / `url.pathname` / method only. `clone_api.get("/main.js?v=:v")` never matches anything. Branch on query/body via the `request_filter` callback.
5. **Snippet changes require SW reload.** A `clone_snippet(create|update)` doesn't take effect until the SW re-registers. Use the `reload_via_sw_message` one-liner (fast path) or navigate to `apply_url` (fallback).

The full gotcha catalog (decode_path, payload_key_match, post_process contract, iframe adoption, etc.) lives in `docs(slug='clone')`. The complete `clone_api` method reference is `docs(slug='clone-api')`.

## End of session

Always file `session_recap` once and `report_friction` once per distinct issue. The `reprise-session-close` skill has the call shapes.
