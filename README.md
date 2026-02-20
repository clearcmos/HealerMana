# HealerMana

A lightweight healer mana tracker for WoW Classic Anniversary Edition (TBC 2.5.5).

## Features

- **Smart Healer Detection** — Automatically identifies healers by inspecting their talent spec. No manual role assignment needed.
- **Mana Display** — Shows each healer's mana percentage with color coding (green → yellow → orange → red)
- **Average Mana** — Displays the average mana across all healers at the top
- **Status Indicators** — Shows when healers are Drinking, have Innervate, or Mana Tide Totem active
- **Potion Tracking** — Tracks potion cooldowns (2 min) via combat log
- **Chat Warnings** — Optional automatic warnings to party/raid when healer mana drops below configurable thresholds
- **Works Everywhere** — Party, raid, battlegrounds, and arena
- **Fully Configurable** — Options panel with font size, scale, opacity, color thresholds, warning thresholds, and more

## Usage

- `/hm` — Open options panel
- `/hm lock` — Toggle frame lock (drag to reposition when unlocked)
- `/hm test` — Show test data to preview the display
- `/hm reset` — Reset all settings to defaults

## How It Works

In Classic Anniversary, players rarely set group roles manually. HealerMana uses a layered detection approach:

1. **Assigned Role** — If someone is assigned as Healer via the group UI, trust it immediately
2. **Class Filter** — Only Priest, Druid, Paladin, and Shaman can be healers
3. **Talent Inspection** — Inspects group members to determine their primary talent tree and whether it's a healing spec

The addon queues inspections automatically, handles range/combat limitations, and caches results so it works seamlessly even during combat.

## Installation

1. Download and extract to your `Interface/AddOns/` folder
2. The folder should be named `HealerMana` containing `HealerMana.toc` and `HealerMana.lua`
3. Restart WoW or `/reload`
