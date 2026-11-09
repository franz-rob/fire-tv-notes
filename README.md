# What can be done on a Fire TV without root

Português: [o-que-pode-ser-feito-na-fire-tv.md](o-que-pode-ser-feito-na-fire-tv.md)

Tested in November 2026 on one device, with ADB (`shell`, uid 2000) and an ordinary app. No root.

| | |
|---|---|
| Device | Fire TV Stick Lite (2020), AFTSS "sheldon" |
| System | Fire OS 7.7.1.7 (PS7717/5746), Android 9 (API 28) |
| Remote | Alexa Voice Remote Lite: TV button and 4 app buttons, **no volume or power buttons** |
| TV | MStar chip, HDMI-CEC 1.4, no soundbar |

Other Fire OS 7 devices may behave the same, but that wasn't tested. Fire OS 8 (Android 11) restricts starting activities from the background, so parts of this may not hold there.

Every change below can be undone, and the undo is given. The last resort is a factory reset: hold **Back + Right for 10 s** on the remote.

## Summary

| Goal | Result | How |
|---|---|---|
| Replace Amazon's launcher | ❌ | Protected package, and `set-home-activity` is ignored (section 1) |
| Open your app at power-on, before Amazon's home | ✅ visible at ~22 s | Accessibility service (section 2) |
| Bring your app back when Amazon's home appears | ✅ ~5 s after Home | Accessibility service, window events |
| Lock Home, Alexa, TV (guide) and the app buttons | ✅ | `RESTRICT_REMOTE_BUTTONS=1` (section 3) |
| Keep that lock after a restart | ✅ | The app sets it again (`WRITE_SECURE_SETTINGS`) |
| Give Home, Alexa, TV or app buttons a new function | ❌ | The system handles them before any app |
| Own Menu, ⏪, ⏯, ⏩ on any screen | ✅ | Accessibility key filter |
| Use the TV remote's keys (over CEC) | ✅ only in your app | `dispatchKeyEvent` |
| Turn the TV on with the Stick | ✅ only from the Fire TV remote | Waking the Stick turns the TV on |
| Turn the TV off when the Stick sleeps | ❌ | Fire OS doesn't send `<Standby>` |
| Put the Stick to sleep when the TV turns off | ✅ | The TV sends `<Standby>` |
| Change the TV's volume from an app | ❌ | Section 5 |
| Turn off Amazon background apps | ✅ many | `pm disable-user` (section 6) |
| Stop Fire OS updates | ⚠️ partly | Section 7 |
| Remove the boot animation | ❌ | It is in `/system`, read-only |

## 1. Amazon's launcher can't be replaced

- `pm disable-user --user 0 com.amazon.tv.launcher` → `SecurityException: Cannot disable a protected package`.
- `cmd package set-home-activity <your.app>/.HomeActivity` answers `Success` and changes nothing.
- HOME candidates by priority: `com.amazon.tv.launcher/.ui.HomeActivity_vNext` (950), `com.amazon.firehomestarter/.HomeStarterActivity` (1), a third-party app (0), `com.amazon.tv.settings.v2/.system.FallbackHome` (-1000). Declaring `HOME` + `DEFAULT` doesn't help.
- There is an official custom-home hook: `com.amazon.policy.keypolicymanager.CustomHomeButtonHandler` looks in the foreground app for the meta-data `com.amazon.metadata.CUSTOM_HOME` and sends `com.amazon.tv.action.CUSTOM_HOME`. It needs `com.amazon.permission.RECEIVE_CUSTOM_HOME`, which is `signature|amazon`. Every Home press logs `CustomHomeButtonHandler: getCustomHomeReceiver: application info meta data is null`. (Read in `/system/framework/oat/arm/fosservices.vdex`; not tried with an app.)

## 2. Opening your app at power-on

**Use an accessibility service.** Android binds accessibility services early in boot, because screen readers must work from the first screen. On Android 9 such a service can start activities.

| Moment (uptime) | What happens |
|---|---|
| ~21.2 s | The accessibility service connects (`onServiceConnected`) and starts your activity |
| ~22.6 s | Your activity is on screen, right after the boot animation |
| ~28.7 s | Amazon's launcher starts behind it and doesn't come to the front |
| ~52–65 s | Amazon resets `RESTRICT_REMOTE_BUTTONS` to `0` (section 3) |
| ~60–68 s | `BOOT_COMPLETED` reaches third-party apps (what "launch on boot" apps rely on) |

Turn the service on from ADB:
```
adb shell settings put secure enabled_accessibility_services <package>/<package>.<Service>
adb shell settings put secure accessibility_enabled 1
```
Undo: `adb shell settings delete secure enabled_accessibility_services` and `adb shell settings put secure accessibility_enabled 0`. Keep any services already listed, separated by `:`.

What the service should do:
- In `onServiceConnected()`, start your activity, and retry every 5 s until it is in front.
- On `TYPE_WINDOW_STATE_CHANGED` from `com.amazon.tv.launcher`, start it again (at most once per second; nothing if it is already in front).
- To read which package is in front outside an event, declare `android:canRetrieveWindowContent="true"`.

Pitfalls:
- **~5 s after Home.** After a Home press, Android delays activity starts from other apps by ~5 s (the app-switch protection), from an accessibility service too. Locking Home (section 3) avoids it.
- **`am force-stop <your.app>` turns the accessibility service off.** Turn it on again with the two commands above.
- **Don't make the power-on start depend on a setting.** With a "redirect Home" setting left off, the app only came up at ~60 s, through `BOOT_COMPLETED`.
- **The network isn't ready at ~22 s**: DNS fails (`UnknownHostException`). Wait for it, or retry.
- **The clock is behind after every boot** (no RTC battery; it's fixed later from the network). Use `SystemClock.elapsedRealtime()`, plus `Settings.Global.BOOT_COUNT` to tell boots apart, for anything timed.
- **Logs vanish right after boot** (logd's `SPAM_DETECTOR`). Write a log file in `getExternalFilesDir(null)` (`/sdcard/Android/data/<package>/files/`) and read it with `adb pull`. No `debuggable` needed.
- **A finished app dies fast.** After `finish()`, Fire OS kills the cached process ~1 s later, before `onStop`. Save first.
- **Waking from standby.** On wake, Amazon's launcher can come back to the front. The window-change return above brings the app back (seen after sleeps of up to ~10 min). As a backup, the app also checks the front window 2 s and 8 s after wake. The one time the app didn't come back, overnight, the service wasn't running.

**Fire TV settings live inside the launcher.** `Settings.ACTION_SETTINGS` opens `com.amazon.tv.launcher/.ui.SettingsActivity`, which shows the settings inside `HomeActivity_vNext`. An app that brings itself back whenever the launcher appears has to pause that while the user is in Settings. A 5-minute pause, counted with `elapsedRealtime` and dropped on reboot, worked: at the end of it, the app came back if the launcher was in front, and otherwise waited for the launcher to appear again.

## 3. Locking Home, Alexa, TV and the app buttons

A setting Amazon uses during the first setup (OOBE):
```
adb shell settings put global com.amazon.tv.oobe.RESTRICT_REMOTE_BUTTONS 1
```
Undo: the same with `0`.

- Home, Alexa, TV (guide) and the 4 app buttons stop working. Only a small Amazon status bar shows at the bottom.
- Arrows, OK, Back, Menu and ⏪ ⏯ ⏩ keep working.
- The log shows `FireTVKeyPolicyManager: block remote keys setting is enabled, won't process home key`.
- A Home sent with `adb shell input keyevent` is blocked too.

**Amazon resets it at every boot.** `com.amazon.tv.oobe/.BootCompleteReceiver` sets it back to `0` at ~52–65 s. To keep it:
1. Declare `android.permission.WRITE_SECURE_SETTINGS`. It is `signature|privileged|development`, so ADB can grant it: `adb shell pm grant <package> android.permission.WRITE_SECURE_SETTINGS` (undo: `pm revoke`).
2. Register a `ContentObserver` on `Settings.Global.getUriFor("com.amazon.tv.oobe.RESTRICT_REMOTE_BUTTONS")` and put `1` back when it turns `0`. In every restart tested, it was back in under 1 s (6 ms in the last one measured).

Trade-off: with the lock on, Home does nothing at all. It no longer takes the user anywhere, so the app needs its own way out to Amazon's home and to Settings.

## 4. Remote keys

Fire TV remote: `dev="Amazon Fire TV Remote"`, src `0x301`. TV remote through HDMI-CEC: `dev="Virtual"`, id `-1`, src `0x2000001`.

| Key | Code | Accessibility filter (`onKeyEvent`) | App in front (`dispatchKeyEvent`) |
|---|---|---|---|
| Fire TV: arrows / OK | 19–22 / 23 | ✅ | ✅ |
| Fire TV: Back | 4 | ✅ | ✅ |
| Fire TV: Menu | 82 | ✅ and can be **consumed** | ✅ |
| Fire TV: ⏪ ⏯ ⏩ | 89 / 85 / 90 | ✅ and can be **consumed** | ✅ |
| Fire TV: Home, Alexa, TV, app buttons | — | ❌ | ❌ (handled by the system first) |
| TV remote (CEC): arrows, OK, ⏪ ⏩ ⏸ ⏹ ⏮ ⏭ | 19–23, 89, 90, 127, 86, 88, 87 | ❌ | ✅ |

- Consuming (returning `true` from `onKeyEvent`) needs `android:canRequestFilterKeyEvents="true"` and `flagRequestFilterKeyEvents`.
- Keys from `adb shell input keyevent` don't reach the accessibility filter, only the app in front. Test the filter with the real remote.
- App buttons are "partner keys": the button → app map comes from Amazon's cloud (`partnerkeyconfig.json`), and the system opens `com.amazon.venezia/.pdi.AppLaunchActivity`. The TV (guide) button opens `com.amazon.tv.livetv/.GuideWrapperActivity`.

## 5. HDMI-CEC: TV power and volume

An ordinary app can't send CEC: that needs `HDMI_CEC`, a system-only permission. ADB can't either.

| Works | How |
|---|---|
| Turning the TV on by waking the Stick | Only from the Fire TV remote: OK wakes the Stick and the TV turns on. Amazon's HDMI service sends it, outside `HdmiControlService`'s history. An app can't wake the Stick |
| Sleeping the Stick when the TV turns off | The TV sends `<Standby>` (`0F:36`) and the Stick sleeps within ~1 s |
| TV remote keys in your app | The TV passes them over CEC |

| Doesn't work | Why |
|---|---|
| Turning the TV off when the Stick sleeps | Fire OS sends no `<Standby>`, even with `hdmi_control_auto_device_off_enabled=1` (`mAutoTvOff:true`). The Stick only drops the HDMI signal, and the TV stays on with "no signal" |
| TV volume from an app | `AudioManager.adjustStreamVolume`, `adjustSuggestedStreamVolume` and injected `VOLUME_UP/DOWN` send nothing (`STREAM_MUSIC` is fixed at 15/15 on HDMI). Only Amazon's "Equipment Control" (`com.amazon.tv.devicecontrol`) sends CEC volume, and Fire TV settings show it as unavailable with this remote: it needs a remote with power and volume buttons, or a Fire TV Blaster. Forcing `dc_volume_control_enabled=1` changed nothing |

## 6. Amazon background apps

Each one was turned off with `adb shell pm disable-user --user 0 <package>`, one at a time and with a restart afterwards. All were accepted, and the device kept working. Undo: `adb shell pm enable <package>`.

| Package | What it is |
|---|---|
| `com.amazon.bueller.photos` | Amazon Photos |
| `com.amazon.tahoe` | Amazon Kids |
| `com.amazon.whad` | Multi-room audio (PERSISTENT: still started at boot) |
| `com.amazon.aca` | Alexa Central for Apps |
| `com.amazon.ceviche` | Metrics |
| `com.amazon.tv.legal.notices` | Legal notices |
| `com.amazon.whisperjoin.middleware.np`, `com.amazon.tv.ffsprovisioneeclient` | Automatic Wi-Fi setup |
| `com.amazon.tv.developer.dataservice` | Developer data service |
| `com.amazon.avsyncslider` | A/V sync adjustment |
| `com.amazon.tv.forcedotaupdater.v2` | Forced Fire OS updater |
| `com.amazon.vizzini` | Alexa core (PERSISTENT: keeps running, and **Amazon enables it again at boot**) |
| `com.amazon.vizzini.ftvcds`, `com.amazon.alexamediaplayer.runtime.ftv`, `com.amazon.alexa.externalmediaplayer.fireos`, `com.amazon.alexadirectivebrokerservice`, `com.amazon.alexa.datastore.app`, `com.amazon.tv.alexanotifications`, `com.amazon.tv.alexaalerts`, `com.amazon.aria` | Alexa and Alexa+ |
| `com.amazon.firebat` | Prime Video |

- **Protected** (`Cannot disable a protected package`): `com.amazon.tv.launcher`, `com.amazon.diode`, `com.amazon.device.software.ota`, `com.amazon.device.software.ota.override`.
- **Left alone on purpose:** `com.amazon.tv.devicecontrol` (CEC equipment control), `com.amazon.device.blepa` and `com.amazon.accessoryconfigservice` (remote pairing: you could lose the remote), `com.amazon.tv.ime` (keyboard), `com.amazon.logan` (VoiceView screen reader), `com.amazon.prism.android.service`.

## 7. Fire OS updates

| Try | Result |
|---|---|
| `pm disable-user` on `com.amazon.device.software.ota` and `...ota.override` | ❌ Protected |
| `pm disable-user` on `com.amazon.tv.forcedotaupdater.v2` | ✅ Off |
| `pm suspend com.amazon.device.software.ota` | ❌ `shell` lacks `SUSPEND_APPS` |
| `cmd appops set com.amazon.device.software.ota RUN_IN_BACKGROUND ignore` | ⚠️ Accepted and kept after a restart. It blocked at least one background run, but `OtaService` still starts at boot |

Not tried yet: blocking the update hosts on the router or with a private DNS.

## Still open

- Confirm with the real remote that Home and the app buttons stay locked after a restart.
- Turning the TV off from the Stick, and waking the Stick from an app.
- A new function for Home, Alexa, TV and the app buttons (they can only be locked).
