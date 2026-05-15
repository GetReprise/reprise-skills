---
name: reprise-mcp
description: Required entry point and router for ALL Reprise MCP work. MUST be invoked immediately whenever the user wants to do anything with Reprise — fixing / building / editing / debugging / rebranding a Reprise demo, preview, tour, replay, or captured app; troubleshooting blank or broken pages in a Reprise context; or any mention of "Reprise", "demo" (in a Reprise context), "replay", "capture", "preview URL", or any Reprise MCP tool. Customers describe tasks in plain English ("fix my demo", "add data to this page", "build a tour of HubSpot", "make this look like a prospect") without naming Reprise's internal product surfaces; this router maps natural-language asks to the right surface skill (`reprise-clone-config`, `reprise-html-capture`, `reprise-html-edit`, `reprise-data-injection`, or `reprise-session-close`) and invokes it explicitly. Always read this skill first on any Reprise turn — it is the entry point.
version: 0.1.0
---

# Reprise MCP — Router

You've landed here because the user wants to do Reprise work. This skill carries **no workflow content** — its only job is to identify the right surface and hand off to the surface-specific skill. Don't try to do the work yourself from this skill; route correctly and the right skill will load with everything you need.

## Step 1 — Confirm Reprise context

You're in scope if:
- The user mentions Reprise, a Reprise preview URL, a Reprise demo, a clone / replay / tour, or any Reprise MCP tool.
- The Reprise MCP is connected and its tools (`clone_*`, `html_*`, `injection_*`, `docs`, `search_patterns`, `session_recap`, `report_friction`) are available.

If neither is true, this skill shouldn't have fired — answer the user's actual question without routing.

## Step 2 — Identify the surface from cues

| Surface | The user is asking for | Cues |
|---|---|---|
| **Clone** (formerly Replicate) | Fix / debug / configure a captured interactive app | `clone_id`, `RB snippet`, `clone_api`, `replay_backend`, any `clone_*` MCP tool, `/sw-setup/` URL, "captured app", "the demo white-screens / goes blank / breaks after login", service-worker issues, snippet editing |
| **HTML Tour, NEW capture** (formerly Replay) | Record a new tour by capturing screens | `html_capture`, `pairing_token`, `deep_link_url`, `capture_now`, a brand-new `draft_id`, "Reprise Builder HTML extension", "record a tour", "build a tour of X", "capture screens" |
| **HTML Tour, EDITING existing** | Modify an existing replay (text, attributes, images, guides, translation, re-skin) | `html_edit`, `html_screen(action='copy_to')`, `html_guide`, `html_variable`, `html_link`, `set_custom`, "edit this replay", "translate to X", "re-skin for prospect Y", "rebrand the tour", "add a guide / variable" |
| **Data Injection** | Populate charts / tables / dropdowns with custom data by swapping API responses | any `injection_*` MCP tool, `dataset_id`, `Dataset → Source → Value`, "Data Studio", "populate the chart", "fake data on this page", "swap the API response", "inject data into the live app" (primary case) |

If exactly one surface matches, go to Step 4.

## Step 3 — Ask if ambiguous

If the prompt doesn't clearly indicate which surface (common with vague phrasings like "fix my Reprise demo," "build me a demo for [prospect]," "the page is broken"), ask **one** clarifying question:

> Are you working on:
> - **Clone** — an interactive captured app you're configuring or debugging (formerly Replicate)?
> - **HTML Tour** — a linear product tour from captured screens (formerly Replay)? *(And: building new, or editing existing?)*
> - **Data Injection** — swapping API responses on a live app to populate charts / tables?

Don't guess; don't proceed without the answer. A wrong surface guess wastes far more time than one clarifying turn.

## Step 4 — Invoke the surface skill

Call the right skill explicitly. Do this BEFORE any other tool call:

- **Clone** → `Skill(skill="reprise:reprise-clone-config")`
- **HTML Tour, new capture** → `Skill(skill="reprise:reprise-html-capture")`
- **HTML Tour, editing existing** → `Skill(skill="reprise:reprise-html-edit")`
- **Data Injection** → `Skill(skill="reprise:reprise-data-injection")`

For HTML Tour, the verb is the signal: "build / record / capture / make a tour" → `reprise-html-capture`; "edit / translate / re-skin / change / fix / add a guide / rebrand" → `reprise-html-edit`.

## Step 5 — At end of session

Regardless of which surface skill ran, when the user signals end-of-session ("we're good", "that's it", "thanks", topic switch) or when you've completed the task and are composing a final summary, invoke:

`Skill(skill="reprise:reprise-session-close")`

That skill carries the `session_recap` and `report_friction` call shapes.

## What this skill is NOT

- It's not a workflow skill. Don't enumerate phases from here.
- It's not the place for gotchas, forcing functions, or method references — those live in the surface skills' bodies and in `docs(slug='...')`.
- It's not a fallback if you forgot which skill to invoke — pick one consciously.
