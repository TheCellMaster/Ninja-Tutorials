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
