# reprise-skills

Workflow skills for the Reprise MCP. URL is shared directly by Reprise; this repo is not indexed for discovery.

One plugin ships from this marketplace:

- **`reprise-mcp-skills`** — targets the atomic per-action Reprise MCP. Tool names like `tour_dom_text_edit`, `injection_dataset_create`, `clone_snippet_create`, `platform_summary_report`.

## Install (Claude Code)

```
/plugin marketplace add GetReprise/reprise-skills
/plugin install reprise-mcp-skills@reprise-skills
```

Other agent surfaces: see your provider's plugin-marketplace docs and point it at this repo.

## Layout

```
plugins/reprise-mcp-skills/skills/    # atomic per-action surface
├── reprise-mcp/                  # router
├── reprise-tour-capture/
├── reprise-tour-edit/
├── reprise-tour-id-model/
├── reprise-clone-config/
├── reprise-injection/
└── reprise-session-close/
```

Each `SKILL.md` is self-contained. Frontmatter carries `version`; the repo tags releases.

## Companion

Skills assume the Reprise MCP is connected: `tour_*` (atomic), `clone_*` (atomic), `injection_*` (atomic), `tour_docs` / `clone_docs` / `injection_docs`, and cross-product `platform_*` (`platform_whoami`, `platform_summary_report`, `platform_friction_report`, `platform_folder_*`, `platform_asset_move`).

Without the MCP, the referenced tools aren't available and the skills won't fire usefully.
