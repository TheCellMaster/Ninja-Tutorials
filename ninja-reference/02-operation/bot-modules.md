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
