---
description: Quickly log something to track or be reminded of with Keep Me Alive, a two-minute countdown included, without asking for details first
argument-hint: [what to track]
---

The user wants to track $ARGUMENTS with Keep Me Alive, or is about to list more than one thing
in the same message. Follow the "Capture beats completeness" rule from the keep-me-alive
skill:

1. Split on the obvious boundaries — one thread per commitment, not per sentence.
2. `create_thread` for each, right away. Give a `due` only where a time was actually stated;
   leave the rest undated on purpose.
3. Put context that is not a title into `first_comment` rather than bloating the title.
4. Confirm once, as a short list, with resolved times in words. Do not interview the user
   first — no "what deadline?", no "should this repeat?".

If the keep-me-alive skill or its MCP tools are not available in this conversation, say so
plainly and stop — there is no local substitute for either.
