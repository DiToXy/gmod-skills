---
name: guthscpbase-integration
description: "How to make a Garry's Mod GLua addon configurable via the guthscpbase in-game VGUI config menu. Use whenever the user wants to add an in-game config panel, make addon values editable without restarting, integrate with guthscpbase, or replace a static config.lua with a live UI. Always produces both a guthscpbase integration AND a fallback config.lua. Trigger on: guthscpbase, ingame config, config menu, VGUI config, configurable addon, DForm config, or any request to make addon values editable in-game."
---

# guthscpbase Integration

`guthscpbase` is a shared Garry's Mod config framework that provides a centralized in-game VGUI admin panel for editing addon settings live — no server restart needed. Config is saved to `data/guthscp/configs/<id>.json` and synced to all clients automatically.

**The golden rule:** every addon must also have a standalone `config.lua` as a fallback. The guthscpbase integration wraps it — it does not replace it.

---

## Workflow

### Step 1 — Scan the addon

Before touching any files, read the addon to find everything that looks like a configurable value:
- Hardcoded numbers, strings, booleans in Lua files (`local DAMAGE = 50`, `local ENABLED = true`)
- Existing `config.lua` / `sh_config.lua` entries
- ConVars
- Table entries that control behavior (allowed ranks, job lists, colors, timers, etc.)

Present a proposed list to the user grouped by category, with a recommendation for each:

```
Found these potential config options:

LIKELY YES (behavior/gameplay values):
  - MYCONFIG.damage = 50          (Number, 0–500)
  - MYCONFIG.enabled = true       (Bool)
  - MYCONFIG.allowed_teams = {}   (Teams)

MAYBE (style/cosmetic):
  - MYCONFIG.color = Color(255,0,0)  (Color)
  - MYCONFIG.title = "My Addon"      (String)

PROBABLY NO (internal/structural):
  - MYCONFIG.net_string = "mynet"    (hardcoded net string name)
  - MYCONFIG.VERSION = "1.0"         (version constant)
```

Ask the user: "Which of these do you want in the in-game config? Which should stay hardcoded in config.lua only?"

Wait for confirmation before writing any code.

### Step 2 — Implement

Once the user has confirmed the list, implement both pieces in parallel:
1. Update (or create) `config.lua` as the canonical fallback
2. Create the guthscpbase module file

---

## File Layout

The correct integration method is the **module system** — place a `main.lua` inside guthscpbase's modules folder. Do NOT use `guthscp.config.add` from an autorun file; the module system is the supported path and is the only one that shows in the panel.

```
myaddon/
├── lua/
│   ├── autorun/
│   │   └── sh_config.lua              ← fallback config (always present)
│   └── guthscp/
│       └── modules/
│           └── <module_id>/
│               └── main.lua           ← guthscpbase module entry point
```

The `<module_id>` must match the folder name exactly — guthscpbase uses it as the config ID.

If the addon already has a `config.lua` or `sh_config.lua`, update it in place rather than creating a duplicate.

---

## Fallback config.lua pattern

The fallback config must work completely independently of guthscpbase. Use the addon's own global config table:

```lua
-- sh_config.lua
MYADDON_CONFIG = MYADDON_CONFIG or {}

-- =============================================
--  MYADDON — Configuration
--  Edit these values if not using guthscpbase.
--  When guthscpbase is active, this file is
--  overridden by the in-game config menu.
-- =============================================

MYADDON_CONFIG.enabled     = true
MYADDON_CONFIG.damage      = 50
MYADDON_CONFIG.color       = Color(255, 100, 50)
MYADDON_CONFIG.allowed_jobs = {}
```

Keep it simple — one value per line, no functions. Admins should be able to edit this without knowing Lua.

---

## guthscpbase module file

```lua
-- lua/guthscp/modules/myaddon/main.lua

local MODULE = {
    name        = "My Addon",
    author      = "Me",
    version     = "1.0.0",
    description = "Short description shown in the guthscpbase menu.",
    icon        = "icon16/brick.png",   -- 16x16 Silkicon or custom material
    dependencies = {
        base = "2.0.0",   -- minimum compatible guthscpbase version
    },
    menu = {
        config = {
            form = {
                -- Categories are plain strings.
                -- Fields are tables nested ONE level inside a table group.
                "General",
                {
                    {
                        type    = "Bool",
                        id      = "enabled",
                        name    = "Enable Addon",
                        desc    = "Optional tooltip shown on hover.",
                        default = true,
                    },
                    {
                        type    = "Number",
                        id      = "damage",
                        name    = "Damage",
                        default = 50,
                        min     = 0,
                        max     = 500,
                    },
                },
                "Access",
                {
                    {
                        type    = "String[]",
                        id      = "allowed_jobs",
                        name    = "Allowed Jobs",
                        default = {},
                    },
                },
            },
        },
    },
}

-- Sync guthscpbase values back into MYADDON_CONFIG whenever config is applied.
-- This keeps the rest of the addon unaware of guthscpbase.
hook.Add("guthscp.config:applied", "myaddon.sync_config", function(id, config)
    if id ~= "myaddon" then return end
    for k, v in pairs(config) do
        MYADDON_CONFIG[k] = v
    end
end)

-- Enable hot-reload during development: save main.lua in-game to reload.
guthscp.module.hot_reload("myaddon")
return MODULE
```

### Critical FORM structure rules

1. **Categories** are bare strings — `"General"`, not `{ type = "Category", name = "General" }`.
2. **Fields** must be **double-nested**: a group table containing field tables.
   ```lua
   -- CORRECT
   "My Category",
   {
       { type = "Bool", id = "foo", name = "Foo", default = true },
       { type = "Number", id = "bar", name = "Bar", default = 5 },
   },

   -- WRONG — fields at top level don't render
   { type = "Bool", id = "foo", name = "Foo", default = true },
   ```
3. Multiple fields in the same group appear side-by-side in the panel.

---

## Config type reference

| Type | Widget | Value | Key fields |
|------|--------|-------|------------|
| `Bool` | Checkbox | `true`/`false` | `default` |
| `Number` | Number input | numeric | `default`, `min`, `max` |
| `String` | Text entry | string | `default` |
| `String[]` | Multiple text entries | `{string, ...}` | `default = {}` |
| `Color` | Color picker | `Color(r,g,b)` | `default` |
| `Vector` | 3-axis input | `Vector(x,y,z)` | `default` |
| `Angle` | 3-axis input | `Angle(p,y,r)` | `default` |
| `Enum` | Dropdown | numeric | `enum = {NAME=1,...}`, `default` |
| `ComboBox` | Dropdown | string | `choice = {{value=...}, ...}`, `default` must equal a `value` |
| `Team` | Job picker | team keyname string | `default = "TEAM_NIL"` |
| `Teams` | Job checkboxes | `{KEYNAME=true,...}` | `default = {}` |
| `InputKey` | Key binder | key code number | `default = KEY_E` |
| `Label` | Static text | — | `name` only |
| `Button` | Button | — | `name`, `action = function(form) end` |

All field tables support an optional `desc` string for tooltip text.

---

## Accessing config values in the addon

Always read from `MYADDON_CONFIG`, never from `guthscp.configs` directly:

```lua
-- Anywhere in the addon (server or client):
if MYADDON_CONFIG.enabled then
    local dmg = MYADDON_CONFIG.damage
end
```

This works whether guthscpbase is installed or not, because:
- Without guthscpbase: `sh_config.lua` populates `MYADDON_CONFIG`
- With guthscpbase: the `guthscp.config:applied` hook keeps `MYADDON_CONFIG` in sync

---

## Load order

`sh_config.lua` (in `lua/autorun/`) loads before guthscpbase initializes its modules, so defaults are always in place before `guthscp.config:applied` fires.

---

## Opening the menu in-game

```
guthscpbase        (console command — opens config menu)
guthscp_menu       (alias)
```

Superadmin-only. Config changes are saved to `data/guthscp/configs/<id>.json` and broadcast to all connected clients.

---

## Hot-reload during development

Call `guthscp.module.hot_reload("<module_id>")` before `return MODULE`. Then save `main.lua` from your editor while in-game — guthscpbase reloads the entire module automatically without a server restart.

---

## What NOT to put in the guthscpbase config

- Net string names, hook IDs, file paths — these are structural, not behavioral
- Version constants
- Values that require a map restart or code reload to take effect (mention this to the user if any of the scanned values fall into this category)
- Passwords or secrets
