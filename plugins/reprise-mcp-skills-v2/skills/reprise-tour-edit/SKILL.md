---
name: reprise-tour-edit
description: Reprise Product Tour editing workflow (v2) — modifying an existing tour. MUST be invoked before any edit action on a published or draft tour. Triggers on `tour_dom_text_edit`, `tour_dom_attributes_edit`, `tour_screen_copy`, `tour_guide_*`, `tour_variable_*`, `tour_link_*`, `tour_injection_set`, or any request to modify / translate / rebrand an existing tour. Usually invoked by the `reprise-mcp` router; can also auto-fire on direct keyword matches. Body has the v2 atomic-tool quick-reference, the re-skin vs translate distinction, the compose-from-screens flow, variables + links basics, and the `guide_css` surface. The full `--rguide-*` token list and complete theming reference live in `tour_docs(slug='tour-theming')`; full compose caveats in `tour_docs(slug='tour-authoring')`. Distinct from `reprise-tour-capture` (recording new tours) and `reprise-tour-id-model` (ID-kind resolution).
version: 0.1.0
---

# Reprise Product Tour Editing (v2)

Editing an existing tour covers text / attribute / image edits, translation, guide management, variables, re-skinning to a new vertical, and stitching new tours together from screens of existing ones. ID kinds (draft vs published vs launch-link) are the most common source of confusion — see `reprise-tour-id-model` if a paste of `/launch/<slug>/` or a `wrong_id_kind` error is what brought you here.

## Inspecting and editing — v2 atomic tool quick-reference

| Action | v2 Tool |
|---|---|
| Details + screens list | `tour_get(draft_id=...)` / `tour_get(published_id=...)` |
| Screen content | `tour_screen_list`, `tour_screen_get`, `tour_screen_search` (list returns per-screen `preview_url` for v4 tours — a deep link to a specific screen) |
| Enumerate captured-screen text | `tour_dom_text_node_list(draft_id=..., screen_node_id=..., text_contains=..., path_contains=...)` |
| Edit text | `tour_dom_text_edit(draft_id=..., edits='[{...}]')` — one call = one undo step; `dry_run=True` previews |
| Edit element attributes (`href`, `alt`, `class`, `aria-*`, `data-*`) | `tour_dom_attribute_node_list` / `tour_dom_attributes_edit` |
| Swap an image | `tour_dom_attributes_edit(...)` with `attribute=src` — also accepts `resource:<id>` shorthand and validates the node is an `<img>` |
| Bulk translate visible text (meaning-preserving) | `tour_translate(...)` (blocks-and-polls; `wait=False` to fire-and-forget) |
| Translate undo | `tour_translate_undo(...)` ; backups list via `tour_translate_backup_list` |
| Compose new tour from existing screens | `tour_screen_copy(...)` (blocks-and-polls) |
| Per-screen anchorable nodes (for guides) | `tour_screen_node_list(screen_id=...)` |
| In-screen guides | `tour_guide_create`, `tour_guide_list`, `tour_guide_get`, `tour_guide_update`, `tour_guide_delete` — **default to `guide_type='tethered'`** with a `target_node_id` from `tour_screen_node_list`. Floating is the fallback for screens with no good anchor, not the easy path. |
| Variables | `tour_variable_list`, `tour_variable_create`, `tour_variable_update`, `tour_variable_delete`, `tour_variable_insert`, `tour_variable_reference_list` |
| Per-tour custom CSS / JS injection | `tour_injection_get(draft_id=..., kind=...)` / `tour_injection_set(draft_id=..., kind='guide_css'\|'replay_wide_css'\|'replay_wide_js', content=...)` |
| Lifecycle | `tour_publish`, `tour_duplicate`, `tour_archive`, `tour_restore`, `tour_move`, `tour_metadata` (consolidates title/description/tags as PATCH-style — pass only the fields you want updated) |
| Shareable published links | `tour_link_create(published_id=..., variables=...)` ; manage via `tour_link_list` / `tour_link_update` / `tour_link_delete` |

`tour_injection_set` returns metadata only (bytes, sha256, last_saved, unchanged, preview_url) so iterative edit loops don't flood context.

## v2 surface notes

- **Every tool is one verb.** Atomic per-action tools — there's no `action=` parameter.
- **Image swaps** use `tour_dom_attributes_edit(attribute='src', value=...)` — accepts the `resource:<id>` shorthand and validates the node is an `<img>`.
- **Authoring guides:** read screens yourself (`tour_screen_get`, `tour_screen_node_list`) and call `tour_guide_create` directly.
- **`include` parameter for response slimming.** `tour_get` defaults to a minimal envelope; opt in to sections with `include="screens,guides,objects,sections,links,variables"`. Same pattern on heavy `_get` tools.

## Re-skin to a new vertical / persona / brand

Use when the user wants to take a template tour and rewrite the visible product story without re-capturing. The story lives in **both** the captured DOM and the in-screen guides — touch both.

1. **Duplicate first.** `tour_duplicate(draft_id=<template>)`. Never edit the template in place.
2. **Rewrite the captured DOM, screen by screen:**
   - `tour_dom_text_node_list(draft_id=<new>, screen_node_id=<scrn-...>, text_contains=<source-term>)` — filter aggressively, surface `node_id`s.
   - `tour_dom_text_edit(draft_id=<new>, edits='[{...}]')` — all edits for a screen as one transformation. `dry_run=True` previews first.
   - Attributes: `tour_dom_attribute_node_list` → `tour_dom_attributes_edit`. Images: `tour_dom_attributes_edit(attribute='src', value=...)` (URL or `resource:<id>`).
3. **Rewrite the guides.** `tour_guide_list` → `tour_guide_get` → `tour_guide_update` per guide.
4. **(Optional) polish + publish.** `tour_publish` + `tour_link_create`.

### `tour_translate` is NOT a re-skin channel

`tour_translate` is honest meaning-preserving translation across languages. Same-language `tour_translate` ignores "remap to <new vertical>" instructions. To re-skin within one language, use the per-screen text/attribute editors.

## Compose a new tour from existing screens

Use when the user wants a tour stitched together from screens that already exist in other tours.

1. **Find candidates.** `tour_search(query=..., search_scope='screens')` returns matched tours with screen IDs nested per tour. Finer control: `tour_screen_search(draft_id=..., query=...)`.
2. **Create the destination.** `tour_create(title='...')`.
3. **Copy each batch.** `tour_screen_copy(draft_id=<source>, target_draft_id=<new>, screen_ids='id1,id2,id3')`. Blocks-and-polls — returns once the copy finishes. Both must be drafts; if you only have a `published_id`, call `tour_get(published_id=...)` first to get the `draft_id`. Order is preserved; screens append. To run async, pass `wait=False` and poll with `tour_screen_copy(target_draft_id=<new>, wait=False)` (status-only).
4. **(Optional) guides + publish.** Walk `tour_screen_node_list` and `tour_guide_create` per screen, then `tour_publish` + `tour_link_create`.

### Verify shared-chrome matches before adding screens

A search hit that comes from a shared sidebar/nav surfaces every screen with that chrome — even when the screen's primary content is unrelated. `tour_screen_search` flags this with `cross_screen_match_count` per snippet, `likely_shared_chrome: true` on flagged screens, and a top-level `verification_hint`. For each flagged screen, call `tour_screen_get(screen_id=...)` and read the headings before adding it.

### Compose caveats

- Both source and target must be **v4+ drafts**. v3 tours are rejected with `error: requires_v4_replay`.
- Source and target must differ. To clone an entire tour, use `tour_duplicate`.
- Search is **literal text matching** against screen HTML + guide markdown — no OCR or LLM-generated descriptions.

## Variables and links

- `tour_variable_list` / `tour_variable_create` / `tour_variable_update` / `tour_variable_delete` / `tour_variable_insert` / `tour_variable_reference_list` — manage variable *definitions* on the draft (source of truth) and wrap screen text with `variable_value` references via `tour_variable_insert`. The list tool returns full rows; there's no separate `_get`. `tour_variable_reference_list` pairs with `tour_variable_list`'s `usage_count` and surfaces where each reference lives — use it before consolidating duplicate-named variables.
- Per-link override values (for a published tour) go via `tour_link_create(published_id=..., variables=...)`.

## `tour_injection_set(kind='guide_css')` writes into the GUIDE iframe

`tour_injection_set(kind='guide_css', draft_id=..., content=...)` writes CSS into the **guide iframe** (the in-page bubble) — not the captured host page. Host-page selectors won't reach it. Use the `.rguide-*` class prefix and the `--rguide-*` CSS custom properties on `:root`.

**Don't guess class prefixes.** There is no `.reprise-tour-*`, `.repcl-*`, `.rps-*`, or `[class*=GuideBubble]` — those will silently miss. Default theme is **dark** (`#27292e` bubble bg), so any light/brand re-skin must override `--rguide-bg-color` or the bubble stays black.

`tour_injection_set` must target a `draft_id`; passing a published ID errors with `wrong_id_kind`. Editor UI needs a refresh to apply MCP-written CSS; published tours apply on republish. Full token list and selector reference at `tour_docs(slug='tour-theming')`.

## After every session

`platform_summary_report` + `platform_friction_report` per distinct issue — see `reprise-session-close`.

The full editor surface lives across the task-area guides: authoring (objects, variables, sections, DOM edits, search semantics, compose) in `tour_docs(slug='tour-authoring')`, the `guide_css` token list in `tour_docs(slug='tour-theming')`, and publish + lifecycle in `tour_docs(slug='tour-publish')`. For ID-kind questions, see `reprise-tour-id-model`.
