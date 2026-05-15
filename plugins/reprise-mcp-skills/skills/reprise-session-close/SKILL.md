---
name: reprise-session-close
description: End-of-session reporting protocol for Reprise MCP sessions. MUST be invoked when the user signals end-of-session ("we're good", "that's it", "thanks", wraps up, switches topic) OR when the agent has completed a Reprise task and is composing its final summary. Usually invoked by the `reprise-mcp` router at session end; can also auto-fire on direct end-of-session signals. Calls `session_recap` once and `report_friction` once per distinct issue. Body has the exact call shapes and the rule that `report_friction` should be submitted even in autonomous sessions.
version: 0.2.0
---

# Reprise Session Close

Every Reprise MCP session should end with one `session_recap` call and one `report_friction` call per distinct friction point encountered. These tools record what happened so the session can be reviewed and patterns can be improved over time.

End-of-session is exactly when models forget to call cleanup tools, which is why this is a skill — a triggered skill at "you're wrapping up" intent is more reliable than a buried instruction inside `session_recap`'s tool description.

## `session_recap` — once per session

```
session_recap(
  outcome_summary="## Summary\n<what was configured>\n\n## Left open\n<what still needs follow-up>",
  time_spent_seconds=<approximate session duration>,
  agent_model="<your model id>",
  app="<name of the target application>",
)
```

- Markdown is fine in `outcome_summary`; use it.
- `time_spent_seconds` is approximate — round generously.
- `agent_model` should be your model id (e.g. `claude-opus-4-7`, `gpt-5-...`, `gemini-...`).
- `app` is the target application — the customer site, the clone label, or the replay title.
- The server returns a `session_id`; subsequent `report_friction` calls in the same session can link to it via the `session_id` arg.

Call this exactly once. It's a plain tool — no elicitation.

## `report_friction` — once per distinct issue

```
report_friction(
  category="bug" | "tool_ergonomics" | "platform_gap" | "docs_gap" | "feature_request",
  severity="blocker" | "major" | "minor",
  product="clone" | "html" | "data_injection",
  summary="<one line, max 80 chars>",
  details="<full description, max 2000 chars>",
  reproducibility="confirmed" | "suspected" | "one_off",
  subsystem="<optional label, e.g. clone_snippet>",
  workaround="<optional>",
  suspected_location="<optional file:line>",
  time_cost_minutes=<optional 0-600>,
  agent_model="<your model id>",
  app="<app name>",
  session_id="<from session_recap response, if available>",
)
```

### Key rules

- **One call per distinct issue.** Never chain multiple friction items into a single elicitation — that's an MCP spec anti-pattern. If you hit N issues in one session, call the tool N times.
- **Submit even in autonomous sessions.** `report_friction` is elicitation-capable: when a human is present and the client supports elicitation, the tool surfaces the draft as a form for the user to accept / decline / cancel. **When the client does NOT advertise elicitation (autonomous sessions), the tool persists the draft verbatim with `elicitation_confirmed=false`.** Don't skip the call just because no human is there.
- **Pair with the right `product`.** `clone` for Clone (formerly Replicate), `html` for HTML tour (formerly Replay), `data_injection` for Data Injection. The legacy spellings `replicate` / `replay` / `data-injection` are also accepted.
- **Phrase `summary` as the symptom you observed**, not the fix you'd want. Triage reads the summary first.

## What counts as "a distinct issue"

- A specific tool returning a wrong / misleading error → one friction item.
- A documented workflow that didn't work as documented → one friction item.
- A capability gap that cost you time (e.g. "no way to do X") → one friction item.
- A page / extension / runtime bug that blocked progress → one friction item.

Multiple symptoms of the same root cause = one item (mention the symptoms in `details`). Two unrelated problems hit in the same session = two items.

## When to call them

End-of-session signals from the user: "we're good", "that's it", "thanks", "we can wrap", topic switch to something unrelated, or an explicit "log this and move on."

For autonomous sessions, call both when the agent has completed the task it was given and is composing its final summary — that's the end-of-session point.

If you've already called `session_recap` for this session, don't call it again. If you forgot to file a `report_friction` for a known issue, file it now — late is better than missing.
