# Debloating a Xiaomi Mi TV (Android TV 9) over Wi-Fi with ADB

A step-by-step guide to make an old Xiaomi Android TV feel lighter: replace the stock home screen with a simple launcher, disable Xiaomi/Google telemetry, PatchWall and preinstalled apps, and shorten animations. Everything is done from a PC on the same network, nothing is rooted or flashed, and every step can be undone.

**Tested on:** Xiaomi `MiTV-MSSP1` (codename `nino`), Android TV 9 (SDK 28), firmware `PI.5925`, MediaTek/MStar `m7322`, 1.8 GB RAM. Other Xiaomi TVs with PatchWall on Android TV 9/10 are very similar, but package names can differ between firmwares — always check your own package list first.

## What to expect

- The home screen becomes a plain grid of your apps: no recommendation rows loading posters, no ads, no PatchWall.
- Animations are twice as short, the heavy Ambient Mode screensaver is gone.
- **Don't expect a big jump in "free RAM".** Measured at the same uptime, free memory stayed about the same (840 MB before, 813 MB after): Android immediately fills freed memory with cached apps, which it drops when needed. The gain is in what runs permanently and in how responsive the UI feels, not in the number.
- 2 GB of RAM and an old CPU are still 2 GB of RAM and an old CPU. It gets noticeably better, not new.

## Safety rules

- Use `pm disable-user`, not uninstall, for system packages. Disabled packages stay on the device and come back with one command.
- **Never disable the stock launcher before a replacement is installed and confirmed working.**
- Make a backup of the package list before changing anything.
- Change things in small batches and check the TV after each batch.
- After disabling packages, **wait ~30 seconds before rebooting.** Android writes package state to disk with a delay; on this TV a reboot a few seconds after `disable-user` silently reverted the whole batch.
- Don't touch anything in the "Do not disable" section below. Disabling system UI, Google Play services or the MediaTek TV stack can leave the TV stuck on the boot logo, and then only a factory reset helps.

## 1. Prepare the PC

1. Download the official [Android SDK Platform-Tools](https://developer.android.com/tools/releases/platform-tools) for your OS and unpack it. You only need `adb` from it.
2. Open a terminal in the unpacked `platform-tools` folder (or add it to `PATH`). The commands below are written as `adb ...`; on Windows PowerShell run them as `.\adb ...` if the folder isn't in `PATH`.

## 2. Enable debugging on the TV

1. **Settings → Device Preferences → About** → press **Build** 7 times until it says you are a developer.
2. **Settings → Device Preferences → Developer options** → turn on **USB debugging**.

On this firmware there is no separate "network debugging" toggle: USB debugging alone makes the TV accept ADB connections on port `5555` over the network.

3. Find the TV's IP address: **Settings → Network & Internet** → your network (or look it up in your router).

## 3. Connect

```bash
adb connect <TV_IP>:5555
```

The first time, the TV shows an **"Allow USB debugging?"** prompt. Accept it and tick **"Always allow from this computer"**. Until you do, `adb devices` shows the TV as `unauthorized`.

```bash
adb devices -l
```

Check what you are working with:

```bash
adb shell getprop ro.product.model
adb shell getprop ro.product.device
adb shell getprop ro.build.version.release
```

If you have more than one device connected, add `-s <TV_IP>:5555` after `adb` in every command.

## 4. Back up the current state

```bash
adb shell pm list packages -f -u > packages_before.txt
adb shell pm list packages -d    > disabled_before.txt
adb shell pm list packages -3    > user_apps_before.txt
adb shell dumpsys meminfo        > meminfo_before.txt
adb shell settings get secure screensaver_components
adb shell cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME
```

Write down the screensaver and home activity values; you'll need them to restore.

`dumpsys meminfo` shows what uses memory right now ("Total PSS by process"). On the tested TV the biggest optional items were the stock launcher (76 MB), Ambient Mode (74 MB) and PatchWall running in the background (67 MB). If you want to compare before/after later, measure both at the same uptime (e.g. 9 minutes after boot) — right after boot the numbers are meaningless.

## 5. Replace the home screen

1. On the TV, install **FLauncher** from the Play Store (package `me.efesser.flauncher`). Projectivy Launcher is a good alternative.
2. Make sure it actually starts:

   ```bash
   adb shell am start -n me.efesser.flauncher/.MainActivity
   ```

3. Set it as the home app and disable the stock launcher:

   ```bash
   adb shell cmd package set-home-activity me.efesser.flauncher/.MainActivity
   adb shell pm disable-user --user 0 com.google.android.tvlauncher
   adb shell input keyevent KEYCODE_HOME
   ```

   While the stock launcher is enabled, Android TV ignores `set-home-activity` (the stock launcher has system priority) — that's why it has to be disabled.

4. Check that Home now resolves to FLauncher:

   ```bash
   adb shell cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME
   ```

The old `com.google.android.leanbacklauncher` can stay: it doesn't run and doesn't use memory.

## 6. Switch the screensaver

Ambient Mode is one of the heaviest processes. Point the screensaver to the tiny built-in one before disabling it:

```bash
adb shell settings put secure screensaver_components com.android.dreams.basic/.Colors
```

## 7. Disable the bloat

Save this as `debloat.txt`. Lines starting with `#` are ignored; optional items are commented out — uncomment what you don't need.

```text
# --- Telemetry ---
com.miui.tv.analytics                      # Xiaomi analytics
com.xiaomi.statistic                       # Xiaomi statistics
com.google.android.feedback                # Google feedback sender
com.google.android.tv.bugreportsender      # bug report sender

# --- PatchWall, promo content, recommendation rows ---
com.mitv.tvhome.atv                        # PatchWall (also contains "Xiaomi TV+")
com.mitv.tvhome.michannel                  # PatchWall channels on the home screen
com.xm.webcontent                          # PatchWall web/promo content
com.xiaomi.floatingframe                   # Xiaomi promo overlay
com.google.android.tvrecommendations       # recommendation rows for the stock launcher
android.autoinstalls.config.xiaomi.amelie  # Play auto-install of "partner" apps

# --- Screensavers (switch the screensaver first, see step 6) ---
com.google.android.backdrop                # Ambient Mode
com.mitv.dream                             # Xiaomi "WallPaper" screensaver

# --- Xiaomi background services ---
com.xiaomi.mitv.tvmanager                  # "TV Manager" cleaner (the rocket tile)
# com.xiaomi.mitv.updateservice            # firmware updates; disable if you don't care about them

# --- Preinstalled apps ---
com.google.android.play.games              # Play Games
com.google.android.videos                  # Play Movies & TV (discontinued)
com.xiaomi.mimusic2                        # Xiaomi music player
com.mitv.gallery                           # Xiaomi gallery
org.chromium.webview_shell                 # WebView test browser
com.android.printspooler                   # printing
com.google.android.marvin.talkback         # TalkBack screen reader; keep it if you need it
# com.google.android.youtube.tvmusic       # YouTube Music
# com.mitv.videoplayer                     # Xiaomi video player
# com.xiaomi.mitv.mediaexplorer            # Xiaomi "Media Player" / USB file browser
```

Before disabling `com.mitv.videoplayer` or `com.xiaomi.mitv.mediaexplorer`, make sure you have another player (VLC, MX Player) and that apps which use an external player (e.g. Lazy Media Deluxe) are set to that player. To check that no app is set to "always open" with the Xiaomi player, look at the "Preferred Activities" section of `adb shell dumpsys package preferred-activities`.

Apply it — bash (Linux/macOS/Git Bash):

```bash
sed 's/#.*//' debloat.txt | awk 'NF {print $1}' | while read -r p; do
  echo "$p: $(adb shell pm disable-user --user 0 "$p" </dev/null)"
done
```

PowerShell (Windows):

```powershell
Get-Content debloat.txt | ForEach-Object { ($_ -split '#')[0].Trim() } | Where-Object { $_ } |
  ForEach-Object { "$_ : $(adb shell pm disable-user --user 0 $_)" }
```

Every line should end with `new state: disabled-user`.

## 8. Shorten animations

```bash
adb shell settings put global window_animation_scale 0.5
adb shell settings put global transition_animation_scale 0.5
adb shell settings put global animator_duration_scale 0.5
```

Use `0` to turn them off completely.

## 9. Reboot and verify

Wait at least 30 seconds after the last `disable-user`, then:

```bash
adb reboot
adb connect <TV_IP>:5555
adb shell getprop sys.boot_completed      # 1 = booted
adb shell pm list packages -d             # everything from debloat.txt + tvlauncher must be here
adb shell logcat -b crash -d              # should be empty
```

If some packages are missing from the disabled list, disable them again, wait 30 seconds and reboot once more.

## 10. Extra steps worth doing

- **Uninstall user apps you don't use.** Some of them start at boot and keep a service running (Spotify, for example, starts a foreground service at boot even if you never open it). List them with `adb shell pm list packages -3` and remove with `adb uninstall <package>` — they can be reinstalled from the Play Store.
- **Turn off Play Store auto-updates** on the TV: Play Store → profile → Settings → Auto-update apps → Don't auto-update. Otherwise the Play Store takes 100+ MB after every power-on while it updates apps. Update manually once in a while.
- **Hide stale tiles in FLauncher.** FLauncher can keep tiles of apps you just disabled. Long-press OK on a tile → Hide.
- **Turn USB debugging off when you're done** (Developer options). ADB on port 5555 is unencrypted; new computers still need to be confirmed on the TV screen, but there's no reason to leave it open.

## Do not disable

| Package(s) | Why |
|---|---|
| `android`, `com.android.systemui`, `com.android.tv.settings`, `com.android.providers.*`, `com.android.shell` | Core system, settings, storage |
| `com.google.android.gms`, `com.google.android.gsf`, `com.android.vending`, `com.google.android.webview` | Google Play services, Play Store, WebView — most apps depend on them |
| `com.google.android.inputmethod.latin` | On-screen keyboard (Gboard) |
| `com.android.bluetooth` | Bluetooth remote |
| `com.mediatek.wwtv.tvcenter`, `com.mediatek.tvinput`, `com.mediatek.tvinputservice.*`, `com.mediatek.tv.agent` | Live TV, tuner **and HDMI inputs** |
| `com.mediatek.hotkey.dispatcher` | Remote buttons |
| `mitv.service`, `com.xiaomi.mitv.res`, `com.mediatek.network` | Xiaomi/MediaTek TV framework |
| `com.google.android.tungsten.setupwraith` | Setup wizard; cleanyourtv reports that disabling it can delete a saved Wi-Fi network |
| `com.google.android.tv.remote.service` | Control from the Google TV app on a phone |

Keep depending on what you use:

- `com.google.android.katniss` — voice search / Assistant (the mic button, also in YouTube).
- `com.google.android.apps.mediashell` — Chromecast built-in (casting from a phone).
- `com.xiaomi.mitv.updateservice` — firmware updates.

## Undo

Re-enable a single package:

```bash
adb shell pm enable --user 0 <package>
```

Re-enable everything from the list (bash / PowerShell):

```bash
sed 's/#.*//' debloat.txt | awk 'NF {print $1}' | while read -r p; do adb shell pm enable --user 0 "$p" </dev/null; done
```

```powershell
Get-Content debloat.txt | ForEach-Object { ($_ -split '#')[0].Trim() } | Where-Object { $_ } |
  ForEach-Object { adb shell pm enable --user 0 $_ }
```

Bring back the stock launcher, screensaver and animations:

```bash
adb shell pm enable --user 0 com.google.android.tvlauncher
adb shell pm enable --user 0 com.google.android.backdrop
adb shell settings put secure screensaver_components com.google.android.backdrop/.Backdrop
adb shell settings put global window_animation_scale 1.0
adb shell settings put global transition_animation_scale 1.0
adb shell settings delete global animator_duration_scale
```

If you ever used `pm uninstall --user 0` on a system package, it comes back with `adb shell cmd package install-existing <package>`.

A firmware update or a factory reset can re-enable some packages. Run `adb shell pm list packages -d` and compare with your list.

## Troubleshooting

- **`unauthorized` in `adb devices`** — accept the prompt on the TV screen. If it doesn't appear, toggle USB debugging off and on and connect again.
- **"Xiaomi TV+ — this feature is not available in your region" on startup** — PatchWall is enabled again (Xiaomi TV+ is part of `com.mitv.tvhome.atv`). Usually it's the "rebooted too fast" problem: disable it again, wait 30 seconds, reboot.
- **Disabled packages are back after a reboot** — same thing: wait ~30 seconds after `disable-user` before rebooting.
- **The Home button still opens the old launcher** — `com.google.android.tvlauncher` isn't disabled; check `pm list packages -d`.
- **A video won't play from another app** — that app was set to use the Xiaomi player; switch it to VLC/MX Player in its settings, or re-enable `com.mitv.videoplayer`.
- **The TV is stuck after a change** — ADB usually still works once the system is up: re-enable the last packages you disabled. If the TV doesn't boot at all, a factory reset is the last resort (that's why you disable in small batches).

## Credits

The package list was cross-checked with the Xiaomi `MiTV-MSSP3` (Android 9) list from [cleanyourtv](https://github.com/Gathin23/cleanyourtv), which has ready debloat lists for ~90 TV models and its own lightweight launcher.
