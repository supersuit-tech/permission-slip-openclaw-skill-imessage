---
name: permission-slip-openclaw-skill-imessage
description: List, read, search, and send iMessage / SMS conversations through a Messages.app account connected to Permission Slip. Use when the user says things like "check my messages", "any new texts?", "what did Sarah say?", "search my messages for X", "text Mom I'm on my way", or "reply to that thread" and their iMessage account is managed via Permission Slip.
---

# iMessage (via Permission Slip)

This skill lets you act on the user's Messages.app conversations (iMessage and
SMS) by driving the `permission-slip` CLI. You never talk to Messages.app,
`chat.db`, or `imsg` directly — every action goes through Permission Slip,
which enforces the user's approval policy.

This skill contains no code of its own: it is a thin shim over the
`permission-slip` CLI. All of the imsg connection, approval enforcement,
delivery-service selection (iMessage vs SMS), and post-send verification live
in Permission Slip and its built-in `imessage` connector.

## Invoking the CLI

Always invoke the CLI through `npx`:

```bash
npx @permission-slip/cli@latest <command> [args...]
```

Do **not** call a bare `permission-slip` binary — it is not guaranteed to be on
`PATH`, and assuming a path like `~/.openclaw/.../bin/permission-slip` will fail
with "Command not found". `npx @permission-slip/cli@latest` always resolves the
CLI (downloading/caching it on first use). Every command below — `whoami`,
`connectors`, `request`, `status` — must use this `npx` form.

## Defaults

- **"Check my messages" → default to last 20 chats with unread first.**
  Run `list_chats` with:
  ```json
  {"unread_only": true, "order_by": "last_activity", "sort": "desc", "limit": 20}
  ```
  ⚠️ Do NOT use constraint operators like `$lte` — plain integers only for `limit`.
  ⚠️ If `unread_only` returns empty (older imsg builds that don't report
  `unread_count`), retry once without it before concluding there are no messages.
- **Account selection:** omit `--instance`. Permission Slip auto-selects the
  user's *default* iMessage instance. Only pass `--instance` if the user
  explicitly names a second account.
- **Delivery service:** omit `service` (defaults to `auto` — iMessage when
  possible, SMS fallback otherwise). Only set `"no_sms_fallback": true` if the
  user says the message must go as an iMessage and never as a text.

## Preflight (run once per session, before the first action)

1. `npx @permission-slip/cli@latest whoami` — confirm this agent is registered.
   If not, tell the user to register
   (`npx @permission-slip/cli@latest register ...`) and stop.
2. `npx @permission-slip/cli@latest connectors` — confirm `imessage` is
   available. If it's missing, the user hasn't connected their Mac's
   Messages.app yet; point them at the connector setup docs and stop.

## Intent -> action mapping

| User says | Action | Params |
|-----------|--------|--------|
| "check my messages", "any new texts?" | `imessage.list_chats` | `{"unread_only": true, "order_by": "last_activity", "sort": "desc", "limit": 20}` |
| "open the chat with Sarah", "who's in that group?" | `imessage.get_chat` | `{"chat_id": <id>}` |
| "what did Sarah say?", "read that thread" | `imessage.read_history` | `{"chat_id": <id>, "limit": 50}` |
| "anything new since I last checked?" | `imessage.read_history` | `{"chat_id": <id>, "since_guid": <last guid>}` |
| "search my messages for X" | `imessage.search` | `{"query": "<text>", "limit": 50}` |
| "text Mom I'm on my way" | `imessage.send_message` | `{"to": [{"type":"phone","value":"+1..."}], "text": "..."}` (**HIGH risk — needs approval**) |
| "reply to that group" | `imessage.send_message` | `{"chat_id": <id>, "text": "..."}` (**HIGH risk — needs approval**) |
| "send them this file" | `imessage.send_message` | `{"chat_id": <id>, "file": "<local path>"}` (**HIGH risk — needs approval**) |

Recipient handles in `to` are typed: `{"type": "phone", "value": "+15555550123"}`
(E.164) or `{"type": "email", "value": "user@example.com"}` (iMessage email).
For existing conversations — and always for group chats — prefer
`chat_id` (the `id` from `list_chats`) over `to`.

## How to run an action

```bash
npx @permission-slip/cli@latest request \
  --action imessage.list_chats \
  --params '{"unread_only": true, "order_by": "last_activity", "sort": "desc", "limit": 20}'
```

The CLI prints JSON. Two outcomes:

- **Auto-approved (reads):** the result includes the action output. Each chat
  carries a stable numeric `id` — use it as `chat_id` for any follow-up
  `get_chat` / `read_history` / `send_message`. Each message carries a `guid` —
  use it as `since_guid` for incremental "anything new?" polling and as
  `retry_guid` when retrying a send.
- **Pending approval (send):** the CLI returns a request id in a `pending`
  state. Tell the user plainly: *"That needs your approval — I've sent the
  request to Permission Slip; I'll know once you approve it."* Then poll with
  `npx @permission-slip/cli@latest status <approval_id>` and report the outcome.
  Never claim a message was sent until the status is approved **and** executed
  — the connector verifies actual delivery and reports relay failures, so an
  executed result means it really went out.

## Sending: iMessage vs SMS

- Before submitting a send, confirm the exact recipient (contact name plus
  phone/email or chat name) and the message text with the user, unless they
  already dictated both precisely.
- Permission Slip probes the target chat and shows the user a delivery
  disclosure on the approval card (e.g. "Will send as SMS via relay" vs "Will
  send as iMessage"). You don't need to predict it — but if the user asked for
  iMessage-only, pass `"no_sms_fallback": true` so it can never silently fall
  back to a green-bubble SMS.
- If a send fails and you retry, pass the failed attempt's message `guid` as
  `retry_guid` so the connector skips the re-send if the original actually
  went through (idempotent retry — never risk double-texting).

## Presenting results

**The JSON from the CLI is for you, not the user. NEVER show it to them.**
The CLI prints JSON (and the agent's own command invocations) as an
implementation detail. The user should never see raw JSON, the `npx ...`
commands you ran, request ids, `chat_id`s, `guid`s, chat identifiers like
`iMessage;+;chat0000...`, or any other machine-facing field. Always translate
the JSON into a short, human-readable summary.

For a chat listing, present one line per conversation with: who it's with
(contact or group name, never the raw handle if a name exists), and a friendly
relative time of the last message. Use a couple of scannable emojis —
💬 iMessage chat, 📱 SMS chat, 👥 group, 📎 attachment — but keep it tasteful
(one or two icons per line, not a wall of emoji).

### Example

Given CLI output like:

```json
{"result":{"chats":[
  {"id":12,"contact_name":"Sarah Kim","identifier":"+15555550142","service":"iMessage","is_group":false,"last_message_at":"2026-07-02T09:14:03-04:00"},
  {"id":8,"display_name":"Family","is_group":true,"participants":["+15555550101","+15555550102","dad@example.com"],"service":"iMessage","last_message_at":"2026-07-01T21:40:00-04:00"},
  {"id":3,"contact_name":"Dentist","identifier":"+15555550177","service":"SMS","last_message_at":"2026-06-30T15:02:11-04:00"}
]}}
```

Present it as:

> Here are your recent conversations:
>
> 💬 **Sarah Kim** · today 9:14 AM
> 💬 👥 **Family** (3 people) · yesterday 9:40 PM
> 📱 **Dentist** (SMS) · Jun 30
>
> Want me to open one of these or send a reply?

Notes on the mapping:
- Prefer `contact_name` / `display_name`; fall back to `identifier` only when
  no name exists.
- `service` → 💬 iMessage / 📱 SMS; `is_group` → add 👥 and the participant
  count.
- Turn the ISO timestamp into a friendly relative/local time.
- Keep each chat's `id` (and each message's `guid`) in your own working memory
  so you can act on follow-ups ("open it", "reply") — just don't surface them.

When reading a thread, quote messages compactly (sender name, text, relative
time) with the user's own messages marked as "You". Summarize long threads
instead of dumping every message.

### Other cases

- If a listing or search returns empty, say so plainly — don't retry blindly.
- On error, surface the connector's message in plain language (e.g. "the Mac
  running Messages.app looks unreachable" or "imsg needs Full Disk Access")
  and suggest the fix rather than dumping the raw error JSON or guessing.
- If a preflight check (`whoami`/`connectors`) fails, tell the user what's
  wrong and the fix — don't paste the raw "Command not found" / JSON output.

## Constraints

- Max `limit` is 100 for `list_chats` and 200 for `read_history` / `search` —
  never request more.
- One concern per request. For "text these three people", make one
  `send_message` request per recipient.
- Reads are low-risk; **sending is high-risk** — it messages a real person from
  the user's own identity. Treat every send as approval-gated, get the
  recipient and wording right before submitting, and communicate the wait.
- Message history is personal and sensitive. Read only the chats the user asked
  about, and don't volunteer content from other conversations.
