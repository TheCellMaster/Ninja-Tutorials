# OGame Ninja — Advanced Reference (PUBLIC bundle, single file)

> All reference docs concatenated for one-shot ingestion by an AI. Anonymized: every value is a fictional placeholder, never real config.

---

<!-- ===== FILE: 01-control-plane/panel-http-api.md ===== -->

---
title: Panel HTTP API & Database Reference
category: http-api
layer: control-plane
audience: advanced
tags: [http-api, rest-api, sqlite, csrf, sse, websocket, playwright, bot-management]
summary: Complete HTTP endpoint + SQLite schema reference for the OGame Ninja panel (127.0.0.1:8080). Create, configure and monitor bots programmatically.
---

# OGame Ninja Panel — API & Database Reference

> Complete technical reference for the OGame Ninja bot management panel.
> Covers HTTP endpoints, request/response contracts, real-time channels,
> and the local SQLite database schema.

| | |
|---|---|
| **Panel version** | 1.15.7 (Go backend) |
| **Base URL** | `http://127.0.0.1:8080` |
| **Database** | `~/.ogame/ogame.db` (SQLite, read-only) |
| **Last updated** | 2026-04-08 |

---

## Table of Contents

- [1. Overview](#1-overview)
- [2. Quick Start](#2-quick-start)
- [3. Authentication & CSRF](#3-authentication--csrf)
- [4. Request / Response Conventions](#4-request--response-conventions)
- [5. Endpoints — Bot Lifecycle](#5-endpoints--bot-lifecycle)
- [6. Endpoints — Bot Settings](#6-endpoints--bot-settings)
- [7. Endpoints — Modules](#7-endpoints--modules)
- [8. Endpoints — Scripts (REST API v1)](#8-endpoints--scripts-rest-api-v1)
- [9. Endpoints — Fingerprints](#9-endpoints--fingerprints)
- [10. Endpoints — Pages (GET)](#10-endpoints--pages-get)
- [11. WebSocket — Real-time Events](#11-websocket--real-time-events)
- [12. Database Schema](#12-database-schema)
- [13. Data Access Matrix — DB vs API](#13-data-access-matrix--db-vs-api)
- [14. Code Examples](#14-code-examples)
- [15. Error Handling](#15-error-handling)
- [16. Common Workflows](#16-common-workflows)
- [Appendix A — Endpoint Quick Reference](#appendix-a--endpoint-quick-reference)
- [Appendix B — Notification Flag Reference](#appendix-b--notification-flag-reference)
- [Appendix C — Field Constraints & Enums](#appendix-c--field-constraints--enums)
- [Appendix D — Response Body Examples](#appendix-d--response-body-examples)
- [Appendix E — Known Gotchas](#appendix-e--known-gotchas)
- [Appendix F — Glossary](#appendix-f--glossary)
- [Appendix G — Integration Status](#appendix-g--integration-status)
- [Appendix H — Game-side Endpoints (out of scope)](#appendix-h--game-side-endpoints-out-of-scope)

---

## 1. Overview

> **Prerequisites**: OGame Ninja panel running at `127.0.0.1:8080`, Python 3.10+, Playwright installed.

OGame Ninja is a self-hosted Go application that manages OGame bot accounts.
It exposes a web panel at `127.0.0.1:8080` with two interfaces:

| Interface | Purpose |
|-----------|---------|
| **HTML pages** | Browser UI for configuration, served via standard HTTP |
| **REST API** (`/api/v1/...`) | Programmatic endpoints for scripts and fingerprints |

All data is persisted in a local SQLite database (`ogame.db`). Some fields are
stored as **encrypted BLOBs** (credentials, module configs); the rest is
**plaintext** and directly queryable.

### Two ways to interact with bot data

| Method | Best for | Speed |
|--------|----------|-------|
| **Direct DB read** (`ogame.db`) | Reading any plaintext data (logs, scripts, player info, module flags, planets) | Instant (~0.05s) |
| **HTTP API / Form POST** | Writing data (creating bots, configuring modules, managing scripts) | ~0.5-2s |

> **Rule of thumb**: Read from the database. Write through the API.

---

## 2. Quick Start

### Verify the panel is running

```bash
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8080/
# Expected: 200
```

### Read a bot from the database (no panel needed)

```python
import sqlite3, os

db_path = os.path.expanduser("~/.ogame/ogame.db")
conn = sqlite3.connect(f"file:{db_path}?mode=ro", uri=True, timeout=5)
cursor = conn.cursor()

cursor.execute("""
    SELECT b.id, b.player_name, s.server_name, s.community, b.active
    FROM bots b JOIN servers s ON b.server_id = s.id
    WHERE b.deleted_at IS NULL
""")
for bot_id, name, server, lang, active in cursor.fetchall():
    print(f"#{bot_id} {name} | {server}.{lang} | active={active}")
# Output: #53 PlayerName | ServerA.en | active=1

conn.close()
```

### Make your first API call (Playwright)

```python
from playwright.async_api import async_playwright

async def main():
    async with async_playwright() as p:
        browser = await p.chromium.launch()
        page = await browser.new_page()

        # Load any page to get CSRF
        await page.goto("http://127.0.0.1:8080/bots/53/scripts")
        csrf = await page.locator('input[name="csrf"]').first.get_attribute("value")

        # Create a script via API
        resp = await page.request.post(
            "http://127.0.0.1:8080/api/v1/bots/53/new-script",
            multipart={"csrf": csrf, "scriptName": "Test.ank"}
        )
        print(f"Status: {resp.status}, Script ID: {(await resp.text()).strip()}")
        # Output: Status: 200, Script ID: 766

        await browser.close()
```

### Path parameters

All endpoints use `{id}` as the bot or resource identifier:

| Placeholder | Source | Example | Description |
|-------------|--------|---------|-------------|
| `{id}` (in `/bots/{id}/...`) | `bots.id` column | `53` | Auto-incremented integer, reused after deletion |
| `{id}` (in `/fingerprints/{id}/...`) | `bot_fingerprints.id` | `56` | Fingerprint profile ID |
| `{script_id}` | `scripts.id` column | `753` | Returned by `POST .../new-script` |

> **Important**: Bot IDs are **reused**. When a bot is deleted and a new one is created,
> the new bot may receive the same ID as the deleted one. Logs from the previous bot
> remain in the database. Always filter by `bots.created_at` when querying logs.

---

## 3. Authentication & CSRF

### Session

The panel runs on localhost without user authentication headers or cookies.
Session state is implicit (single-user local process).

### CSRF Token

Every `POST` request requires a `csrf` parameter. The token is:
- **Per-session** — one token stays valid for the entire panel session
- Embedded as a hidden `<input>` on every HTML page
- Sent as a form body parameter (never as a header)

**Extraction** (from any page that has been loaded):
```python
csrf = await page.locator('input[name="csrf"]').first.get_attribute("value")
```

---

## 4. Request / Response Conventions

### Content Types

| Content-Type | Used by |
|-------------|---------|
| `application/x-www-form-urlencoded` | All form endpoints (bot creation, settings, modules, fingerprints) |
| `multipart/form-data` | Script API (`new-script`, `save-script`, `run-script`, `reorder-scripts`), file uploads (`/bots/import`) |
| `application/json` | `GET /api/v1/random-fingerprint` response only |

### Response Pattern (Post-Redirect-Get)

The panel follows the PRG pattern for form submissions:

| Outcome | HTTP Status | Body |
|---------|-------------|------|
| **Success** | `302 Found` | `Location` header redirects back to the originating page |
| **Failure** | `200 OK` | Re-rendered HTML form with inline error messages |

The REST API (`/api/v1/...`) uses standard status codes instead:

| Status | Meaning |
|--------|---------|
| `200 OK` | Operation completed (body may contain result data) |
| `202 Accepted` | Async operation started (e.g., script execution) |

### Rate Limiting

Observed on the root endpoint:
```
X-Ratelimit-Limit: 4
X-Ratelimit-Remaining: 3
X-Ratelimit-Reset: <unix_timestamp>
```

### Honeypot Fields

The bot creation form (`POST /bots/new`) includes three anti-scraping
honeypot fields. They **must** be present with these exact values:

| Field | Required value |
|-------|---------------|
| `hack_username` | `john` |
| `hack_email` | `noreply@ogame.ninja` |
| `hack_password` | `nopassword` |

Omitting or changing these values will silently reject the submission.

---

## 5. Endpoints — Bot Lifecycle

### `POST /bots/new` — Create bot

Creates a new bot account. This is the most complex endpoint with 22 parameters.

| | |
|---|---|
| **Method** | `POST` |
| **Content-Type** | `application/x-www-form-urlencoded` |
| **Success** | `302` -> `/bots/{new_id}/initialization` |
| **Failure** | `200` with re-rendered HTML form |

**Parameters:**

| Parameter | Type | Required | Example | Description |
|-----------|------|----------|---------|-------------|
| `csrf` | string | Yes | | Session CSRF token |
| `hack_username` | string | Yes | `john` | Honeypot — fixed value |
| `hack_email` | string | Yes | `noreply@ogame.ninja` | Honeypot — fixed value |
| `hack_password` | string | Yes | `nopassword` | Honeypot — fixed value |
| `sp_port_used` | int | Yes | `0` | Special port flag |
| `ls_used` | string | Yes | `` | Reserved (send empty) |
| `challenge_id` | string | Yes | `` | Reserved (send empty) |
| `fingerprint` | int | Yes | `56` | Fingerprint profile ID |
| `createLobby` | int | Yes | `1` | Create through lobby flow |
| `email` | string | Yes | `user@example.com` | OGame account email |
| `password` | string | Yes | `EXAMPLE_PASSWORD` | OGame account password |
| `lang` | string | Yes | `en` | Community code (lowercase) |
| `universe` | string | Yes | `ServerB` | Server name |
| `lobby` | string | Yes | `lobby` | Lobby type (fixed value) |
| `proxy_type` | string | Yes | `http` | `http` / `socks5` / `none` |
| `proxy` | string | Yes | `//myproxy.example:12321` | Proxy address |
| `proxy_username` | string | Yes | `proxy-user-example` | Proxy auth username |
| `proxy_password` | string | Yes | `EXAMPLE_PROXY_PASSWORD` | Proxy auth password |
| `otp_secret` | string | No | `` | TOTP 2FA secret (empty if none) |
| `bearer_token` | string | No | `` | Bearer token (empty if none) |
| `player_id` | string | No | `` | Player ID (auto-detected if empty) |
| `proxy_login_only` | int | No | `1` | `1` = use proxy only during login |

---

### `POST /bots/{id}/delete` — Delete bot

| | |
|---|---|
| **Method** | `POST` |
| **Content-Type** | `application/x-www-form-urlencoded` |
| **Success** | `200` (HTML, redirects to home) |
| **Failure** | `500` (JSON — e.g., nil pointer for blocked bots) |

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `csrf` | string | `EXAMPLE_CSRF_TOKEN` | Session CSRF token |

---

### `GET /bots/{id}/export` — Export bot as XML

Downloads a full XML backup of the bot configuration.

| | |
|---|---|
| **Method** | `GET` |
| **Response** | `200`, `Content-Type: application/xml` |
| **Header** | `Content-Disposition: attachment; filename="bot-Name-Server-lang.xml"` |

No parameters required. Authentication is implicit via session.

---

### `POST /bots/import` — Import bot from XML

| | |
|---|---|
| **Method** | `POST` |
| **Content-Type** | `multipart/form-data` |
| **Success** | `302` (redirect to home) |

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `csrf` | string | `EXAMPLE_CSRF_TOKEN` | Session CSRF token |
| `file` | file | `bot-PlayerName-ServerA-en.xml` | `.xml` backup file |

---

## 6. Endpoints — Bot Settings

### `POST /bots/{id}/settings` — General settings

| | |
|---|---|
| **Method** | `POST` |
| **Content-Type** | `application/x-www-form-urlencoded` |
| **Success** | `302` |

This endpoint handles **two distinct forms** on the same URL,
distinguished by the `formName` parameter.

#### Form A: `formName=general`

Saves general bot configuration (fingerprint, timezone, slots, etc.).

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `csrf` | string | `EXAMPLE_CSRF_TOKEN` | Session CSRF token |
| `formName` | string | `general` | **Required** — identifies this form variant |
| `bot_fingerprint` | int | `56` | Fingerprint profile ID |
| `bot_timezone` | string | `host` | Timezone mode |
| `homeworld` | string | `` | Homeworld selector (empty = auto-detect) |
| `dropdownPlanetsSort` | int | `0` | Planet sort order in dropdown |
| `preferred_transport` | int | `202` | Ship ID for transport missions |
| `preferred_expedition_cargo` | int | `202` | Ship ID for expedition cargo |
| `nb_probes` | int | `0` | Number of probes to send |
| `conqueror_max_reports` | int | `25` | Max conqueror reports to keep |
| `reserved_slots` | int | `1` | Reserved fleet slots |
| `max_farmer_slots` | int | `0` | Max farmer slots (0 = unlimited) |
| `max_expeditions_slots` | int | `0` | Max expedition slots |
| `max_brain_slots` | int | `0` | Max brain slots |
| `max_repatriate_slots` | int | `0` | Max repatriate slots |
| `max_colonizer_slots` | int | `0` | Max colonizer slots |
| `max_discovery_slots` | int | `0` | Max discovery slots |
| `max_spyer_slots` | int | `0` | Max spyer slots |
| `max_script_slots` | int | `0` | Max script slots |
| `logLoginIP` | int | `1` | `1` = log login IP addresses |
| `logProxyPassword` | int | `1` | `1` = log proxy password in logs |
| `user_agent` | string | `Mozilla/5.0 ...` | Browser user-agent string |

#### Form B: `formName=validateAccount`

Submits the email validation link to activate a new bot account.

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `csrf` | string | `EXAMPLE_CSRF_TOKEN` | Session CSRF token |
| `formName` | string | `validateAccount` | **Required** — identifies this form variant |
| `code` | string | `https://lobby.ogame.gameforge.com/validation/{uuid}` | Full validation URL from email |

---

### `POST /bots/{id}/settings/notifications` — Discord notifications

Configures Discord webhook and notification preferences.

| | |
|---|---|
| **Method** | `POST` |
| **Content-Type** | `application/x-www-form-urlencoded` |
| **Success** | `302` |

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `csrf` | string | `EXAMPLE_CSRF_TOKEN` | Session CSRF token |
| `discord_webhook_url` | string | `https://discord.com/api/webhooks/...` | Discord webhook URL |
| `discord_user_ids` | string | `My Label` | Discord user IDs or label |
| `telegram` | string | `` | Telegram config (empty if unused) |
| `email` | string | `` | Email notification config (empty if unused) |

Additionally, include one `int` (`1` = enabled) for each notification flag.
See [Appendix B](#appendix-b--notification-flag-reference) for the full list of 21 Discord flags.

---

## 7. Endpoints — Modules

### `POST /bots/{id}/brain` — Configure Brain module

| | |
|---|---|
| **Method** | `POST` |
| **Content-Type** | `application/x-www-form-urlencoded` |
| **Success** | `302` |

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `csrf` | string | `EXAMPLE_CSRF_TOKEN` | Session CSRF token |
| `formName` | string | `brain_configs` | Required discriminator |
| `brainActive` | int | `1` | `1` = enable Brain module |
| `mode_{celestial_id}` | int | `2` | **Dynamic** — one field per celestial. Values: `0` = off, `1` = manual, `2` = AI |
| `delay_after_export_min` | int | `10` | Minimum delay after export (seconds) |
| `delay_after_export_max` | int | `30` | Maximum delay after export (seconds) |

> The `mode_{celestial_id}` fields are dynamic: the panel generates one per
> celestial (planet/moon) owned by the bot. The `{celestial_id}` is the OGame
> planet ID (e.g., `mode_438`). To set all celestials to AI, iterate over all
> such fields and set each to `2`.

---

### `POST /bots/{id}/scanner` — Configure Scanner module

| | |
|---|---|
| **Method** | `POST` |
| **Content-Type** | `application/x-www-form-urlencoded` |
| **Success** | `302` |

This endpoint handles **three distinct actions** via the `formName` parameter
(or its absence).

#### Action A: Configure scanner settings (no `formName`)

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `csrf` | string | `EXAMPLE_CSRF_TOKEN` | Session CSRF token |
| `scannerActive` | int | `1` | `1` = enable Scanner |
| `scan_frequency` | int | `1` | Scan frequency (hours) |
| `page_interval` | int | `250` | Delay between page loads (ms) |
| `celestial_id` | int | `0` | Home celestial ID (`0` = default) |

#### Action B: Start highscore crawl

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `csrf` | string | `EXAMPLE_CSRF_TOKEN` | Session CSRF token |
| `formName` | string | `crawlHighscores` | Fixed value — triggers highscore crawl |

#### Action C: Start universe scan

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `csrf` | string | `EXAMPLE_CSRF_TOKEN` | Session CSRF token |
| `formName` | string | `scanUniverse` | Fixed value — triggers full universe scan |

---

### `POST /bots/{id}/hunter` — Configure Hunter module

| | |
|---|---|
| **Method** | `POST` |
| **Content-Type** | `application/x-www-form-urlencoded` |
| **Success** | `302` |

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `csrf` | string | `EXAMPLE_CSRF_TOKEN` | Session CSRF token |
| `hunterActive` | int | `1` | `1` = enable Hunter |
| `scan_frequency` | int | `5` | Scan frequency (minutes) |
| `celestial_id` | int | `0` | Home celestial ID (`0` = default) |

---

## 8. Endpoints — Scripts (REST API v1)

These are true REST endpoints under `/api/v1/`. Unlike the form endpoints,
they accept `multipart/form-data` and return status codes (not redirects).

### `POST /api/v1/bots/{id}/new-script`

Creates a new empty script file.

| | |
|---|---|
| **Method** | `POST` |
| **Content-Type** | `multipart/form-data` |
| **Success** | `200` — body contains the new script ID (plain text) |

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `csrf` | string | `EXAMPLE_CSRF_TOKEN` | Session CSRF token |
| `scriptName` | string | `My_Script.ank` | Script filename |

---

### `POST /api/v1/bots/{id}/save-script`

Saves script content and the "run at start" flag.

| | |
|---|---|
| **Method** | `POST` |
| **Content-Type** | `multipart/form-data` |
| **Success** | `200` — empty body |

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `csrf` | string | `EXAMPLE_CSRF_TOKEN` | Session CSRF token |
| `scriptID` | int | `753` | Script ID (returned by `new-script`) |
| `runAtStart` | string | `true` | `true` = enabled, omit field = disabled |
| `script` | string | `func main() { ... }` | Full source code (`.ank` content) |

---

### `POST /api/v1/bots/{id}/run-script`

Executes a script immediately.

| | |
|---|---|
| **Method** | `POST` |
| **Content-Type** | `multipart/form-data` |
| **Success** | `202 Accepted` — empty body |

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `csrf` | string | `EXAMPLE_CSRF_TOKEN` | Session CSRF token |
| `scriptID` | int | `753` | Script ID |
| `script` | string | `func main() { ... }` | Source code to execute |

---

### Additional script endpoints

All follow the same pattern: `POST`, `multipart/form-data`, CSRF required.

| Method | Endpoint | Description | Parameters (with examples) |
|--------|----------|-------------|---------------------------|
| `POST` | `.../start-script` | Start script in background | `csrf`=`EXAMPLE...`, `scriptID`=`753` |
| `POST` | `.../stop-script` | Stop running script | `csrf`=`EXAMPLE...`, `scriptID`=`753` |
| `POST` | `.../delete-script` | Delete a script | `csrf`=`EXAMPLE...`, `scriptID`=`753` |
| `POST` | `.../rename-script` | Rename a script | `csrf`=`EXAMPLE...`, `scriptID`=`753`, `scriptName`=`NewName.ank` |
| `POST` | `.../check-script` | Syntax check | `csrf`=`EXAMPLE...`, `scriptID`=`753`, `script`=`func main()...` |
| `POST` | `.../validate-script` | Validate script | `csrf`=`EXAMPLE...`, `scriptID`=`753`, `script`=`func main()...` |
| `POST` | `.../reorder-scripts` | Change script order | `csrf`=`EXAMPLE...`, `scriptIDs`=`[753,754,755]` |

---

## 9. Endpoints — Fingerprints

### `GET /api/v1/random-fingerprint` — Generate random fingerprint

| | |
|---|---|
| **Method** | `GET` |
| **Response** | `200`, `Content-Type: application/json` (~1.9 KB) |

No parameters required. Returns a JSON object with all fingerprint fields
(user agent, hashes, screen dimensions, etc.) that can be submitted to
`POST /fingerprints/new`.

---

### `POST /fingerprints/new` — Create fingerprint

Submits a complete browser fingerprint profile.

| | |
|---|---|
| **Method** | `POST` |
| **Content-Type** | `application/x-www-form-urlencoded` |
| **Success** | `302` -> `/fingerprints` |

**Identity fields:**

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `csrf` | string | `EXAMPLE_CSRF_TOKEN` | Session CSRF token |
| `fingerprint_name` | string | `ServerA.EN - Random (DD/MM/YY at HH:MM:SS)` | Display name |
| `constant_version` | int | `12` | Fingerprint format version |

**Browser fields:**

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `user_agent` | string | `Mozilla/5.0 (Windows NT 10.0; Win64; x64)...` | Full user-agent string |
| `browser_name` | string | `Chrome` | Browser name |
| `navigator_vendor` | string | `Google Inc.` | `navigator.vendor` value |
| `languages` | string | `en-US` | `navigator.languages` |

**Hardware fields:**

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `os_name` | string | `Windows` | Operating system |
| `version` | string | `10` | OS version |
| `memory` | int | `8` | `navigator.deviceMemory` (GB, `0` if not exposed) |
| `hardware_concurrency` | int | `8` | `navigator.hardwareConcurrency` (CPU cores) |
| `screen_width` | int | `1920` | Screen width in pixels |
| `screen_height` | int | `1040` | Screen height in pixels |
| `timezone` | string | `Europe/Istanbul` | IANA timezone string |

**Fingerprint hash fields:**

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `webgl_info` | string | `Google Inc. (NVIDIA),ANGLE (NVIDIA, ...)` | WebGL `vendor,renderer` |
| `offline_audio_ctx` | float | `124.04345259929687` | `OfflineAudioContext` fingerprint value |
| `canvas_2d_info` | int | `275641662` | Canvas 2D rendering hash |
| `webgl_render_hash` | hex(64) | `1a3a60f1e2b3c4d5...` | WebGL rendering hash |
| `permissions_states_hash` | hex(64) | `a1b2c3d4e5f6a7b8...` | Permissions API state hash |
| `media_devices_hash` | hex(64) | `f0e1d2c3b4a59687...` | `navigator.mediaDevices` hash |
| `plugins_hash` | hex(64) | `1234567890abcdef...` | `navigator.plugins` hash |
| `fonts_hash` | hex(64) | `abcdef1234567890...` | Available fonts hash |
| `audio_hash` | hex(64) | `9876543210fedcba...` | Audio fingerprint hash |
| `audio_ctx_hash` | hex(64) | `0a1b2c3d4e5f6789...` | AudioContext hash |
| `video_hash` | hex(64) | `fedcba9876543210...` | Video capabilities hash |

**Anti-bot challenge fields (from Gameforge `game1.js`):**

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `date_iso` | string | `2026-01-01T00:00:00.000Z` | ISO timestamp from game |
| `x_game` | string | `EXAMPLE_GAME_TOKEN` | Game session token |
| `calc_delta_ms` | int | `199` | Calculated time delta (ms) |
| `game1_date_header` | string | `2026-01-01T00:00:00.000Z` | `game1.js` Date header |

---

### `POST /fingerprints/{id}/delete` — Delete fingerprint

| | |
|---|---|
| **Method** | `POST` |
| **Content-Type** | `application/x-www-form-urlencoded` |
| **Success** | `302` -> `/fingerprints` |

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `csrf` | string | `EXAMPLE_CSRF_TOKEN` | Session CSRF token |

---

## 10. Endpoints — Pages (GET)

HTML pages served by the panel. Primarily used for browser navigation
and CSRF token extraction. No parameters required.

| Method | Endpoint | Description | Response size |
|--------|----------|-------------|---------------|
| `GET` | `/` | Dashboard — lists all bots | ~36 KB |
| `GET` | `/admin/bots` | Admin panel — creation dates, IDs | varies |
| `GET` | `/bots/new` | Bot creation form (includes server/universe dropdowns) | ~487 KB |
| `GET` | `/bots/{id}/initialization` | Post-creation initialization status | ~19 KB |
| `GET` | `/bots/{id}/settings` | General bot settings | ~148 KB |
| `GET` | `/bots/{id}/settings/notifications` | Notification preferences | ~65 KB |
| `GET` | `/bots/{id}/brain` | Brain module configuration | ~193 KB |
| `GET` | `/bots/{id}/scanner` | Scanner module configuration | ~49 KB |
| `GET` | `/bots/{id}/hunter` | Hunter module configuration | varies |
| `GET` | `/bots/{id}/scripts` | Script list (redirects to first script) | `302` |
| `GET` | `/bots/{id}/scripts/{script_id}` | Individual script editor | ~111-232 KB |
| `GET` | `/bots/{id}/logs` | Bot activity logs viewer | varies |
| `GET` | `/fingerprints` | Fingerprint profiles list | ~23 KB |
| `GET` | `/fingerprints/new` | Fingerprint creation form | ~34 KB |

---

## 11. WebSocket — Real-time Events

### `GET /sse/{channels}`

Upgrades to WebSocket (`ws://127.0.0.1:8080/sse/...`).
Returns `101 Switching Protocols`.

Channels are comma-separated in the URL path. The panel subscribes
to different channels depending on the active page.

### Channel patterns

| Pattern | Example | Purpose |
|---------|---------|---------|
| `bot_{id}` | `bot_52` | Bot status updates (running, stopped, errors) |
| `bott_{id}_init` | `bott_52_init` | Initialization progress (note: double `t`) |
| `scanner_{id}` | `scanner_52` | Live scanner results |
| `script_{id}` | `script_751` | Script execution status |
| `script_log_{id}` | `script_log_751` | Script execution output logs |
| `global_notifications` | — | System-wide notifications |
| `user_{id}` | `user_1` | User-specific notifications |

### Typical subscriptions by page

| Page | Channels |
|------|----------|
| Dashboard | `bot_47,bot_51,bot_52,...,global_notifications,user_1` |
| Bot detail | `bot_52,global_notifications,user_1` |
| Scanner | `scanner_52` |
| Scripts | `script_log_751,script_751,...,bot_52,global_notifications,user_1` |
| Other pages | `global_notifications,user_1` |

---

## 12. Database Schema

### Connection

```python
import sqlite3
conn = sqlite3.connect("file:~/.ogame/ogame.db?mode=ro", uri=True, timeout=5)
```

Access is **read-only**. The panel process owns the database; external
readers must use `mode=ro` and short-lived connections to avoid lock contention.

### Tables overview

**Core tables (with typical row counts):**

| Table | Rows | Description |
|-------|------|-------------|
| `bots` | 3 | Bot records — central hub for all relationships |
| `scripts` | 40 | Script source code (`.ank` files, full content) |
| `logs` | 5,000 | Bot activity logs with level and category |
| `bot_planets` | 30 | Complete planet data: buildings, ships, defenses, resources (293 columns) |
| `bot_fingerprints` | 3 | Browser fingerprint profiles |
| `bot_logins` | 20 | Login history |
| `sessions` | 40 | Panel login sessions |
| `servers` | 2 | OGame server definitions |

**Universe data:**

| Table | Rows | Description |
|-------|------|-------------|
| `players` | 5,000 | All known players across servers |
| `player_history` | 300,000 | Historical rank/points snapshots |
| `alliances` | 500 | All known alliances |
| `alliance_history` | 150,000 | Historical alliance snapshots |
| `planets` | 10,000 | Galaxy scan data (all planets in universe) |

**Combat & fleet:**

| Table | Rows | Description |
|-------|------|-------------|
| `espionage_reports` | 1,000 | Full spy reports (buildings, fleet, research, resources) |
| `sent_fleets` | 2,000 | Record of fleets sent by bots |
| `fleets` | 2,000 | Fleet tracking (ID, initiator, recall status) |
| `hunter_targets` | 50 | Players being tracked by Hunter |
| `hunter_target_activities` | 300 | Activity observations for hunter targets |

**Enums:**

| Table | Rows | Description |
|-------|------|-------------|
| `expedition_categories` | 13 | `alien`, `dm`, `resource`, `fleet`, `pirate`, etc. |
| `farm_target_states` | 11 | `waiting`, `spying_1`, `attacking_1`, etc. |
| `fleet_initiators` | 11 | `player`, `farmer`, `brain`, `defender`, `script`, etc. |

---

### Table: `bots`

The `bots` table is the central entity. 24 child tables reference `bots.id`.

#### Plaintext columns (directly queryable)

| Category | Columns |
|----------|---------|
| **Identity** | `id`, `player_name`, `player_id`, `points`, `rank`, `total`, `honour_points`, `character_class`, `alliance_class` |
| **Status** | `active`, `created_at`, `updated_at`, `deleted_at` |
| **Server** | `server_id` (FK), `server_number`, `lobby`, `nb_systems`, `galaxies`, `systems`, `homeworld` |
| **Module flags** | `brain_active`, `scanner_active`, `hunter_active`, `farmer_active`, `defender_active`, `expeditions_active`, `colonizer_active`, `discovery_active`, `spyer_active`, `auctioneer_active`, `crawler_active`, `repatriate_active` |
| **Brain** | `brain_mode` (0=off, 1=auto) |
| **Proxy** | `proxy` (URL), `proxy_username`, `proxy_type` (`http`/`socks5`/`none`), `proxy_login_only`, `socks5_address`, `socks5_username` |
| **User agent** | `user_agent` |
| **Sleep schedule** | `sleep_active`, `sleep_mode`, `sleep_delay`, `wake_delay`, `sleep_from_hour_{day}`/`sleep_to_hour_{day}` (per weekday) |
| **Slot limits** | `max_farmer_slots`, `max_brain_slots`, `max_repatriate_slots`, `max_script_slots`, `max_expeditions_slots`, `max_colonizer_slots`, `max_discovery_slots`, `max_spyer_slots`, `reserved_slots` |
| **Research** | All 16 research fields: `espionage_technology`, `computer_technology`, `weapons_technology`, `shielding_technology`, `armour_technology`, `energy_technology`, `hyperspace_technology`, `combustion_drive`, `impulse_drive`, `hyperspace_drive`, `laser_technology`, `ion_technology`, `plasma_technology`, `intergalactic_research_network`, `astrophysics`, `graviton_technology` |
| **Officers** | `has_commander`, `has_admiral`, `has_engineer`, `has_geologist`, `has_technocrat` |
| **Notifications** | ~70 boolean flags: `notif_discord_*`, `notif_telegram_*`, `notif_slack_*` |
| **Other** | `idx`, `tz_offset`, `bot_fingerprint` (FK), `notepad`, `commander_ui`, `augments_enabled`, `conqueror_max_reports`, `nb_probes`, `auto_fleet_save`, `fleet_save_recall`, `log_login_ip`, `log_proxy_password` |

#### Encrypted columns (BLOB — require panel UI or API to read)

| Category | Columns | Typical size |
|----------|---------|-------------|
| **Credentials** | `email`, `password`, `universe`, `lang` | 30-48 bytes |
| **Proxy secrets** | `proxy_password`, `socks5_password` | varies |
| **Discord** | `discord_webhook_url`, `discord_user_ids` | 40-149 bytes |
| **Module configs** | `brain_configs`, `scanner_configs`, `hunter_configs`, `defender_configs`, `crawler_configs`, `expeditions_configs`, `colonizer_configs`, `discovery_configs`, `spyer_configs`, `auctioneer_configs` | 808 B to 99 KB |
| **2FA** | `two_factor_secret`, `two_factor_refresh_token`, `two_factor_user_id`, `two_factor_recovery_token` | varies |
| **Notification endpoints** | `notif_telegram_chat_id`, `notif_email`, `notif_phone`, `notif_slack` | varies |
| **Other** | `bearer_token`, `hotkeys`, `topraider_username`, `topraider_password`, `plugins` | varies |

> **Exception**: `lf_bonuses` is typed as BLOB but contains readable JSON (lifeform bonus data). `plugins` contains `{}` in UTF-8.

---

### Table: `scripts`

| Column | Type | Description |
|--------|------|-------------|
| `id` | INTEGER | Primary key — this is the `scriptID` used in the API |
| `script_name` | VARCHAR(50) | Filename (e.g., `My_Script.ank`) |
| `bot_id` | INTEGER | FK -> `bots.id` |
| `run_at_start` | TINYINT | `0` = disabled, `1` = auto-run on bot start |
| `script` | TEXT | **Full source code** — plaintext, directly readable |
| `idx` | INTEGER | Display/execution order (0-based) |
| `binary_script` | BLOB | Not used (always `NULL`) |
| `created_at` | DATETIME | |
| `updated_at` | DATETIME | |

> **Special script names**: `!sleep.ank` (runs before sleep mode), `!wake.ank` (runs on wake).

---

### Table: `logs`

| Column | Type | Description |
|--------|------|-------------|
| `id` | INTEGER | Primary key |
| `bot_id` | INTEGER | FK -> `bots.id` |
| `message` | TEXT | Log content, format: `[module:line] message text` |
| `level` | INTEGER | `1` = DEBUG, `2` = INFO, `3` = WARN, `4` = ERROR |
| `category` | VARCHAR | Module category (see below) |
| `created_at` | DATETIME | Timestamp |

**Log categories (by frequency):**

| Category | Count | Description |
|----------|-------|-------------|
| `VM` | 3,774 | Virtual machine events |
| `HEARTBEAT` | 1,087 | Keep-alive pings |
| `DEFENDER` | 685 | Defense operations |
| `DISCOVERY` | 655 | Discovery module events |
| `BRAIN` | 573 | Brain module decisions |
| `SPYER` | 485 | Spyer operations |
| `AUDIT` | 315 | Audit trail |
| `COLONIZER` | 195 | Colonizer operations |
| `GENERAL` | 170 | General events |
| `SCANNER` | 83 | Scanner results |
| `SLOTS` | 65 | Fleet slot management |
| `CRAWLER` | 41 | Crawler events |
| `LOGIN_MANAGER` | 38 | Login operations |
| `HUNTER` | 1 | Hunter events |

---

### Table: `bot_planets` (293 columns)

Comprehensive planet data for each celestial owned by a bot.

| Category | Example columns |
|----------|----------------|
| **Location** | `galaxy`, `system`, `position`, `planet_id`, `planet_type` (`1`=planet, `3`=moon) |
| **Buildings** | `metal_mine`, `crystal_mine`, `deuterium_synthesizer`, `solar_plant`, `fusion_reactor`, `nanite_factory`, ... |
| **Ships** | `light_fighter`, `heavy_fighter`, `cruiser`, `battleship`, `bomber`, `destroyer`, `deathstar`, `reaper`, `pathfinder`, ... |
| **Defenses** | `rocket_launcher`, `light_laser`, `heavy_laser`, `gauss_cannon`, `ion_cannon`, `plasma_turret`, ... |
| **Resources** | `metal`, `crystal`, `deuterium`, `energy` |
| **Production** | `metal_mine_setting`, `crystal_mine_setting`, etc. (0-100%) |
| **Brain config** | `brain_mode`, `evacuation_mode`, `producer`, `exporter`, `importer` |
| **Lifeform** | `lifeform_type`, all LF buildings and researches |

---

### Table: `servers`

| Column | Example values |
|--------|---------------|
| `server_name` | `ServerA`, `ServerB`, `ServerC`, `ServerD` |
| `community` | `en`, `de` |
| `server_number` | `123`, `124`, `125`, `126` |

---

### Child tables of `bots`

24 tables reference `bots.id`:

`bot_planets`, `scripts`, `logs`, `fleets`, `sent_fleets`, `espionage_reports`,
`bot_logins`, `farm_sessions`, `evacuations`, `flights`, `cronjobs`,
`bots_hunter_targets`, `ignored_players`, `ignored_alliances`, `ignored_planets`,
`defender_whitelisted_players`, `expedition_messages`, `scheduled_flights`,
`recent_flights`, `multi_phalanx_reports`, `phalanx_sessions`, `spyer_sessions`,
`shared_bots`, `user_bots`

---

## 13. Data Access Matrix — DB vs API

### When to use each

| Data | Source | Method |
|------|--------|--------|
| Player name, rank, points | DB | `SELECT player_name, rank, points FROM bots WHERE id = ?` |
| Server / community | DB | `JOIN bots ON servers` |
| Creation date | DB | `SELECT created_at FROM bots WHERE id = ?` |
| Active status | DB | `SELECT active FROM bots WHERE id = ?` |
| Module on/off flags | DB | `SELECT brain_active, scanner_active, ... FROM bots` |
| Brain mode | DB | `SELECT brain_mode FROM bots WHERE id = ?` |
| Proxy URL / username / type | DB | `SELECT proxy, proxy_username, proxy_type FROM bots` |
| User agent | DB | `SELECT user_agent FROM bots WHERE id = ?` |
| All 16 research levels | DB | `SELECT espionage_technology, ... FROM bots` |
| Officer flags | DB | `SELECT has_commander, ... FROM bots` |
| Sleep schedule | DB | `SELECT sleep_active, sleep_from_hour_*, ... FROM bots` |
| Slot limits | DB | `SELECT max_farmer_slots, ... FROM bots` |
| Notification toggle flags | DB | `SELECT notif_discord_*, ... FROM bots` |
| Activity logs | DB | `SELECT * FROM logs WHERE bot_id = ?` |
| Script source code | DB | `SELECT script_name, script, run_at_start FROM scripts WHERE bot_id = ?` |
| Planet data (buildings, ships, resources) | DB | `SELECT * FROM bot_planets WHERE bot_id = ?` |
| Espionage reports | DB | `SELECT * FROM espionage_reports WHERE bot_id = ?` |
| Fleet history | DB | `SELECT * FROM sent_fleets WHERE bot_id = ?` |
| Hunter targets | DB | `SELECT * FROM hunter_targets ...` |
| Galaxy scan (all players/planets) | DB | `SELECT * FROM planets` / `SELECT * FROM players` |
| | | |
| Email address | API | Playwright: extract from dashboard HTML |
| Password | API | Playwright: extract from settings HTML |
| Discord webhook URL | API | Playwright: extract from notifications HTML |
| Discord user IDs | API | Playwright: extract from notifications HTML |
| Proxy password | API | Playwright: extract from settings HTML |
| Module detailed configs | API | Playwright: encrypted BLOB in DB |
| 2FA secrets | API | Playwright: encrypted BLOB in DB |

---

## 14. Code Examples

### Reading data from the database (Python)

```python
import sqlite3

DB_PATH = "~/.ogame/ogame.db"

def get_connection():
    return sqlite3.connect(f"file:{DB_PATH}?mode=ro", uri=True, timeout=5)

# Get bot info
conn = get_connection()
cursor = conn.cursor()
cursor.execute("""
    SELECT b.id, b.player_name, b.active, s.server_name, s.community
    FROM bots b
    JOIN servers s ON b.server_id = s.id
    WHERE b.deleted_at IS NULL
    ORDER BY b.id
""")
for row in cursor.fetchall():
    print(f"Bot #{row[0]}: {row[1]} | {row[3]}.{row[4]} | active={row[2]}")
conn.close()
```

### Reading script source code from the database

```python
conn = get_connection()
cursor = conn.cursor()
cursor.execute("""
    SELECT script_name, run_at_start, script, idx
    FROM scripts WHERE bot_id = ? ORDER BY idx ASC
""", (53,))
for name, run_at_start, content, idx in cursor.fetchall():
    print(f"[{idx}] {name} | run_at_start={run_at_start} | {len(content)} chars")
conn.close()
```

### Making API calls with Playwright (Python)

```python
# 1. Extract CSRF token (once per session)
csrf = await page.locator('input[name="csrf"]').first.get_attribute("value")

# 2. POST to any endpoint using page.request (shares browser cookies)
# Note: Script API v1 endpoints require multipart/form-data (not form-urlencoded)
response = await page.request.post(
    "http://127.0.0.1:8080/api/v1/bots/53/new-script",
    multipart={"csrf": csrf, "scriptName": "MyScript.ank"}
)
script_id = (await response.text()).strip()  # e.g., "753"

# 3. Save content with run-at-start enabled
await page.request.post(
    "http://127.0.0.1:8080/api/v1/bots/53/save-script",
    multipart={
        "csrf": csrf,
        "scriptID": script_id,
        "runAtStart": "true",
        "script": 'func main() { log("Hello") }'
    }
)

# 4. Run the script
response = await page.request.post(
    "http://127.0.0.1:8080/api/v1/bots/53/run-script",
    multipart={"csrf": csrf, "scriptID": script_id, "script": 'func main() { log("Hello") }'}
)
assert response.status == 202  # Accepted
```

### Configuring a module via direct POST (Python)

```python
# Configure scanner — no page.goto() needed, just POST directly
response = await page.request.post(
    "http://127.0.0.1:8080/bots/53/scanner",
    form={
        "csrf": csrf,
        "scannerActive": "1",
        "scan_frequency": "1",
        "page_interval": "250",
        "celestial_id": "0"
    }
)
# Success = 302 redirect, Failure = 200 with HTML errors
print(f"Status: {response.status}")
```

### Deleting a bot via API (Python)

```python
# Navigate to settings first to get a valid CSRF token
await page.goto("http://127.0.0.1:8080/bots/53/settings")
csrf = await page.locator('input[name="csrf"]').first.get_attribute("value")

# Direct HTTP POST — no click needed
response = await page.request.post(
    "http://127.0.0.1:8080/bots/53/delete",
    form={"csrf": csrf}
)
if response.status == 200:
    print("Bot deleted")
elif response.status == 500:
    error = await response.json()
    print(f"Error: {error}")
```

---

## 15. Error Handling

### Form endpoints (302/200 pattern)

| Status | Meaning | How to detect |
|--------|---------|---------------|
| `302` | Success — follow `Location` header | `response.status == 302` |
| `200` | Failure — form re-rendered with errors | `response.status == 200` + parse HTML for `.alert-danger` |
| `500` | Server error (rare — e.g., nil pointer) | `response.status >= 500` + body may be JSON |

### API v1 endpoints (status code pattern)

| Status | Meaning | How to detect |
|--------|---------|---------------|
| `200` | Success — body may contain data | Check `response.status == 200` |
| `202` | Accepted — async operation started | Check `response.status == 202` |
| `400` | Bad request — invalid parameters | Check response body for details |
| `404` | Bot or script not found | Verify bot ID exists |
| `500` | Internal server error | Retry or report |

### Common failure scenarios

| Scenario | Endpoint | Status | Cause |
|----------|----------|--------|-------|
| Invalid CSRF | Any POST | `403` or `200` (re-rendered) | Token expired or missing |
| Bot creation - Gameforge down | `POST /bots/new` | `200` | HTML contains error message inline |
| Bot creation - account exists | `POST /bots/new` | `200` | "Account already in lobby" in HTML |
| Delete blocked bot | `POST /bots/{id}/delete` | `500` | JSON body with nil pointer error |
| Script not found | `POST .../save-script` | `404` | Invalid `scriptID` |
| Missing honeypot | `POST /bots/new` | `200` | Silent rejection — form re-rendered |

---

## 16. Common Workflows

### Workflow 1: Replace a banned bot

```
1. READ  DB    → query_bot_scripts_full(old_bot_id) → get scripts with content
2. READ  DB    → query_bot_module_status(old_bot_id) → get brain/scanner/hunter flags
3. GET   API   → /bots/{old_id}/export                → download XML backup
4. POST  API   → /bots/{old_id}/delete                 → delete old bot
5. POST  API   → /bots/new                             → create new bot
6. POST  API   → /bots/{new_id}/settings (validate)    → validate account
7. POST  API   → /bots/{new_id}/brain                  → configure brain
8. POST  API   → /bots/{new_id}/scanner                → configure scanner
9. POST  API   → /bots/{new_id}/hunter                 → configure hunter
10. POST API   → /bots/{new_id}/settings/notifications  → configure discord
11. POST API   → /api/v1/.../new-script (per script)    → create scripts
12. POST API   → /api/v1/.../save-script (per script)   → save content + run_at_start
13. POST API   → /api/v1/.../run-script (per script)    → start scripts
```

### Workflow 2: Backup bot configuration

```
1. READ  DB  → bots table        → player_name, module flags, proxy, research
2. READ  DB  → scripts table     → full script source code
3. READ  DB  → bot_planets table → buildings, ships, defenses, resources
4. GET   API → /bots/{id}/export → XML backup (contains encrypted fields)
5. READ  HTML → notifications page → discord webhook, user IDs (encrypted in DB)
```

### Workflow 3: Monitor bot health

```
1. READ  DB  → logs table        → filter by bot_id + last 48h
2. READ  DB  → bots.active       → check if still active
3. READ  DB  → bots.*_active     → verify modules are enabled
4. WS    SSE → bot_{id} channel  → real-time status updates
```

---

## Appendix A — Endpoint Quick Reference

| # | Method | Endpoint | Content-Type | Success |
|---|--------|----------|-------------|---------|
| 1 | POST | `/bots/new` | form-urlencoded | `302` -> init page |
| 2 | POST | `/bots/{id}/delete` | form-urlencoded | `200` |
| 3 | GET | `/bots/{id}/export` | — | `200` XML |
| 4 | POST | `/bots/import` | multipart | `302` |
| 5 | POST | `/bots/{id}/settings` | form-urlencoded | `302` |
| 6 | POST | `/bots/{id}/settings/notifications` | form-urlencoded | `302` |
| 7 | POST | `/bots/{id}/brain` | form-urlencoded | `302` |
| 8 | POST | `/bots/{id}/scanner` | form-urlencoded | `302` |
| 9 | POST | `/bots/{id}/hunter` | form-urlencoded | `302` |
| 10 | POST | `/api/v1/bots/{id}/new-script` | multipart | `200` + ID |
| 11 | POST | `/api/v1/bots/{id}/save-script` | multipart | `200` |
| 12 | POST | `/api/v1/bots/{id}/run-script` | multipart | `202` |
| 13 | POST | `/api/v1/bots/{id}/start-script` | multipart | `200` |
| 14 | POST | `/api/v1/bots/{id}/stop-script` | multipart | `200` |
| 15 | POST | `/api/v1/bots/{id}/delete-script` | multipart | `200` |
| 16 | POST | `/api/v1/bots/{id}/rename-script` | multipart | `200` |
| 17 | POST | `/api/v1/bots/{id}/check-script` | multipart | `200` |
| 18 | POST | `/api/v1/bots/{id}/validate-script` | multipart | `200` |
| 19 | POST | `/api/v1/bots/{id}/reorder-scripts` | multipart | `200` |
| 20 | GET | `/api/v1/random-fingerprint` | — | `200` JSON |
| 21 | POST | `/fingerprints/new` | form-urlencoded | `302` |
| 22 | POST | `/fingerprints/{id}/delete` | form-urlencoded | `302` |

---

## Appendix B — Notification Flag Reference

All notification flags are `int` (`1` = enabled, absent = disabled).

### Discord notifications (21 flags)

| Flag | Event |
|------|-------|
| `notif_discord_login_fails` | Login failure |
| `notif_discord_player_chat` | Player chat message |
| `notif_discord_alliance_chat` | Alliance chat message |
| `notif_discord_celestial_destroyed` | Planet or moon destroyed |
| `notif_discord_new_attack` | Incoming attack |
| `notif_discord_new_missiles_attack` | Incoming missile attack |
| `notif_discord_evacuation_fails` | Fleet evacuation failure |
| `notif_discord_phalanx_watched_fleet_recall` | Phalanx-watched fleet recalled |
| `notif_discord_multi_phalanx_report_completed` | Multi-phalanx report done |
| `notif_discord_farm_session_started` | Farm session started |
| `notif_discord_farm_session_completed` | Farm session completed |
| `notif_discord_enter_sleep` | Bot entering sleep mode |
| `notif_discord_exit_sleep` | Bot exiting sleep mode |
| `notif_discord_fleet_save_recalled` | Fleet save recalled |
| `notif_discord_fleet_save_failed` | Fleet save failed |
| `notif_discord_expe_loss` | Expedition loss |
| `notif_discord_hunter_target_inactive` | Hunter target went inactive |
| `notif_discord_hunter_target_entered_vacation_mode` | Target entered vacation |
| `notif_discord_hunter_target_left_vacation_mode` | Target left vacation |
| `notif_discord_hunter_target_banned` | Target was banned |
| `notif_discord_spyer_completed` | Spyer session completed |

---

## Appendix C — Field Constraints & Enums

### String length limits (observed)

| Field | Max length | Example |
|-------|-----------|---------|
| `player_name` | 17 chars | `PlayerName Sample` |
| `proxy` | 23 chars | `//myproxy.example:12321` |
| `user_agent` | 121+ chars | `Mozilla/5.0 (Windows NT 10.0; Win64; x64)...` |
| `script_name` | 50 chars | `My_Long_Script_Name.ank` |

### Integer enums

| Field | Values | Description |
|-------|--------|-------------|
| `character_class` | `0` = none, `1` = Collector, `2` = General, `3` = Discoverer | OGame player class |
| `brain_mode` | `0` = off, `1` = auto | Brain automation level |
| `sleep_mode` | `0` = simple, `1` = per-weekday schedule | Sleep configuration type |
| `planet_type` | `1` = planet, `3` = moon | In `bot_planets` table |
| `log level` | `1` = DEBUG, `2` = INFO, `3` = WARN, `4` = ERROR | In `logs` table |
| `mode_{id}` (Brain) | `0` = off, `1` = manual, `2` = AI | Per-celestial brain mode |

### String enums

| Field | Values | Description |
|-------|--------|-------------|
| `proxy_type` | `http`, `socks5`, `none` | Proxy protocol type |
| `formName` (settings) | `general`, `validateAccount` | Distinguishes forms on same URL |
| `formName` (brain) | `brain_configs` | Required form discriminator |
| `formName` (scanner) | `crawlHighscores`, `scanUniverse` | Triggers scan actions |
| `runAtStart` (scripts API) | `true` (present) / omit field (disabled) | Script auto-start flag |
| `lobby` | `lobby` | Fixed value — always `"lobby"` |

### Boolean fields

All boolean fields in the database are `TINYINT`: `0` = false, `1` = true.
In POST form data, send `"1"` for true; **omit the field** or send `"0"` for false.

Exception: `runAtStart` in the Scripts API uses `"true"` (include field) or omits the field entirely, instead of `1` / `0`.

---

## Appendix D — Response Body Examples

### `POST /api/v1/bots/{id}/new-script` — Success (200)

```
753
```
Plain text body containing the newly created script ID.

### `POST /api/v1/bots/{id}/save-script` — Success (200)

```
(empty body)
```

### `POST /api/v1/bots/{id}/run-script` — Success (202)

```
(empty body)
```

### `POST /bots/{id}/delete` — Failure (500)

```json
{"error": "runtime error: invalid memory address or nil pointer dereference"}
```
Occurs when trying to delete a bot that is blocked or in an invalid state.

### `POST /bots/new` — Failure (200)

The HTML body is re-rendered with the creation form. Error messages appear
as inline elements near the relevant field. Look for `class="alert-danger"`
or similar patterns in the HTML.

Common error messages observed in the HTML:
- `"account is blocked"` — Gameforge banned the account
- `"account already in lobby"` — Email already registered
- Challenge/captcha failures from `challenge.gameforge.com`

### `GET /api/v1/random-fingerprint` — Success (200)

```json
{
  "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 ...",
  "browser_name": "Chrome",
  "navigator_vendor": "Google Inc.",
  "os_name": "Windows",
  "version": "10",
  "memory": 8,
  "hardware_concurrency": 8,
  "screen_width": 1920,
  "screen_height": 1040,
  "timezone": "Europe/Berlin",
  "languages": "de-DE",
  "webgl_info": "Google Inc. (NVIDIA),ANGLE (NVIDIA, ...)",
  "offline_audio_ctx": 124.04345259929687,
  "canvas_2d_info": 275641662,
  "webgl_render_hash": "1a3a60f1e2b3c4d5...",
  "permissions_states_hash": "a1b2c3d4e5f6...",
  "media_devices_hash": "f0e1d2c3b4a5...",
  "plugins_hash": "1234567890ab...",
  "fonts_hash": "abcdef123456...",
  "audio_hash": "9876543210fe...",
  "audio_ctx_hash": "0a1b2c3d4e5f...",
  "video_hash": "fedcba987654..."
}
```

### Database: Sample log entry

```
level:      1
category:   "VM"
created_at: "2026-01-01 12:00:00.0000000+00:00"
message:    "[ninjaVM:126] My_Script.ank [✅ #8 OK | 1s | [12 00 00]]"
```

The `created_at` field includes sub-second precision and timezone offset (ISO 8601).
The `message` field uses the format `[module:line] content`.

### Database: Sample bot_planets row (partial)

```
id:                    439
bot_id:                53
galaxy:                1
system:                234
position:              8
planet_id:             12345678
name:                  "Homeworld"
diameter:              12800
fields_built:          37
fields_total:          188
temperature_min:       47
temperature_max:       87
metal_mine:            7
crystal_mine:          6
deuterium_synthesizer: 6
solar_plant:           10
fusion_reactor:        0
```

### Database: Sample espionage_report row (partial)

```
id:        1
server_id: 3
bot_id:    7
galaxy:    1
system:    100
position:  4
metal:     0          (always present — 0 if no resources)
crystal:   0
deuterium: 0
metal_mine: None      (None = not visible in this spy report)
```

> Resource fields are always integers (0 if empty). Building/research/fleet fields
> are **nullable** — `None` means the spy level was too low to see that data.

---

## Appendix E — Known Gotchas

### 1. Bot IDs are reused
When a bot is deleted and a new one is created, the new bot may receive the
**same ID**. Logs from the previous bot remain in the `logs` table. Always
filter by `bots.created_at` to avoid analyzing stale logs.

### 2. CSRF token is session-scoped
One CSRF token works for the entire session. However, if the panel process
restarts, all tokens are invalidated. If you get silent failures, re-extract
the CSRF from any page.

### 3. Honeypot fields are required
`POST /bots/new` will silently reject the request if the three honeypot fields
(`hack_username`, `hack_email`, `hack_password`) are missing or have wrong values.
There is no error message — the form just re-renders.

### 4. `formName` discriminates forms on the same URL
`POST /bots/{id}/settings` serves two different forms. Sending the wrong
`formName` (or omitting it) will cause unexpected behavior. Always include
`formName=general` or `formName=validateAccount` explicitly.

### 5. Encrypted BLOBs cannot be read from the DB
Fields like `email`, `password`, `discord_webhook_url`, and all `*_configs`
columns are AES-encrypted. Reading them from SQLite returns binary garbage.
Use the HTML panel (Playwright) to extract these values.

### 6. Script `idx` ordering may be zero for all scripts
Newly created scripts via the API all get `idx=0`. Use `POST .../reorder-scripts`
if you need specific ordering, or rely on creation order.

### 7. `302` means success, `200` means failure (for form endpoints)
This is counter-intuitive. The panel uses Post-Redirect-Get: a successful
form submission returns `302 Found`, while a failure returns `200 OK` with
the form re-rendered containing error messages.

### 8. Rate limiting on the root endpoint
The panel enforces `X-Ratelimit-Limit: 4` on `GET /`. If you poll the
dashboard frequently, you may hit rate limits. Use the database for
reading bot status instead.

### 9. `scripts.script` column contains full source code
The `script` column in the `scripts` table is **not** a reference — it contains
the complete `.ank` source code as plaintext. Scripts can be up to 182 KB+.
Always use `LENGTH(script)` before loading large scripts into memory.

### 10. `bot_planets` has 293 columns
Be selective with `SELECT` queries on this table. Use specific columns
instead of `SELECT *` to avoid loading unnecessary data.

### 11. `runAtStart` requires `"true"`, not `"on"`
The Go backend parses the `runAtStart` field with `strconv.ParseBool()`.
HTML checkbox convention sends `"on"`, but Go does **not** recognize `"on"` as a
boolean. Use `"true"` when the flag is enabled, or omit the field entirely when
disabled. Sending `"on"` via `application/x-www-form-urlencoded` will silently
store `false` in the database even though the HTTP response is `200 OK`.

### 12. Delete endpoint returns `200`, not `302`
Unlike all other form POST endpoints which follow the PRG pattern (`302` = success),
`POST /bots/{id}/delete` returns `200` with an HTML body on success. Server errors
return `500` with a JSON body. Do not check `!= 302` for deletion — use `>= 400`
to detect failure.

---

## Appendix F — Glossary

| Term | Description |
|------|-------------|
| **Bot** | An automated OGame account managed by the Ninja panel |
| **Celestial** | A planet or moon owned by a bot. Identified by `planet_id` |
| **Brain** | AI module that automates building, researching, and fleet management |
| **Scanner** | Module that crawls the universe and collects player data |
| **Hunter** | Module that tracks specific players and monitors their activity |
| **Spyer** | Module that sends espionage probes to targets |
| **Farmer** | Module that farms inactive players for resources |
| **Defender** | Module that evacuates fleets when an attack is detected |
| **Fingerprint** | Browser fingerprint profile used to avoid detection (user agent, WebGL hashes, etc.) |
| **game1.js** | Gameforge anti-automation script (v12) that collects browser fingerprint data |
| **Lobby** | Gameforge login portal (`lobby.ogame.gameforge.com`) |
| **Community** | Language/region code (e.g., `en`, `fr`, `de`) |
| **Universe** | An OGame server instance (e.g., ServerA, ServerB, ServerC) |
| **Server number** | Numeric ID of the universe (e.g., `123` for ServerA) |
| **CSRF** | Cross-Site Request Forgery token — required on all POST requests |
| **PRG** | Post-Redirect-Get — the panel's response pattern (302 = success, 200 = failure) |
| **Anko (.ank)** | Scripting language used by OGame Ninja for custom bot scripts |
| **SSE** | Server-Sent Events — real-time communication channel (WebSocket upgrade) |
| **Honeypot** | Anti-scraping fields that must be sent with specific dummy values |
| **Residential proxy** | Proxy provider used for geographic consistency |

---

## Appendix G — Integration Status

Current implementation status for each writable endpoint (updated 2026-04-08).

| Operation | Endpoint | Method | Status |
|-----------|----------|--------|--------|
| Delete bot | `POST /bots/{id}/delete` | `page.request.post` | **Done** |
| Export backup | `GET /bots/{id}/export` | `page.request.get` | **Done** |
| New script | `POST /api/v1/.../new-script` | `page.request.post` | **Done** |
| Save script | `POST /api/v1/.../save-script` | `page.request.post` | **Done** |
| Run script | `POST /api/v1/.../run-script` | `page.request.post` | **Done** |
| Reorder scripts | `POST /api/v1/.../reorder-scripts` | `page.request.post` | **Done** |
| Config Scanner | `POST /bots/{id}/scanner` | `page.request.post` | **Done** |
| Start Crawl | `POST /bots/{id}/scanner` | `page.request.post` | **Done** |
| Start Scan | `POST /bots/{id}/scanner` | `page.request.post` | **Done** |
| Config Hunter | `POST /bots/{id}/hunter` | `page.request.post` | **Done** |
| Config Brain | `POST /bots/{id}/brain` | `page.request.post` | **Done** |
| Config Notifications | `POST .../notifications` | `page.request.post` | **Done** |
| Validate Account | `POST /bots/{id}/settings` | `page.request.post` | **Done** |
| Delete Fingerprint | `POST /fingerprints/{id}/delete` | `page.request.post` | **Done** |
| Import Bot XML | `POST /bots/import` | `page.request.post` (multipart) | **Done** |
| | | | |
| Config Log Settings | `POST /bots/{id}/settings` | `evaluate + form.submit` | Intentional |
| Create Bot | `POST /bots/new` | Playwright UI (click) | Intentional |
| Create Fingerprint | `POST /fingerprints/new` | Playwright UI (click) | Intentional |
| Login | `POST /` | Playwright UI (fill + click) | N/A |

**15/19 endpoints migrated to direct API calls** (`page.request.post/get`).

The 3 "Intentional" entries remain on Playwright UI for good reason:
- **Config Log Settings**: `evaluate + form.submit` preserves all 22 form fields without enumerating each one.
- **Create Bot**: Form has dynamic fields (fingerprint dropdown, universe select) that depend on page state.
- **Create Fingerprint**: 28+ dynamically generated fields — serialization would be fragile and maintenance-heavy.

## Appendix H — Game-side endpoints (out of scope)

Endpoints served by the OGame **game** server (`sXXX-<community>.ogame.gameforge.com`) are **outside the
scope** of this bundle, which covers only the Ninja **panel** (`127.0.0.1:8080`). For game-side
automation, refer to the official OGame Ninja documentation (<https://www.ogame.ninja/doc>).


<!-- ===== FILE: 01-control-plane/panel-pages-and-selectors.md ===== -->

---
title: Panel Pages, URLs & HTML Selectors
category: ui-automation
layer: control-plane
audience: advanced
tags: [urls, css-selectors, forms, playwright, navigation]
summary: Panel URLs, form fields and CSS selectors for browser automation (Playwright) of the OGame Ninja panel.
---

# OGame Ninja — Pages, URLs and HTML Selectors

## URLs used by NINJA-MANAGER

| Page | URL | Usage in code |
|---|---|---|
| Home | `/` | Check bot list |
| Create bot | `/bots/new` | `bot_creator_service.py` |
| Import bot | `/bots/import` | `backup.py → import_bot_from_backup()` |
| Initialization | `/bots/{id}/initialization` | Wait after creation |
| Settings | `/bots/{id}/settings` | `backup.py`, `bot_deletion_service.py` |
| Notifications | `/bots/{id}/settings/notifications` | `notifications.py` |
| Brain | `/bots/{id}/brain` | `brain_service.py` |
| Scanner | `/bots/{id}/scanner` | `scanner_service.py` |
| Hunter | `/bots/{id}/hunter` | `hunter_service.py` |
| Logs | `/bots/{id}/logs` | `log_analyzer_service.py` (fallback) |
| Scripts | `/bots/{id}/scripts` | `scripts.py` |
| Backup | `/bots/{id}/settings` → button | `backup.py → backup_bot_before_deletion()` |
| Delete | `/bots/{id}/settings` → button | `bot_deletion_service.py` |
| Admin bots | `/admin/bots` | `bot_deletion_service.py → delete_bot_via_admin()` |
| Fingerprints | `/fingerprints` | `fingerprint_service.py` |
| Fingerprint new | `/fingerprints/new` | `fingerprint_service.py → create_fingerprint()` |

## Bot creation form (`/bots/new`)

```
select#fingerprint              — Fingerprints dropdown
input[name="email"]             — OGame account email
input[name="password"]          — Account password
select#community                — Community (en, de, fr)
select#server                   — Server (ServerA, ServerB)
input[name="proxy"]             — Proxy URL (e.g. //myproxy.example:12321)
input[name="proxy_username"]    — Proxy username
input[name="proxy_password"]    — Proxy password (with country and session)
input#proxy_login_only          — Checkbox: proxy only during login
input#create_lobby              — Checkbox: create lobby account
button[type="submit"]           — Create button
```

## Fingerprint form (`/fingerprints/new`)

```
input[name="fingerprint_name"]  — Fingerprint name
button "Generate random fingerprint" — Generates random values

— Fields auto-filled after Generate: —
input[name="user_agent"]
input[name="browser_name"]
input[name="navigator_vendor"]
input[name="webgl_info"]
input[name="os_name"]
input[name="version"]
input[name="memory"]
input[name="hardware_concurrency"]
input[name="screen_width"]
input[name="screen_height"]
input[name="timezone"]
input[name="languages"]
input[name="constant_version"]     — Set to "12" (current version)
input[name="browser_engine_name"]  — Set to "Blink"
+ 14 hash fields (audio, canvas, webgl, fonts, etc.)

button "Create"                 — Submit fingerprint
```

## Import form (`/bots/import`)

```
input[type="file"][name="file"]  — Upload .xml or .tar
input[type="submit"]             — Upload button
```

## Notifications page (`/bots/{id}/settings/notifications`)

```
input[name="discord_webhook_url"]    — Discord webhook URL
input[name="discord_user_ids"]       — Discord user IDs
input[name^="notif_discord_"]        — Notification checkboxes (14+)
button[type="submit"] "Save"         — Save
```

## Brain page (`/bots/{id}/brain`)

```
input#brainActive                    — Checkbox to enable/disable brain
table th:has-text("AI")              — Header to set all as AI
input[name="try_buy_offer_of_the_day"]   — Offer of the day option
input[name="export_same_galaxy_only"]    — Export same galaxy only
input[name="prevent_import_sleep_time"]  — Prevent import during sleep
form:has(input#brainActive)          — Brain form
button[type="submit"] "Save"         — Save
```

## Settings page (`/bots/{id}/settings`)

```
a "Backup this bot"              — Link to download the .xml
button "Delete this bot"         — Delete button (requires confirmation)
input[name="log_login_ip"]       — Checkbox: log IP
input[name="log_proxy_password"] — Checkbox: log proxy password
```

## Internal API endpoints

| Endpoint | Method | Usage |
|---|---|---|
| `/api/v1/fingerprint` | POST | Submit the browser fingerprint |
| `/sse/{topics}` | WebSocket | Real-time events |
| `/api/v1/notifications/recent-html` | POST | Recent notifications |

## CSRF Token

Every POST action requires a CSRF token:
```html
<input type="hidden" name="csrf" value="TOKEN_HERE">
```


<!-- ===== FILE: 02-operation/bot-modules.md ===== -->

---
title: Bot Modules Reference
category: modules
layer: operation
audience: advanced
tags: [brain, scanner, hunter, defender, farmer, colonizer, spyer, discovery]
summary: What each automation module does (Brain, Scanner, Hunter, Defender, Farmer, Colonizer, Spyer, Discovery) and how it is configured.
---

# OGame Ninja — Bot Modules

## Brain — Planet development automation

**Page:** `/bots/{id}/brain`
**Activation selector:** `input#brainActive`

### Modes
- **AI Mode** — Auto-building with smart prioritization (cheapest mines first)
- **Queue Mode** — Builds only manually queued items
- **None Mode** — Disabled

### Features
- Resource importer/exporter between planets
- Mining limits per resource (metal, crystal, deuterium)
- Energy configuration (solar plants, fusion, satellites)
- Automatic storage expansion
- Lunar priority: Lunar Base → Robotics Factory (lvl 8) → Jump Gate → Sensor Phalanx

### Configuration in NINJA-MANAGER
- `brain_service.py → configure_brain()` enables the checkbox, sets celestials as AI, disables default options
- Backup: `backup.py` extracts `brain_active` from the DB via `query_bot_module_status()`
- `FORCE_BRAIN_ENABLED` in config.py forces activation on all bots

## Scanner — Automatic universe scan

**Page:** `/bots/{id}/scanner`

- Configurable scan frequency (in hours)
- Interval between system scans (in ms)
- `scanner_service.py → configure_scanner()` restores settings from backup

## Hunter — Player monitoring

**Page:** `/bots/{id}/hunter`

- Monitors activity of selected players
- Generates online/offline graphs
- CPU intensive — keep the list small
- Added via galaxy page → "Add to hunter list"

## Defender — Automatic protection

- Attack detection with a random interval (e.g. 10-20 min)
- Evacuation with priority: buildings → research → deuterium → crystal → metal → defenses
- Modes: Dangerous (>1/3 firepower), Always, Never
- Fleet recall after attack
- Alerts: email (SMTP), Telegram, desktop, audio (21s LOUD)
- Probe of suspicious attackers
- Preset message for new attackers

## Farmer — Automatic farming

- Scans galaxies and spies inactive players
- Attacks only safe targets (no defense, no fleet)
- **Intentional throttle:** uses 5-10 of 20 slots (to look human)
- Settings: scan range, probes per mission, resource threshold
- Filters: defense, rank, storage level, activity
- Fast attacking mode (immediate vs waiting for a full scan)
- Origin: nearest moon/planet/hybrid

### Farm Session States
```
pending      → Needs espionage
spy_1        → Reconnaissance en route
spy_2        → Scout returning
spied        → Suitable for assault
attacking_1  → Combat fleet en route
attacking_2  → Fleet returning
completed    → Done
aborted      → Terminated mid-process
```

## Colonizer — Automatic colonization

- Dispatches colony ships to specified coordinates
- Configurable galaxy/system range and position
- Auto-builds colony ships when unavailable
- Continues until it finds a planet with minimum fields

## Spyer — Probe deployment

- Filtering: active, inactive, strongest, by activity
- Probes from the nearest planet/moon with a reserve

## Discovery — Lifeform exploration

- Max discovery slots
- System range for missions
- Leave-open option for reserved slots


<!-- ===== FILE: 02-operation/errors-and-troubleshooting.md ===== -->

---
title: Errors & Troubleshooting
category: troubleshooting
layer: operation
audience: advanced
tags: [errors, troubleshooting, bot-creation, http-403, session]
summary: Known error types during bot creation/operation and the recommended reaction for each.
---

# OGame Ninja — Known Errors and Troubleshooting

## Bot creation errors

### FORBIDDEN 403
**Message:** `FORBIDDEN: 403 Forbidden - https://www.ogame.ninja/faq/new-bot-403-forbidden`

**Causes:**
1. Wrong credentials (Gameforge email/password)
2. Fingerprint problem — creating a new fingerprint fixes it
3. IP temporarily blocked by Gameforge
4. Geographic inconsistency (fingerprint timezone ≠ IP region)

**Action in NINJA-MANAGER:** Switch proxy IP (new country)

### BAD_CREDENTIALS
**Message:** `bad credentials`

**Causes:**
1. Wrong email/password for OGame
2. Gameforge rate limiting per email base (too many aliases)
3. Gameforge account blocked

**Action in NINJA-MANAGER:** Reuse proxy (not an IP issue), progressive backoff, ban the email base after a threshold

### INTERNAL_SERVER_ERROR 500
**Message:** `gameforge internal server error : 500 Internal Server Error`

**Cause:** Unstable Gameforge server or an internal crash

**Action in NINJA-MANAGER:** Switch proxy + fingerprint (reset_all), retry after 5s

### EMAIL_EXISTS
**Message:** `Failed to create new lobby, email already used`

**Cause:** Email alias already used to create an OGame account

**Action in NINJA-MANAGER:** Reuse proxy, generate a new email alias

### NO_LICENSE_AVAILABLE
**Message:** `/bots/new` page redirects to home

**Cause:** All license slots are occupied

**Action in NINJA-MANAGER:** Break the retry loop (no point retrying)

### FINGERPRINT_NOT_IN_LIST
**Message:** Fingerprint dropdown does not contain the desired fingerprint

**Cause:** Fingerprint was deleted or the name does not match

**Action in NINJA-MANAGER:** Generate a new fingerprint (`reset_identity()`, keeps proxy)

### CHALLENGE_ERROR
**Message:** Connection error with `challenge.gameforge.com`

**Cause:** Gameforge challenge server rejected the connection (possibly a blocked IP)

**Action in NINJA-MANAGER:** Switch proxy + fingerprint (`reset_all()`), retry after 5s

### GAME1JS_ERROR
**Message:** Error loading `gameforge.com/tra/game1.js`

**Cause:** Gameforge fingerprinting script unavailable for the IP

**Action in NINJA-MANAGER:** Switch proxy + fingerprint (`reset_all()`), retry after 5s

### LOBBY_ERROR
**Message:** Connection error with `lobby.ogame.gameforge.com`

**Cause:** Lobby server failed or rejected the connection

**Action in NINJA-MANAGER:** Switch proxy + fingerprint (`reset_all()`), retry after 5s

### AUTH_ERROR
**Message:** Authentication error with `gameforge.com/api`

**Cause:** Authentication API failed or rejected the request

**Action in NINJA-MANAGER:** Switch proxy + fingerprint (`reset_all()`), retry after 5s

### SERVER_ERROR
**Message:** Generic OGame server error

**Cause:** OGame server failed to process the request

**Action in NINJA-MANAGER:** Switch proxy + fingerprint (`reset_all()`), retry after 5s

### SESSION_EXPIRED
**Message:** Session expired in the OGame Ninja panel

**Cause:** Session token expired during the retry loop

**Action in NINJA-MANAGER:** Automatic re-login. If it fails, break the loop

### ACCOUNT_IN_LOBBY
**Message:** Account is already in the lobby

**Cause:** Email alias already has an active account in the Gameforge lobby

**Action in NINJA-MANAGER:** Reuse proxy, generate a new email (`reset_email()`)

## Post-creation errors

### Ghost Bot (player_id = 0)
**Detection:** `is_ghost_bot()` checks `player_id == 0` in ogame.db

**Cause:** Bot created but the OGame account was not associated (Gameforge failure)

**Action in NINJA-MANAGER:** Delete the ghost, re-import the original bot from the XML backup

### Validation Failed
**Message:** `context deadline exceeded (Client.Timeout exceeded while awaiting headers)`

**Cause:** OGame server took too long to process account validation

**Action in NINJA-MANAGER:** Delete the non-validated bot, reset state, retry

### Account Blocked
**Message:** `account is blocked`

**Cause:** Account banned by Gameforge (Scripting, Botusing)

**Action in NINJA-MANAGER:** Replace the bot (detected in log analysis)

## Configuration errors

### Suspicious Login
**Cause:** Direct login on the OGame site while the bot is running, or a geographic location change

**Solution:** Change the OGame password and update it in the bot

### Browser Loading Forever
**Cause:** Browser limit of 6 simultaneous connections (OGame fleet page uses 5, websocket 1)

**Solution:** Enable HTTPS in admin settings (enables HTTP/2 multiplexing)

### Clock Sync
**Cause:** System clock out of sync (fingerprint depends on a precise timestamp)

**Solution:** Sync via NTP (Windows: Settings → Time, Linux: `ntpd`)

