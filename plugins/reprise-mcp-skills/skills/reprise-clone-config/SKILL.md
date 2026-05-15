---
name: reprise-clone-config
description: Reprise Clone (formerly Replicate) configuration workflow. MUST be invoked before any action on a Reprise clone — including writing or editing RB snippets, diagnosing white screens / blank panels / stale data on a clone preview, routing API endpoints, or walking a capture path. Triggers on `clone_id`, `RB snippet`, `clone_api`, `replay_backend`, any `clone_*` MCP tool, `/sw-setup/`, or any clone debug/preview URL. Usually invoked by the `reprise-mcp` router; can also auto-fire on direct keyword matches. Body has the phase workflow, the two forcing functions before any snippet, RB vs C snippet distinction, two-stage routing model, hallucination warnings, baseline snippets, state management, and the gotcha catalog.
version: 0.2.0
---

# Reprise Clone Configuration

A Reprise Clone is a captured, interactive replica of a web app, served by a service worker (SW) from cached network responses. Configuring a clone means making the captured content work correctly when served from the clone — fixing request matching, API routing, telemetry noise, stale dates, and other capture-time-vs-serve-time mismatches.

The MCP doesn't drive a browser; it reads + mutates the Reprise-side clone object that the **Reprise Builder: Clone** Chrome extension wrote to during capture, plus inspects the SW's runtime behavior via the debug surface.

## Terminology — get these straight before writing snippets

- **RB snippet** (Replicate Backend / Clone Backend) — JavaScript that runs **inside the service worker**, BEFORE the response leaves the SW. Use for request matching, routing, and response transformation. This is the right tool for almost every clone-fix.
- **C snippet** (Custom JS) — JavaScript that runs **in the page context**, AFTER the SW has served. DOM patches, UI tweaks, monkey-patching `fetch` on the page side. **Cannot fix SW-level request issues** — `window.fetch` overrides in C don't intercept what the SW handles first.
- **`clone_api`** — the SW JavaScript global available inside RB snippets. Also exposed as **`replay_backend`** (its original name, kept permanently). Both names point to the same instance. **Don't rewrite `replay_backend.*` to `clone_api.*` in existing snippets** as part of an unrelated edit — the snippet is correct as-is.
- **Capture session** — a recording of user interactions during clone capture. **Path** — the sequence of steps (navigate, click, input) inside a capture session.

## Two forcing functions — do these BEFORE writing any snippet

Skip either and you will write broken snippets.

1. **`docs(slug='clone-api')`** — the authoritative method reference for `clone_api`. **Never guess a method name.** Calling a non-existent method (e.g. `match_url()`, `match_prefix()`, `match_host()`, `match_path()`, `route()`, `when()`, `clone_api.fetch()`, `clone_api.request()`) throws at snippet-init and **crashes the entire service worker, taking the clone down**. If the reference doesn't list what you need, fall back to `match_domain().pre_process()` / `.handle()` and build the behaviour explicitly.
2. **`search_patterns(product='clone', symptom='...')`** — phrase the symptom as the behavior you're seeing, not the fix you think you need. Good phrasings: *"fetch requests bypass my wrapper after DataDog loads"*, *"service worker returns 404 on uncaptured GraphQL queries"*, *"clone redirects to the live app instead of serving from cache"*. Returns top-k matches by semantic similarity; treat hits with `score < ~0.5` as weak guesses.

## Workflow

### Phase 1 — Assess

1. **`clone(clone_id)`** — read notes, existing snippets, and capture sessions left by prior agents.
2. **`clone_request(action='analyze')`** — domain breakdown, telemetry count, API routing candidates, version drift.
3. **`clone_path(clone_id=..., session_uuid=..., include_health=True)`** — cross-references path steps with content health. Surfaces `has_content` / `empty` / `not_captured`. Use this instead of calling `check_health` separately.
4. **`clone_request(action='check_duplicates')`** — for any `empty` step, find a non-empty duplicate and plan a `match_id` snippet pointing at the `recommended_id` (the hashid string, not a numeric pk).

### Phase 1.5 — Offline test (CRITICAL, do not skip)

Before spending time on URL rewriting or domain interception, verify the SW actually serves cached assets. Ask the user to open the preview URL with devtools Network tab open and "Offline" throttling enabled:

- Page still renders offline → SW is serving from cache. Focus on API routing; skip URL rewriting.
- Page goes blank offline → call `clone_request(action='check_health')` for 0-byte or missing content.

A 404 in the Network panel does **not** mean the file wasn't served — the CDN plugin rewrites requests internally and the panel shows the original URL. Always verify with the offline check.

### Phase 2 — Baseline snippets

Create these before any browser testing. They're standard for every clone.

**Block telemetry** (always; reduces error noise):

```javascript
// Analytics & tracking
clone_api.match_domain("google-analytics", true).block();
clone_api.match_domain("googletagmanager", true).block();
clone_api.match_domain("segment", true).block();
clone_api.match_domain("mixpanel", true).block();
clone_api.match_domain("amplitude", true).block();
clone_api.match_domain("heap", true).block();
clone_api.match_domain("fullstory", true).block();
// Error tracking
clone_api.match_domain("sentry", true).block();
clone_api.match_domain("datadog", true).block();
clone_api.match_domain("newrelic", true).block();
// User engagement
clone_api.match_domain("pendo", true).block();
clone_api.match_domain("intercom", true).block();
clone_api.match_domain("hotjar", true).block();
// A/B testing
clone_api.match_domain("optimizely", true).block();
clone_api.match_domain("launchdarkly", true).block();
```

**Block original service worker** (prevents SW conflicts):

```javascript
clone_api.get("/sw.js").handle(async () => {
  return new Blob([""], { type: "application/javascript" });
});
```

**`match_id` snippets** for any duplicates found in Phase 1. Pass the **hashid string** (e.g. `"lnwjy9j"` from `clone_request(action='search')` / `check_duplicates`); a numeric pk will not match.

**API routing snippets** for endpoints flagged by `clone_request(action='analyze')`.

### Phase 3 — Walk the path with the user

Always start a debug session even if the page renders fine — debug captures JS errors and failed network requests invisible to the agent.

1. `clone_debug(action='start')` → debug URL + preview URL.
2. User opens debug URL (registers SW), then navigates to preview URL.
3. Walk each path step one at a time. After each step: `clone_debug(action='errors')`, then `clone_debug(action='errors', clear=True)` before the next step.
4. If a snippet route looks right but isn't firing: `clone_debug(action='routes')` shows every registered route + the last 50 SW fetch decisions. A route that doesn't appear as a `matched.label` for the URL you expected is the pattern-mismatch signal — typical causes: case (`/Report/` vs `/report/`), stale path segment, or a same-origin `get()` used for a cross-origin URL.
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
- `patterns(action='append', product='clone', scope='client')` — for a reusable client-specific technique.

## Two-stage routing model — internalize this

Every intercepted request goes through **two independent stages**. Confusing them is the #1 source of "I added `no_hash_check()` but it didn't help" friction.

| Stage | What it decides | What it inspects | Tuned by |
|---|---|---|---|
| **1. Route match** | Does this snippet's chain run for this live request? | `url.host` (for `match_domain`), `url.pathname` (for `get`/`post`/etc.), HTTP method, optional `request_filter`. **Query string and request body are NOT inspected.** | Rewrite chain opener, or scope with `request_filter` arg on the opener |
| **2. Capture lookup** | Once a chain matches, which captured response does the SW serve? | Full target URL (path + query) + body hash + method + (optionally) timestamp | `no_hash_check()`, `payload_key_match()`, `match_id()`, `fallback_path_match()`, `check_timestamps()` |

A route written as `clone_api.get("/main.js?v=:v")` **never matches anything** — the matcher ignores the query string. To branch on a query value, pass `(request) => …` as the second arg to the chain opener. `no_hash_check()` doesn't change which routes match — it tunes the stage-2 capture lookup after the route already matched.

If you find yourself reaching for `no_hash_check()` to fix a route that "isn't matching," you almost certainly want stage-1 plumbing instead.

## Top RB-snippet gotchas

Each is a silent failure mode that costs real session time.

1. **One broken snippet crashes ALL routes.** Any RB snippet that throws at init breaks every other snippet's routes. Errors only surface in the browser's SW devtools console, not MCP debug output. If everything suddenly stops working, check the most recently created/updated snippet for a syntax or runtime error first.
2. **`handle()` always fires when the route matches — no cache-miss fallthrough.** Don't chain `.handle()` expecting it to stub only uncaptured operations; it hijacks every matching request. To target one operation, scope with a `request_filter` callback on the chain opener. To modify a cached response, use `post_process()`.
3. **Path routes are same-origin only.** `clone_api.get("/api/data")` only matches the clone's own domain. For cross-origin APIs use `clone_api.match_domain("api.example.com")`. Passing a full URL like `clone_api.get("https://cdn.example.com/...")` throws `TypeError` synchronously at registration.
4. **`:param` matches ONE path segment only.** `/contacts/:rest` matches `/contacts/foo` but NOT `/contacts/foo/bar`. For deep paths use `new RegExp("^/contacts/")`. Bare `(.*)` in path strings does NOT work — path-to-regexp v6 requires named parameters for regex groups.
5. **Snippet changes require SW reload.** A `clone_snippet(create|update)` doesn't take effect until the SW re-registers. Use the `reload_via_sw_message` one-liner (fast path) or navigate to `apply_url` (fallback).
6. **`pre_process` cannot bail out of a route — scope at match time with `filter`.** Returning the original request from `pre_process` does NOT skip the route; the SW still serves the cached response. To intercept some requests but not others on the same path, pass a `filter` to the chain opener.
7. **GraphQL `handle()` on `/graphql` breaks all cached queries.** The shared endpoint amplifies the always-fires contract. Use a POST body filter to target specific operations:
   ```javascript
   clone_api.post("/graphql", (req, body) => {
     return JSON.parse(body).operationName === "UncapturedQuery";
   }).handle(async () => JSON.stringify({ data: { /* stub */ } }));
   ```
8. **JSON-RPC-style endpoints — `payload_key_match` is a header modifier, NOT a route scoper.** On `/customer/list` invoked with many `meth` values, `payload_key_match('meth')` tells the backend hasher to match on the `meth` sub-key but does NOT scope the chain to a single value. Two chains opened with `payload_key_match('meth')` both fire on every request; the last-registered wins. Scope per-value with the chain-opener's `request_filter` arg.
9. **Double-encoded path segments — use `decode_path()`, not per-endpoint `pre_process`.** When the live URL is `%2540` (double-encoded `@`) but the capture stored the literal `@`, the Django matcher tolerates one `unquote` pass — no more. `clone_api.get("/users/:handle").decode_path(2).pass()` handles it built-in.
10. **`handle()` must return string or Blob — never `new Response()`.** Returning `new Response(null, {status: 302})` stringifies to `[object Response]` on the page. For redirects use `.block({status: 302, headers: {"Location": url}})`.
11. **`post_process(cb)` receives `(response, request)` — read per-request data from the second arg, not `clone_api.state`.** `state` is a single shared object across every in-flight request; concurrent fetches clobber each other before `post_process` reads.
12. **Modify body, keep headers — return the body directly; don't wrap in `Response`.** Returning `Blob`/`string` from `post_process` is auto-rewrapped with the original status + headers. Build a `Response` only to override headers. `new Response(stringBody)` auto-sets `Content-Type: text/plain` — when the body type differs from the original, set `Content-Type` explicitly.

## State management — `state` vs `storage`

Two mechanisms for persisting state across requests inside RB snippets:

- **`clone_api.state`** — plain JS object, synchronous, **lost when the SW restarts** (page hard-refresh, `/sw-setup/`). Use for: dedup flags within a session, conditional responses based on prior requests in the same SW lifetime. **Never use to forward per-request data between `pre_process` and `post_process`** — concurrent requests race.
- **`clone_api.storage`** — IndexedDB-backed key/value, async, persists across SW restarts and page reloads. `set(key, val)`, `get(key)`, `remove(key)`, `clear()`. Use for: cross-session counters, mock data that must survive navigations, async-task polling state (serve QUEUED first, then SUCCESS).

Quick chooser: persist across page refresh → `storage`. Synchronous flag for "have I already handled this?" → `state`. POST → GET sequencing for mock flow → `state` for simple, `storage` for multi-page.

## Common hallucinations to avoid

These names have been hallucinated in real sessions and **do not exist** on `clone_api`. Calling any of them crashes the SW at init.

- `match_url()`, `match_prefix()`, `match_host()`, `match_path()`, `route()`, `when()`
- `clone_api.fetch()`, `clone_api.request()`

If a method you want isn't in `docs(slug='clone-api')`, don't invent it. Fall back to `match_domain().pre_process()` or `.handle()` and build the behavior explicitly.

## Visual symptoms — quick diagnosis

| Symptom | Likely cause |
|---|---|
| White / blank screen | HTML page not captured or 0-byte → `clone_path(include_health=True)` |
| Spinner that never stops | API call hanging (not matched by SW) |
| Page shell renders, content blank | SPA API routing wrong → `clone_request(action='analyze')` |
| `[object Response]` on page | `handle()` returning `new Response()` — return string/Blob, or use `block()` |
| Raw JSON text on page | Redirect or API response rendered as HTML; missing `default_page` for SPA shell |
| Filter / dropdown doesn't update table | Filtered API call not matched — add `Custom-Details-Filter` via `pre_process`, or scope with `request_filter` |
| Stale dates | Content frozen at capture time — needs date-refresh snippet |

**Diagnosis order** to minimize wasted effort: offline test → content health → API routing → telemetry noise → auth/JWT → request matching → UI/display.

## C snippet timing slot

C snippets default to running just before the first captured `<script>` in `<head>`. For patches that need to happen **before any captured-page script runs** (monkey-patching `fetch`, `XMLHttpRequest`, `Node.prototype.appendChild`, etc.), pass `timing='head'`:

```python
clone_snippet(action='create', clone_id=..., title='patch fetch',
              code="window.fetch = (...args) => { /* ... */ };",
              timing='head')
```

`timing='head'` snippets run before `document.body` exists. Use `MutationObserver` or `DOMContentLoaded` if you need DOM access. `head` and `load` are CUSTOM_JS-only — ignored for `backend` (RB) and `websocket` snippets.

## End of session

Always file `session_recap` once and `report_friction` once per distinct issue. The `reprise-session-close` skill has the call shapes.

The full gotcha catalog (third-party iframes, AST rewrite, `default_page` / `default_handler`, `decode_path` details, POST header patterns, complex DOM-snapshot handling, RB-runs-once-at-SW-boot semantics, etc.) lives in `docs(slug='clone')`. The full `clone_api` method reference is `docs(slug='clone-api')`.
