# ⚡ StreamerPlugin

> Advanced live stream announcement plugin for Paper 1.21.x Minecraft servers.

When a player on your server goes live on **Twitch**, **YouTube**, **TikTok**, or **Instagram**,
StreamerPlugin notifies the entire server with a full suite of customizable announcements —
automatically, repeatedly, and smartly.

---

## ✨ Features

### 🔴 Live Detection
- **Twitch** — Auto-detected via Twitch Helix API (Client Credentials flow)
- **YouTube** — Auto-detected via YouTube Data API v3
- **TikTok / Instagram** — Supported via IFTTT / Zapier / Make.com webhooks
- **Any platform** — Manual announce with `/golive`
- Configurable polling interval
- Automatic offline detection when streams end

### 📢 Notification Channels
Every channel is independently toggleable in `config.yml`:

| Channel | Description |
|---|---|
| Screen Title + Subtitle | Large centered text with configurable fade/stay timing |
| Boss Bar | Top-of-screen bar with color, style and auto-dismiss |
| Action Bar | Subtle above-hotbar line |
| Chat Broadcast | Styled clickable message with stream URL |
| Sound Effects | Per-event sounds with volume and pitch control |
| Stream Book | Written book with WATCH NOW link and all social links |

### 🔁 Repeat Announcements
- Repeats title, boss bar, action bar and chat every N minutes while live
- Configurable interval via `repeat-announcement-interval-minutes`
- Books are **never** re-given on repeat cycles

### 📖 Smart Book System
- Scans player inventory before giving — **never duplicates**
- Given to players who join after a stream started
- Re-given on rejoin (in case they dropped it)
- Drops at feet if inventory is full (configurable)

### 🖥️ Interactive GUIs
- **Admin Panel** (`/streamadmin gui`) — toggle notifications, view live count, reload, see live heads
- **Streamer Profile** (`/streamer profile`) — view/manage linked social accounts, see live status

### ⚙️ Fully Configurable
- All messages use **MiniMessage format** (gradients, bold, hover, click events)
- World filter (whitelist or blacklist mode)
- Per-sound control (name, volume, pitch)
- Boss bar color and style
- Title timing (fade-in, stay, fade-out in ticks)
- Webhook secret validation
- Debug mode

---

## 📦 Requirements

| Requirement | Details |
|---|---|
| Server | Paper 1.21.x **only** (not Spigot, not Folia) |
| Java | 21+ |
| Twitch Detection | [Twitch Developer App](https://dev.twitch.tv/console/apps) (free) |
| YouTube Detection | [Google Cloud API Key](https://console.cloud.google.com) with YouTube Data API v3 enabled (free) |
| TikTok / Instagram | IFTTT, Zapier, or Make.com account (free tier works) |

---

## 🛠️ Installation

1. Drop `StreamerPlugin.jar` into your `/plugins` folder
2. Start the server to generate config files
3. Open `plugins/StreamerPlugin/config.yml`
4. Fill in your Twitch and YouTube credentials
5. For TikTok/Instagram: set up a webhook pointing to:
