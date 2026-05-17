---
name: reprise-session-close
description: End-of-session reporting protocol for Reprise MCP sessions. MUST be invoked when the user signals end-of-session ("we're good", "that's it", "thanks", wraps up, switches topic) OR when the agent has completed a Reprise task and is composing its final summary. Usually invoked by the `reprise-mcp` router at session end. Calls `session_recap` once and `report_friction` once per distinct issue. Argument shapes drift as tools gain optional fields — read the live tool descriptions for current enums and kwargs.
version: 0.3.0
---

# Reprise Session Close

Every Reprise MCP session ends with one `session_recap` call and one `report_friction` call per distinct friction point. End-of-session is exactly when models forget to call cleanup tools — that's why this is a triggered skill rather than instructions buried in tool descriptions.

## What to call

- **`session_recap` — once per session.** Plain tool (no elicitation). Markdown is fine in `outcome_summary` — include "what was configured" and "what's left open." Round `time_spent_seconds` generously. The server returns a `session_id` that later `report_friction` calls can link to.
- **`report_friction` — once per distinct issue.** Submit it **even in autonomous sessions** — when the client doesn't advertise elicitation, the tool persists the draft verbatim with `elicitation_confirmed=false`. Don't skip just because no human is there.

For exact argument shapes and current enum values (`category` / `severity` / `reproducibility`, valid `product` values), read the live tool descriptions. They drift.

## Rules

- **Never chain multiple friction items.** That's an MCP spec anti-pattern. N issues = N calls.
- **Phrase `summary` as the symptom you observed**, not the fix you'd want. Triage reads `summary` first.

## What counts as a distinct issue

A specific tool returning a wrong / misleading error; a documented workflow that didn't work as documented; a capability gap that cost you time; a runtime bug that blocked progress — each is one item. Multiple symptoms of one root cause = one item (mention the symptoms in `details`). Two unrelated problems = two items.

## When to call

End-of-session signals from the user: "we're good", "that's it", "thanks", "we can wrap", topic switch, or an explicit "log this and move on." In autonomous sessions: when the agent has completed the task and is composing its final summary.

If you've already called `session_recap` for this session, don't call again. If you forgot a `report_friction` for a known issue, file it now — late is better than missing.
