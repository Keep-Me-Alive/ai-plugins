# Time, recurrence and notifications

Everything about *when* Keep Me Alive speaks. Read this before building a `recur` spec,
before choosing between a snooze and a reschedule, and whenever the user says a reminder
arrived at the wrong time.

## Resolving what the user said

`get_settings` gives you `tz`. Resolve every relative phrase in that zone and send an
absolute instant with an offset.

| The user says | You send |
| --- | --- |
| "in an hour" | now + 1h, in their zone, with offset |
| "tomorrow at 10" | 10:00 local tomorrow |
| "Friday" | the next Friday; pick a sane hour (their digest hour or 09:00) and say which |
| "end of the month" | the last day, at a stated hour |
| "next week" | ambiguous — pick Monday, say so, and move on |

Two habits that avoid nearly every complaint:

1. **Say the resolved time back in words.** "Thursday 14:00" gives the user a chance to
   correct you; an ISO string does not.
2. **Never assume a bare date means midnight.** A reminder at 00:00 is a reminder nobody
   reads. Choose a waking hour and name it.

## Building a `recur` spec

```jsonc
{
  "n": 1,                                // every n units, 1–999
  "unit": "hour" | "day" | "week" | "month",
  "anchor": "schedule" | "completion",   // default "schedule"
  "at": { "hour": 9, "minute": 0 },      // local fire time
  "weekdays": [1, 4],                    // ISO 1=Mon … 7=Sun — `week` only
  "day": 15                              // 1–31 — `month` only
}
```

Rules the schema enforces, so getting them wrong is a rejected call:

- `weekdays` only with `unit: "week"`; `day` only with `unit: "month"`.
- For `unit: "hour"`, only `at.minute` matters — the hour comes from when it starts.
- Hourly is the finest cadence there is. Nothing repeats by the minute.

Behaviour worth knowing:

- **`anchor: "schedule"`** (default) is a fixed cadence: "every Monday 09:00", whatever you
  do. Missed occurrences are **skipped, not queued** — a week offline does not owe the user
  seven pings.
- **`anchor: "completion"`** means "n units after I last did it": the next occurrence is
  computed from the completion. Right for "water the plants every 3 days", wrong for rent.
  It stays overdue until actually completed, which is the point.
- **Wall clock wins over elapsed time.** A 09:00 daily thread fires at 09:00 local on both
  sides of a DST change.
- **Monthly remembers the day you meant.** `day: 31` lands on 28 February and returns to the
  31st in March.
- A recurring thread created without `due` starts at the next slot, and uses the current
  period if it is still ahead: "every day at 09:00", created at 08:00, fires *today*.

## Snooze, reschedule, or stop

| Situation | Call | Why |
| --- | --- | --- |
| "not now, later today" | `snooze_thread` | One-shot override of the next push; the series is untouched |
| "actually it's due Thursday, not today" | `update_thread {patch: {due: …}}` | The schedule itself was wrong |
| "make it weekly instead of daily" | `update_thread {patch: {recur: …}}` | New cadence |
| "stop asking about this" | `stop_recurring` | Ends the series and closes the thread |
| "it's done for now, but it'll come back" | `complete_thread` | On a recurring thread this advances rather than closes |

A snooze on a recurring thread defers only the pending push. Tomorrow's occurrence still
lands at its normal time — if the user wants the whole cadence moved, that is an
`update_thread`.

## Nagging

`nag: true` means: when the thread fires, keep re-notifying every `nag_every_sec` (or the
account default, 900 seconds) until the user acknowledges. Acknowledgement is `ack_nag`, the
👌 button in Telegram, a snooze, or completion — all four end the current run. The *next*
occurrence starts a fresh run.

Reserve it for things with real consequences. A nag on something optional teaches the user to
ignore the one that mattered. `list_threads {filter: "nagging"}` shows what is currently
buzzing.

## Quiet hours

Pushes inside the quiet window (default 22:00–07:00 local) are **held, not dropped**, and go
out when the window ends. The window is half-open: a `quiet_end` of 07:00 means the 07:00
push is delivered.

There are **three** states, and `get_settings` gives you both halves of the answer —
`quietStart` / `quietEnd`, and `quietEnabled`:

| State | The account | What happens |
| --- | --- | --- |
| **On** | `quietEnabled: true`, and the window has length | Pushes inside the window are held until it ends |
| **Off** | `quietEnabled: false` | Nothing is ever held back. The window is still stored, so switching back on restores it |
| **No window** | `quietStart` equals `quietEnd` | Nothing is held back either, whatever the switch says. A legacy state — collapsing the window used to be the only way to stop the 3am pushes — so an account in it usually *meant* "off". Offer a real window, or `quiet_enabled: false` said out loud |

**"Turn quiet hours off" is `update_settings {quiet_enabled: false}`.** Never collapse the
window to a single time to achieve it: that produces the third row, and it throws away the
hours the user chose. From the phone the same switch is `/quiet off`, `/quiet on`, and
`/quiet 23:00-08:00` to set the window.

So "I set it for 03:00 and heard nothing" is usually correct behaviour, and worth saying out
loud rather than debugging — but read `quietEnabled` before you say it. On an account with
quiet hours switched off, quiet hours are not the explanation.

## The morning digest

Sent once a day at `digestHour` local, in three fixed sections:

1. 🔥 **overdue**
2. 📌 **undated open threads** — the daily resurface that makes an undated thread safe to
   create
3. 📅 **due later today**

It is capped at roughly 25 lines, with every non-empty section guaranteed a share, so a pile
of overdue items cannot crowd out the undated list. Threads due beyond today are absent by
design — the digest is about today, not a forecast.

`digest_preview` returns exactly the text that would be sent **if the digest went out now** —
same builder, current instant, same "today only" cut.

## Where notifications actually go

Telegram, to the chat the user linked. No linked chat means no pushes, however perfect the
schedule is: `link_telegram` returns a one-time link that fixes it. Inline buttons on a push
(✅ Done / 💤 Snooze / 👌 Ack) run the same commands these tools do, so a thread the user
handled from their phone may already have moved when you next read it — one more reason to
send `base_versions` on an edit you based on an earlier read.
