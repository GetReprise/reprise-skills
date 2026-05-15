---
name: reprise-data-injection
description: Reprise Data Injection configuration workflow. MUST be invoked before any `injection_*` MCP call. Primarily for injecting data into a LIVE application via the Reprise extension; secondarily into a Reprise clone. Covers populating charts, tables, KPI tiles, dropdowns by swapping API responses at runtime. Triggers on any `injection_*` MCP tool, `dataset_id`, `Dataset → Source → Value`, `data injection`, or `Data Studio`. Usually invoked by the `reprise-mcp` router; can also auto-fire on direct keyword matches. Body has the two-phase wiring/data framework, the canonical entry point, the activation handshake, and the top 5 silent-failure gotchas. NOT a target — data injection cannot inject into an HTML replay directly.
version: 0.2.0
---

# Reprise Data Injection

Data Injection swaps specific web request responses on a target site with structured data the agent supplies, so charts / tables / KPI tiles / dropdowns render with the data the demo needs.

## Targets, in priority order

1. **Live application + Reprise extension** (primary). The user loads the live site with the Reprise Builder: Data Injection extension installed; injected rules intercept matching requests at runtime and substitute responses.
2. **Reprise clone** (secondary). Same rules can fire inside a clone preview when configured.
3. **NOT a target — HTML replays.** Data injection cannot target an HTML replay directly. To get injected data into a replay, inject into the live app first, then capture the live, injected app via the HTML capture flow.

## Requirements

- **Browser-automation MCP** (e.g. Playwright) launching **real Chrome** — not bundled Chromium. Chrome Web Store installs need real Chrome.
- **Reprise Builder: Data Injection extension** installed in that Chrome. Without it, `injection_dataset(activate)` returns `extension_not_installed` and nothing injects.

## The two-phase framework — don't mix them

Data injection has two loops with distinct failure axes. Mixing them is the #1 wasted-time pattern.

- **Phase A — Plumbing.** Wire up the source + adapter + response_template with **sentinel values** (obvious fakes like `99999`, `"DUMMY"`, dates `1999-01-01..30`). Failure here means the wiring is wrong: source not matching the request, wrong injection point, adapter not linked, response_template shape off.
- **Phase B — Data.** Replace sentinels with the user's intended data. Failure here means values, not wiring.

**Key principle: data lives in the Value (DataObject), nowhere else.** Don't put data values:

- In the **adapter** — its `parse_function` / `serialize_function` are pure transforms.
- In the **source's `conditions`** — they match requests, not data.
- Via **browser automation typing** — bypasses Reprise entirely; the next reload reverts.

If you find yourself typing values somewhere other than the Value, you're off-track.

## Vocabulary (v2 model)

Data Studio v2 uses a flat **Dataset → Source → Value** model. Older docs/tools use legacy terms; map them on the fly:

| v2 term | Legacy term(s) |
|---|---|
| Dataset | Dataset |
| `section` (string on a Source) | ProductArea (first-class entity, now a name on a Source) |
| Source | Widget + DataObjectConfig + InterceptionRule (fused) |
| Value | DataObject |
| `response_template` | `template` + `data_map` + `injection_point` (fused — give it the raw body and the backend infers the triple) |
| `response_template_raw` | The same triple, supplied verbatim — escape hatch when inference can't cover the response shape |
| `target_url` | `interception_rule.target` |
| `conditions` | `interception_rule.rules.conditions` |

The MCP tool names (`injection_dataset`, `injection_widget`, `injection_config`, `injection_rule`, `injection_object`, ...) keep their legacy names, but their bodies call the v2 endpoints whenever you pass a `dataset_id`.

## Workflow

### Phase A — Plumbing with sentinel values

1. **Identify the target request.** Drive the BMA to the page, let it settle, find the request carrying the data. Record URL + HTTP method + raw response body. Note `fetch` vs `XMLHttpRequest` — adapters only intercept `fetch`. **Dual-content-type trap:** some pages return HTML by default and JSON only with `Accept: application/json` + `X-Requested-With: XMLHttpRequest` headers.
2. **Pattern-search.** `search_patterns(symptom='...', product='data_injection')`. If the top hit is weak (`score < ~0.5`), fall back to `injection_studio(action='read_patterns')`.
3. **Pick the injection point and derive the schema from the captured body.** Don't invent `template` / `data_map`. Two options:
   - **Let the backend infer it.** Supply the raw captured body as `response_template` on the source payload — inference produces `template` + `data_map` + `injection_point` for you.
   - **Supply the triple verbatim** under `response_template_raw` when inference can't cover the response shape (deeply nested, mixed types, etc.). Skips inference; stored as-is.
4. **Build the dataset + source + Value in ONE call** with sentinel values. See "Canonical entry point" below for the shape.
5. **Open the dataset editor in a side tab** at `https://<host>/data-studio/datasets/<dataset_id>`. This is ground truth for what Reprise has stored. Keep open through Phase B — it disambiguates wiring failures (editor wrong) from data failures (editor right but page wrong).
6. **Activate the dataset.** Two-step:
   1. `injection_dataset(action='activate', dataset_id=...)` returns `{handshake_expression, play_config, dataset_id}`.
   2. Run `handshake_expression` via the BMA (`browser_evaluate(function=...)`). Returns structured `{success, reason}`:
      - `{success: true, action: 'play', status: 'received'}` — extension acked; page reloads with injection.
      - `{success: false, reason: 'extension_not_installed'}` — install the extension.
      - `{success: false, reason: 'extension_outdated'}` — update the extension; or fall back to `preview(start, demo_id=...)` for a demo wrapper.
      - `{success: false, reason: 'extension_not_responding'}` — reload the page and retry.
7. **Verify sentinels render.** After the post-activate reload settles, check the page via the BMA:
   - Sentinels visible → Phase A passes.
   - Originals still visible → plumbing fails (target_url? XHR/fetch mismatch? adapter unlinked? conditions shape wrong?).
   - Page broken / blank → injection over-replaced; narrow the source.
8. **Iterate plumbing only** (on miss). PATCH source via `injection_widget(action='update', dataset_id=..., widget_id=..., attributes='{...}')`, then `deactivate` + `activate` again to refresh. **Don't touch data values in Phase A.**

### Phase B — Replace sentinels with intended data

9. **Re-confirm data shape.** Rows-not-columns (a 30-day chart = 30 rows × few columns, not 30 columns). Scalar cells only — no nested structs, arrays, or stringified JSON. Cross-check the editor tab: Display Names in `data_map` must match the keys you're about to write; mismatches land as empty cells.
10. **Replace sentinels** via `injection_object(action='update', dataset_id=..., source_id=..., object_id=..., attributes='{"data":[...]}')`, then re-activate. Glance at the editor tab to confirm rows updated.
11. **Verify against intent.** Reload, compare to the user's described shape. Off → adjust data and re-verify. Editor matches intent but page doesn't → wiring (back to Phase A); editor rows wrong → data (back to step 10).
12. **Hand off + record.** Tell the user where the injection is live (URL + what changed) and share the editor URL so they can keep iterating. Then `session_recap` + one `report_friction` per distinct issue.

## Canonical entry point — one atomic call

**`injection_dataset(action='create', name=..., sources_json=...)`** creates the dataset + every source + every Value in a single transaction. Reach for this **before** `injection_widget(action='create_with_rule', ...)` — the latter is a back-compat alias and costs N+1 calls.

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

- One entity per row; keys are Display Names from the inferred `data_map` (e.g. `"Park Id"`), not paths (`"park_id"`). Keys that don't match the data_map = empty cells.
- Sentinels = obvious fakes so step 7 can distinguish "injection working" from "no-op".
- Multiple sources / values: just add more entries to `sources_json` and `values` respectively — still one atomic call.

Pass the captured raw body in `response_template` to let inference work. If you already have the structured `{template, data_map, injection_point}` triple, pass it in `response_template_raw` instead — the backend rejects a triple passed as `response_template` at preflight.

## Top 5 silent-failure gotchas

1. **`interception_rule_metadata.sync = true` or the rule never fires.** Without it, the rule isn't included in `play_config.sync_interception_rules` and the maestro runtime filter treats it as inactive. Set on every source you create unless you specifically want the rule inactive.
2. **Conditions shape — typed-dict form is mandatory.** Flat shorthand `{"method": "GET"}` or `{"url": {"contains": "/api"}}` is accepted at create time but **never matches at runtime** — the matcher reads `rule.type` / `rule.value`, falls through, and the request passes uninjected with no error. Always use the explicit typed form: `{"type": "method", "value": "GET"}`, `{"type": "url", "test": "contains", "value": "/api"}`.
3. **One source per `(target_url, injection_point)` pair — last `replace` wins.** N sources sharing the same target URL + array injection point will have `apply_rule_modifications` walk them in order; the last `replace` wins and the others' values are silently overwritten. If one captured response feeds N charts, model it as ONE source whose `data_map` has N columns (one per chart's metric); each chart-side widget reads its own column. Different sources only when the charts genuinely come from different endpoints or disambiguating conditions.
4. **`response_template` expects a raw body; `response_template_raw` expects the triple.** Passing `{template, data_map, injection_point}` under `response_template` is rejected at preflight — inference would walk the triple as a body and produce garbage. Pass triples in `response_template_raw`; pass raw response bodies in `response_template`.
5. **PATCH of `response_template` / `response_template_raw` deletes every existing Value attached to the source.** The PATCH response returns `deleted_value_ids` and `warnings`. If you only want to refine `template` / `data_map` / `injection_point` without losing values, PATCH the legacy `data_object_config` via `injection_config(action='update', config_id=..., attributes={...})` — that path syncs renames instead of deleting children.

Full gotcha catalog (data-map renderer contract, adapter linking, double-encoded conditions, the multi-injection pattern, extension version notes, etc.) lives in `docs(slug='data-injection')`.

## Inspection

- `injection_dataset(action='get', dataset_id=..., include='templates,metadata')` — full source bodies. Default omits `response_template` to keep responses small.
- `injection_dataset(action='read_play_config', dataset_id=...)` — confirms the rule is wired into the runtime config.
- `injection_dataset(action='recent')` — list recently-edited datasets / sources / values for resuming prior work.

## Image fields

For any image field in injected data (logos, screenshots, brand assets): use PNG or JPG (SVG doesn't render in injection widgets), verify the URL returns image content before using it, and don't rely on the Clearbit Logo API (404s are common and unpredictable).

## After every session

`session_recap` + `report_friction` per distinct issue — see the `reprise-session-close` skill.

For the full reference (data_map renderer contract, adapter create/link, the legacy `create_with_rule` flow, the failure-mode table, response-shape trims, resuming prior sessions): `docs(slug='data-injection')`.
