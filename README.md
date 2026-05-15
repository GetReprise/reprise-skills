# reprise-skills

Workflow skills for the Reprise MCP — Clone configuration, HTML tour capture, HTML tour editing, data injection, and end-of-session reporting.

Authored once as standard `SKILL.md` files; install from this Git repo on every major agent surface. Designed to make the Reprise MCP land tasks first-time-right.

## What's in the box

Five skills under `plugins/reprise/skills/`:

| Skill | Fires on |
|---|---|
| `reprise-clone-config` | Configuring, debugging, or fixing a Reprise Clone (RB snippets, white screens, API routing, SW issues) |
| `reprise-html-capture` | Building a new HTML tour via the Reprise Builder: HTML extension (pairing, capture loop, page-settle) |
| `reprise-html-edit` | Editing an existing HTML replay (text/attribute/image, translate, re-skin, compose from screens, guides, variables) |
| `reprise-data-injection` | Configuring Data Injection (primarily into a live app via the Reprise extension; secondarily into a clone) |
| `reprise-session-close` | End-of-session — calls `session_recap` once and `report_friction` once per distinct issue |

A sixth skill (`reprise-pattern-search-first`) is being held for v2; the "call `search_patterns` first" forcing function currently lives inside each product skill.

## Install — one-line per provider

| Provider | Command |
|---|---|
| **Claude** | `/plugin marketplace add GetReprise/reprise-skills` then toggle auto-update on for the marketplace |
| **Codex CLI** | `codex plugin marketplace add GetReprise/reprise-skills` (the repo also ships `.agents/plugins/marketplace.json` for repo-scoped install) |
| **GitHub Copilot** | `gh skill install GetReprise/reprise-skills <skill-name>` per skill; `gh skill update` updates whatever's installed |
| **Gemini CLI** | `npx skills install github.com/GetReprise/reprise-skills` |
| **ChatGPT (Business / Enterprise / EDU / Team)** | Upload `plugins/reprise/skills/*` through OpenAI's Skills upload flow (admin) |
| **Cursor** | `git clone https://github.com/GetReprise/reprise-skills.git` then copy `plugins/reprise/skills/*` into `.cursor/skills/`; `git pull` for updates |

> While the repo is private, install commands that point at a GitHub repo need either (a) the repo flipped to public, or (b) authenticated access for whichever provider is doing the install (typically a PAT with `repo:read`). For internal testing on Claude Code, a local-path install also works: `claude plugin marketplace add /path/to/local/clone`.

Repo lives at https://github.com/GetReprise/reprise-skills.

## Companion: the Reprise MCP

These skills assume the Reprise MCP is connected. Without it, the tool calls referenced in each skill (`html_capture`, `clone_snippet`, `injection_dataset`, `docs`, `search_patterns`, `session_recap`, `report_friction`, etc.) aren't available and the skills won't fire usefully. The MCP itself also exposes `docs(slug='clone' | 'html' | 'data-injection' | 'clone-api' | 'html-id-model')` for full reference material — every skill points there for depth.

## Repo layout

```
reprise-skills/
├── .claude-plugin/
│   └── marketplace.json          # Claude marketplace manifest
├── .agents/
│   └── plugins/
│       └── marketplace.json      # Codex repo-scoped marketplace manifest
├── plugins/
│   └── reprise/
│       ├── .claude-plugin/
│       │   └── plugin.json       # Plugin manifest
│       └── skills/
│           ├── reprise-clone-config/SKILL.md
│           ├── reprise-html-capture/SKILL.md
│           ├── reprise-html-edit/SKILL.md
│           ├── reprise-data-injection/SKILL.md
│           └── reprise-session-close/SKILL.md
└── README.md
```

## To verify before publishing

- **Codex marketplace schema.** `.agents/plugins/marketplace.json` was authored against the convention described in the cross-provider plan; verify against the current Codex plugin spec before publishing. Repo marketplaces work today without going through the official Plugin Directory.
- **ChatGPT Skills upload flow.** OpenAI's Apps + Skills are distributed separately; confirm the current admin upload path (vs. marketplace listing vs. per-user) before treating this as a one-step ship.
- **Frontmatter dialect parity across loaders.** SKILL.md is an open standard, but each provider may be stricter about which frontmatter keys they accept. Smoke-test the bundle on at least Claude + ChatGPT + Gemini + Cursor before assuming portability is friction-free.
- **Trigger-evaluation parity across providers.** Skill descriptions are portable; whether they *fire* on the same intent across providers may vary. Run the same canonical task against each provider's loader as part of release CI.

## Versioning

`SKILL.md` files carry `version: 0.1.0` in frontmatter. Cut a Git tag per release so customers reporting "skill X said Y" can be pinned to a commit / tag.

## Distribution & updates

- **Auto-update for free:** Claude, Codex, Copilot, Gemini all install from this repo and pull updates via their native mechanisms after a `git push` to the canonical branch.
- **Manual coordinated release:** ChatGPT Apps + Skills go through OpenAI's separate upload channels — coordinate the two on every release.
- **DIY pull:** Cursor users `git pull` (or re-clone) to update.

## License

(TBD before public publication.)
