# KDE Plasma 6 — Per-Monitor Wallpaper

Reference for setting a wallpaper on one specific screen in a multi-monitor Plasma 6 setup. No personal paths/hostnames — fill in the placeholders.

**Shell note:** fish syntax differs from bash — use `for ... end` instead of `for ... do ... done`, and single-quote any string containing `$`.

---

## 1. Identify monitors

```bash
xrandr --listmonitors
```

Gives: connector names, x-offset, y-offset. Capture **both** x and y — x-only matching breaks on stacked/portrait layouts.

## 2. Identify monitor model (optional, to confirm which connector is which physical screen)

```bash
for f in /sys/class/drm/card*-<CONNECTOR>/edid
    strings "$f"
end
```

## 3. Find containment IDs

```bash
grep -B2 'Image=' ~/.config/plasma-org.kde.plasma.desktop-appletsrc
```

Gives: containment ID → current wallpaper path.

> Multiple containment IDs can map to the same physical screen across different activities. Disambiguate:

```bash
grep -B1 'activityId=' ~/.config/plasma-org.kde.plasma.desktop-appletsrc
qdbus6 org.kde.plasmashell /org/kde/ActivityManager/Activities org.kde.ActivityManager.Activities.CurrentActivity
```

Only the containment under the **current active activity** will visibly update.

## 4. Apply wallpaper to a specific screen

Method: Plasma scripting via D-Bus. `screenName` is unreliable on Plasma 6 — match by geometry offset instead.

- Binary name varies by distro: try `qdbus6`, then `qdbus`, then `qdbus-qt6`.
- Path has spaces/special chars: URL-encode before inserting into the `file://` URI, or symlink to a path without spaces.
- Prereq: desktop widgets must be unlocked, or `writeConfig` fails silently with `DBus.Error.Failed`.

```bash
qdbus6 org.kde.plasmashell /PlasmaShell org.kde.PlasmaShell.evaluateScript '
var d = desktops();
for (i=0;i<d.length;i++) {
    var g = screenGeometry(d[i].screen);
    if (g.x == <TARGET_X_OFFSET> && g.y == <TARGET_Y_OFFSET>) {
        d[i].wallpaperPlugin = "org.kde.image";
        d[i].currentConfigGroup = Array("Wallpaper","org.kde.image","General");
        d[i].writeConfig("Image", "file://<ABSOLUTE_IMAGE_PATH>");
    }
}
'
```

## 5. Fallback (single screen only)

```bash
plasma-apply-wallpaperimage <ABSOLUTE_IMAGE_PATH>
```

Limitation: applies to all screens or the current screen only — no per-screen targeting.

## 6. Verify it applied

```bash
grep -A2 'Containments\]\[<CONTAINMENT_ID>\]\[Wallpaper' ~/.config/plasma-org.kde.plasma.desktop-appletsrc
```

## 7. Persistence

Writes to `~/.config/plasma-org.kde.plasma.desktop-appletsrc`. Survives reboot, logout, and `plasmashell` restart.

Breaks if:
- Wallpaper plugin is `slideshow`, not `org.kde.image`
- Image file gets moved or deleted
