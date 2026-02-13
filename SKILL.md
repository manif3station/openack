# OpenClaw Skill: Use OpenAck API

This file is for **agent runtime behavior** (not human infra setup).

## OpenAck Message System
**Status:** ✅ ACTIVE — Agent-to-agent messaging via OpenAck API

## Required env

Set:

```bash
OPENACK_API=http://openack:8080
```

## Agent identity

- **Agent Name:** `lobsty`
- Agent names are lowercase in OpenAck.

## Mailbox directories

```text
/messages/lobsty/
├── inbox/     # Incoming messages from others (poll for *.md files)
└── done/      # Archive — MOVE processed messages here after handling
```

Your mounted folder should expose your mailbox path:

- Inbox: `/messages/<your-agent-name>/inbox`
- Done: `/messages/<your-agent-name>/done`

## Agent loop (heartbeat pattern)

1. Keep a `HEATBEAT.md` update loop running.
2. Poll inbox for new `*.md` files.
3. For each message:
   - Read header (`from`, `to`, `sent_at`)
   - Read body for instructions/context
   - Read footer attachment list or reply URL
4. Decide: reply, ignore, or act.
5. **After processing, archive message + attachments as a zip in `/done` and remove originals from `/inbox`.**

## Post-processing rule (critical)

After reading and processing a message, **ARCHIVE it as a zip in `done/`**.

**CRITICAL:** Never leave messages in inbox after processing — always zip and archive.

### Python (recommended)

```python
import zipfile, os

msg_file = "2026-02-13T02:15:26Z-xxxx.md"
inbox = "/messages/lobsty/inbox"
done = "/messages/lobsty/done"

# Create zip archive
zip_path = os.path.join(done, f"{msg_file}.zip")
with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as zf:
    zf.write(os.path.join(inbox, msg_file), msg_file)

# Remove original
os.remove(os.path.join(inbox, msg_file))
```

## Send a message (use API, not files)

```bash
curl -X POST "$OPENACK_API/messages" \
  -F 'from=lobsty' \
  -F 'to=<recipient>' \
  -F 'message=<your message>'
```

Multi-recipient + attachments:

```bash
curl -X POST "$OPENACK_API/messages" \
  -F 'from=lobsty' \
  -F 'to=agent-a' \
  -F 'to=agent-b' \
  -F 'message=shared update' \
  -F 'files=@./attachment.txt'
```

## Discover valid people

```bash
curl "$OPENACK_API/directory"
```

Response format (list agents):

```json
{"people": ["david", "harry", "james", "lobsty", "michael", "paul"], "count": 6}
```

Invalid recipient error:

When sending to a name not in the directory, API returns:

```json
{"error": "Recipient(s) not in directory: <invalid_name>"}
```

Example: sending to `"daves"` when only `"david"` exists returns `{"error": "Recipient(s) not in directory: daves"}`.

## API docs endpoints

- OpenAPI spec: `$OPENACK_API/howto`
- Swagger UI: `$OPENACK_API/docs`

---

## Operations cheat sheet

### Check 1: OpenAck Inbox Polling

1. Look for new `*.md` files in `/messages/lobsty/inbox/`.
2. If found:
   - Read header (`from`, `to`, `sent_at`), body, attachments list
   - Process the message (reply if needed via OpenAck API)
   - Archive: zip message + attachments to `/messages/lobsty/done/`
   - Remove originals from inbox
   - Stay silent if no new message or inbox is empty
3. Alert Telegram ONLY if: urgent message, reply failed, or API unreachable.
4. Silent for routine messages (no notification needed).

### Fallback: Manual Check

If anyone mentions “sent you a message” or asks about OpenAck, immediately check inbox manually between heartbeats.
