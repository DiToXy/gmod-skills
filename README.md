# gmod-skills

A [Claude Code](https://claude.ai/code) plugin containing skills for **Garry's Mod GLua addon development**. Covers SAM admin integration and the guthscpbase in-game config system.

## Install

```
/plugin marketplace add DiToXy/gmod-skills
/plugin install sam-addon-integration
```

> Requires Claude Code with the plugin system enabled.

---

## Skills

### `sam-addon-integration`

Teaches Claude how to gate addon features behind **SAM admin ranks and permissions**.

**Covers:**
- `sam.permissions.add()` — register named permissions visible in SAM's in-game UI
- `ply:HasPermission()`, `ply:CheckGroup()`, `ply:IsAdmin()` — correct rank check methods
- Rank inheritance — why `CheckGroup("admin")` beats `GetUserGroup() == "admin"`
- SAM command builder — `sam.command.new():SetPermission():OnExecute():End()`
- Server-side security rules (always re-check on server, never trust client)

**Triggers on:** SAM, admin rank, permission check, HasPermission, GetUserGroup, CheckGroup, rank gate, sv_/sh_ files restricting access to staff roles.

---

### `guthscpbase-integration`

Teaches Claude how to make an addon **configurable in-game via the guthscpbase VGUI config panel**, with a standalone `config.lua` fallback.

**Covers:**
- Scanning an addon to propose what to make configurable vs. what to keep hardcoded
- `guthscp.config.add()` — registering config blocks with the framework
- All config types: Bool, Number, String, Color, Teams, Enum, InputKey, etc.
- Mirroring guthscpbase values into the addon's own config table (so nothing else needs to know about guthscp)
- Fallback `sh_config.lua` that works without guthscpbase installed

**Triggers on:** guthscpbase, ingame config, config menu, VGUI config, make values configurable ingame, DForm config.

> Requires the [guthscpbase](https://github.com/guthscpbase) addon installed on the server.

---

## Updating

```
/plugin update sam-addon-integration
/reload-plugins
```

## Stack

- Language: Lua (GLua — Garry's Mod API)
- Admin system: SAM v162
- Config framework: guthscpbase
