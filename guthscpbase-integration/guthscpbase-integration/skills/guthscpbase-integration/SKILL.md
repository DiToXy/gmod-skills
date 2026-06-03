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
2. Create the guthscpbase integration file

---

## File Layout

```
myaddon/lua/autorun/
├── sh_config.lua          ← fallback config (always present)
└── sh_guthscp_config.lua  ← guthscpbase integration (guarded by if guthscp)
```

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
MYADDON_CONFIG.allowed_jobs = { ["TEAM_GUARD"] = true }
```

Keep it simple — one value per line, no functions. Admins should be able to edit this without knowing Lua.

---

## guthscpbase integration file

```lua
-- sh_guthscp_config.lua

-- If guthscpbase isn't installed, silently skip.
-- The fallback sh_config.lua handles the defaults.
if not guthscp then return end

local ID = "myaddon"   -- unique ID for this addon's config block

-- ── Shared registration (both realms call this) ─────────────────
local FORM = {
    { type = "Category", name = "General" },
    {
        type    = "Bool",
        id      = "enabled",
        name    = "Enable Addon",
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
    { type = "Category", name = "Appearance" },
    {
        type    = "Color",
        id      = "color",
        name    = "UI Color",
        default = Color(255, 100, 50),
    },
    { type = "Category", name = "Access" },
    {
        type    = "Teams",
        id      = "allowed_jobs",
        name    = "Allowed Jobs",
        default = {},
    },
}

-- ── Server registration ──────────────────────────────────────────
if SERVER then
    guthscp.config.add(ID, {
        form    = FORM,
        receive = function(config)
            -- Called when an admin saves the config in-game.
            -- Mirror the values back into MYADDON_CONFIG so the
            -- rest of the addon doesn't need to know about guthscp.
            for k, v in pairs(config) do
                MYADDON_CONFIG[k] = v
            end
        end,
        parse = function(config)
            -- Optional: post-process or validate after load/apply.
        end,
    })

    -- Also sync to MYADDON_CONFIG on initial load / config:applied
    hook.Add("guthscp.config:applied", "myaddon.sync_config", function(id, config)
        if id ~= ID then return end
        for k, v in pairs(config) do
            MYADDON_CONFIG[k] = v
        end
    end)
end

-- ── Client registration ──────────────────────────────────────────
if CLIENT then
    guthscp.config.add(ID, {
        form = FORM,
    })

    hook.Add("guthscp.config:applied", "myaddon.sync_config", function(id, config)
        if id ~= ID then return end
        for k, v in pairs(config) do
            MYADDON_CONFIG[k] = v
        end
    end)
end
```

The `receive` + `hook.Add("guthscp.config:applied")` pattern keeps `MYADDON_CONFIG` as the single source of truth everywhere else in the addon — no code outside this file needs to know about `guthscp.configs`.

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
| `ComboBox` | Dropdown | string/data | `choice = {{label=..., data=...}}` |
| `Team` | Job picker | team keyname string | `default = "TEAM_NIL"` |
| `Teams` | Job checkboxes | `{KEYNAME=true,...}` | `default = {}` |
| `InputKey` | Key binder | key code number | `default = KEY_E` |
| `Category` | Section header | — | `name` only |
| `Label` | Static text | — | `name` only |
| `Button` | Button | — | `name`, `action = function(form) end` |

---

## Accessing config values in the addon

Always read from `MYADDON_CONFIG`, never from `guthscp.configs` directly:

```lua
-- Anywhere in the addon (server or client):
if MYADDON_CONFIG.enabled then
    local dmg = MYADDON_CONFIG.damage
    -- ...
end
```

This works whether guthscpbase is installed or not, because:
- Without guthscpbase: `sh_config.lua` populates `MYADDON_CONFIG`
- With guthscpbase: the `receive`/`applied` hook keeps `MYADDON_CONFIG` in sync

---

## Load order note

`sh_config.lua` must load before `sh_guthscp_config.lua`. Since GMod autoloads `autorun/` files alphabetically, naming them with `sh_` prefix ensures this. If the addon uses a manual loader, include `sh_config.lua` first.

---

## Opening the menu in-game

```
guthscpbase        (console command — opens config menu)
guthscp_menu       (alias)
```

Superadmin-only. Config changes are saved automatically to `data/guthscp/configs/<id>.json` and broadcast to all connected clients.

---

## What NOT to put in the guthscpbase config

- Net string names, hook IDs, file paths — these are structural, not behavioral
- Version constants
- Values that require a map restart or code reload to take effect (mention this to the user if any of the scanned values fall into this category)
- Passwords or secrets
