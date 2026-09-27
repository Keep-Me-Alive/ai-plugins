---
description: Run the Keep Me Alive morning triage — what's overdue, undated, and due today
---

Run the "Morning triage" workflow from the keep-me-alive skill:

1. Call `digest_preview` — the digest rebuilt for right now, in the user's language.
2. Read it back grouped, shortest form possible: overdue first, then undated, then later today.
3. Handle each answer as it comes (`complete_thread`, `snooze_thread`, `add_comment`,
   `stop_recurring`) — one call per thread, each named before you make it.
4. Close with what is left, not with what was cleared.

If the keep-me-alive skill or its MCP tools are not available in this conversation, say so
plainly and stop — there is no local substitute for either.
