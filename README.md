# 🎮 MCStatusAPI

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![API Version](https://img.shields.io/badge/API-v1%20%7C%20v2%20WebSocket-brightgreen.svg)]()
[![Node.js](https://img.shields.io/badge/Node.js-%3E%3D18.0.0-informational.svg)]()

**MCStatusAPI** is a Minecraft server status and monitoring API hosted at `mc.fizzystudio.xyz`. It provides access to server data including player counts, MOTD (formatted, raw, HTML), latency, favicons, software info, and WebSocket updates for Minecraft Java Edition servers.

---

## ✨ Features

- ⚡ **REST API (v1)**: Fetch server status, ping, version, player lists, and favicons with server-side caching.
- 📡 **WebSockets (v2)**: Subscribe to server status updates over a WebSocket connection.
- 🖼️ **Direct Icon Endpoint**: Retrieve Minecraft server favicons as PNG images.
- 🔑 **Developer Dashboard**: Sign in via Discord OAuth2 to generate API keys and track key usage.
- 🧩 **SRV & IP Resolution**: Lookup for `_minecraft._tcp` SRV DNS records and numeric IP resolution.
- 🛡️ **Rate Limiting**: Request throttling and connection management for server stability.

---

## 📌 Table of Contents

- [REST API Reference](#-rest-api-reference)
  - [1. Get Server Status](#1-get-server-status)
  - [2. Get Server Icon](#2-get-server-icon)
  - [3. Get API Plans & Limits](#3-get-api-plans--limits)
- [WebSocket API (v2) Reference](#-websocket-api-v2-reference)
  - [Connecting](#connecting)
  - [Client Messages](#client-messages)
  - [Server Events](#server-events)
  - [Heartbeat & Pings](#heartbeat--pings)
  - [Error Handling](#error-handling)
- [🔑 Authentication](#-authentication)
- [⚡ Quickstart & Code Examples](#-quickstart--code-examples)
  - [cURL](#curl)
  - [JavaScript (Node.js & Web)](#javascript-nodejs--web)
  - [Python](#python)
- [📊 Rates & Limits](#-rates--limits)
- [📄 License](#-license)

---

## 📡 REST API Reference

### Base URL
```text
https://mc.fizzystudio.xyz
```

---

### 1. Get Server Status

Retrieves current server status, player counts, formatted MOTD, Minecraft version, server software, and base64 favicon.

#### Endpoint
`GET /status/:address`  
`GET /status/v1/:address`

#### Parameters
| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `address` | `string` | **Yes** | Server domain or IP address (e.g. `mc.hypixel.net` or `127.0.0.1:25565`). If port is omitted, `25565` or DNS SRV records are automatically used. |

#### Example Request
```http
GET /status/mc.hypixel.net HTTP/1.1
Host: mc.fizzystudio.xyz
```

#### Example Response (`200 OK`)
```json
{
  "ip": "mc.hypixel.net",
  "numericIp": "209.222.115.11",
  "port": 25565,
  "motd": {
    "raw": "§aHypixel Network  §c[1.8-1.20]\n§b§lSKYBLOCK §7| §e§lBEDWARS",
    "clean": "Hypixel Network  [1.8-1.20]\nSKYBLOCK | BEDWARS",
    "html": "<span style=\"color: #55FF55;\">Hypixel Network  </span><span style=\"color: #FF5555;\">[1.8-1.20]</span><br><span style=\"color: #55FFFF; font-weight: bold;\">SKYBLOCK </span><span style=\"color: #AAAAAA;\">| </span><span style=\"color: #FFFF55; font-weight: bold;\">BEDWARS</span>"
  },
  "players": {
    "online": 32410,
    "max": 200000,
    "sample": []
  },
  "version": "Requires MC 1.8-1.20",
  "online": true,
  "protocol": 47,
  "icon": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEAAAABACAYAAACqaX...",
  "software": "Spigot",
  "eula_blocked": false
}
```

#### Server Offline / Error Response (`400 Bad Request`)
```json
{
  "message": "Server at mc.hypixel.net is offline or unreachable."
}
```

---

### 2. Get Server Icon

Retrieves the Minecraft server favicon directly as a standard PNG image stream.

#### Endpoint
`GET /icon/:address`  
`GET /icon/v1/:address`

#### Parameters
| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `address` | `string` | **Yes** | Server domain or IP address (e.g. `mc.hypixel.net` or `play.example.com:25565`). |

#### Response Headers & Behavior
- **Status `200 OK`**: `Content-Type: image/png`, cached in browser for 24 hours (`Cache-Control: public, max-age=86400`).
- **Status `404 Not Found`**: Returned if the Minecraft server does not have a custom favicon configured.

#### HTML Usage Example
```html
<img src="https://mc.fizzystudio.xyz/icon/mc.hypixel.net" alt="Hypixel Icon" width="64" height="64" />
```

---

### 3. Get API Plans & Limits

Fetches public details about available subscription plans, rate limits, and feature permissions.

#### Endpoint
`GET /api/plans`

#### Example Response (`200 OK`)
```json
{
  "defaultLimit": -1,
  "plans": [
    {
      "name": "basic",
      "limit": -1,
      "limitLabel": "Unlimited",
      "ws": true,
      "priority": false,
      "featured": false
    },
    {
      "name": "standard",
      "limit": 90000,
      "limitLabel": "90,000",
      "ws": true,
      "priority": false,
      "featured": false
    },
    {
      "name": "premium",
      "limit": 150000,
      "limitLabel": "150,000",
      "ws": true,
      "priority": true,
      "featured": true
    }
  ]
}
```

---

## 🔄 WebSocket API (v2) Reference

The WebSocket API (v2) provides a persistent bidirectional connection for listening to live server updates without HTTP polling overhead.

### Connecting

Connect to the WebSocket URL using your API Key via query parameter or Authorization header:

```text
wss://mc.fizzystudio.xyz/ws/v2?api_key=YOUR_API_KEY
```

**Alternative Header Authentication:**
```text
Authorization: Bearer YOUR_API_KEY
```

---

### Client Messages

All messages sent over the WebSocket connection must be valid JSON strings.

#### 1. Subscribe (`subscribe`)
Subscribe to real-time status updates for a server.
```json
{
  "type": "subscribe",
  "server": "mc.hypixel.net"
}
```

#### 2. Unsubscribe (`unsubscribe`)
Stop monitoring updates for a server.
```json
{
  "type": "unsubscribe",
  "server": "mc.hypixel.net"
}
```

#### 3. Heartbeat Response (`pong`)
Respond to server ping requests to keep your session active.
```json
{
  "type": "pong"
}
```

---

### Server Events

#### 1. Subscribed Confirmation (`subscribed`)
Received when a server subscription is confirmed.
```json
{
  "type": "subscribed",
  "server": "mc.hypixel.net"
}
```

#### 2. Live Status Update (`status_update`)
Broadcast whenever player count, MOTD, latency, or server status changes.
```json
{
  "type": "status_update",
  "server": "mc.hypixel.net",
  "online": true,
  "players": {
    "online": 32450,
    "max": 200000
  },
  "latency": 42,
  "version": {
    "name": "Requires MC 1.8-1.20",
    "protocol": 47
  },
  "motd": {
    "raw": "§aHypixel Network",
    "clean": "Hypixel Network",
    "html": "<span style=\"color: #55FF55;\">Hypixel Network</span>"
  },
  "icon": "data:image/png;base64,iVBORw0KG...",
  "software": "Spigot",
  "eula_blocked": false,
  "timestamp": 1770000000
}
```

#### 3. Server Offline Notification (`server_offline`)
Sent when a monitored server goes offline or becomes unreachable.
```json
{
  "type": "server_offline",
  "server": "mc.hypixel.net",
  "timestamp": 1770000000
}
```

#### 4. Heartbeat Ping (`ping`)
Sent periodically by the server. Clients must respond with `{ "type": "pong" }` to prevent disconnection.
```json
{
  "type": "ping"
}
```

---

### Error Handling

If a request fails or rate limits are hit, the server sends an error message:

```json
{
  "error": true,
  "code": "RATE_LIMIT_EXCEEDED",
  "message": "Max subscriptions reached"
}
```

#### Common Error Codes
| Code | Description |
| :--- | :--- |
| `INVALID_API_KEY` | Key is missing or invalid. |
| `RATE_LIMIT_EXCEEDED` | Maximum connection, message, or subscription limit exceeded. |
| `INVALID_SERVER` | Address string is malformed or exceeds maximum length. |
| `INVALID_JSON` | Payload is not valid JSON. |
| `UNKNOWN_TYPE` | Unrecognized message type. |

---

## 🔑 Authentication

Most REST endpoints operate publicly without authentication. API Keys are required for **WebSocket v2 streaming** and rate limit allocation:

1. Click **Login with Discord** at `https://mc.fizzystudio.xyz/auth/discord`.
2. Access your **User Dashboard** at `https://mc.fizzystudio.xyz/dashboard`.
3. Create up to **3 active API keys** (`mcsapi_...`). Free Basic plan keys include **unmetered / unlimited requests** for a limited time.
4. Attach your API key to WebSocket connection URIs or request headers.

---

## ⚡ Quickstart & Code Examples

### cURL

```bash
# Fetch Minecraft Server Status
curl -X GET "https://mc.fizzystudio.xyz/status/mc.hypixel.net"

# Download Server Icon
curl -o favicon.png "https://mc.fizzystudio.xyz/icon/mc.hypixel.net"
```

---

### JavaScript (Node.js & Web)

#### Fetching REST Status
```javascript
async function getServerStatus(serverIp) {
  try {
    const response = await fetch(`https://mc.fizzystudio.xyz/status/${serverIp}`);
    const data = await response.json();

    if (data.online) {
      console.log(`Server Online! ${data.players.online}/${data.players.max} Players`);
    } else {
      console.log('Server is currently offline.');
    }
  } catch (err) {
    console.error('API Error:', err);
  }
}

getServerStatus('mc.hypixel.net');
```

#### WebSocket Real-Time Monitoring
```javascript
import WebSocket from 'ws';

const apiKey = 'mcsapi_your_api_key_here';
const ws = new WebSocket(`wss://mc.fizzystudio.xyz/ws/v2?api_key=${apiKey}`);

ws.on('open', () => {
  console.log('Connected to WebSocket stream');
  
  // Subscribe to server
  ws.send(JSON.stringify({
    type: 'subscribe',
    server: 'mc.hypixel.net'
  }));
});

ws.on('message', (buffer) => {
  const data = JSON.parse(buffer.toString());

  if (data.type === 'status_update') {
    console.log(`[UPDATE] ${data.server} | Players: ${data.players.online} | Latency: ${data.latency}ms`);
  } else if (data.type === 'ping') {
    ws.send(JSON.stringify({ type: 'pong' }));
  }
});
```

---

### Python

#### REST Request
```python
import requests

response = requests.get("https://mc.fizzystudio.xyz/status/mc.hypixel.net")
data = response.json()

if data.get("online"):
    print(f"Status: Online | Players: {data['players']['online']}/{data['players']['max']}")
else:
    print("Status: Offline")
```

#### Real-Time WebSocket Client
```python
import asyncio
import json
import websockets

API_KEY = "mcsapi_your_api_key_here"
WS_URL = f"wss://mc.fizzystudio.xyz/ws/v2?api_key={API_KEY}"

async def connect_websocket():
    async with websockets.connect(WS_URL) as ws:
        # Subscribe
        await ws.send(json.dumps({
            "type": "subscribe",
            "server": "mc.hypixel.net"
        }))
        
        async for message in ws:
            event = json.loads(message)
            if event.get("type") == "status_update":
                print(f"[LIVE UPDATE] {event['server']} - {event['players']['online']} players online")
            elif event.get("type") == "ping":
                await ws.send(json.dumps({"type": "pong"}))

asyncio.run(connect_websocket())
```

---

## 📊 Rates & Limits

> ℹ️ **Notice**: For a limited time, the **Basic (Free) Plan** provides **unmetered / unlimited daily requests** access.

| Plan Tier | Daily Request Limit | WS Connections / Key | Subscriptions / Connection | Messages / Minute | Idle Timeout |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Basic (Free)** | **Unlimited** *(Limited-Time)* | 3 Connections | 10 Servers | 60 | 5 Minutes |
| **Standard** | 90,000 / day | 3 Connections | 10 Servers | 60 | 5 Minutes |
| **Premium** | 150,000 / day | Priority Access | Custom Limits | Higher Capacity | 5 Minutes |

---

## ⚠️ Disclaimer

MCStatusAPI is provided "as is" without warranties, promises, or SLA uptime guarantees of any kind. Service availability, features, and rate limits are subject to change without notice.

---

## 📄 License

Distributed under the MIT License. See `LICENSE` for details.
