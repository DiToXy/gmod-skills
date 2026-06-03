---
name: sam-addon-integration
description: "How to integrate SAM admin permissions and commands into a Garry's Mod GLua addon. Use whenever adding rank gates, permission checks, or admin commands to any addon — new or existing. Covers sam.permissions.add(), ply:HasPermission(), ply:CheckGroup(), rank inheritance, SAM command registration, and server-side security patterns. Trigger on: SAM, admin rank, permission check, HasPermission, GetUserGroup, CheckGroup, rank gate, or any sv_/sh_ file restricting access to staff."
---

# SAM Addon Integration

SAM (version 162) is the admin mod on this server. It overrides Garry's Mod's default usergroup system so that `ply:GetUserGroup()` always returns the SAM rank string rather than the default GMod group.

---

## How SAM Ranks Work

### Built-in ranks
- `"superadmin"` — always has every permission (hardcoded bypass in all checks)
- `"admin"` — base staff rank
- `"user"` — default for regular players

### Custom ranks
Servers can define custom ranks (e.g. `"moderator"`, `"operator"`) that **inherit** from one of the built-in ranks. A custom rank inherits all permissions from its parent unless explicitly overridden. This chain can be multiple levels deep.

### Immunity levels
```lua
SAM_IMMUNITY_SUPERADMIN = 100
SAM_IMMUNITY_ADMIN      = 50
SAM_IMMUNITY_USER       = 1
```
Custom ranks have immunity values set in the SAM config panel.

---

## Player Permission Methods (added by SAM)

These are available server-side and client-side after SAM loads:

```lua
ply:GetUserGroup()          -- raw rank string: "superadmin", "admin", "moderator", "user", etc.
ply:IsAdmin()               -- true if rank inherits from "admin" (includes superadmin)
ply:IsSuperAdmin()          -- true if rank is "superadmin"
ply:CheckGroup("rank")      -- true if rank inherits from the given rank name (handles custom ranks)
ply:HasPermission("key")    -- true if the rank has the named SAM permission granted
ply:CanTarget(ply2)         -- true if this player's immunity >= ply2's immunity
ply:CanTargetRank("rank")   -- true if this player's immunity >= the named rank's immunity
```

**Key rule:** always use `ply:CheckGroup("admin")` instead of `ply:GetUserGroup() == "admin"` when you want "at least admin level". A custom `"moderator"` inheriting from `"admin"` passes `CheckGroup` but fails a direct string comparison.

---

## Approach 1 — Native SAM Permission (recommended)

Register a named permission with SAM so it appears in SAM's in-game permissions UI and can be granted/revoked per rank by admins without touching code.

### Step 1 — Register the permission (shared file, e.g. `sh_config.lua`)

```lua
-- Only register if SAM is loaded; the permission persists in SAM's DB
if sam then
    -- sam.permissions.add(key, category_label, default_rank)
    -- default_rank is the rank that gets the permission by default on first load
    sam.permissions.add("myaddon.use_feature", "MyAddon", "admin")
    sam.permissions.add("myaddon.manage",       "MyAddon", "superadmin")
end
```

The `category_label` groups permissions in the SAM UI (e.g. `"MyAddon"`).
`default_rank` is only applied the first time — after that SAM's DB controls it.

### Step 2 — Check the permission

**Server-side** (inside a `net.Receive` or concommand):
```lua
net.Receive("myaddon.do_thing", function(len, ply)
    if not IsValid(ply) then return end
    if not ply:HasPermission("myaddon.use_feature") then
        return
    end
    -- proceed
end)
```

**Client-side** (UI gating only — always re-check on the server):
```lua
if LocalPlayer():HasPermission("myaddon.use_feature") then
    -- show the button / open the panel
end
```

---

## Approach 2 — SAM Command Registration

Use the SAM command builder to register a chat/console command that appears in SAM's command list, respects the permission system, and gets tab-completion.

```lua
-- Typically in a sv_ file, inside an `if SERVER then` block or sv_ autorun
sam.command.new("mycommand")
    :Help("Does something useful")
    :SetPermission("myaddon.use_feature", "admin")   -- (perm_key, default_rank)
    :Aliases("mycmd")                                 -- optional
    :AddArg("player", { hint = "target", optional = false })
    :AddArg("text",   { hint = "reason", optional = true })
    :OnExecute(function(ply, args)
        local target = args[1]
        local reason = args[2] or "no reason"
        sam.player.send_message(ply, "Done: " .. target:Name())
    end)
    :End()
```

**Notes:**
- `:SetPermission(key, default_rank)` registers the permission AND gates the command. SAM enforces the check before `OnExecute` fires.
- `:AddArg("player", {...})` — SAM resolves the target player for you; `args[1]` is already a valid `Player` entity.
- Common arg types: `"player"`, `"text"`, `"number"`, `"rank"`, `"steamid"`.
- If you need to check this permission elsewhere too, register it separately with `sam.permissions.add()` in a shared file.

---

## Rank Inheritance in Practice

If a server has a `"moderator"` rank that inherits from `"admin"`:

| Check | moderator | admin | superadmin |
|---|---|---|---|
| `ply:GetUserGroup() == "admin"` | false | true | false |
| `ply:CheckGroup("admin")` | true | true | true |
| `ply:IsAdmin()` | true | true | true |
| `ply:HasPermission("myaddon.use_feature")` | depends on SAM config | true (if default) | always true |

Always prefer `CheckGroup` or `HasPermission` over direct string equality unless you need to match one exact rank.

---

## File Organisation Pattern

```
myaddon/lua/
├── autorun/
│   └── sh_config.lua      -- SHARED: sam.permissions.add() calls + MYADDON_CONFIG table
├── autorun/server/
│   └── sv_main.lua        -- SERVER: net.Receive, concommands, command registration
└── autorun/client/
    └── cl_ui.lua          -- CLIENT: UI gating (cosmetic only — server is authority)
```

In `sh_config.lua`, guard against double-load and missing SAM:
```lua
MYADDON_CONFIG = MYADDON_CONFIG or {}

if sam then
    sam.permissions.add("myaddon.use_feature", "MyAddon", "admin")
end

function MYADDON_CONFIG.HasPermission(ply)
    if not IsValid(ply) then return false end
    return ply:HasPermission("myaddon.use_feature")
end
```

---

## Security Rules

1. **Always re-check on the server.** Client-side checks are UI-only. Every `net.Receive` and `concommand.Add` must check permissions before acting.
2. **Never trust the client.** A player can send any net message regardless of their UI state.
3. **Use `IsValid(ply)` before any rank check.** The console entity is not a valid player and will crash `GetUserGroup`.
4. **`superadmin` always passes.** SAM's `has_permission` short-circuits for superadmin — no need to special-case it.
5. **Register permissions in a shared file.** `sam.permissions.add()` must run on both realms so `ply:HasPermission()` works client-side too.
