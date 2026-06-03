# guthscpbase-integration

A Claude Code skill that teaches Claude how to make any Garry's Mod GLua addon configurable via the **guthscpbase** in-game VGUI admin panel — without breaking addons that don't have guthscpbase installed.

## Install

```
/plugin install guthscpbase-integration
```

## What this skill does

When invoked, Claude will:

1. **Scan the addon** — reads all Lua files and identifies hardcoded values that could be configurable (damage numbers, colors, enabled flags, job lists, etc.)
2. **Propose a config list** — presents grouped options with recommendations (what should be in the menu vs. what should stay hardcoded) and waits for your confirmation
3. **Implement both layers:**
   - `sh_guthscp_config.lua` — registers the config block with guthscpbase, with all selected options shown in the in-game panel
   - `sh_config.lua` (updated) — standalone fallback with the same defaults, works without guthscpbase

## When it triggers

Automatically when you're working on a GLua addon and mention:
- guthscpbase
- in-game config / config menu / VGUI config
- "make X configurable in-game"
- DForm, config panel, admin menu

## Requirements

- [guthscpbase](https://steamcommunity.com/workshop/filedetails/) must be installed on the server for the in-game UI to appear
- The fallback `sh_config.lua` always works without it

## Config types supported

| Type | Widget |
|------|--------|
| `Bool` | Checkbox |
| `Number` | Number input with min/max |
| `String` | Text entry |
| `Color` | Color picker |
| `Teams` | Job checkbox grid |
| `Team` | Job dropdown |
| `Enum` | Dropdown from a Lua table |
| `ComboBox` | Custom dropdown |
| `InputKey` | Key binder |
| `String[]` | Multiple text entries |
| `Category` | Section header (UI only) |
| `Label` | Static text (UI only) |
| `Button` | Action button (UI only) |

## Design principle

The skill mirrors all guthscpbase values back into the addon's own `MYADDON_CONFIG` table via the `guthscp.config:applied` hook. The rest of the addon never imports or references guthscp directly — it just reads `MYADDON_CONFIG` as always. Dropping or adding guthscpbase requires no changes outside the integration file.
