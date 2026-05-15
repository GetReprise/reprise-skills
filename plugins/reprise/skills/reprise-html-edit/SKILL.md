---
name: reprise-html-edit
description: Reprise HTML tour editing workflow — modifying an existing HTML replay. MUST be invoked before any edit action on a published or draft replay. Triggers on `html_edit`, `html_screen(action='copy_to')`, `html_guide`, `html_variable`, `html_link`, `set_custom`, or any request to modify / translate / rebrand an existing replay. Usually invoked by the `reprise-mcp` router; can also auto-fire on direct keyword matches. Body has the draft-vs-published ID model, the re-skin workflow, the compose-from-screens workflow, and the `--rguide-*` CSS surface. Distinct from `reprise-html-capture` (which is for recording new tours).
version: 0.1.0
---

# Reprise HTML Tour Editing

Editing an existing HTML replay covers text/attribute/image edits, translation, guide management, variables, re-skinning to a new vertical, and stitching new replays together from screens of existing ones.

## Draft vs published IDs (the one-paragraph version)

Every HTML replay has two IDs: `draft_id` (`Replay.id`, always exists) and `published_id` (`PublishedReplay.id`, only after `html_lifecycle(action='publish')`). Each `html_*` tool's signature accepts only the kind it operates on. There is **no auto-flip** — if you have the wrong kind, call `html(draft_id=... | published_id=...)` first; the response includes both IDs labeled so you can hand the right one to the strict-kind tool. A `<slug>` from a `/launch/<slug>/` URL is a third kind (`PublishedReplayLink.id`); `html(...)` resolves it transparently and echoes `link_id` back. Full table at `docs(slug='html-id-model')`.

## Re-skin to a new vertical / persona / brand

Use when the user wants to take a template replay and rewrite the visible product story without re-capturing. The story lives in **both** the captured DOM and the in-screen guides — touch both.

1. **Duplicate first.** `html_lifecycle(action='duplicate', draft_id=<template>)`. Never edit the template in place.
2. **Rewrite the captured DOM, screen by screen:**
   - `html_edit(action='list_text_nodes', draft_id=<new>, screen_node_id=<scrn-...>, text_contains=<source-term>)` — filter aggressively, surface `node_id`s.
   - `html_edit(action='edit_text', draft_id=<new>, edits='[{...}]')` — all edits for a screen as one transformation. `dry_run=True` previews first.
   - Attributes: `list_attribute_nodes` → `edit_attributes`. Images: `replace_image` (to a URL or `resource:<resource_id>`).
3. **Rewrite the guides.** `html_guide(action='list')` → `action='get'` → `action='update'` per guide.
4. **(Optional) polish + publish.** `html_lifecycle(action='publish')` + `html_link(action='create')`.

### `translate` is NOT a re-skin channel

`html_edit(action='translate')` is honest meaning-preserving translation across languages. Same-language `translate` ignores "remap to <new vertical>" instructions. To re-skin within one language, use the per-screen text/attribute editors.

## Compose a new replay from existing screens

Use when the user wants a tour stitched together from screens that already exist in other replays.

1. **Find candidates.** `html_search(query=..., search_scope='screens')` returns matched replays with screen IDs nested per replay. Finer control: `html_screen(action='search', draft_id=..., query=...)`.
2. **Create the destination.** `html(action='create', title='...')`.
3. **Copy each batch.** `html_screen(action='copy_to', draft_id=<source>, target_draft_id=<new>, screen_ids='id1,id2,id3')`. Both must be drafts; if you only have a `published_id`, call `html(published_id=...)` first to get the `draft_id`. Order is preserved; screens append.
4. **Wait between copies.** `html_screen(action='copy_status', target_draft_id=<new>)` until `in_progress=false`. Each copy is async; the destination locks during a copy and subsequent calls reject.
5. **(Optional) guides + publish.** `html_guide(action='generate')` then publish + link.

### Verify shared-chrome matches before adding screens

A search hit that comes from a shared sidebar/nav surfaces every screen with that chrome — even when the screen's primary content is unrelated. `html_screen(action='search')` flags this with `cross_screen_match_count` per snippet, `likely_shared_chrome: true` on flagged screens, and a top-level `verification_hint`. For each flagged screen, call `html_screen(action='get', screen_id=...)` and read the headings before adding it to a new tour.

## Variables and links

- `html_variable(action='list'|'get'|'create'|'update'|'delete'|'insert'|'references', draft_id=...)` — manage variable *definitions* on the draft (source of truth) and wrap screen text with `variable_value` references via `insert`. The `references` action pairs with `list`'s `usage_count` and surfaces where each reference lives — use it before consolidating duplicate-named variables.
- Per-link override values (for a published replay) go via `html_link(action='create', published_id=..., variables=...)`.

## `set_custom(kind='guide_css')` writes into the GUIDE iframe

`html_lifecycle(action='set_custom', kind='guide_css', draft_id=..., content=...)` writes CSS into the **guide iframe** (the in-page bubble) — not the captured host page. Host-page selectors won't reach it. Use the `.rguide-*` class prefix and the `--rguide-*` CSS custom properties on `:root`.

**Don't guess class prefixes.** There is no `.reprise-tour-*`, `.repcl-*`, `.rps-*`, or `[class*=GuideBubble]` — those will silently miss. Default theme is **dark** (`#27292e` bubble bg), so any light/brand re-skin must override `--rguide-bg-color` or the bubble stays black.

`set_custom` must target a `draft_id`; passing a published ID errors with `wrong_id_kind`. Editor UI needs a refresh to apply MCP-written CSS; published replays apply on republish. Full token list and selector reference at `docs(slug='html')`.

## After every session

`session_recap` + `report_friction` per distinct issue — see the `reprise-session-close` skill.

The full editor surface (lifecycle actions, custom kinds, search semantics, v3-vs-v4 constraints, set_auto continuous capture) lives in `docs(slug='html')`.
