# HealerMana - Development Guide

## Overview

HealerMana is a WoW Classic Anniversary Edition addon that tracks healer mana in group content. It automatically detects healers via talent inspection (since role assignment is unreliable in Classic) and displays their mana percentages with color coding and status indicators. It also tracks raid-wide cooldowns (Innervate, Mana Tide, Bloodlust/Heroism, Power Infusion, Divine Intervention, Rebirth, Lay on Hands) via combat log.

## Architecture

**Single-file addon** — all logic in `HealerMana.lua` (~2020 lines). No XML, no external dependencies.

### Key Systems

1. **Healer Detection Engine** (layered approach):
   - Layer 1: `UnitGroupRolesAssigned()` — trusts assigned roles
   - Layer 2: Class filter — only Priest/Druid/Paladin/Shaman can heal
   - Layer 3: Talent inspection via `C_SpecializationInfo.GetSpecializationInfo()` — checks `role` field first, falls back to `HEALING_TALENT_TABS` mapping when `role` is nil (which is the norm in Classic Anniversary). 5-man fallback assumes healer-capable classes are healers if API returns no data.

2. **Inspection Queue** — async system that queues `NotifyInspect()` calls with 2.5s cooldown, pauses in combat, validates range via `CanInspect()`. Runs on a separate `BackgroundFrame` (always shown) to avoid the hidden-frame OnUpdate deadlock. Periodically re-queues unresolved members.

3. **Raid Cooldown Tracker** — monitors `SPELL_CAST_SUCCESS` (and `SPELL_AURA_APPLIED` for Soulstone) in combat log for key raid cooldowns. Class-baseline cooldowns (Innervate, Rebirth, Lay on Hands, Divine Intervention, Soulstone, Bloodlust/Heroism) are pre-seeded as "Ready" on group scan; player talent cooldowns (Mana Tide, Power Infusion) detected via `IsSpellKnown()`. Multi-rank spells use canonical spell IDs for consistent keys. Displays with class-colored caster names, spell names, and countdown timers (or green "Ready") in a text-only layout matching the healer rows above.

4. **Display System** — movable, resizable frame with BackdropTemplate, object-pooled row frames per healer (class-colored name + mana % + status indicators). Resize handle appears on hover (checked in display OnUpdate to avoid conflicts with drag OnUpdate), clamped to content-driven minimums.

5. **Options GUI** — uses the native WoW Settings API (`Settings.RegisterVerticalLayoutCategory` + `Settings.RegisterAddOnCategory`) so options appear in the AddOns tab of the built-in Options panel (ESC > Options > AddOns > HealerMana). Proxy settings with get/set callbacks to `db`. `/hm` opens directly to the category.

### Code Sections (in order)

| Section | Lines (approx) | Description |
|---------|----------------|-------------|
| S1      | 1-38           | Header, DEFAULT_SETTINGS |
| S2      | 41-81          | Local performance caches |
| S3      | 83-231         | Constants (classes, healing tabs, potions, raid cooldowns, canonical IDs, class/talent mappings) |
| S4      | 233-285        | State variables |
| S5      | 287-405        | Utility functions (iteration, colors, measurement, status formatting) |
| S6      | 407-590        | Healer detection engine (self-spec, inspect results, inspect queue) |
| S7      | 592-762        | Group scanning + class cooldown seeding |
| S8      | 764-800        | Mana updating |
| S9      | 802-849        | Buff/status tracking |
| S10     | 851-858        | Raid cooldown cleanup (no-op, entries kept for Ready state) |
| S11     | 860-897        | Potion + raid cooldown tracking (CLEU) |
| S12     | 899-941        | Warning system |
| S13     | 943-1058       | Display frame + resize handle (hover-to-show) |
| S14     | 1060-1105      | Row frame pool |
| S15     | 1106-1149      | Cooldown row frame pool |
| S16     | 1150-1432      | Display update (healer rows + cooldown rows with Ready state) |
| S17     | 1434-1520      | OnUpdate handler + BackgroundFrame |
| S18     | 1522-1647      | Preview system (mock healers + mock cooldowns) |
| S19     | 1649-1834      | Options GUI (native Settings API) |
| S20     | 1836-1949      | Event handling |
| S21     | 1951-2019      | Slash commands + init |

## Features

- Auto-detect healers via talent inspection
- Color-coded mana display (green/yellow/orange/red with configurable thresholds)
- Dead/DC detection with grey indicators
- Status indicators: Drinking, Innervate, Mana Tide Totem (with optional durations)
- Potion cooldown tracking (2min timer from combat log)
- Soulstone status indicator on dead healers (purple "SS"/"Soulstone")
- Raid cooldown tracking with Ready/on-cooldown states: Innervate, Mana Tide, Bloodlust/Heroism, Power Infusion, Divine Intervention, Rebirth, Lay on Hands, Soulstone
- Average mana across all healers
- Sort healers by lowest mana or alphabetically
- Optional chat warnings at configurable thresholds with cooldown
- Movable, lockable, resizable frame with configurable scale/font/opacity
- Live preview system with animated mock data
- Native options panel in AddOns tab (ESC > Options > AddOns > HealerMana, or `/hm`)
- Show when solo option

## Development Workflow

1. Edit `HealerMana.lua` in the AddOns directory
2. `/reload` in-game to test
3. `/hm test` to show fake healer data without a group
4. Copy changes back to `~/git/mine/HealerMana/`
5. Update version in `.toc` and `CHANGELOG.md`
6. Commit, tag, push to deploy

### New Feature Checklist

When adding any new trackable feature (buff, cooldown, status indicator, etc.), **always** include all three:

1. **Options GUI toggle** — add a `showFeatureName` entry to `DEFAULT_SETTINGS` and a corresponding checkbox in the appropriate section of `RegisterSettings()`. Gate the display logic behind `db.showFeatureName`.
2. **Preview system coverage** — add mock data to `PREVIEW_DATA` and/or `StartPreview()` so the feature is visible in `/hm test` and the settings panel preview. If the feature has a timer, add it to the OnUpdate preview loop logic.
3. **Feature list consistency** — update the Features section in both `CLAUDE.md` and `README.md` to mention the new feature.

## Key APIs Used

- `C_SpecializationInfo.GetSpecializationInfo(i, isInspect, isPet, target, sex, group)` — returns `pointsSpent` per talent tree; `role` field is nil in Classic Anniversary so we fall back to `HEALING_TALENT_TABS` mapping
- `C_SpecializationInfo.GetActiveSpecGroup(isInspect, isPet)` — active talent group (required for dual spec)
- `NotifyInspect(unit)` / `INSPECT_READY` / `ClearInspectPlayer()` — inspection workflow
- `UnitGroupRolesAssigned(unit)` — assigned role check (rarely set in Classic)
- `CombatLogGetCurrentEventInfo()` — potion and raid cooldown tracking
- `UnitBuff(unit, index)` — buff scanning
- `GetPlayerInfoByGUID(guid)` — class lookup for cooldown casters
- `IsSpellKnown(spellId)` — player talent cooldown detection
- `Settings.RegisterVerticalLayoutCategory()` / `Settings.RegisterAddOnCategory()` / `Settings.RegisterProxySetting()` / `Settings.OpenToCategory()` — native options panel
