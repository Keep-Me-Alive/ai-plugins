# Workflows

Multi-step routines worth doing the same way every time. Each one assumes `get_settings` has
already been called this conversation.

## Brain dump → threads

The user lists five things in one message. Capture all of them, then confirm once.

1. Split on the obvious boundaries; one thread per commitment, not per sentence.
2. `create_thread` for each. Give a `due` only where the user gave a time — the rest stay
   undated on purpose.
3. Put context that is not a title into `first_comment` rather than bloating the title.
4. Confirm as a short list, resolved times in words:
   > Tracking 5: invoice → Thu 10:00 · Vlad's contract → undated · …

Do not stop halfway to ask a clarifying question. Capture everything, then ask at the end if
one genuinely needs it.

## Morning triage

When the user opens with "what's on today?", "what am I sitting on?", or answers the digest.

1. `digest_preview` — the digest rebuilt for right now, in their language. Asked in the
   morning, that is what their push said; asked later, it has moved on, and either way it
   stops at the end of today.
2. Read it back grouped, shortest form possible: overdue first, then undated, then later
   today.
3. Then handle the answers as they come:
   - "did that" → `complete_thread`
   - "not today" → `snooze_thread` to a stated time
   - "waiting on X" → `add_comment`, leave the schedule alone
   - "that's not a thing any more" → `complete_thread`, or `stop_recurring` if it recurs
4. Close with what is left, not with what was cleared.

Batch the reading, never the writing: each completion is a separate call, and each one gets
named before you make it.

## Chase sweep — "what am I waiting on?"

1. `list_threads {filter: "overdue"}` and `list_threads {filter: "undated"}`.
2. For anything the user reacts to, `get_thread` for its history — the timeline is what says
   how long they have been waiting and how many times they have asked.
3. Offer the concrete next action per thread: chase again today, set a hard date, or drop it.
4. Whatever they decide lands as a comment even if nothing else changes. "Third ask, no
   reply" is the fact that justifies escalating next week.

## Logging progress from a conversation

The user is telling you about their day, not asking for anything. Anything that touches a
tracked thread still belongs on it.

1. Recognise the reference ("Vlad finally replied", "the migration slipped").
2. `search` if the phrase is vague, `get_thread {title_query}` if it is distinctive.
3. `add_comment` with what was said, in their words, not a summary of your own.
4. If it changes the timing, propose the reschedule rather than doing it silently.
5. One-line acknowledgement. Do not turn a side remark into a status meeting.

## Weekly review

1. `stats` for the shape of it — open, overdue, undated, nagging.
2. `list_threads {filter: "undated", limit: 200}` — this is where rot accumulates. For each:
   still real? Then either give it a date or leave it deliberately undated. Not real?
   Complete it.
3. `list_threads {filter: "nagging"}` — anything permanently nagging is a nag that has stopped
   working. Suggest `nag: false` or a real deadline.
4. `list_threads {status: "done", limit: 50}` for what actually got finished — worth saying
   out loud, it is the only part of the week that is invisible by default.

## Answering a reminder that just fired

The user replies to a push. The thread is already in an active state, so the mapping is
tight:

| They say | Call |
| --- | --- |
| "done" | `complete_thread` (say the next occurrence if it recurs) |
| "seen it, stop buzzing" | `ack_nag` |
| "in an hour" / "tomorrow" | `snooze_thread {until}` |
| "it's blocked on someone else" | `add_comment`, then ask if the date should move |
| "cancel it" | `complete_thread`, or `stop_recurring` for a series |

## Migrating a list from somewhere else

The user pastes a todo list, a set of Slack follow-ups, or an email backlog.

1. `search` a couple of the distinctive ones first — do not create duplicates of threads that
   already exist.
2. Create the rest, preserving whatever dates the source carried and leaving the others
   undated.
3. Report the count, and name anything you skipped as an existing thread.

## Handing a thread to a calendar

`calendar_link {id}` returns a Google Calendar URL for the next occurrence. Give the user the
link — it opens the event and lets them choose which calendar it lands in. It does not create
anything, so do not claim it did, and it never replaces the thread: the thread is what chases
them, the calendar entry is just a block of time.
