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
