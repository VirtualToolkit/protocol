# VirtualToolkit Desktop Protocol

Communication spec between the C# desktop client and the VirtualToolkit web server.

---

## Overview

The desktop client maintains a persistent WebSocket connection to the server and forwards events sourced from VRChat's log files, OSC, and OSCQuery. The server stores per-user state and forwards relevant events to any open browser sessions in real time.

```
VRChat ──(logs/OSC)──► C# Client ──(WebSocket)──► VT Server ──(WebSocket)──► Browser
```

---

## Connection

| Property | Value |
|---|---|
| Endpoint | `ws://<host>/ws/desktop` |
| Protocol | WebSocket (RFC 6455) |
| Encoding | UTF-8 JSON, one message per frame |
| Auth | Bearer token (see below) |

### Authentication

Provide your desktop token using **either** method:

**Header (preferred):**
```
Authorization: Bearer <token>
```

**Query parameter (fallback):**
```
ws://localhost:3000/ws/desktop?token=<token>
```

The token is generated from the VirtualToolkit web UI under Settings. The server will close the connection immediately with code `1008` if the token is missing or invalid.

---

## Message Envelope

Every message — in both directions — uses this JSON envelope:

```json
{
  "type": "<event_name>",
  "ts": 1713531600000,
  "data": { }
}
```

| Field | Type | Description |
|---|---|---|
| `type` | `string` | Event discriminator (see types below) |
| `ts` | `number` | Unix timestamp in **milliseconds** (`Date.now()` / `DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()`) |
| `data` | `object` | Event-specific payload — never `null`, use `{}` if empty |

---

## Connection Lifecycle

```
Client                          Server
  │                               │
  ├──── WebSocket upgrade ────────►│
  │◄─── 101 Switching Protocols ───┤  (or 1008 if auth fails)
  │                               │
  ├──── hello ────────────────────►│
  │                               │  Server broadcasts desktop_connected to browser
  │                               │
  ├──── [events as they occur] ───►│
  │                               │
  ├──── heartbeat (every 30s) ────►│
  │                               │
  ├──── [disconnect] ─────────────►│
  │                               │  Server clears player state
  │                               │  Server broadcasts desktop_disconnected to browser
```

### Reconnection

- On unexpected disconnect, reconnect with **exponential backoff**: start at 2s, double each attempt, cap at 60s
- Re-send `hello` after every reconnect
- Do **not** re-send historical events after reconnect — only new events going forward

---

## Desktop → Server Messages

### `hello`

Sent once immediately after connecting. Advertises client version and capabilities so the server can adjust behaviour.

```json
{
  "type": "hello",
  "ts": 1713531600000,
  "data": {
    "version": "1.0.0",
    "features": ["logs", "osc", "oscquery"]
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `version` | `string` | Yes | Semver client version |
| `features` | `string[]` | Yes | Active data sources: `"logs"`, `"osc"`, `"oscquery"` |

---

### `heartbeat`

Sent every **30 seconds** to keep the connection alive. No payload.

```json
{
  "type": "heartbeat",
  "ts": 1713531600000,
  "data": {}
}
```

---

### `player_joined`

Fired when VRChat logs emit `[Behaviour] OnPlayerJoined <name>`.

```json
{
  "type": "player_joined",
  "ts": 1713531600000,
  "data": {
    "displayName": "SomeUser",
    "userId": "usr_xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `displayName` | `string` | Yes | VRChat display name as it appears in logs |
| `userId` | `string` | No | VRChat user ID — include if available via OSCQuery |

> **Note:** The local player (you) also fires `OnPlayerJoined` when you enter an instance. Include this event — the server will add you to the player list too.

---

### `player_left`

Fired when VRChat logs emit `[Behaviour] OnPlayerLeft <name>`.

```json
{
  "type": "player_left",
  "ts": 1713531600000,
  "data": {
    "displayName": "SomeUser",
    "userId": "usr_xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `displayName` | `string` | Yes | Must match the `displayName` sent in `player_joined` |
| `userId` | `string` | No | VRChat user ID if available |

---

### `instance_changed`

Fired when VRChat logs emit a `[RoomManager]` join line. Send this **before** the player join events for the new instance. The server clears the player list on receipt.

Relevant log patterns:
```
[RoomManager] Joining wrld_xxx:12345~...
[RoomManager] Successfully joined room: wrld_xxx:12345~...
```

Use the **"Successfully joined"** line, not the initial "Joining" line, to avoid firing on cancelled loads.

```json
{
  "type": "instance_changed",
  "ts": 1713531600000,
  "data": {
    "worldId": "wrld_xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "instanceId": "12345",
    "worldName": "The Great Pug",
    "instanceType": "public"
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `worldId` | `string` | Yes | e.g. `wrld_xxxxxxxx-...` |
| `instanceId` | `string` | Yes | The instance ID segment (before `~`) |
| `worldName` | `string` | No | Friendly name if parseable from logs |
| `instanceType` | `string` | No | `public` \| `friends` \| `invite` \| `group` etc. |

---

### `avatar_changed`

Fired when the local user changes avatar. Available via OSCQuery at `/avatar/change`.

```json
{
  "type": "avatar_changed",
  "ts": 1713531600000,
  "data": {
    "avatarId": "avtr_xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "avatarName": "My Cool Avatar"
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `avatarId` | `string` | Yes | VRChat avatar ID |
| `avatarName` | `string` | No | Avatar name if available |

---

### `osc_parameter`

Fired on OSC parameter changes received on UDP port **9000** (VRChat default output port).

Send only parameters your app actively monitors — do not forward every parameter update if volume is high, as this will flood the browser.

```json
{
  "type": "osc_parameter",
  "ts": 1713531600000,
  "data": {
    "parameter": "/avatar/parameters/VRCEmote",
    "value": 1
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `parameter` | `string` | Yes | Full OSC address, e.g. `/avatar/parameters/Viseme` |
| `value` | `number \| boolean \| string` | Yes | Matches the OSC type tag: `f`/`i` → `number`, `T`/`F` → `boolean`, `s` → `string` |

---

### `chatbox`

Fired on OSC messages received at `/chatbox/input`.

```json
{
  "type": "chatbox",
  "ts": 1713531600000,
  "data": {
    "text": "Hello world!",
    "typing": false
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `text` | `string` | Yes | Current chatbox content |
| `typing` | `boolean` | Yes | `true` while the keyboard is open, `false` when submitted |

---

## Server → Browser Messages

These are forwarded to the user's open browser tabs over `/ws/browser`. The desktop client does not receive these.

### `desktop_connected`

Sent when the desktop client connects (or reconnects).

```json
{
  "type": "desktop_connected",
  "ts": 1713531600000,
  "data": {
    "version": "1.0.0",
    "features": ["logs", "osc"]
  }
}
```

### `desktop_disconnected`

Sent when the desktop client disconnects. The browser should treat the player list as stale.

```json
{
  "type": "desktop_disconnected",
  "ts": 1713531600000,
  "data": {}
}
```

### All Desktop Events

`player_joined`, `player_left`, `instance_changed`, `avatar_changed`, `osc_parameter`, and `chatbox` are forwarded to the browser verbatim with their original envelope.

---

## Close Codes

| Code | Meaning |
|---|---|
| `1000` | Normal closure |
| `1008` | Auth failure — missing or invalid token |
| `1011` | Internal server error |

---

## Log Parsing Reference

Key VRChat log patterns the client should watch for:

| Event | Log pattern |
|---|---|
| Instance joined | `[RoomManager] Successfully joined room: <worldId>:<instanceId>~...` |
| Player joined | `[Behaviour] OnPlayerJoined <displayName>` |
| Player left | `[Behaviour] OnPlayerLeft <displayName>` |
| Avatar changed | OSCQuery — `/avatar/change` event |

VRChat log files are located at:
```
%AppData%\..\LocalLow\VRChat\VRChat\output_log_<datetime>.txt
```

The client should tail the **most recently modified** log file and watch for new files being created when VRChat restarts.

---

## OSC Reference

VRChat OSC defaults:

| Direction | Port | Description |
|---|---|---|
| VRChat → Client (receive) | `9000` | Parameter updates, chatbox |
| Client → VRChat (send) | `9001` | Parameter writes, chatbox input |

OSCQuery runs on port `9001` (HTTP) and provides the current avatar parameter tree.
