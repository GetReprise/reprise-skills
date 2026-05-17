# reprise-skills

Workflow skills for the Reprise MCP. URL is shared directly by Reprise; this repo is not indexed for discovery.

## Install (Claude Code)

```
/plugin marketplace add GetReprise/reprise-skills
/plugin install reprise-mcp-skills@reprise-skills
```

Other agent surfaces: see your provider's plugin-marketplace docs and point it at this repo.

## Layout

```
plugins/reprise-mcp-skills/skills/
├── reprise-mcp/              # router
├── reprise-tour-capture/
├── reprise-tour-edit/
├── reprise-tour-id-model/
├── reprise-clone-config/
├── reprise-injection/
└── reprise-session-close/
```

Each `SKILL.md` is self-contained. Frontmatter carries `version`; the repo tags releases.

## Companion

Skills assume the Reprise MCP is connected. Without it, the referenced tools (`tour_*`, `clone_*`, `injection_*`, `docs`, `search_patterns`, `session_recap`, `report_friction`) aren't available and the skills won't fire usefully.
