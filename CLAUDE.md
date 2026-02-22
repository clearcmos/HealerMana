# Triage - Development Guide

## Overview

Triage is a WoW Classic Anniversary Edition addon that tracks healer mana in group content. It automatically detects healers via talent inspection (since role assignment is unreliable in Classic) and displays their mana percentages with color coding and status indicators. It also tracks raid-wide cooldowns (Innervate, Mana Tide, Bloodlust/Heroism, Power Infusion, Rebirth, Lay on Hands, Soulstone, Symbol of Hope) via combat log.

## Architecture

**Single-file addon** — all logic in `Triage.lua` (~3840 lines). No XML, no external dependencies.

### Key Systems

1. **Healer Detection Engine** (layered approach):
   - Layer 1: `UnitGroupRolesAssigned()` — trusts assigned roles
   - Layer 2: Class filter — only Priest/Druid/Paladin/Shaman can heal
   - Layer 3: Talent inspection via `C_SpecializationInfo.GetSpecializationInfo()` — checks `role` field first, falls back to `HEALING_TALENT_TABS` mapping when `role` is nil (which is the norm in Classic Anniversary). 5-man fallback assumes healer-capable classes are healers if API returns no data.

2. **Inspection Queue** — async system that queues `NotifyInspect()` calls with 2.5s cooldown, pauses in combat, validates range via `CanInspect()`. Runs on a separate `BackgroundFrame` (always shown) to avoid the hidden-frame OnUpdate deadlock. Periodically re-queues unresolved members.

3. **Raid Cooldown Tracker** — monitors `SPELL_CAST_SUCCESS` (and `SPELL_AURA_APPLIED` for Soulstone) in combat log for key raid cooldowns. Class-baseline cooldowns (Innervate, Rebirth, Lay on Hands, Soulstone, Bloodlust/Heroism) are pre-seeded as "Ready" on group scan; player talent cooldowns (Mana Tide, Power Infusion) detected via `IsSpellKnown()`. Multi-rank spells use canonical spell IDs for consistent keys. Displays with class-colored caster names, spell names, and countdown timers (or green "Ready"). Supports text, icon, and icon+label display modes.

4. **Display System** — two independently movable, resizable frames (`TriageFrame` for healer rows, `CooldownFrame` for raid cooldowns) with BackdropTemplate. `splitFrames` setting (default false) controls whether cooldowns render in their own frame or merge into TriageFrame. Object-pooled row frames reparented on acquire. Resize handles appear on hover, clamped to content-driven minimums. All periodic logic (preview animation, mana updates, display refresh) runs on BackgroundFrame to avoid hidden-frame OnUpdate deadlock.

5. **Cooldown Request System** — click-to-whisper system for requesting cooldowns. Clicking a healer row or cooldown row opens a context menu to whisper a request. Includes target selection submenus, caster priority logic (prefers longest time since last cast), and subgroup-aware mana checks for targeted spells.

6. **Options GUI** — uses the native WoW Settings API (`Settings.RegisterVerticalLayoutCategory` + `Settings.RegisterAddOnCategory`) so options appear in the AddOns tab of the built-in Options panel (ESC > Options > AddOns > Triage). Proxy settings with get/set callbacks to `db`. `/triage` opens directly to the category.

### Code Sections (in order)

| Section | Lines (approx) | Description |
|---------|----------------|-------------|
| S1      | 1-58           | Header, DEFAULT_SETTINGS (incl. splitFrames, cdFrame* keys, per-cooldown cd* toggles) |
| S2      | 59-105         | Local performance caches |
| S3      | 106-336        | Constants (classes, healing tabs, potions, raid cooldowns, canonical IDs, COOLDOWN_SETTING_KEY, class/talent/tank mappings) |
| S4      | 337-412        | State variables (incl. cdFrame resize state, context menu, subgroup tracking) |
| S5      | 413-581        | Utility functions (iteration, colors, measurement, status formatting) |
| S6      | 582-834        | Healer detection engine (self-spec, inspect results, inspect queue) |
| S7      | 835-1108       | Group scanning + class cooldown seeding + cooldown grouping |
| S8      | 1109-1147      | Mana updating |
| S9      | 1148-1208      | Buff/status tracking |
| S10     | 1209-1293      | Potion + raid cooldown tracking (CLEU) |
| S11     | 1294-1322      | Warning system |
| S12     | 1323-1455      | Triage display frame + resize handle |
| S13     | 1456-1563      | CooldownFrame + resize handle (split mode) |
| S14     | 1564-1644      | Row frame pool (UIParent-parented, reparented on acquire) |
| S15     | 1645-1714      | Cooldown row frame pool (persistent, reused in place) |
| S16     | 1715-2325      | Cooldown request menu (click-to-whisper, target selection, caster menus) |
| S17     | 2326-3012      | Display update (PrepareHealerRowData, RenderHealerRows, RenderCooldownRows, RefreshHealerDisplay, RefreshCooldownDisplay, RefreshMergedDisplay, RefreshDisplay dispatcher) |
| S18     | 3013-3099      | OnUpdate handler (all logic on BackgroundFrame; per-frame resize-hover only) |
| S19     | 3100-3342      | Preview system (mock healers + mock cooldowns, both frames) |
| S20     | 3343-3597      | Options GUI (native Settings API, splitFrames checkbox, per-cooldown toggles) |
| S21     | 3598-3746      | Event handling (both frame positions restored) |
| S22     | 3747-3837      | Slash commands + init (lock/reset apply to both frames) |

## Features

- Auto-detect healers via talent inspection
- Color-coded mana display (green/yellow/orange/red with configurable thresholds)
- Dead/DC detection with grey indicators
- Status indicators: Drinking, Innervate, Mana Tide Totem, Symbol of Hope (text or icon mode, with optional durations)
- Potion cooldown tracking (2min timer from combat log)
- Soulstone status indicator on dead healers (purple "SS"/"Soulstone")
- Raid cooldown tracking with Ready/on-cooldown states: Innervate, Mana Tide, Bloodlust/Heroism, Power Infusion, Rebirth, Lay on Hands, Soulstone, Symbol of Hope (each individually toggleable)
- Cooldown display modes: text only, icons only, or icons with labels
- Click-to-request cooldowns via whisper (healer rows and cooldown rows)
- Average mana across all healers
- Sort healers by lowest mana or alphabetically
- Optional chat warnings at configurable threshold with cooldown
- Split or merged display: raid cooldowns in their own frame or combined with healer mana
- Movable, lockable, resizable frames with configurable scale/font/opacity
- Row highlights and header backgrounds
- Live preview system with animated mock data
- Native options panel in AddOns tab (ESC > Options > AddOns > Triage, or `/triage`)

## Development Workflow

1. Edit `Triage.lua` in the AddOns directory
2. `/reload` in-game to test
3. `/triage test` to show fake healer data without a group
4. Copy changes back to `~/git/mine/HealerMana/`
5. Update version in `.toc` and `CHANGELOG.md`
6. Commit, tag, push to deploy

### New Feature Checklist

When adding any new trackable feature (buff, cooldown, status indicator, etc.), **always** include all three:

1. **Options GUI toggle** — add a `showFeatureName` entry to `DEFAULT_SETTINGS` and a corresponding checkbox in the appropriate section of `RegisterSettings()`. Gate the display logic behind `db.showFeatureName`.
2. **Preview system coverage** — add mock data to `PREVIEW_DATA` and/or `StartPreview()` so the feature is visible in `/triage test` and the settings panel preview. If the feature has a timer, add it to the OnUpdate preview loop logic.
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
