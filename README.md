# permission-slip-openclaw-skill-imessage

An [OpenClaw](https://github.com/supersuit-tech) agent skill that lets an agent
list, read, search, and send **iMessage / SMS** conversations on the user's
behalf — entirely through the [Permission Slip](https://github.com/supersuit-tech/permission-slip)
CLI, so every action respects the user's approval policy.

With this skill installed, a user can simply say **"check my messages"** and the
agent lists their recent Messages.app conversations via the iMessage connector,
or **"text Sarah I'm running late"** and the agent submits a send request for
the user to approve in Permission Slip.

## How it works

This skill contains **no code of its own**. It is a thin instruction layer over
the `permission-slip` CLI:

- The [imsg](https://github.com/openclaw/imsg) connection to Messages.app,
  approval enforcement, delivery-service disclosure (iMessage vs SMS), and
  post-send verification all live in Permission Slip and its built-in
  `imessage` connector.
- The skill's only job is mapping natural language to the right CLI command
  (`npx @permission-slip/cli@latest request --action imessage.<action> ...`)
  and presenting the JSON result.

Reads (`list_chats`, `get_chat`, `read_history`, `search`) are low-risk and
typically auto-approve. Sending is **high-risk** and approval-gated: the agent
submits the `send_message` request and waits for the user to approve it in
Permission Slip before reporting success. The approval card discloses how the
message will be delivered (blue-bubble iMessage vs green-bubble SMS relay).

## Prerequisites

- An OpenClaw machine configured as an agent for a Permission Slip server.
- The `permission-slip` CLI available via `npx` and registered
  (`npx @permission-slip/cli@latest whoami` should succeed).
- A connected iMessage account in Permission Slip, which requires a Mac signed
  into the user's Apple ID with [imsg](https://github.com/openclaw/imsg)
  installed. See the
  [iMessage connector docs](https://github.com/supersuit-tech/permission-slip/blob/main/connectors/imessage/README.md)
  for the full setup (Full Disk Access, Automation permission, Text Message
  Forwarding for SMS threads).

## Installation

Install the skill onto your OpenClaw machine when you're ready to use it — there
is no auto-install. Clone or copy `SKILL.md` into your agent's skills directory.

## See also

- [Permission Slip](https://github.com/supersuit-tech/permission-slip)
- [iMessage connector docs](https://github.com/supersuit-tech/permission-slip/blob/main/connectors/imessage/README.md)
- [permission-slip-openclaw-skill-protonmail](https://github.com/supersuit-tech/permission-slip-openclaw-skill-protonmail) — Proton Mail skill
- [permission-slip-openclaw-skill-gmail](https://github.com/supersuit-tech/permission-slip-openclaw-skill-gmail) — Gmail skill
- [permission-slip-openclaw-skill-google-calendar](https://github.com/supersuit-tech/permission-slip-openclaw-skill-google-calendar) — Google Calendar skill
- [permission-slip-openclaw-skill-google-drive](https://github.com/supersuit-tech/permission-slip-openclaw-skill-google-drive) — Google Drive skill
- [permission-slip-openclaw-skill-slack](https://github.com/supersuit-tech/permission-slip-openclaw-skill-slack) — Slack skill
