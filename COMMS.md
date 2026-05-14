# FM-COMMS.md — Forgemaster Communicator Protocol

> How Oracle1 and Forgemaster talk. Persisted across sessions.

## Architecture

```
FM (RTX 4050) ──Matrix──→ Oracle1 (Oracle Cloud) ──Telegram──→ Casey
                ←──────────                        ←──────────
```

**Matrix homeserver:** Conduwuit at `http://147.224.38.131:6167`
**Bridge API:** `http://147.224.38.131:6168` (port 6168, bound 0.0.0.0)
**Broadcast room (fleet-coord):** `!z5oIJTqor4UUZliQp1:147.224.38.131`
**Presence room:** `!VKhbIyYhCOYbIvFPv4:147.224.38.131`

## Key PLATO Rooms

| Room | Purpose |
|------|---------|
| `oracle1-forgemaster-bridge` | Primary FM↔Oracle1 channel |
| `forge` | FM's broadcast room (also sync'd to Matrix) |
| `fleet-coord` | Shared coordination room (also sync'd to Matrix) |

## Running Services

- **plato-matrix-bridge.py** — daemon, syncs PLATO↔Matrix every 3s. Checks both directions.
- Located at: `fleet/comms/plato-matrix-bridge.py`
- Config at: `/tmp/plato-matrix-oracle1.json`
- Log: `/tmp/plato-matrix-daemon.log`
- Repo: `github.com/SuperInstance/plato-matrix-bridge`

## How to Check for FM Messages

**Fastest: PLATO bridge room**
```bash
curl -s http://localhost:8847/room/oracle1-forgemaster-bridge/history | \
  python3 -c "import sys,json; d=json.load(sys.stdin); [print(f'[{t[\"source\"]}] {t[\"question\"][:80]}') for t in d['tiles'] if any(x in t['source'] for x in ['forgemaster','dancer','fm_'])]"
```

**Matrix fleet-coord (raw):**
```bash
MATRIX="http://localhost:6167"
TOKEN="cZpdJNoUymtMLcHPbAoMY8GpsNv4Qie7"
ROOM="!z5oIJTqor4UUZliQp1:147.224.38.131"
curl -s "$MATRIX/_matrix/client/v3/rooms/$(python3 -c 'import urllib.parse; print(urllib.parse.quote(\"'$ROOM'\", safe=\"\"))')/messages?dir=b&limit=5" -H "Authorization: Bearer $TOKEN"
```

**Alert file:** `/tmp/fm-com badge-alert.txt` — latest unread message
**Seen log:** `/tmp/fm-com badge-seen.txt` — all messages ever received
**Answering machine state:** `/tmp/communicator-state.json`

## Answering Machine Protocol

1. New message from FM → alert file written immediately
2. On heartbeat: check alert file. If non-empty, I have a message.
3. After reading: clear alert file (`> /tmp/fm-com badge-alert.txt`)
4. If unanswered messages accumulate, they stay in the state file
5. The blinking light: state file tracks `unacknowledged_count`
6. On every heartbeat with unacknowledged > 0: 📞 reminder to Casey

## Forgemaster's Matrix Accounts

| Account | Used by |
|---------|---------|
| `@forgemaster:147.224.38.131` | Forgemaster's module (primary) |
| `@fm-bot:147.224.38.131` | Fleet-matrix-bridge (legacy) |

## Credentials

Forgemaster's Matrix password: `fleet-fm-2026`
Forgemaster's Matrix token: `wa1ViGSmGnbu0jMrlPSQuj6KL1sBJgTi`
(Stored in: `data/matrix/fleet-matrix-credentials.md`)

## Startup Sequence

On session start:
1. Check `ps aux | grep plato-matrix-bridge` — daemon should be running
2. If daemon down: restart with config `/tmp/plato-matrix-oracle1.json`
3. Check alert file for unread messages
4. Check bridge room for any new FM tiles since last session
5. Report state to Casey on Telegram
