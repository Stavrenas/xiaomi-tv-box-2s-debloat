# Xiaomi TV Box 2S debloat

Notes from cleaning up a **Xiaomi TV Box 2S** (`MiTV-AFKR0`, Android TV / Google TV) over the local network with ADB.

Device discovered on LAN as **Xiaomi TV Box** (Cast / Android TV Remote / ADB). Example IP used during this session: `192.168.2.4` (yours may differ).

## Prerequisites

- Same LAN as the TV box
- On the box: **Developer options** → **USB debugging** + **Network debugging** (or equivalent)
- Accept the RSA fingerprint prompt on the TV the first time you connect
- Host tools: `adb` (Android platform-tools)

```bash
adb connect <TV_IP>:5555
adb devices   # should show <TV_IP>:5555  device
```

Replace `<TV_IP>` below, or set:

```bash
export SERIAL=<TV_IP>:5555
```

---

## What we did

### 1. Find the box on the network

- Ping-sweep / ARP on the LAN
- Confirm via mDNS:
  - `_androidtvremote2._tcp` → **Xiaomi TV Box**
  - `_googlecast._tcp` → **MiTV-AFKR0-…**
- Useful ports when online: `5555` (ADB), `6466`/`6467` (Android TV Remote), `8008`/`8009` (Cast)

### 2. Inventory apps

```bash
adb -s $SERIAL shell pm list packages -3
adb -s $SERIAL shell cmd package query-activities --brief \
  -a android.intent.action.MAIN \
  -c android.intent.category.LEANBACK_LAUNCHER
```

**User-installed (3rd party) at the time:**

| App | Package | Version (then) |
|---|---|---|
| TV Browser | `com.internet.tvbrowser` | 2.18.3 |
| Moonlight | `com.limelight` | 12.2 |
| Steam Link | `com.valvesoftware.steamlink` | 1.4.5 |
| Stremio | `com.stremio.one` | 1.12.1 |
| Disney+ | `com.disney.disneyplus` | 26.20.0 |

Also present on the Leanback launcher: Play Store, YouTube, YouTube Music, Netflix, Prime Video, PatchWall / Mi TV Home, Xiaomi TV Plus, Miracast / Smart Share, Gallery, Media Explorer, Google TV (Videos), Play Games, etc.

### 3. Remove streaming / Xiaomi bloat (first pass)

```bash
# User app
adb -s $SERIAL shell pm uninstall com.disney.disneyplus

# Updated system apps: drop Play/update layer, then remove for user 0
for pkg in \
  com.amazon.amazonvideo.livingroom \
  com.mitv.tvhome.atv \
  com.mitv.tvhome.mitvplus
do
  adb -s $SERIAL shell pm uninstall "$pkg"           # may only remove update
  adb -s $SERIAL shell pm uninstall --user 0 "$pkg"  # hide/remove for current user
done

# Privileged system app
adb -s $SERIAL shell pm uninstall --user 0 com.xiaomi.mitv.smartshare
```

| Removed | Package | Role |
|---|---|---|
| Disney+ | `com.disney.disneyplus` | Streaming |
| Prime Video | `com.amazon.amazonvideo.livingroom` | Streaming (system) |
| PatchWall | `com.mitv.tvhome.atv` | Xiaomi home / content |
| Xiaomi TV Plus | `com.mitv.tvhome.mitvplus` | Xiaomi content |
| Miracast | `com.xiaomi.mitv.smartshare` | Cast / Wi‑Fi Display |

### 4. Safe Xiaomi removals (ads / suggestions / clutter)

Home was **Google TV** (`com.google.android.apps.tv.launcherx`). Remaining Xiaomi packages still fed OEM tabs, channels, telemetry, and extra launcher tiles.

```bash
for pkg in \
  com.mitv.tvhome.oemtab \
  com.mitv.tvhome.michannel \
  com.xiaomi.statistic \
  com.xiaomi.mimusic2 \
  com.mitv.gallery \
  com.xiaomi.mitv.mediaexplorer \
  com.xiaomi.tvqs.overseas
do
  adb -s $SERIAL shell pm uninstall "$pkg" 2>/dev/null
  adb -s $SERIAL shell pm uninstall --user 0 "$pkg" 2>/dev/null
done
```

| Removed | Package | Why |
|---|---|---|
| OEM tab | `com.mitv.tvhome.oemtab` | Xiaomi tab in Google TV |
| Mi Channel | `com.mitv.tvhome.michannel` | Promo / channel content |
| MiTV Stat | `com.xiaomi.statistic` | Telemetry |
| Mi Music | `com.xiaomi.mimusic2` | Unused music app |
| Gallery | `com.mitv.gallery` | Gallery / screensaver surface |
| Media Explorer | `com.xiaomi.mitv.mediaexplorer` | File browser tile |
| TvQS | `com.xiaomi.tvqs.overseas` | Xiaomi quick-settings extras |

**Left alone (do not remove casually):** Google TV launcher (until replaced), TV Settings, Play Store, core system packages.

### 5. Install Projectivy Launcher (clean home)

Sideloaded **Projectivy Launcher 4.68** from the official GitHub release:

- Release: https://github.com/spocky/miproja1/releases/tag/4.68  
- APK: `ProjectivyLauncher-4.68-c82-xda-release.apk`  
- Package: `com.spocky.projengmenu`  
- Home activity: `.ui.home.MainActivity`  
- Accessibility service: `.services.ProjectivyAccessibilityService`

```bash
adb -s $SERIAL install -r ProjectivyLauncher-4.68-c82-xda-release.apk

# Preferred home + accessibility override
adb -s $SERIAL shell cmd package set-home-activity --user 0 \
  com.spocky.projengmenu/.ui.home.MainActivity

adb -s $SERIAL shell settings put secure enabled_accessibility_services \
  com.spocky.projengmenu/com.spocky.projengmenu.services.ProjectivyAccessibilityService
adb -s $SERIAL shell settings put secure accessibility_enabled 1

# Stop Google TV / setup from winning the HOME intent
adb -s $SERIAL shell pm disable-user --user 0 com.google.android.apps.tv.launcherx
adb -s $SERIAL shell pm disable-user --user 0 com.google.android.tungsten.setupwraith

adb -s $SERIAL shell am start -n com.spocky.projengmenu/.ui.home.MainActivity
```

After this, `HOME` should resolve to Projectivy. Complete onboarding on the TV remote.

---

## Restore Google TV home (optional)

```bash
adb -s $SERIAL shell pm enable com.google.android.apps.tv.launcherx
adb -s $SERIAL shell pm enable com.google.android.tungsten.setupwraith
# Then pick Google TV again when prompted, or:
adb -s $SERIAL shell cmd package set-home-activity --user 0 \
  com.google.android.apps.tv.launcherx/.home.HomeActivity
```

---

## Notes & caveats

- `pm uninstall` on **updated system apps** often only removes the update; follow with `pm uninstall --user 0 <pkg>` (or `pm disable-user --user 0 <pkg>` if uninstall fails).
- Factory reset / some OTAs can restore preinstalled Xiaomi / Amazon packages.
- Disabling `launcherx` without another HOME app can leave you on Fallback Home — keep Projectivy installed first.
- IP addresses and ADB authorization are local to your network/session.

## Session leftovers (kept)

Still installed as user apps after cleanup: TV Browser, Moonlight, Steam Link, Stremio, plus Projectivy.
