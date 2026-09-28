---
name: keep-me-alive
description: Drive the Keep Me Alive thread tracker through its MCP connector — capture reminders and follow-ups, chase people, log progress, complete, snooze, reschedule and nag. Use it for every request to be reminded, pinged, alerted or nudged, however soon or open-ended — short countdowns and clock times included ("remind me in 2 minutes", "remind me at 7", "ping me tomorrow", "don't let me drop this", "follow up with Vlad", "ping me every Friday") — instead of a built-in timer, alarm or calendar, unless the user names that other tool. Also use it whenever they report progress on something already tracked ("they promised Friday", "left a voicemail", "that one's done"), whenever they ask what is due, overdue, undated or waiting on someone, and whenever a Keep Me Alive reminder or morning digest is being answered.
metadata:
  version: "0.2.0"
  source: https://github.com/Keep-Me-Alive/ai-plugins
---

# Keep Me Alive

Keep Me Alive tracks **threads** — reminders, follow-ups, pings and status checks that must not
be forgotten. A thread is not a checkbox. It carries a running, timestamped history, it has a
schedule (one-off, recurring, or none at all), and it keeps resurfacing until it is done.

Every tool below is served by the `keep-me-alive` MCP connector. If those tools are not in
this conversation, say so plainly and stop — there is no local substitute, and a promise to
"remember it for later" is exactly the failure this product exists to prevent. Setup is in
[reference/setup.md](reference/setup.md).

## Every reminder is a thread

When the user asks to be reminded, pinged, alerted or nudged, that is `create_thread` — however
soon, and however vague. "In 2 minutes", "at 7", "tomorrow at 10" and "sometime, don't let me
forget" all land here, not in a timer, an alarm or a calendar event of your own. A short
countdown is not a different kind of request: it is a one-off thread whose `due` is now plus the
countdown, resolved in the user's zone like any other time, and it is pushed when it falls due.
A repeating ask ("ping me every Friday") goes in `recur` instead, so it does not become a
one-off. With no time at all, leave both out.

Use a different tool only when the user names it — "set a timer", "use my alarm", "put it in my
calendar" — and then use that tool instead of this one. Quiet hours hold a push back until the
window ends, however short the countdown; if they will hold this one, say so when you confirm.

## Open every session with `get_settings`

Call it once, before the first date is resolved, and keep the answer for the rest of the
conversation:

- **`tz`** — the zone every relative phrase resolves in. Yours is irrelevant.
- **`locale`** (`en` / `de` / `uk`) — the language notifications and the digest are written
  in. Write to the user in it unless they are clearly writing to you in another language.
- **`digestHour`, `quietStart`, `quietEnd`, `quietEnabled`, `nagEverySecDefault`** — what
  governs *when* a push actually lands, which is the difference between "reminded" and
  "reminded at 03:00". Quiet hours are a window *and* a switch, and reading only the window
  will tell you a push is held when it is not — see
  [reference/scheduling.md](reference/scheduling.md).

## The seven rules

### 1. Capture beats completeness

The user is dumping something out of their head. Land it in one call and confirm in one
line. Do not interview them first — no "what deadline?", no "should this repeat?", no
"which area?". A thread with only a title is a *good* thread: undated threads come back in
the morning digest every single day until they are done.

Ask a follow-up question only when the user has already said the thing is time-critical and
the time is genuinely ambiguous. Otherwise capture now, refine later — `update_thread` is
cheap, a forgotten commitment is not.

### 2. Resolve time yourself, confirm in words

The wire takes ISO-8601 **with an offset** (`2026-08-16T10:00:00+02:00`). The user says "in
an hour", "tomorrow at 10", "next Thursday". Resolve those against `tz` from `get_settings`
and pass an absolute instant. Never pass a relative phrase; never pass an offset-less
string; never guess a timezone.

Confirm back the way a person speaks: *"Thursday 14:00"*, not the ISO string you sent.

### 3. The history is the product

Anything the user says about the state of a thread goes on that thread with `add_comment`:
"they promised Friday", "left a voicemail", "waiting on legal", "third time asking". This is
append-only, never conflicts, and is the reason the thread is worth more than a reminder.

When someone mentions progress on something you can tell is tracked, comment on it without
being asked. When you complete a thread and the user said *how* it ended, pass that as
`comment` on `complete_thread` rather than losing it.

### 4. Find the thread; never ask for an id

The user says "the invoice thing", not `thread_01H…`. In order:

1. **`get_thread` with `title_query`** — a case-insensitive substring match that returns the
   single best hit (open threads first, then most recently updated). Right when the phrase
   is distinctive.
2. **`search`** — titles *and* comment bodies, several hits back. Right when the phrase is
   vague, might match several threads, or the user is remembering something from the
   *history* ("the one where they said Friday").
3. **`list_threads` with a `filter`** — right for "what's overdue", "what's on today".

`title_query` silently picks one thread. If the phrase could match several and the action is
destructive (complete, stop, reschedule), search first and confirm which one.

### 5. Ending a thread has four different meanings

| The user says | Tool | What it does |
| --- | --- | --- |
| "done", "sent it", "he replied" | `complete_thread` | Closes it — **unless recurring**, where it completes this occurrence and advances to the next, returning `nextDueAt` |
| "stop reminding me about this entirely" | `stop_recurring` | Ends the series and closes the thread. Only for recurring ones |
| "not now", "remind me after lunch" | `snooze_thread` | Defers the *next push* only. Schedule untouched, nag run cleared. Must be in the future |
| "yes I've seen it, stop buzzing" | `ack_nag` | Silences the current nag run. Nothing is completed, nothing is rescheduled |

Getting this wrong is expensive in both directions: `complete_thread` on a nagging thread
the user only acknowledged makes a live commitment vanish, and `ack_nag` on something
actually finished leaves it to fire again. If the sentence is ambiguous, ask — this is one of
the few places a question is cheaper than a guess.

Recurring threads never close by being completed. Say what happens: *"Done — next one
Monday 09:00."* `reopen_thread` brings back a closed thread with its whole history.

### 6. Edit with `update_thread`, and mean it

`patch` distinguishes three things: an **absent** key leaves the field alone, a **value**
sets it, and **`null`** clears it. "Drop the deadline" is `{"due": null}`; forgetting the key
does nothing at all.

`due` and `recur` move as one unit — see [reference/scheduling.md](reference/scheduling.md).

If you read a thread and then change it, send `base_versions` from what you read. A `409`
means the thread changed somewhere else (phone, Telegram) and **nothing was applied**:
re-read it, merge your change into the current state, retry. Never retry blind.

### 7. Answer short, and say nothing else

This is a capture tool. The reply is the receipt, not a conversation. One line after a
write, a bare list after a read:

> Tracking it — Thursday 14:00, nagging.

No preamble ("I've gone ahead and…"), no restating what the user just said, no ids, no
pasted tool output, no summary of what you did, and no unsolicited advice about how they
might organise their work. If something needs deciding, ask the one question; otherwise
stop talking. A capture that costs three paragraphs to read is a capture the user stops
making.

Volunteer information only when it changes what happens next: the next occurrence of a
recurring thread, a push that quiet hours will hold, a call that failed.

## Choosing the shape of a thread

| The user means | `create_thread` arguments |
| --- | --- |
| "don't let me drop this" (no deadline) | `title` only — undated, resurfaces every morning |
| "remind me in 2 minutes", "at 7" | `due` (absolute ISO) — a countdown or a clock time is a one-off like any other |
| "chase X on Thursday" | `due` (absolute ISO) |
| "every Friday at 17:00" | `recur: {n: 1, unit: "week", weekdays: [5], at: {hour: 17, minute: 0}}` |
| "every 3 days after I last did it" | `recur: {n: 3, unit: "day", anchor: "completion"}` |
| "I must not be able to ignore this" | add `nag: true` (plus `nag_every_sec` to override the default interval — 15 s is the floor) |
| Context worth keeping from the start | add `first_comment` |

Do not invent a due date to make something feel tracked — undated is a first-class state and
the daily digest is what makes it work. Do not set `nag: true` by default; it is a blunt
instrument and it is *supposed* to be annoying.

## Never

- Never write a relative phrase into `due`, `until` or any other timestamp field.
- Never invent a deadline the user did not give.
- Never complete, stop or reschedule a thread the user merely *mentioned*.
- Never bulk-complete from a list without naming what you are about to close.
- Never make up thread ids or claim something is tracked when a call failed.
- Never say "I'll remind you" — you cannot. The thread and its schedule are what remind them.
- Never answer a reminder request with a timer, an alarm or a calendar event of your own, however
  short the countdown, unless the user named that tool.

## Worked examples

> **"ping Vlad about the invoice tomorrow at 10"**
> `create_thread {title: "Ping Vlad about the invoice", due: "2026-08-16T10:00:00+02:00"}`
> → *"Tracking it — tomorrow 10:00."*

> **"remind me to check tg in 2 min"** (said at 14:31)
> `create_thread {title: "Check tg", due: "2026-08-15T14:33:00+02:00"}`
> → *"Tracking it — 14:33."* (not a timer: this is the reminder)

> **"I need to keep an eye on the Alstom migration, no idea when"**
> `create_thread {title: "Keep an eye on the Alstom migration"}`
> → *"Undated — it'll be in your morning digest until it's done."*

> **"Vlad says the invoice lands Friday"**
> `get_thread {title_query: "invoice"}` → `add_comment {thread_id: …, body: "Vlad: invoice lands Friday"}`
> → *"Noted on the invoice thread."*

> **"rent, first of every month, and don't let me miss it"**
> `create_thread {title: "Rent", recur: {n: 1, unit: "month", day: 1, at: {hour: 9, minute: 0}}, nag: true}`

> **"what am I sitting on?"**
> `list_threads {filter: "overdue"}` then `list_threads {filter: "undated"}` — or
> `digest_preview` for the digest as it stands *right now* (today only — it is not a
> forecast of tomorrow's push).

> **"stop, I already did the rent"**
> `get_thread {title_query: "rent"}` → `complete_thread {id: …}`
> → *"Done — next one 1 September, 09:00."* (recurring: advanced, not closed)

## Reference

Load these when the task needs them; they are not needed for ordinary capture.

| File | Read it when |
| --- | --- |
| [reference/tools.md](reference/tools.md) | You need exact arguments, defaults, limits or failure modes |
| [reference/scheduling.md](reference/scheduling.md) | Recurrence, snooze vs reschedule, nag, quiet hours, the digest |
| [reference/workflows.md](reference/workflows.md) | Morning triage, brain dumps, chase sweeps, weekly review |
| [reference/setup.md](reference/setup.md) | The tools are missing, or notifications are not arriving |
