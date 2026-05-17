---
name: reprise-tour-edit
description: Reprise Product Tour editing workflow — modifying an existing tour. MUST be invoked before any edit action on a published or draft tour. Triggers on `tour_edit`, `tour_screen(action='copy_to')`, `tour_guide`, `tour_variable`, `tour_link`, `set_custom`, or any request to modify / translate / rebrand an existing tour. Usually invoked by the `reprise-mcp` router; can also auto-fire on direct keyword matches. Body has the tool quick-reference, the re-skin vs translate distinction, the compose-from-screens flow, variables + links basics, and the `guide_css` surface. The full `--rguide-*` token list, complete theming reference, and full compose caveats live in `docs(slug='tour')`. Distinct from `reprise-tour-capture` (recording new tours) and `reprise-tour-id-model` (ID-kind resolution).
version: 0.3.2
---

# Reprise Product Tour Editing

Editing an existing tour covers text / attribute / image edits, translation, guide management, variables, re-skinning to a new vertical, and stitching new tours together from screens of existing ones. ID kinds (draft vs published vs launch-link) are the most common source of confusion — see `reprise-tour-id-model` if a paste of `/launch/<slug>/` or a `wrong_id_kind` error is what brought you here.

## Inspecting and editing — tool quick-reference

| Action | Tool |
|---|---|
| Details + screens list | `tour(draft_id=...)` / `tour(published_id=...)` |
| Screen content | `tour_screen(action='list'\|'get'\|'search', ...)` (list returns per-screen `preview_url` for v4 tours — a deep link to a specific screen) |
| Enumerate captured-screen text | `tour_edit(action='list_text_nodes', ...)` — filter with `text_contains` / `path_contains` / `screen_node_id` |
| Edit text | `tour_edit(action='edit_text', edits='[{...}]')` — one call = one undo step; `dry_run=True` previews |
| Edit element attributes (`href`, `alt`, `class`, `aria-*`, `data-*`) | `tour_edit(action='list_attribute_nodes'\|'edit_attributes', ...)` |
| Swap an image | `tour_edit(action='replace_image', src=<URL or resource:id>)` |
| Bulk translate visible text (meaning-preserving) | `tour_edit(action='translate', ...)` |
| Compose new tour from existing screens | `tour_screen(action='copy_to'\|'copy_status', ...)` |
| In-screen guides | `tour_guide(action='list_nodes'\|'create'\|'generate'\|'list'\|'get'\|'update'\|'delete', ...)` — **default to `guide_type='tethered'`** with a `target_node_id` from `list_nodes`. Floating is the fallback for screens with no good anchor, not the easy path. |
| Variables | `tour_variable(action='list'\|'get'\|'references'\|'create'\|'update'\|'delete'\|'insert', ...)` |
| Per-tour custom CSS / JS injection | `tour_lifecycle(action='get_custom'\|'set_custom', kind='guide_css'\|..., ...)` |
| Lifecycle (publish, duplicate, archive, restore, update_title, etc.) | `tour_lifecycle(action='...', ...)` |
| Shareable published links | `tour_link(action='create', published_id=..., variables=...)` |

`set_custom` returns metadata only (bytes, sha256, last_saved, unchanged, preview_url) so iterative edit loops don't flood context.

## Pattern search before custom JS or diagnosis

Before writing custom JS for a screen or diagnosing a broken tour, call `search_patterns(symptom='...', product='tour')`. Treat `score < ~0.5` as weak guesses — the tour pattern corpus is thinner than clone's.

## Re-skin to a new vertical / persona / brand

Use when the user wants to take a template tour and rewrite the visible product story without re-capturing. The story lives in **both** the captured DOM and the in-screen guides — touch both.

1. **Duplicate first.** `tour_lifecycle(action='duplicate', draft_id=<template>)`. Never edit the template in place.
2. **Rewrite the captured DOM, screen by screen:**
   - `tour_edit(action='list_text_nodes', draft_id=<new>, screen_node_id=<scrn-...>, text_contains=<source-term>)` — filter aggressively, surface `node_id`s.
   - `tour_edit(action='edit_text', draft_id=<new>, edits='[{...}]')` — all edits for a screen as one transformation. `dry_run=True` previews first.
   - Attributes: `list_attribute_nodes` → `edit_attributes`. Images: `replace_image` (to a URL or `resource:<resource_id>`).
3. **Rewrite the guides.** `tour_guide(action='list')` → `action='get'` → `action='update'` per guide.
4. **(Optional) polish + publish.** `tour_lifecycle(action='publish')` + `tour_link(action='create')`.

### `translate` is NOT a re-skin channel

`tour_edit(action='translate')` is honest meaning-preserving translation across languages. Same-language `translate` ignores "remap to <new vertical>" instructions. To re-skin within one language, use the per-screen text/attribute editors.

## Compose a new tour from existing screens

Use when the user wants a tour stitched together from screens that already exist in other tours.

1. **Find candidates.** `tour_search(query=..., search_scope='screens')` returns matched tours with screen IDs nested per tour. Finer control: `tour_screen(action='search', draft_id=..., query=...)`.
2. **Create the destination.** `tour(action='create', title='...')`.
3. **Copy each batch.** `tour_screen(action='copy_to', draft_id=<source>, target_draft_id=<new>, screen_ids='id1,id2,id3')`. Both must be drafts; if you only have a `published_id`, call `tour(published_id=...)` first to get the `draft_id`. Order is preserved; screens append.
4. **Wait between copies.** `tour_screen(action='copy_status', target_draft_id=<new>)` until `in_progress=false`. Each copy is async; the destination locks during a copy and subsequent calls reject.
5. **(Optional) guides + publish.** `tour_guide(action='generate')` then publish + link.

### Verify shared-chrome matches before adding screens

A search hit that comes from a shared sidebar/nav surfaces every screen with that chrome — even when the screen's primary content is unrelated. `tour_screen(action='search')` flags this with `cross_screen_match_count` per snippet, `likely_shared_chrome: true` on flagged screens, and a top-level `verification_hint`. For each flagged screen, call `tour_screen(action='get', screen_id=...)` and read the headings before adding it.

### Compose caveats

- Both source and target must be **v4+ drafts**. Legacy v3 tours are rejected with `error: requires_v4_replay`.
- Source and target must differ. To clone an entire tour, use `tour_lifecycle(action='duplicate')`.
- Search is **literal text matching** against screen HTML + guide markdown — no OCR or LLM-generated descriptions.

## Variables and links

- `tour_variable(action='list'|'get'|'create'|'update'|'delete'|'insert'|'references', draft_id=...)` — manage variable *definitions* on the draft (source of truth) and wrap screen text with `variable_value` references via `insert`. The `references` action pairs with `list`'s `usage_count` and surfaces where each reference lives — use it before consolidating duplicate-named variables.
- Per-link override values (for a published tour) go via `tour_link(action='create', published_id=..., variables=...)`.

## `set_custom(kind='guide_css')` writes into the GUIDE iframe

`tour_lifecycle(action='set_custom', kind='guide_css', draft_id=..., content=...)` writes CSS into the **guide iframe** (the in-page bubble) — not the captured host page. Host-page selectors won't reach it. Use the `.rguide-*` class prefix and the `--rguide-*` CSS custom properties on `:root`.

**Don't guess class prefixes.** There is no `.reprise-tour-*`, `.repcl-*`, `.rps-*`, or `[class*=GuideBubble]` — those will silently miss. Default theme is **dark** (`#27292e` bubble bg), so any light/brand re-skin must override `--rguide-bg-color` or the bubble stays black.

`set_custom` must target a `draft_id`; passing a published ID errors with `wrong_id_kind`. Editor UI needs a refresh to apply MCP-written CSS; published tours apply on republish. Full token list and selector reference at `docs(slug='tour')`.

## After every session

`session_recap` + `report_friction` per distinct issue — see `reprise-session-close`.

The full editor surface (lifecycle actions, custom kinds, search semantics, v3-vs-v4 constraints, `set_auto` continuous capture, full theming token list) lives in `docs(slug='tour')`. For ID-kind questions, see `reprise-tour-id-model`.
