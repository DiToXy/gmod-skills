# rebornui-development-guide

A Claude Code skill for building Garry's Mod GLua UI with the **RebornUI** shared
VGUI library — themed Derma components, render primitives, the theme/override
system, and IMGUI-backed world-space 3D2D surfaces.

## What this skill teaches

- Depending on RebornUI: mounting + the `RebornUI.Ready` load-order contract
- Conventions: builders vs stylers, `RebornUI.NoBg` (not `SetPaintBackgroundEnabled`),
  `RebornUI.Scale`, fonts, caching discipline
- Theme tokens + `RebornUI.SetTheme` and how the live theme switch works
- The full component catalog (structural, form, containers, big-data, feedback,
  rich, text fx, experiential, primitives, systems)
- `RebornUI.Surface` — interactive world 3D2D screens (tablets/terminals/wall panels),
  the IMGUI interaction model, and the `angles:Up()` outward-normal rule
- Anti-patterns (white DPanel backgrounds, breaking the live reskin, paint-hook allocs)

## Live reference

Run `rebornui_open` in console to open the gallery: every component rendered live
and interactive, plus a theme switch and the world-surface showcases.

## When it triggers

Automatically when you're building GLua UI and mention RebornUI / reborn_ui /
RebornSM, themed VGUI, a 3D2D world screen/tablet, `RebornUI.Frame`/`Button`/`Surface`,
`rebornui_open`, a white DPanel background, or theme tokens / accent color.
