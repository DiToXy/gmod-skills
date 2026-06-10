---
name: rebornui-development-guide
description: "How to build Garry's Mod GLua UI with the RebornUI shared VGUI library — themed Derma components, primitives, world-space 3D2D surfaces, and the theme/override system. Use whenever an addon should render a panel, menu, HUD, frame, button, list, tab, modal, toggle, slider, card, badge, progress bar, model panel, or an interactive world screen/tablet/terminal, and you want it to match the REBORN look instead of raw Derma. Covers mounting + the RebornUI.Ready load-order contract, theme tokens + SetTheme, the builder/styler conventions, RebornUI.NoBg (not SetPaintBackgroundEnabled), RebornUI.Scale/Sound/Settings, the full component catalog, and RebornUI.Surface (IMGUI-backed 3D2D). Trigger on: RebornUI, reborn_ui, RebornSM, themed VGUI, RebornUI.Frame/Button/Label/Surface, 3D2D world UI, world tablet/terminal, rebornui_open, white DPanel background, theme token, accent color."
---

# RebornUI Development Guide

`RebornUI` is the REBORN project's standalone clientside VGUI styling library: a
dark themed Derma component set, render primitives, a theme-token/override system,
and an IMGUI-backed world-space 3D2D surface system. Build your addon's UI with it
and everything matches the REBORN look and reskins with the live theme switch.

**The implemented library is the source of truth.** Run `rebornui_open` in console
to open the live gallery — every component rendered interactively, plus a theme
switch. When unsure of a signature, open the gallery or read `reborn_ui/README.md`.

**Golden rule:** never paint a raw Derma panel. Use the builders/stylers below, and
suppress unwanted backgrounds with `RebornUI.NoBg(panel)` — never
`SetPaintBackgroundEnabled`.

---

## Depending on RebornUI (load order)

The `reborn_ui` addon must be mounted. It owns its own loader and announces
readiness with a flag + hook. Because consumers often capture
`local Theme = RebornUI.Theme` at file scope, the library is named to load first;
still, gate any setup that calls `RebornUI.*` on readiness:

```lua
local function setup()
    -- safe to call RebornUI.* here
end

if RebornUI and RebornUI.Ready then
    setup()
else
    hook.Add("RebornUI.Ready", "myaddon.ui_setup", setup)
end
```

Never depend on a third-party addon to sequence RebornUI. See the
`load-order-management` skill for the full pattern.

---

## Conventions

- **Builders** return a constructed panel: `RebornUI.Frame(title, w, h)`,
  `RebornUI.Button(parent, text, danger, accent)`, etc. Parent-first.
- **Stylers** theme an existing widget in place and return it:
  `RebornUI.StyleEntry(dtextentry)`, `RebornUI.StyleCombo(dcombobox)`, etc.
- **`RebornUI.NoBg(panel)`** suppresses a panel's default skin background. A plain
  `DPanel` paints a light/white skin background that `SetPaintBackgroundEnabled`
  does NOT stop (it gates the engine flag, not DPanel's Lua-drawn `m_bBackground`).
  Use `NoBg` for any transparent container, or give the panel a custom `Paint`.
- **`RebornUI.Scale(px)` / `ScaleH(px)` / `ScaleMin(px)`** size against a 1920x1080
  reference. Size new UI through these.
- **Fonts:** `RebornUI_Title` (20, 600), `RebornUI_Body` (16, 400),
  `RebornUI_Small` (13, 500). Don't create your own unless it's addon-specific.
- **No icon font.** Components use drawn glyphs (arcs, chevrons, magnifier) or text.
  If you need icons, draw them or flag that a Glyphter font should be generated.
- **Caching discipline:** nothing allocates a `Color`/`Vector`/poly inside a paint
  hook. Cache at module scope and mutate `.r/.g/.b`, or use `RebornUI.LerpColor`.

---

## Theme

Tokens on `RebornUI.Theme`:

```
bg surface surface2 accent accentMut line text textMut danger dangerHov
success warning radius(8) btnRadius(4)
```

Derived (read-only, recomputed): `RebornUI.Derived.{accentHov, titleBar, closeHov, border}`.

```lua
-- Reskin the whole library live. Mutates tokens IN PLACE (stable table identity),
-- so open panels reskin instantly and captured references stay valid.
RebornUI.SetTheme({ accent = Color(60, 185, 120), accentMut = Color(60, 185, 120, 40) })
```

Read tokens at paint time (`Theme.accent.r` inside `Paint`) so the theme switch
reskins your panel too. Do not copy a token into a one-off `Color` at build time.

---

## Component catalog (one line each)

### Structural / core
| Call | Result |
|------|--------|
| `RebornUI.Frame(title, w, h)` | draggable window with titlebar + close |
| `RebornUI.Button(parent, text, danger, accent)` | themed button (SetEnabled(false) = disabled look) |
| `RebornUI.Label(parent, text, font)` | DLabel |
| `RebornUI.Row(parent, labelText)` | docked labelled row; dock your widget FILL into it |
| `RebornUI.Menu()` | themed right-click menu (`:AddOption`, `:AddSubMenu`, `:Open`) |

### Stylers (theme an existing widget)
`StyleEntry(dtextentry)`, `StyleCombo(dcombobox)`, `StyleListView(dlistview)`,
`StyleCheckbox(dcheckboxlabel)`, `StyleSlider(dnumslider)`, `StyleScrollBar(dvscrollbar)`.

### Form controls
`Toggle(parent, state, onChange)`, `Slider(parent, min, max, decimals, onChange)`,
`ScrollPanel(parent)` (themed scrollbar), `SearchBox(parent, placeholder, onChange)`,
`Tooltip(panel, text)`, `Confirm(title, msg, onOk, onCancel)`, `Alert(title, msg, onOk)`,
`Prompt(title, msg, default, onSubmit, onCancel)`.

### Form primitives
`Segmented(parent, options, onSelect)`, `Stepper(parent, min, max, step, onChange)`,
`TextArea(parent)`, `RangeSlider(parent, min, max, decimals, onChange)`,
`KeybindField(parent, key, onBind)`, `Wizard(parent)` (`:AddStep(title, panel)`).

### Containers & navigation
`Card(parent, title)` (content in `card.Body`), `Badge(parent, text, variant)`
(default/accent/success/warning/danger), `NavRail(parent, width)` (`:AddItem`, `.onSelect`),
`Tabs(parent)` (`:AddTab(label, panel)`), `Accordion(parent)` (`:AddSection`),
`Table(parent, columns)` (styled DListView).

### Big data / nav
`VirtualList(parent, rowHeight, buildRow, updateRow)` (`:SetData`, pooled rows for
thousands), `Pagination(parent, total, onPage)`, `Breadcrumbs(parent, items, onClick)`,
`Drawer(side, size)` (edge slide-out, `:Close()`), `Popover(target, build, w, h)`.

### State & feedback
`Spinner(parent, size)`, `IndeterminateBar(parent)`, `Skeleton(parent, lines)`,
`EmptyState(parent, title, msg, ctaText, ctaFn)`,
`ValidatedField(parent, placeholder, validateFn)` (validateFn(text) -> ok, message).

### Rich / game
`ModelPanel(parent, model)`, `Avatar(parent, steamID64orPlayer, size)`,
`ProgressBar(parent)` (`:SetFraction`), `RadialGauge(parent, size)` (`:SetFraction`),
`StatTile(parent, label, value, accent)`, `Slot(parent, size)`,
`SlotGrid(parent, cols, rows, slotSize, gap)` (`:GetSlot(i)`).

### Text effects
`Typewriter(parent, text, cps, font)`, `GlitchText(parent, text, dur, font)`,
`DrawVGradient(x, y, w, h, top, bottom)`, `DrawVignette(x, y, w, h, strength)`.

### Experiential (world / HUD; each handle has `:Remove`)
`World3D2D(pos, ang, scale, w, h, drawFn)`, `Keypad(opts)`, `RadialMenu(options, onSelect)`,
`Banner(opts)`, `Hotbar(count)`, `InteractionPrompt(text, key)`, `Waypoint(pos, label)`.

### Primitives (use inside paint hooks)
`DrawRoundedPoly`, `StartMask`/`DrawInsideMask`/`EndMask`, `DrawCircle`, `DrawArc`,
`BlurBehind`, `DrawShadow`, `DrawGlow`, `Ease`, `AnimateHover`, `LerpColor`.

### Systems
`RebornUI.Sound.Play("hover"|"click"|"open"|"close"|"error")` / `SetHandler(fn)` /
`SetEnabled(b)`; `RebornUI.Settings.Get(ns, key, default)` / `Set(ns, key, value)`;
`RebornUI.Inspector.Toggle()`.

---

## Example: a panel

```lua
local function openPanel()
    local frame = RebornUI.Frame("My Addon", 360, 220)

    local content = vgui.Create("DPanel", frame)
    content:Dock(FILL)
    content:DockMargin(8, 8, 8, 8)
    RebornUI.NoBg(content)               -- transparent: do NOT use SetPaintBackgroundEnabled

    local nameRow = RebornUI.Row(content, "Name")
    local entry = vgui.Create("DTextEntry", nameRow)
    entry:Dock(FILL)
    RebornUI.StyleEntry(entry)

    local save = RebornUI.Button(content, "Save", false, true)  -- accent
    save:Dock(BOTTOM)
    save:SetTall(30)
    save.DoClick = function()
        RebornUI.Sound.Play("click")
        net.Start("myaddon_save") net.WriteString(entry:GetValue()) net.SendToServer()
    end
end
```

---

## World surfaces (interactive 3D2D)

`RebornUI.Surface` renders UI on entities/props/world positions, built on the
bundled IMGUI lib (correct cursor projection, culling, obstruction). Interaction is
IMGUI's model: look at the surface, press `+use` / `+attack` to click.

```lua
local s = RebornUI.Surface.Attach(ent, {
    offset = Vector(0, 0, 0),   -- local top-left corner on the model
    angle  = Angle(90, 0, 0),   -- IMPORTANT: angles:Up() must be the OUTWARD normal
    scale  = 0.05, width = 300, height = 400,
}, function(surface)
    surface:Label{ x = 16, y = 12, text = "TERMINAL", font = "RebornUI_Title" }
    surface:Tabs{ x = 16, y = 46, w = 268, h = 28, tabs = { "Status", "Doors" },
        onChange = function(i) end }
    surface:List{ x = 16, y = 86, w = 268, h = 200, items = myRows,
        onSelect = function(i, item) end }
    surface:Button{ x = 16, y = 350, w = 268, h = 34, text = "Lock", accent = true,
        onPress = function() net.Start("myaddon_lock") net.SendToServer() end }
end)
-- s:SetData(tbl) to rebuild; s:Remove() on teardown (and remove the entity if you spawned it).
```

Rules for surfaces:
- **`angles:Up()` is the outward normal.** IMGUI culls surfaces facing away. Orient
  the surface so `Up()` points at viewers, then tune offset/angle/scale empirically.
- Use `Surface.Attach(ent, ...)` for entity-mounted screens (IMGUI ignores that prop
  in its obstruction trace). `Surface.World(pos, ang, ...)` for map-fixed.
- Widgets: `surface:Label/Button/Toggle/Tabs/List{ x, y, w, h, ... }`. Interactions
  fire the widget callback and `surface.onPress = function(id, w) end`.
- The library is standalone: interactions are callbacks; wire server effects through
  your own net messages.

---

## Anti-patterns

- `SetPaintBackgroundEnabled(false)` on a `DPanel` to make it transparent — it does
  not stop the skin background. Use `RebornUI.NoBg(panel)`.
- Capturing a theme token into a one-off `Color` at build time — breaks the live
  reskin. Read `Theme.accent.r` etc. inside `Paint`.
- Replacing `RebornUI.Theme` wholesale — mutate via `SetTheme` so identity is stable.
- Allocating `Color()`/`Vector()`/poly tables inside a paint hook.
- Hardcoding a 3D2D surface transform without verifying against the model — tune the
  offset/angle/scale in-game (the on-surface cursor is the alignment aid).
- Calling `RebornUI.*` before readiness — gate on `RebornUI.Ready` / the ready hook.
