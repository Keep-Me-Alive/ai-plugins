# Tool reference

Every tool the `keep-me-alive` connector publishes, with the arguments it actually accepts.

Two rules hold everywhere:

- **Inputs are strict.** An unknown key is an error, not a silently ignored typo. `due_at`
  is not `due`, and the difference costs a reminder.
- **Timestamps are ISO-8601 with an offset** (`2026-08-16T10:00:00+02:00` or `…Z`). An
  offset-less string is rejected. Resolve relative phrases yourself against `tz` from
  `get_settings`.

Field names on the wire are `snake_case`; the structured result is `camelCase`
(`dueAt`, `nagEverySec`, `fieldVersions`).

## Writing

### `create_thread`

| Argument | Type | Notes |
| --- | --- | --- |
| `title` | string, 1–500 | Required. Trimmed |
| `due` | ISO datetime | Present → one-off. With `recur`, it pins the *first* occurrence |
| `recur` | RecurSpec | See [scheduling.md](scheduling.md). Present → recurring |
| `nag` | boolean | Re-notify until acknowledged. Default false |
| `nag_every_sec` | int 15–86400 | Overrides the account default (900 s = 15 min) for this thread |
| `first_comment` | string, 1–8000 | Seeds the history |
| `area` | string, 1–100 | Free-text grouping label. Nothing behaves differently because of it |

Neither `due` nor `recur` → an **undated** thread, which is a deliberate state: it appears in
the morning digest every day until it is completed.

### `add_comment`

`thread_id` (required), `body` (1–8000). Append-only, versionless, never conflicts. The most
frequently useful tool in the set.

### `update_thread`

`id`, `patch`, optional `base_versions`.

`patch` accepts `title`, `area`, `due`, `recur`, `nag`, `nag_every_sec` and must change at
least one. **Absent** leaves a field alone; **`null`** clears it (`area`, `due`, `recur`,
`nag_every_sec` are nullable; `title` and `nag` are not).

`due` and `recur` are resolved together into the thread's schedule:

| After the patch | Resulting schedule |
| --- | --- |
| `recur` non-null | recurring; a missing `due` is derived from the spec |
| `recur` null, `due` non-null | one-off |
| both null | undated |

So turning a recurring thread into a single reminder takes `{"recur": null, "due": "…"}` —
patching only `due` leaves it recurring.

`base_versions` takes the counters from a thread you just read (`fieldVersions`: `title`,
`area`, `schedule`, `nag`). Omitting the argument entirely is last-write-wins.

**A counter you leave out is not checked — it is skipped, not compared against 0.** That field
quietly falls back to last-write-wins, and it bites hardest on young threads: a thread nobody
has edited yet has `fieldVersions: {}`, so "send back what you read" is `base_versions: {}`,
which checks nothing at all. Send an explicit `0` for every field you are about to change.

A conflict returns **409 with nothing applied** — re-read, merge, retry. All schedule-shaped
fields share one `schedule` counter, so `due`, `recur` and a snooze all move it together.

### `complete_thread`

`id`, optional `comment`.

- One-off or undated → closed (`status: "done"`).
- **Recurring → not closed.** The occurrence is completed, the thread advances, and
  `nextDueAt` comes back in the result. Say when the next one is.

### `reopen_thread`

`id`. Brings back a completed thread with its entire history intact.

### `stop_recurring`

`id`. Ends the series for good and **closes** the thread — "stop" means stop asking, not
"leave it lying around undated". Fails with *thread is not recurring* on anything else.

### `snooze_thread`

`id`, `until` (ISO, strictly in the future — a past instant is rejected).

Defers the next push only. The recurrence anchor is untouched, so a daily thread snoozed to
this evening still fires tomorrow at its usual time. An active nag run is cleared, and the
snooze is written into the history.

### `ack_nag`

`id`. Silences the current nag run without completing anything or changing the schedule. The
next occurrence will nag again.

### `update_settings`

At least one of `tz` (IANA zone), `locale` (`en` | `de` | `uk`), `digest_hour` (0–23 local),
`quiet_start` / `quiet_end` (`HH:MM`, may cross midnight), `quiet_enabled` (boolean),
`nag_every_sec_default` (15–86400 seconds; 900 is the default).

Quiet hours are two independent settings. `quiet_start` / `quiet_end` are the *window*;
`quiet_enabled` is whether it applies. **"Turn quiet hours off" is `quiet_enabled: false`** —
never collapse the window to a single time. The window is remembered while switched off, so
`quiet_enabled: true` hands the user back the hours they configured instead of asking them to
type them again. The three states this produces are in [scheduling.md](scheduling.md).

`locale` changes the language of *everything the user reads* — notifications, the digest, the
tool descriptions themselves. Confirm before changing it on a guess.

### `link_telegram`

No arguments. Returns a single-use link (valid 15 minutes) plus a typeable code. Telegram is
where notifications and the morning digest are delivered, so a user with no linked chat gets
no pushes at all — worth checking when they say reminders are not arriving.

## Reading

### `list_threads`

| Argument | Values |
| --- | --- |
| `status` | `open` \| `done`. Omitted → both |
| `filter` | `overdue` \| `today` \| `undated` \| `upcoming` \| `nagging` |
| `query` | Case-insensitive substring, **titles only** |
| `limit` | 1–200, default 50 |

Filter semantics, all evaluated in the user's local calendar and snooze-aware:

- `overdue` — effective due time is in the past.
- `today` — overdue **plus** still to come today. Something that slipped past 09:00 is very
  much part of today.
- `upcoming` — later today plus everything beyond.
- `undated` — no due date at all; these are digest regulars.
- `nagging` — currently in an active nag run.

Order is fixed: overdue first (oldest first), then by due time, then undated by recency.
`total` is the count *before* `limit`, so "showing 20 of 53" is honest. The prose result is
capped at 25 lines; the structured payload carries the rest.

### `get_thread`

Exactly one of `id` **or** `title_query`. Returns the thread plus its full timeline, oldest
first, including `source: "system"` entries the app wrote itself (completed, reopened,
snoozed, nag acknowledged).

`title_query` is a substring match that returns one thread: open ones before closed, then
most recently updated. It cannot tell you it was unsure — use `search` when the phrase is
vague.

### `search`

`query` (1–200), `include_done` (default false), `limit` (1–100, default 25).

Searches titles **and** comment bodies; a title hit outranks a comment hit, and the matching
comment comes back with the hit so you can quote what was found.

### `digest_preview`

No arguments. Runs the digest builder **at the current instant** and returns what it produces:
overdue, then undated open threads, then what is still due later today.

It is the same code the morning push runs, not a copy — but it is evaluated *now*, and the
digest never reaches past the end of today. Called at 22:00 it will not list tomorrow's 10:00
item, which tomorrow's 08:00 digest certainly will. So: the best available answer to "what am I
sitting on?", and **not** a preview of tomorrow's push. Call it in the morning and it is the
push; call it in the evening and say what it is.

### `stats`

No arguments. Counts of open, overdue, undated and nagging threads. Cheap; good for an
opening line before a longer review.

### `calendar_link`

`id`. Builds a Google Calendar link for the thread's next occurrence, which the user opens to
pick which calendar it lands in. It does not create anything — hand over the link.

### `get_settings`

No arguments. Timezone, language, digest hour, quiet hours — the window (`quietStart`,
`quietEnd`) **and** whether it applies (`quietEnabled`) — and the default nag interval. Call it
once at the start of a conversation, before resolving any relative date.

## Failure modes

| Message | Meaning | What to do |
| --- | --- | --- |
| *thread not found* | No such thread for this account | Re-find it; never guess an id |
| *thread changed on another device* (409) | Concurrent edit, **nothing applied** | Re-read, merge, retry |
| *snooze target must be in the future* | `until` was in the past | Recompute in the user's `tz` |
| *thread is not recurring* | `stop_recurring` on a one-off | Use `complete_thread` |
| *Invalid arguments — …* | Schema rejection | Read the detail; usually a stray key or a naked datetime |
| *Telegram is not configured…* | Deployment has no bot | Nothing the user can fix from here |

Errors come back as readable text rather than a protocol failure — read them and adapt
rather than repeating the same call.

The quoted wording is the `en` one. Every message here is localised, so a `de` or `uk` account
gets the same failures in German or Ukrainian down the same path. **Match on meaning, never on
the literal string.**
