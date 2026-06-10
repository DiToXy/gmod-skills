---
name: load-order-management
description: "How to make Garry's Mod GLua addons load in the correct order — especially a shared library (theme/UI lib, framework) that must finish loading before the addons that depend on it. Use whenever an addon throws 'attempt to index a nil value (global X)' at load, a consumer runs before its dependency, or you need a deterministic loader. Covers GMod autorun ordering, a self-contained loader pattern, sv_/cl_/sh_ realm detection, ready-flag + ready-hook signalling, your own modules/extension folder, require() for exact-point loading, and file-scope capture pitfalls. The addon always owns its own loader and never depends on a third-party admin/framework for load order. Trigger on: load order, loads before, dependency not loaded, nil global at load, autorun order, AddCSLuaFile, shared library, ready hook, MyLib.Ready, lib loads after consumer, deterministic loader."
---

# Load-Order Management (GLua)

Garry's Mod gives you **no guaranteed cross-addon load order**. When addon B depends on a global from library A, and B's code runs before A finished loading, you get `attempt to index a nil value (global 'A')`. This skill shows how to make load order deterministic with a **self-contained loader the addon owns** — no reliance on any third-party admin mod or framework to sequence your files.

**The golden rule:** never rely on alphabetical autorun filenames to order a dependency, and never depend on another addon's loader. Make the dependency *announce* it is ready, or have your own single loader include everything in an explicit sequence.

---

## Why cross-addon order is fragile

GMod runs autorun by calling `file.Find("autorun/*.lua", "LUA")` (and `autorun/client/`, `autorun/server/`) across the merged virtual filesystem of every mounted addon, then `include`s the results **sorted by filename** — not by addon folder, not by mount order.

Consequences:
- `cl_mylib.lua` loads before `cl_consumer.lua` only because `l` < `o`. Add a consumer named `cl_alpha.lua` and it now loads **before** the library and crashes.
- Workshop vs legacy addons mount differently, so the order you see locally may differ on a live server.
- Renaming a file silently changes load order.

So filename ordering "works" until it doesn't. Treat it as a fallback, never a guarantee.

---

## The loader pattern (recommended)

A robust addon owns a single loader and announces when it's ready. Four ideas do all the work.

### 1. One loader, guarded, with a ready flag

```lua
-- lua/autorun/myaddon_load.lua  (top)
if MYADDON_LOADED then return end   -- idempotent: never double-load

MyAddon = MyAddon or {}
MyAddon.Version = 1

-- ... load everything in order ...

MYADDON_LOADED = true               -- (bottom) the public "I am ready" flag
hook.Run("MyAddon.Ready")           -- the public ready signal for consumers
```

A single entry file owns the whole sequence. `MYADDON_LOADED` is the global consumers check; `MyAddon.Ready` (a hook) is what they wait on.

### 2. A realm-aware load helper

Don't scatter raw `include` / `AddCSLuaFile` calls. Route every file through one helper that picks the realm from the **filename prefix** (`sv_`, `cl_`, `sh_`):

```lua
local types = {
    sv_ = SERVER and include or function() end,          -- server-only: run on server, ignore on client
    cl_ = SERVER and AddCSLuaFile or include,            -- client file: ship from server, run on client
    sh_ = function(name)                                  -- shared: ship + run on both
        if SERVER then AddCSLuaFile(name) end
        return include(name)
    end,
}

local function load(name, realm)
    -- explicit `realm` wins; else detect from first 3 chars of the filename; else default shared
    local func = types[realm and realm.."_"]
              or types[name:GetFileFromFilename():sub(1, 3)]
              or types.sh_
    return func(name)
end
```

This is the single most copyable pattern in this skill: **name your files `sv_`/`cl_`/`sh_` and a one-line call handles realm + clientside shipping correctly.**

### 3. An explicit ordered list — dependencies first

```lua
-- order is hand-written, dependency-first; NOT alphabetical
load("myaddon/libs/sh_util.lua")
load("myaddon/sh_config.lua")
load("myaddon/sh_permissions.lua")
load("myaddon/core/sh_core.lua")      -- needs config + util above
load("myaddon/sv_main.lua")
load("myaddon/cl_ui.lua")             -- UI last
```

Inside one addon you control the order absolutely by writing the list. Order by dependency, not by name.

### 4. Your own extension folder, loaded *after* core

If you want a plugin/module system inside your own addon, scan a folder and load it **after** your core is ready:

```lua
for _, module in ipairs(file.Find("myaddon/modules/*.lua", "LUA")) do
    load("myaddon/modules/" .. module)   -- runs after util/config/core exist
end
```

Anything dropped in `myaddon/modules/` is now guaranteed to load with your whole API available. This is how *your* addon lets sub-modules or first-party add-ons hook in — owned entirely by you, not by a third party.

### Pulling in a library at an exact point with `require`

If your loader needs a library available at a precise line (e.g. just before UI code), load it synchronously with `require`:

```lua
require("mylib")   -- synchronously runs lua/includes/modules/mylib.lua right now
```

`require("name")` runs `lua/includes/modules/<name>.lua` immediately. Because it's synchronous and placed *before* the code that needs it, the library is guaranteed ready — no hooks, no ordering guess. If you control both the library and the loader, this is the strongest guarantee of all.

### Ready signals your library should expose

| Signal | Set where | Meaning |
|---|---|---|
| `MYADDON_LOADED` (global bool) | end of the loader | core fully loaded |
| `hook.Run("MyAddon.Ready")` | end of the loader | the hook consumers wait on |
| `MyAddon.Version` (number) | top of the loader | version gate for consumers |

Optionally fire finer-grained hooks (`MyAddon.ConfigLoaded`, etc.) if consumers need to integrate at a specific stage.

---

## Patterns to apply

Pick by your situation.

### Pattern A — Deterministic loader for your own addon

Use when your addon has more than ~3 files. One guarded entry file, the realm helper, an explicit ordered list. You then never worry about intra-addon order again.

```lua
-- lua/autorun/myaddon_load.lua
if MYADDON_LOADED then return end
MyAddon = MyAddon or {}

local types = {
    sv_ = SERVER and include or function() end,
    cl_ = SERVER and AddCSLuaFile or include,
    sh_ = function(n) if SERVER then AddCSLuaFile(n) end return include(n) end,
}
local function load(name)
    return (types[name:GetFileFromFilename():sub(1,3)] or types.sh_)(name)
end

load("myaddon/sh_config.lua")     -- shared first
load("myaddon/sv_core.lua")       -- server logic
load("myaddon/cl_ui.lua")         -- client UI last

MYADDON_LOADED = true
hook.Run("MyAddon.Ready")
```

### Pattern B — Shared library that must load before its consumers

This is the shared-theme/UI-lib case. Two layers, belt and suspenders:

**Library side** — announce readiness:
```lua
-- end of the lib's bootstrap
MyLib.Ready = true
hook.Run("MyLib.Ready")
```
Optionally also name the lib's autorun to sort early (`aa_mylib.lua`) so it usually wins the filename race — but never *rely* on that alone.

**Consumer side** — run setup now if ready, else wait for the hook:
```lua
local function setup()
    -- safe to call MyLib.* here
end

if MyLib and MyLib.Ready then
    setup()
else
    hook.Add("MyLib.Ready", "myaddon.setup", setup)
end
```

> **File-scope capture trap.** If your other files do `local Theme = MyLib.Theme` at the top of the file (file scope), deferring `setup()` to a hook is too late — those captures already ran and grabbed `nil`. Two fixes:
> 1. **Guarantee order** so the lib is loaded before any consumer file runs (own loader, `require()`, or sort-first naming), OR
> 2. **Don't capture at file scope** — reference `MyLib.Theme.accent` through the table inside functions/paint hooks, so the lookup happens at call time (after the lib loaded). The library keeping a *stable table identity* (mutating the existing `Theme` table on retheme rather than replacing it) is what makes late references safe.

### Pattern C — Your own modules / extension folder

When you want first-party sub-modules to hook in after your core is ready, have *your* loader scan a folder you own:

```
myaddon/lua/myaddon/modules/sh_feature_a.lua   -- your loader load()s this after core
myaddon/lua/myaddon/modules/sv_feature_b.lua
```

Inside those files your whole API is already available, and realm follows the filename prefix. The folder belongs to your addon — you never drop files into another addon's tree or depend on its loader to sequence you.

### Pattern D — Defer to a dependency's ready signal

When your addon depends on a shared library you (or a teammate) own, wait for its ready signal instead of guessing order:

```lua
local function integrate()
    -- safe: MyLib.* is fully loaded here
    MyLib.Register("myaddon", { ... })
end

if MyLib and MyLib.Ready then
    integrate()
else
    hook.Add("MyLib.Ready", "myaddon.integration", integrate)
end
```

The `if ready then / else hook` shape is the universal safe form — it handles both "dependency already loaded" and "dependency loads later" with one block.

---

## Which pattern when

| Situation | Pattern |
|---|---|
| Ordering files *within* one addon | A — one guarded loader + ordered list |
| A shared lib others depend on (UI/theme/framework) | B — ready flag + ready hook (+ stable table identity) |
| Letting first-party sub-modules hook into your addon | C — your own `myaddon/modules/` folder |
| Your addon depends on a shared lib you own | D — `if READY then else hook.Add(ReadyHook)` |
| You control both lib and loader, lib needed at an exact point | `require("lib")` synchronously, right before use |

---

## Anti-patterns

- **Relying on alphabetical autorun filenames** to order a cross-addon dependency. Coincidence, not a contract.
- **Depending on a third-party admin mod or framework to load your files.** Own your loader; you can't control or predict theirs.
- **`timer.Simple(0, ...)` or `timer.Create` retry loops** to "wait for" a dependency. Fragile, racy, and still runs after file-scope captures. Use a ready hook.
- **`pcall` spin / "try again next frame"** polling for a global. A ready hook fires exactly once, at the right time.
- **Assuming `InitPostEntity` is late enough.** It's fine for runtime calls, but it runs long after file-scope `local X = Lib.Thing` captures — those already failed.
- **Replacing a shared table on retheme** (`Lib.Theme = {...}`). Any consumer that captured the old table keeps the stale one. Mutate fields in place to preserve identity.

---

## Realm cheat-sheet

| Prefix | Server does | Client does |
|---|---|---|
| `sv_` | `include` (runs) | nothing |
| `cl_` | `AddCSLuaFile` (ships to client) | `include` (runs) |
| `sh_` | `AddCSLuaFile` + `include` | `include` |

A `cl_` file **must** be `AddCSLuaFile`'d on the server or clients never receive it. The `load` helper above does this automatically from the prefix — which is the whole reason to route through one helper instead of scattering raw `include`/`AddCSLuaFile` calls.

---

## Verifying load order

1. **Force the bad case:** temporarily rename a consumer's autorun so it sorts *before* the library (e.g. `aaa_consumer.lua`). If it still loads clean, your ordering is real and not luck.
2. **Check the console** at server start and on first client join for `attempt to index a nil value (global '<Lib>')` — that's a load-order failure, not a logic bug.
3. **Confirm clientside delivery:** a `cl_` file that wasn't `AddCSLuaFile`'d throws "Couldn't include file" only on clients, not the listen-server host — test with a real client or `sv_allowcslua 0`.
