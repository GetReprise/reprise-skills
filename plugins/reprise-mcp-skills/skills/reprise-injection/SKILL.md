---
name: reprise-injection
description: Reprise Data Injection configuration workflow. MUST be invoked before any `injection_*` MCP call. Primarily for injecting data into a LIVE application via the Reprise extension; secondarily into a Reprise clone. Covers populating charts, tables, KPI tiles, dropdowns by swapping API responses at runtime. Triggers on any `injection_*` MCP tool, `dataset_id`, `Dataset → Source → Value`, `data injection`, or `Data Studio`. Usually invoked by the `reprise-mcp` router; can also auto-fire on direct keyword matches. Body has the two-phase wiring/data framework, the load-bearing principles, the canonical entry point, and the top catastrophic gotchas. Full vocabulary table, complete gotcha catalog, response-template derivation algorithm, adapter create/link/swap, failure-modes table, response-shape trims, and activation-reason taxonomy all live in `docs(slug='injection')`. NOT a target — data injection cannot target a Product Tour directly.
version: 0.3.1
---

# Reprise Data Injection

Data Injection swaps specific web request responses on a target site with structured data, so charts / tables / KPI tiles / dropdowns render with the data the demo needs.

## Targets, in priority order

1. **Live application + Reprise extension** (primary). The user loads the live site with the Reprise Builder: Data Injection extension installed; injected rules intercept matching requests at runtime.
2. **Reprise clone** (secondary). Same rules can fire inside a clone preview when configured.
3. **NOT a target — Product Tours.** To get injected data into a tour, inject into the live app first, then capture the live, injected app via the tour capture flow.

## Requirements

- **Browser-automation MCP** launching **real Chrome** (not bundled Chromium — Chrome Web Store installs need real Chrome).
- **Reprise Builder: Data Injection extension** installed in that Chrome. Without it, `injection_dataset(activate)` returns `extension_not_installed`.

## The two-phase framework — don't mix them

Two loops with distinct failure axes. Mixing them is the #1 wasted-time pattern.

- **Phase A — Plumbing.** Wire up source + adapter + response_template with **sentinel values** (`99999`, `"DUMMY"`, dates `1999-01-01..30`). Failure means wiring is wrong.
- **Phase B — Data.** Replace sentinels with the user's intended data. Failure means values, not wiring.

**Key principle: data lives in the Value (DataObject), nowhere else.** Don't put data values in the adapter (its `parse_function` / `serialize_function` are pure transforms), in the source's `conditions` (they match requests, not data), or via browser-automation typing (bypasses Reprise entirely; the next reload reverts). If you find yourself typing values somewhere other than the Value, you're off-track.

## Vocabulary

Data Studio uses a flat **Dataset → Source → Value** model. A Source fuses what some legacy tool names still call Widget / DataObjectConfig / InterceptionRule; a Value is what some still call DataObject. Tool names keep their legacy spellings for back-compat; their bodies call the current endpoints when you pass `dataset_id`. Full vocabulary table at `docs(slug='injection')`.

## Workflow

**Phase A — Plumbing with sentinels:**

1. **Identify the target request** via the BMA. Record URL + method + raw response body. Note `fetch` vs `XMLHttpRequest` (adapters only intercept `fetch`).
2. **Patterns.** `docs(slug='injection-patterns')` — the data-injection pattern corpus, read in full (TOC with excerpts by default; pass `section='<name>'` for one topic's full text).
3. **Derive the schema from the captured body** — don't invent it. Either pass the raw body as `response_template` (inference produces the triple) or pass the `{template, data_map, injection_point}` triple under `response_template_raw` when inference can't cover the shape. Full derivation algorithm at `docs(slug='injection')`.
4. **Build the dataset + source + Value in ONE atomic call.** See "Canonical entry point" below.
5. **Open the dataset editor** at `https://<host>/data-studio/datasets/<dataset_id>` in a side tab — ground truth for what Reprise has stored. Keep open through Phase B to disambiguate wiring vs data failures.
6. **Activate.** `injection_dataset(action='activate', dataset_id=...)` returns a `handshake_expression` — run it via the BMA. Structured `{success, reason}` reply distinguishes real success from `extension_not_installed` / `extension_outdated` / `extension_not_responding`.
7. **Verify sentinels render.** Sentinels visible → Phase A passes. Originals still visible → plumbing fails. Page blank → injection over-replaced.
8. **Iterate plumbing only** on miss via `injection_widget(action='update', ...)`, then `deactivate` + `activate`. Don't touch data values here.

**Phase B — Real data:**

9. **Re-confirm data shape.** Rows-not-columns (30-day chart = 30 rows × few columns). Scalar cells only. Display Names in `data_map` must match the keys you're about to write.
10. **Replace sentinels** via `injection_object(action='update', ...)`, re-activate, glance at the editor tab to confirm rows updated.
11. **Verify against intent.** Editor matches intent but page doesn't → wiring (back to Phase A). Editor rows wrong → data (back to step 10).
12. **Hand off.** Share the editor URL with the user so they can keep iterating.

## Canonical entry point — one atomic call

**`injection_dataset(action='create', name=..., sources_json=...)`** creates the dataset + every source + every Value in a single transaction. Reach for this **before** `injection_widget(action='create_with_rule', ...)` — the latter is back-compat and costs N+1 calls.

```json
[
  {
    "section": "<group name>",
    "name": "<source name>",
    "target_url": "<step-1 URL>",
    "conditions": [
      {"type": "method", "value": "<step-1 method>"},
      {"type": "url", "test": "contains", "value": "<step-1 path fragment>"}
    ],
    "response_template": <raw body from step 1>,
    "interception_rule_metadata": {"sync": true},
    "values": [
      {"name": "sentinel", "data": [{"type": "replace", "entities": [<sentinel rows>]}]}
    ]
  }
]
```

Sentinels keyed by Display Names from the inferred `data_map`. Mismatched keys = empty cells. Multiple sources / values: more entries — still one atomic call. Pass raw body in `response_template`; pass the triple in `response_template_raw` (backend rejects the triple in `response_template` at preflight).

## Top catastrophic gotchas

The full catalog is in `docs(slug='injection')`. These three are silent — accepted at create time, break at runtime, no error:

1. **`interception_rule_metadata.sync = true` or the rule never fires.** Without it, the rule isn't included in `play_config.sync_interception_rules` and the runtime filter treats it as inactive.
2. **Conditions shape — typed-dict form is mandatory.** Flat shorthand (`{"method": "GET"}`) is accepted at create but **never matches at runtime** — the matcher reads `rule.type` / `rule.value`. Always: `{"type": "method", "value": "GET"}`, `{"type": "url", "test": "contains", "value": "/api"}`.
3. **One source per `(target_url, injection_point)` pair — last `replace` wins.** N sources sharing the same target URL + array `injection_point` have `apply_rule_modifications` walk them in order; the last `replace` wins and the others' values are silently overwritten. One response feeding N charts → ONE source whose `data_map` has N columns.

## After every session

`session_recap` + `report_friction` per distinct issue — see `reprise-session-close`.

For everything else — full vocabulary, complete gotcha catalog, response-template derivation algorithm, adapter mechanics, legacy `create_with_rule` flow, response-shape trims, failure-modes table, activation-reason taxonomy, resuming prior sessions — `docs(slug='injection')`.
