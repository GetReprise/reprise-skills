# reprise-skills

Workflow skills for the Reprise MCP. URL is shared directly by Reprise; this repo is not indexed for discovery.

Two plugins ship from this marketplace — pick the one that matches the MCP you're connected to:

- **`reprise-mcp-skills`** — the v1 plugin, targets the polymorphic `action=` Reprise MCP at `/mcp/`.
- **`reprise-mcp-skills-v2`** — the v2 plugin, targets the atomic per-action Reprise MCP at `/v2/mcp/` (and per-product scoped paths). Tool names like `tour_dom_text_edit`, `injection_dataset_create`, `summary_report`.

## Install (Claude Code)

```
/plugin marketplace add GetReprise/reprise-skills

# pick one:
/plugin install reprise-mcp-skills@reprise-skills        # v1
/plugin install reprise-mcp-skills-v2@reprise-skills     # v2
```

Other agent surfaces: see your provider's plugin-marketplace docs and point it at this repo.

## Layout

```
plugins/reprise-mcp-skills/skills/         # v1 — polymorphic action= surface
├── reprise-mcp/                  # router
├── reprise-tour-capture/
├── reprise-tour-edit/
├── reprise-tour-id-model/
├── reprise-clone-config/
├── reprise-injection/
└── reprise-session-close/

plugins/reprise-mcp-skills-v2/skills/      # v2 — atomic per-action surface
├── reprise-mcp/                  # router
├── reprise-tour-capture/
├── reprise-tour-edit/
├── reprise-tour-id-model/
├── reprise-injection/
└── reprise-session-close/
```

Each `SKILL.md` is self-contained. Frontmatter carries `version`; the repo tags releases.

The v2 plugin omits `reprise-clone-config` for now — Clone is rebuilt later in the v2 roadmap. Clone work continues against the v1 polymorphic surface until then.

## Companion

Skills assume the corresponding Reprise MCP is connected.

- v1: `tour_*`, `clone_*`, `injection_*`, `docs`, `search_patterns`, `session_recap`, `report_friction`.
- v2: `tour_*` (atomic), `injection_*` (atomic), `tour_docs`, `tour_guidance`, `whoami`, `summary_report`, `friction_report`.

Without the relevant MCP, the referenced tools aren't available and the skills won't fire usefully.
