# StreamerPlugin — Full Platform Linking Guide

> **Covers:** Twitch, YouTube, TikTok, and Instagram — API setup, config, in-game commands, and webhook wiring.

---

## Table of Contents

1. [Overview](#overview)
2. [Twitch](#twitch)
3. [YouTube](#youtube)
4. [TikTok (via Webhook)](#tiktok-via-webhook)
5. [Instagram (via Webhook)](#instagram-via-webhook)
6. [Linking Players In-Game](#linking-players-in-game)
7. [Linking Players via streamers.yml](#linking-players-via-streamerssyml)
8. [Testing Your Setup](#testing-your-setup)
9. [Webhook JSON Reference](#webhook-json-reference)
10. [Troubleshooting](#troubleshooting)

---

## Overview

StreamerPlugin uses **two detection strategies**:

| Platform | Strategy | Requires |
|---|---|---|
| Twitch | API Polling (automatic) | Twitch Developer App |
| YouTube | API Polling (automatic) | Google Cloud API Key |
| TikTok | Webhook (external trigger) | IFTTT / Zapier / Make.com |
| Instagram | Webhook (external trigger) | IFTTT / Zapier / Make.com |

**Polling** means the plugin checks the platform every X minutes on its own.  
**Webhook** means an external automation tool watches TikTok/Instagram and notifies the plugin via HTTP POST.

---

## Twitch

### Step 1 — Create a Twitch Developer Application

1. Go to [https://dev.twitch.tv/console/apps](https://dev.twitch.tv/console/apps)
2. Log in with your Twitch account
3. Click **Register Your Application**
4. Fill in the form:
   - **Name:** anything (e.g. `MyServerBot`)
   - **OAuth Redirect URLs:** `http://localhost` (required but unused)
   - **Category:** `Application Integration`
5. Click **Create**
6. On the next page, click **Manage** next to your new app
7. Click **New Secret** to generate a Client Secret
8. Copy both the **Client ID** and **Client Secret**

### Step 2 — Add to config.yml

```yaml
twitch:
  enabled: true
  client-id: "abc123youridhere"
  client-secret: "xyz789yoursecrethere"
```

### Step 3 — Link Each Player's Twitch Handle

Each player needs their Twitch username registered. Do this via command:

```
/streamer setlink twitch <twitchUsername>
```

**Example:**
```
/streamer setlink twitch ninja
```

Or edit `streamers.yml` directly (see [Linking via streamers.yml](#linking-players-via-streamerssyml)).

### How It Works

Every `polling-interval-minutes` minutes the plugin calls the **Twitch Helix `/streams` endpoint** with all registered handles. If a handle returns `"type": "live"`, the announcement fires. When the player goes offline the plugin detects the missing entry on the next poll and clears their live status automatically.

> **Note:** Twitch OAuth tokens are fetched automatically using `client_credentials` flow — you never need to log in or authorize manually.

---

## YouTube

### Step 1 — Create a Google Cloud Project

1. Go to [https://console.cloud.google.com](https://console.cloud.google.com)
2. Click the project dropdown at the top → **New Project**
3. Name it anything (e.g. `StreamerPlugin`) → **Create**

### Step 2 — Enable the YouTube Data API v3

1. With your project selected, go to **APIs & Services → Library**
2. Search for `YouTube Data API v3`
3. Click it → **Enable**

### Step 3 — Create an API Key

1. Go to **APIs & Services → Credentials**
2. Click **Create Credentials → API Key**
3. Copy the generated key
4. (Recommended) Click **Edit API Key** → under **API restrictions**, select **YouTube Data API v3** to limit its scope

### Step 4 — Add to config.yml

```yaml
youtube:
  enabled: true
  api-key: "AIzaSyYOURKEYHERE"
```

### Step 5 — Find Each Player's YouTube Channel ID

The plugin needs a **Channel ID** (starts with `UC`), not a handle or username.

To find a Channel ID:
1. Go to the channel's YouTube page
2. Click **About** or scroll to the bottom of the page
3. Click **Share Channel → Copy Channel ID**

Alternatively, go to:
```
https://www.youtube.com/@HandleName/about
```
and the URL or page source will contain `"channelId":"UCxxxxxxxx"`.

### Step 6 — Link Each Player's Channel

```
/streamer setlink youtube UC1234567890abcdefg
```

Or in `streamers.yml`:
```yaml
youtube_channel: "UC1234567890abcdefg"
```

### How It Works

The plugin calls the YouTube Data API `search` endpoint with `eventType=live&type=video` for each registered channel ID. If a live video is found, the announcement fires. The free API quota is **10,000 units/day** — each search costs 100 units, so with 10 players polled every 5 minutes you'd use ~2,880 units/day, well within limits.

---

## TikTok (via Webhook)

TikTok has no public livestream API, so the plugin uses an **internal HTTP webhook server**. An external automation service watches TikTok and sends a POST request to your server when someone goes live.

### Step 1 — Verify the Webhook Server is Running

In `config.yml`:
```yaml
settings:
  webhook-port: 8765
  webhook-secret: "mysecretpassword123"

tiktok:
  enabled: true
```

On startup you should see in the server console:
```
[StreamerPlugin] Webhook server started on port 8765
```

### Step 2 — Open the Port

On your server machine, allow inbound TCP on port `8765`:

**Linux (ufw):**
```bash
sudo ufw allow 8765/tcp
```

**Linux (iptables):**
```bash
sudo iptables -A INPUT -p tcp --dport 8765 -j ACCEPT
```

**Windows Firewall:**
```
netsh advfirewall firewall add rule name="StreamerPlugin" protocol=TCP dir=in localport=8765 action=allow
```

Test it is reachable:
```
http://YOUR_SERVER_IP:8765/webhook/health
```
You should get: `StreamerPlugin webhook server running.`

### Step 3 — Set Up IFTTT

1. Go to [https://ifttt.com](https://ifttt.com) and sign in
2. Click **Create** → **If This**
3. Search for **TikTok** → select **New Live Video Started**
4. Connect your TikTok account when prompted
5. Click **Then That** → search for **Webhooks** → select **Make a web request**
6. Fill in the fields:

| Field | Value |
|---|---|
| URL | `http://YOUR_SERVER_IP:8765/webhook/tiktok` |
| Method | `POST` |
| Content Type | `application/json` |
| Body | See below |

**Body:**
```json
{
  "secret": "mysecretpassword123",
  "player_uuid": "PLAYER-UUID-HERE",
  "platform": "tiktok",
  "action": "live",
  "url": "https://tiktok.com/@yourtiktokhandle/live",
  "title": "Live on TikTok!"
}
```

7. Click **Continue** → name your applet → **Finish**

> **IFTTT Note:** IFTTT does not currently have a native "TikTok live ended" trigger. To mark a player offline when they stop streaming, create a second applet using a timer or **TikTok → No new videos** as a proxy, sending `"action": "offline"`.

### Step 4 — Set Up with Zapier (Alternative)

1. Go to [https://zapier.com](https://zapier.com) → **Create Zap**
2. **Trigger:** Search `TikTok` → event: **New Live Broadcast**
3. **Action:** Search `Webhooks by Zapier` → event: **POST**
4. URL: `http://YOUR_SERVER_IP:8765/webhook/tiktok`
5. Payload Type: `JSON`
6. Data:
```
secret: mysecretpassword123
player_uuid: PLAYER-UUID-HERE
platform: tiktok
action: live
url: https://tiktok.com/@handle/live
title: Live on TikTok!
```
7. **Publish Zap**

### Step 5 — Link the Player's TikTok

```
/streamer setlink tiktok @yourtiktokhandle
```

Or with the full URL:
```
/streamer setlink tiktok https://tiktok.com/@yourtiktokhandle
```

---

## Instagram (via Webhook)

Instagram Live also has no public API, so the same webhook approach is used.

### Step 1 — Ensure Instagram Webhook is Enabled

```yaml
instagram:
  enabled: true
```

The webhook endpoint is: `http://YOUR_SERVER_IP:8765/webhook/instagram`

### Step 2 — Set Up IFTTT for Instagram

> **Important:** Instagram has removed many third-party integrations over time. As of 2025, IFTTT's Instagram triggers work best for **Business/Creator accounts**.

1. Go to [https://ifttt.com](https://ifttt.com) → **Create**
2. **If This:** Search `Instagram` → **New photo/video by you** (use this as a proxy for going live, or use a third-party Instagram live monitor service)
3. **Then That:** `Webhooks → Make a web request`

| Field | Value |
|---|---|
| URL | `http://YOUR_SERVER_IP:8765/webhook/instagram` |
| Method | `POST` |
| Content Type | `application/json` |
| Body | See below |

**Body:**
```json
{
  "secret": "mysecretpassword123",
  "player_uuid": "PLAYER-UUID-HERE",
  "platform": "instagram",
  "action": "live",
  "url": "https://instagram.com/@yourhandle",
  "title": "Live on Instagram!"
}
```

### Step 2 (Alternative) — Make.com (Recommended for Instagram)

[Make.com](https://make.com) (formerly Integromat) has more reliable Instagram integration:

1. Create a new **Scenario**
2. Add module: **Instagram → Watch Live Videos** (requires Instagram Business account connected via Facebook)
3. Add module: **HTTP → Make a Request**
   - URL: `http://YOUR_SERVER_IP:8765/webhook/instagram`
   - Method: `POST`
   - Body type: `Raw`
   - Content type: `JSON (application/json)`
   - Request content:
```json
{
  "secret": "mysecretpassword123",
  "player_uuid": "{{playerUUID}}",
  "platform": "instagram",
  "action": "live",
  "url": "https://instagram.com/@yourhandle",
  "title": "{{streamTitle}}"
}
```
4. Set schedule → **Activate**

### Step 3 — Link the Player's Instagram

```
/streamer setlink instagram yourinstagramhandle
```

---

## Linking Players In-Game

Any player (with `streamerplugin.streamer` permission) can link their own accounts:

```
/streamer setlink <platform> <handle or URL>
```

| Example | Result |
|---|---|
| `/streamer setlink twitch ninja` | Sets Twitch handle to `ninja` |
| `/streamer setlink youtube UCxxxxxxxx` | Sets YouTube Channel ID |
| `/streamer setlink tiktok @myhandle` | Sets TikTok handle |
| `/streamer setlink instagram myhandle` | Sets Instagram handle |
| `/streamer setlink twitch https://twitch.tv/ninja` | Sets full Twitch URL directly |

View your profile and current live status:
```
/streamer profile
```

Manually announce yourself as live:
```
/golive twitch https://twitch.tv/myhandle My Stream Title
```

Mark yourself offline:
```
/streamer gooffline
```

---

## Linking Players via streamers.yml

Admins can pre-configure players directly in `plugins/StreamerPlugin/streamers.yml`:

```yaml
streamers:
  # Replace with the player's actual UUID
  "550e8400-e29b-41d4-a716-446655440000":
    name: "CoolStreamer"
    twitch_user: "coolstreamer"
    youtube_channel: "UCabcdefghijk12345"
    tiktok_user: "coolstreamer"
    instagram_user: "coolstreamer"
    # Optional: override auto-generated URLs
    twitch_url: "https://twitch.tv/coolstreamer"
    youtube_url: "https://youtube.com/@coolstreamer"
    tiktok_url: "https://tiktok.com/@coolstreamer"
    instagram_url: "https://instagram.com/coolstreamer"
    live: false
    live_platform: ""
    live_url: ""
    live_title: ""
```

After editing, run `/streamadmin reload` to apply changes without restarting.

**To find a player's UUID:** run this command in the server console:
```
whitelist list
```
Or visit [https://mcuuid.net](https://mcuuid.net) and enter the player's username.

---

## Testing Your Setup

### Test Twitch Polling

1. Have a registered Twitch streamer go live
2. Wait up to `polling-interval-minutes` (default: 5 min) or set it to `1` temporarily
3. Watch the console for: `[PollingEngine] Running poll cycle...`
4. The in-game broadcast should fire automatically

To force an immediate announcement for testing:
```
/streamadmin announce PlayerName twitch https://twitch.tv/test TestStream
```

### Test YouTube Polling

Same as Twitch — ensure the channel is actually live on YouTube, then wait for the next poll cycle.

### Test TikTok/Instagram Webhook

Send a test POST with `curl` from any machine that can reach your server:

```bash
curl -X POST http://YOUR_SERVER_IP:8765/webhook/tiktok \
  -H "Content-Type: application/json" \
  -d '{
    "secret": "mysecretpassword123",
    "player_uuid": "550e8400-e29b-41d4-a716-446655440000",
    "platform": "tiktok",
    "action": "live",
    "url": "https://tiktok.com/@test/live",
    "title": "Test Stream"
  }'
```

Expected response: `OK`  
Expected in-game: announcement fires immediately.

### Test Offline Webhook

```bash
curl -X POST http://YOUR_SERVER_IP:8765/webhook/tiktok \
  -H "Content-Type: application/json" \
  -d '{
    "secret": "mysecretpassword123",
    "player_uuid": "550e8400-e29b-41d4-a716-446655440000",
    "platform": "tiktok",
    "action": "offline"
  }'
```

### Health Check

```bash
curl http://YOUR_SERVER_IP:8765/webhook/health
# Expected: StreamerPlugin webhook server running.
```

---

## Webhook JSON Reference

Full schema for POST requests to `/webhook/tiktok` or `/webhook/instagram`:

| Field | Type | Required | Description |
|---|---|---|---|
| `secret` | string | If set in config | Must match `webhook-secret` in config.yml |
| `player_uuid` | string | **Yes** | The player's Minecraft UUID (with dashes) |
| `platform` | string | No | `"tiktok"` or `"instagram"` (auto-set by endpoint) |
| `action` | string | No | `"live"` (default) or `"offline"` |
| `url` | string | No | Direct link to the stream |
| `title` | string | No | Stream title shown in the announcement |

**Minimal live payload:**
```json
{
  "secret": "yourwebhooksecret",
  "player_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "action": "live"
}
```

**Full payload:**
```json
{
  "secret": "yourwebhooksecret",
  "player_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "platform": "tiktok",
  "action": "live",
  "url": "https://tiktok.com/@myhandle/live",
  "title": "Playing Minecraft with viewers!"
}
```

**Offline payload:**
```json
{
  "secret": "yourwebhooksecret",
  "player_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "action": "offline"
}
```

---

## Troubleshooting

### Twitch — No announcements firing

- Check `config.yml` has the correct `client-id` and `client-secret` (not the example placeholders)
- Ensure `twitch.enabled: true`
- Check the player has a `twitch_user` set (run `/streamer profile` in-game)
- Set `settings.debug: true` and watch console logs during a poll cycle
- Verify the Twitch app is not suspended at [dev.twitch.tv/console](https://dev.twitch.tv/console)

### YouTube — "quotaExceeded" error in console

- You've used your 10,000 daily units. Wait until midnight Pacific Time for reset.
- Increase `polling-interval-minutes` to reduce API calls (10–15 min recommended for YouTube).
- Consider creating a second Google Cloud project with a second API key for backup.

### Webhook — Connection refused

- Confirm port `8765` is open on your server firewall (see [Open the Port](#step-2--open-the-port))
- Confirm the plugin started successfully: look for `Webhook server started on port 8765` in the console
- If behind NAT (home server), forward port 8765 on your router to the machine's local IP
- If using a hosting panel (e.g. Pterodactyl), add port `8765` as an allocation and use the node's public IP

### Webhook — 403 Forbidden

- The `secret` in your JSON payload doesn't match `webhook-secret` in `config.yml`
- Double-check for leading/trailing spaces in both places

### Duplicate announcements

- The `LiveCacheManager` deduplicates by UUID + platform. A duplicate fires only if the player was marked offline between streams.
- If a player uses `/golive` AND the poll detects them live, only the first triggers an announcement.

### Player's UUID is wrong

- UUIDs are case-sensitive and must include dashes: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`
- Look up the correct UUID at [https://mcuuid.net](https://mcuuid.net)
- If the player has logged in to your server, check `plugins/StreamerPlugin/streamers.yml` — it's auto-populated on join

### Admin commands quick reference

| Command | Effect |
|---|---|
| `/streamadmin reload` | Reloads all YAML configs |
| `/streamadmin gui` | Opens admin panel GUI |
| `/streamadmin announce <player> <platform> <url> [title]` | Force-announces a player as live |
| `/streamer profile` | View your own profile + live status |
| `/streamer setlink <platform> <value>` | Link a social account |
| `/streamer gooffline` | Clear your live status |
| `/golive <platform> [url] [title]` | Manually announce yourself live |
