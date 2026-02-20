# HealerMana - Development Guide

## Overview

HealerMana is a WoW Classic Anniversary Edition addon that tracks healer mana in group content. It automatically detects healers via talent inspection (since role assignment is unreliable in Classic) and displays their mana percentages with color coding and status indicators.

## Architecture

**Single-file addon** — all logic in `HealerMana.lua` (~1770 lines). No XML, no external dependencies.

### Key Systems

1. **Healer Detection Engine** (layered approach):
   - Layer 1: `UnitGroupRolesAssigned()` — trusts assigned roles
   - Layer 2: Class filter — only Priest/Druid/Paladin/Shaman can heal
   - Layer 3: Talent inspection via `C_SpecializationInfo.GetSpecializationInfo()` — checks `role` field first, falls back to `HEALING_TALENT_TABS` mapping when `role` is nil (which is the norm in Classic Anniversary). 5-man fallback assumes healer-capable classes are healers if API returns no data.

2. **Inspection Queue** — async system that queues `NotifyInspect()` calls with 2.5s cooldown, pauses in combat, validates range via `CanInspect()`. Runs on a separate `BackgroundFrame` (always shown) to avoid the hidden-frame OnUpdate deadlock. Periodically re-queues unresolved members.

3. **Display System** — movable frame with BackdropTemplate, object-pooled row frames per healer (class-colored name + mana % + status indicators)

4. **Options GUI** — Ace3-style widgets (sliders, checkboxes, dropdowns) created in Lua, lazy-loaded on first `/hm` call

### Code Sections (in order)

| Section | Lines (approx) | Description |
|---------|----------------|-------------|
| S1-S2   | 1-35           | Header, DEFAULT_SETTINGS |
| S3      | 37-68          | Local performance caches |
| S4      | 69-134         | Constants (classes, healing tabs, potions, spell names) |
| S5      | 135-171        | State variables |
| S6      | 173-237        | Utility functions |
| S7      | 238-394        | Healer detection engine |
| S8      | 396-478        | Group scanning |
| S9      | 480-515        | Mana updating |
| S10     | 517-559        | Buff/status tracking |
| S11     | 560-576        | Potion tracking (CLEU) |
| S12     | 577-620        | Warning system |
| S13     | 621-667        | Display frame |
| S14     | 668-721        | Row frame pool |
| S15     | 722-887        | Display update |
| S16     | 888-951        | OnUpdate handler + BackgroundFrame |
| S17     | 952-1015       | Preview system |
| S18     | 1016-1587      | Options GUI |
| S19     | 1589-1696      | Event handling |
| S20     | 1697-1772      | Slash commands + init |

## Features

- Auto-detect healers via talent inspection
- Color-coded mana display (green/yellow/orange/red)
- Status indicators: Drinking, Innervate, Mana Tide Totem
- Potion cooldown tracking (2min timer from combat log)
- Average mana across all healers
- Optional chat warnings at configurable thresholds
- Movable, lockable frame with configurable scale/font/opacity
- Full options GUI via `/hm`

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
- `CombatLogGetCurrentEventInfo()` — potion tracking
- `UnitBuff(unit, index)` — buff scanning
