# sam-addon-integration

A Claude Code skill for integrating SAM (admin mod v162) permissions and commands into Garry's Mod GLua addons.

## What this skill teaches

- `sam.permissions.add()` — register named permissions that appear in SAM's in-game UI
- `ply:HasPermission()`, `ply:CheckGroup()`, `ply:IsAdmin()` — the correct rank check methods
- Rank inheritance — why `CheckGroup("admin")` beats `GetUserGroup() == "admin"`
- SAM command builder — `sam.command.new():SetPermission():OnExecute():End()`
- Server-side security patterns (always re-check on server, never trust client)

## Install

```
/plugin install DiToX/sam-addon-integration
```

## When it triggers

Automatically activates when you're working on any GLua file that involves:
- SAM rank/permission checks
- Admin-gated net messages or console commands
- `HasPermission`, `GetUserGroup`, `CheckGroup`, `sam.permissions`, `sam.command`
