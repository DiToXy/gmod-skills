# load-order-management

A Claude Code skill for making Garry's Mod GLua addons load in the correct order — especially a shared library (UI/theme lib, framework) that must finish loading before the addons that depend on it. The addon always owns its own loader and never depends on a third-party framework to sequence its files.

## What this skill teaches

- Why GMod cross-addon load order is fragile (autorun is sorted by filename, not folder)
- A self-contained loader: one guarded entry file, a realm-aware `load` helper, an explicit dependency-first order, and a ready flag (`MYADDON_LOADED`)
- Your own `myaddon/modules/*.lua` extension folder — letting first-party sub-modules hook in *after* your core is ready
- `require()` for synchronous, exact-point library loading
- Ready-flag + ready-hook signalling for shared libraries (`MyLib.Ready` + `hook.Run("MyLib.Ready")`)
- The file-scope capture trap (`local Theme = Lib.Theme`) and how stable table identity fixes it
- `sv_`/`cl_`/`sh_` realm detection and clientside `AddCSLuaFile` delivery

## Patterns covered

| Situation | Pattern |
|---|---|
| Ordering files within one addon | Guarded loader + ordered list |
| A shared lib others depend on | Ready flag + ready hook |
| Letting first-party sub-modules hook into your addon | Your own `myaddon/modules/` folder |
| Your addon depends on a shared lib you own | `if READY then else hook.Add(ReadyHook)` |
| You control lib + loader | `require("lib")` synchronously before use |

## When it triggers

Automatically when you're working on GLua and hit:
- `attempt to index a nil value (global X)` at load
- a consumer addon running before its dependency
- load order / autorun order / `AddCSLuaFile` / shared library bootstrap
- ready hooks, `MyLib.Ready`, deterministic loaders
