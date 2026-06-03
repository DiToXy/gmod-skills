---
name: cns-reborn-integration
description: How to send notifications using the CNS Reborn notification system in any Garry's Mod GLua addon. Use whenever an addon needs to show a toast/alert to players — confirmation of an action, a warning, an error, a broadcast announcement. Covers SendGlobalNotification, SendPlayerNotification, CNS.SendLocal, all four notification types, both positions, permission gating, and the admin GUI. Trigger on: CNS, cns_reborn, notify, notification, SendGlobalNotification, SendPlayerNotification, toast, alert popup, in-game notification, broadcast message to players.
---

# CNS Reborn — Notification System

CNS Reborn (`cns_reborn`) is the server-wide notification system. It sends stylised toast popups to players via a DHTML overlay. Notifications are one-way: the server pushes them, clients display them.

---

## Notification Types

| Type | Colour | Use for |
|------|--------|---------|
| `"success"` | Green | Positive confirmation — action completed, item granted, warning issued |
| `"error"` | Red | Failures, rejected actions, access denied |
| `"warning"` | Orange/Yellow | Cautions, player-facing warn notifications, risky operations |
| `"info"` | Blue/Neutral | General announcements, status updates, informational messages |

## Positions

| Position | Placement |
|----------|-----------|
| `"top"` | Top-centre of the screen |
| `"right"` | Right-side rail |

**Duration:** integer or float, between `1` and `60` seconds.

---

## Server-Side API

These functions are global and available anywhere server-side after CNS loads.

### Broadcast to all players

```lua
-- SendGlobalNotification(type, position, duration, title, message)
SendGlobalNotification("info",    "top",   8,  "Maintenance",   "The server will restart in 5 minutes.")
SendGlobalNotification("warning", "top",   10, "Code Rouge",    "Une alerte a été déclenchée.")
SendGlobalNotification("success", "right", 5,  "Mise à jour",   "Les salaires ont été versés.")
```

### Send to a single player

```lua
-- SendPlayerNotification(target, type, position, duration, title, message)
-- target must be a valid Player entity
SendPlayerNotification(ply, "success", "right", 5, "Action effectuée",  "Votre demande a été traitée.")
SendPlayerNotification(ply, "error",   "right", 6, "Accès refusé",      "Vous n'avez pas les permissions.")
SendPlayerNotification(ply, "warning", "right", 8, "Avertissement reçu", "Raison : " .. reason)
```

Both functions return `true` on success, or `false, errorMsg` on failure (invalid type, out-of-range duration, empty title/message).

---

## Client-Side API

### Show a notification only to the local player (no network call)

```lua
-- CNS.SendLocal(type, position, duration, title, message)
-- CLIENT only — never leaves the local machine.
CNS.SendLocal("info", "right", 4, "Chargement", "Interface prête.")
```

---

## Common Patterns

### Confirm an admin action (server-side)

```lua
net.Receive("MYADDON.DoAction", function(len, ply)
    if not IsValid(ply) then return end
    if not CNS_NOTIFY_CONFIG.HasPermission(ply) then
        SendPlayerNotification(ply, "error", "right", 5, "Accès refusé", "Vous n'avez pas les permissions.")
        return
    end

    -- ... do the action ...

    SendPlayerNotification(ply, "success", "right", 4, "Action effectuée", "Terminé.")
end)
```

### Notify both parties when an admin acts on a player

```lua
SendPlayerNotification(target, "warning", "right", 8, "Avertissement reçu",  "Raison : " .. reason)
SendPlayerNotification(admin,  "success", "right", 5, "Avertissement envoyé", target:Nick() .. " a été averti.")
```

### Broadcast a server event

```lua
hook.Add("SomeGameEvent", "CNS_EventNotify", function()
    SendGlobalNotification("info", "top", 10, "Événement", "Un événement spécial a commencé !")
end)
```

---

## Permission Check

The admin GUI and console commands are gated by SAM. The permission key is `"cns.send_notification"`, registered under the `"CNS Notifications"` category with default rank `"admin"`.

```lua
-- Check whether a player can use CNS admin features:
CNS_NOTIFY_CONFIG.HasPermission(ply)  -- returns bool
-- Internally: ply:HasPermission("cns.send_notification")
```

The raw API functions (`SendGlobalNotification`, `SendPlayerNotification`) have **no built-in permission gate** — they are internal API calls for server-side code. Always add your own gate in the calling `net.Receive` or hook if the action is player-triggered.

---

## Console Commands (staff use)

```
notify_send <type> <position> <duration> "<title>" "<message>"
    -- Broadcast to all players. Requires cns.send_notification.

notify_send_ply <player_name> <type> <position> <duration> "<title>" "<message>"
    -- Send to a single player by partial name match. Requires cns.send_notification.
```

Open the admin GUI in-game: `notify_gui` (requires the SAM permission).

---

## Security Rules

1. `SendGlobalNotification` and `SendPlayerNotification` are **server-only** — never expose them to client triggers without a server-side permission check first.
2. The client-side admin panel sends `CNSNotify.AdminRequest` to the server, which re-checks `CNS_NOTIFY_CONFIG.HasPermission` before acting. The client cannot bypass this.
3. Sanitise any player-supplied input before passing it as `title` or `message`.
