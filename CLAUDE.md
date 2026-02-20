# HealerMana - Development Guide

## Overview

HealerMana is a WoW Classic Anniversary Edition addon that tracks healer mana in group content. It automatically detects healers via talent inspection (since role assignment is unreliable in Classic) and displays their mana percentages with color coding and status indicators. It also tracks raid-wide cooldowns (Innervate, Mana Tide, Bloodlust/Heroism, Power Infusion, Divine Intervention, Rebirth, Lay on Hands) via combat log.

## Architecture

**Single-file addon** — all logic in `HealerMana.lua` (~2340 lines). No XML, no external dependencies.

### Key Systems

1. **Healer Detection Engine** (layered approach):
   - Layer 1: `UnitGroupRolesAssigned()` — trusts assigned roles
   - Layer 2: Class filter — only Priest/Druid/Paladin/Shaman can heal
   - Layer 3: Talent inspection via `C_SpecializationInfo.GetSpecializationInfo()` — checks `role` field first, falls back to `HEALING_TALENT_TABS` mapping when `role` is nil (which is the norm in Classic Anniversary). 5-man fallback assumes healer-capable classes are healers if API returns no data.

2. **Inspection Queue** — async system that queues `NotifyInspect()` calls with 2.5s cooldown, pauses in combat, validates range via `CanInspect()`. Runs on a separate `BackgroundFrame` (always shown) to avoid the hidden-frame OnUpdate deadlock. Periodically re-queues unresolved members.

3. **Raid Cooldown Tracker** — monitors `SPELL_CAST_SUCCESS` in combat log for key raid cooldowns. Tracks per-caster, auto-expires, displays with spell icons and countdown timers in a separate section below the healer mana rows. Multi-rank spells (Rebirth, Lay on Hands) share a single info table.

4. **Display System** — movable, resizable frame with BackdropTemplate, object-pooled row frames per healer (class-colored name + mana % + status indicators). Resize handle visible when unlocked, clamped to content-driven minimums.

5. **Options GUI** — Ace3-style widgets (sliders, checkboxes, dropdowns) created in Lua, lazy-loaded on first `/hm` call. Live preview with animated mock data while options are open.

### Code Sections (in order)

| Section | Lines (approx) | Description |
|---------|----------------|-------------|
| S1      | 1-39           | Header, DEFAULT_SETTINGS |
| S2      | 41-81          | Local performance caches |
| S3      | 83-211         | Constants (classes, healing tabs, potions, spell names, raid cooldowns) |
| S4      | 213-265        | State variables |
| S5      | 267-385        | Utility functions (iteration, colors, measurement, status formatting) |
| S6      | 387-570        | Healer detection engine (self-spec, inspect results, inspect queue) |
| S7      | 572-663        | Group scanning |
| S8      | 665-700        | Mana updating |
| S9      | 702-749        | Buff/status tracking |
| S10     | 751-762        | Raid cooldown cleanup |
| S11     | 764-800        | Potion + raid cooldown tracking (CLEU) |
| S12     | 802-844        | Warning system |
| S13     | 846-954        | Display frame + resize handle |
| S14     | 956-999        | Row frame pool |
| S15     | 1001-1044      | Cooldown row frame pool |
| S16     | 1046-1314      | Display update (healer rows + cooldown rows) |
| S17     | 1316-1396      | OnUpdate handler + BackgroundFrame |
| S18     | 1398-1523      | Preview system (mock healers + mock cooldowns) |
| S19     | 1525-2148      | Options GUI |
| S20     | 2150-2257      | Event handling |
| S21     | 2259-2336      | Slash commands + init |

## Features

- Auto-detect healers via talent inspection
- Color-coded mana display (green/yellow/orange/red with configurable thresholds)
- Dead/DC detection with grey indicators
- Status indicators: Drinking, Innervate, Mana Tide Totem (with optional durations)
- Potion cooldown tracking (2min timer from combat log)
- Raid cooldown tracking: Innervate, Mana Tide, Bloodlust/Heroism, Power Infusion, Divine Intervention, Rebirth, Lay on Hands
- Average mana across all healers
- Sort healers by lowest mana or alphabetically
- Optional chat warnings at configurable thresholds with cooldown
- Movable, lockable, resizable frame with configurable scale/font/opacity
- Live preview system with animated mock data
- Full options GUI via `/hm`
- Show when solo option

## Development Workflow

1. Edit `HealerMana.lua` in the AddOns directory
2. `/reload` in-game to test
3. `/hm test` to show fake healer data without a group
4. Copy changes back to `~/git/mine/HealerMana/`
5. Update version in `.toc` and `CHANGELOG.md`
6. Commit, tag, push to deploy

## Key APIs Used

- `C_SpecializationInfo.GetSpecializationInfo(i, isInspect, isPet, target, sex, group)` — returns `pointsSpent` per talent tree; `role` field is nil in Classic Anniversary so we fall back to `HEALING_TALENT_TABS` mapping
- `C_SpecializationInfo.GetActiveSpecGroup(isInspect, isPet)` — active talent group (required for dual spec)
- `NotifyInspect(unit)` / `INSPECT_READY` / `ClearInspectPlayer()` — inspection workflow
- `UnitGroupRolesAssigned(unit)` — assigned role check (rarely set in Classic)
- `CombatLogGetCurrentEventInfo()` — potion and raid cooldown tracking
- `UnitBuff(unit, index)` — buff scanning
- `GetPlayerInfoByGUID(guid)` — class lookup for cooldown casters
