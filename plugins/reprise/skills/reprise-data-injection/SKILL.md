---
name: reprise-data-injection
description: Reprise Data Injection configuration workflow. MUST be invoked before any `injection_*` MCP call. Primarily for injecting data into a LIVE application via the Reprise extension; secondarily into a Reprise clone. Covers populating charts, tables, KPI tiles, dropdowns by swapping API responses at runtime. Triggers on any `injection_*` MCP tool, `dataset_id`, `Dataset → Source → Value`, `data injection`, or `Data Studio`. Usually invoked by the `reprise-mcp` router; can also auto-fire on direct keyword matches. Body has the v2 flat model, the canonical entry point, and the sync-metadata gotcha. NOT a target — data injection cannot inject into an HTML replay directly.
version: 0.1.0
---

# Reprise Data Injection

Data Injection swaps specific web request responses on a target site with structured data the agent supplies, so charts / tables / KPI tiles / dropdowns render with the data the demo needs.

## Targets, in priority order

1. **Live application + Reprise extension** (primary). The user has the Reprise extension installed and is loading the live site; injected rules intercept matching requests at runtime and substitute responses.
2. **Reprise clone** (secondary). The same rules can fire inside a clone preview when configured.
3. **NOT a target — HTML replays.** Data injection cannot target an HTML replay directly. To get injected data into a replay, inject into the live app first, then use the HTML capture flow to capture the live, injected app as a replay.

## Vocabulary (v2 model)

Data Studio v2 uses a flat **Dataset → Source → Value** model. Older docs/tools use legacy terms; map them on the fly:

| v2 term | Legacy term(s) |
|---|---|
| Dataset | Dataset |
| `section` (string field on a Source) | ProductArea (was a first-class entity, now a name on a Source) |
| Source | Widget + DataObjectConfig + InterceptionRule (three legacy entities, fused) |
| Value | DataObject |
| `response_template` | `template` + `data_map` + `injection_point` (three legacy fields, fused) |
| `response_template_raw` | The same triple, supplied verbatim — escape hatch when inference can't cover the response shape |
| `target_url` | `interception_rule.target` |
| `conditions` | `interception_rule.rules.conditions` |

The MCP tool names (`injection_dataset`, `injection_widget`, `injection_config`, `injection_rule`, `injection_object`, `injection_product_area`, ...) keep their legacy names for back-compat, but their bodies call the v2 endpoints whenever you pass a `dataset_id`.

## Canonical entry point

**`injection_dataset(action='create', name=..., sources_json=...)`** atomically creates a dataset + its sources + their values in one transaction. Reach for this **before** `injection_widget(action='create_with_rule', ...)` — the latter is a back-compat alias that translates into the same v2 source payload under the hood.

`sources_json` is a JSON array of source objects. Each source uses the v2 flat shape:

```json
{
  "section": "<group name>",
  "name": "<source name>",
  "target_url": "<url to intercept>",
  "conditions": [
    {"type": "method", "operator": "equals", "value": "GET"},
    {"type": "url", "operator": "contains", "value": "/api/..."}
  ],
  "response_template": {
    "template": <inferred template>,
    "data_map": <inferred data_map>,
    "injection_point": {"path": "...", "type": "array" }
  },
  "interception_rule_metadata": {"sync": true},
  "values": [
    {"name": "Row 1", "data": [{"type": "replace", "entities": [...]}]}
  ]
}
```

### Set `interception_rule_metadata.sync = true` or the rule won't fire

This is the single most-missed gotcha. Without `sync: true`, the rule doesn't show up in `play_config.sync_interception_rules` and the runtime never fires it. Set it on every source you create unless you specifically want the rule inactive.

## Inspection

- `injection_dataset(action='get', dataset_id=..., include='templates,metadata')` returns the dataset with full source bodies. Default omits `response_template` to keep responses compact.
- `injection_dataset(action='read_play_config', dataset_id=...)` shows whether the rule is wired into the runtime config.

## Image fields

For any image field in injected data (logos, screenshots, brand assets), follow the rules in the `reprise-mcp-assets` skill if installed. Key constraints regardless: no SVG (PNG/JPG only), no Clearbit, verify the URL returns image content before using it.

## After every session

`session_recap` + `report_friction` per distinct issue — see the `reprise-session-close` skill.

The full reference — response-template inference, the `response_template_raw` escape hatch, condition operators, value layering, the legacy `injection_widget` / `injection_config` / `injection_rule` flow — lives in `docs(slug='data-injection')`.
