# When something isn't wired up

## The `keep-me-alive` tools are not in this conversation

Nothing in this skill works without them, and there is no fallback worth offering — say that
plainly rather than keeping a list in the chat that nobody will ever see again.

**Check the connector first.** The Keep Me Alive plugin brings this skill and the connector
together, so when this skill is loaded the plugin is usually installed — but installing a plugin
does not sign its connector in, and may not even add it to the account. On claude.ai:
**Settings → Connectors**, then **Connect** on Keep Me Alive. If it is not listed there, the
install did not add it: open the plugin under **Customize → Plugins**, add the connector from the
plugin's own **Connectors** tab, then connect it. Claude then asks to sign in, and the request
arrives in the user's Telegram chat with the bot: approving it there is what connects the
connector to that account.

To set it up from the start:

1. Open the Keep Me Alive bot in Telegram and send `/start`, then accept the terms. That
   creates the account and makes the chat the first notification channel.
2. Send `/connect_ai` to the bot (`/claude` still works — it is the same command under its old
   name). It replies with the steps below, and a button to switch to the Claude Code install
   instead.
3. On claude.ai: **Customize → Plugins → Add → Add marketplace**, enter
   `Keep-Me-Alive/ai-plugins`, and install **Keep Me Alive** from it.
4. Connect it and approve the sign-in in Telegram, as above. It then works wherever the user
   talks to Claude with that account, the mobile apps included.

**Without plugins** — they need a paid plan — the connector can be added on its own:
**Settings → Connectors → Add custom connector**, pasting the address `/connect_ai` ends with,
`https://<api-host>/mcp`, the same for everybody. The sign-in is the same.

**The address is not a credential and does not need protecting.** It used to be — a
`https://<api-host>/mcp?k=…` URL that was the whole of the account's access — and that scheme
was retired. A `?k=…` URL kept from before now opens nothing at all.

What *is* worth caring about is the approval in Telegram. It names the app, the address the
data would be sent to, and a short code that must match the screen the request was started
from. If the user is looking at an approval they did not start, the answer is **don't
connect** — and a connection that was approved cannot presently be withdrawn from Telegram
short of `/delete`, so it is the tap, not the URL, that deserves the care.

## Threads exist but no notifications arrive

Work down this list before assuming a bug:

1. **Quiet hours.** `get_settings` returns the window (`quietStart` / `quietEnd`, default
   22:00–07:00 local) *and* the switch (`quietEnabled`). Quiet hours explain a missing push
   only when they are **on** and the missing push fell inside the window — it is held until
   the window ends, not dropped. If `quietEnabled` is false, or the window starts and ends at
   the same time, nothing was ever held back and the answer is further down this list.
   `update_settings {quiet_enabled: false}` switches them off, and `/quiet` does the same from
   Telegram — but only offer that once you know quiet hours are what swallowed the push.
2. **No linked Telegram chat.** Notifications and the digest are delivered there and nowhere
   else. `link_telegram` returns a one-time link (15 minutes) that fixes it.
3. **Nothing to say.** An empty digest is not sent. `digest_preview` shows whether there was
   anything for it to report.
4. **Undated threads have no push.** They appear in the morning digest only — that is what
   undated means.
5. **The thread is snoozed.** A snooze suppresses the next push without changing anything
   visible in the schedule.

## The account speaks the wrong language

`update_settings {locale: "en" | "de" | "uk"}` changes every word the user reads —
notifications, the digest, and the tool descriptions themselves. Existing history is left in
the language it was written in, deliberately: it is a record of what happened, not a label.

## Timezone looks wrong

`update_settings {tz: "Europe/Berlin"}` takes any IANA zone. Everything is recomputed from it
— digest hour, quiet hours, and the local wall-clock time recurring threads fire at — so
confirm before changing it, and re-read anything you had already resolved.
