# custom-mudita-launcher Kompakt Launcher Mods



Patch file: `kompakt_launcher_mods_clean.patch`

## What this patch does

Turns the stock Mudita Kompakt launcher (`com.mudita.launcher`, version 1.3.1) into
a side-by-side "custom" build (`com.mudita.launcher.custom`) with the following
additions:

### Identity and manifest
- Renames applicationId to `com.mudita.launcher.custom` so it installs alongside
  the stock launcher.
- Renames the signature-protected `DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION` to
  match the new applicationId.
- Renames ContentProvider authorities (androidx-startup, SentryInitProvider,
  SentryPerformanceProvider) to match.
- Adds two activity declarations: `PickerActivity` (app-chooser helper) and
  `HubActivity` (settings screen). Neither has a `LAUNCHER` intent-filter, so
  they don't show in the app drawer.

### Visual / theme
- `Theme.Launcher` overrides: pure white windowBackground and colorBackground,
  pure black text, pure black/white status and nav bars, `forceDarkAllowed=false`.
- Compose Material3 color palette (`LT1/a`): background `0xCFCFCF` -> `0xFFFFFF`
  and the 50%-gray `0xFF808080` -> `0xFF000000`, for higher contrast on e-ink.

### Dispatcher / overrides
- `Lv2/a;->d(Context, String)` is the generic launch dispatcher. Patched so:
  - when called with `com.mudita.messages`, checks SharedPrefs `kompakt_overrides`
    -> `sms_override` first; if set, launches the override instead.
  - when called with `com.mudita.alarm` (the clock tap target), does the same
    with `clock_override`.
- Default behaviour (no override saved) remains: launches Mudita's stock apps.

### Hub and Picker activities
- `HubActivity`: simple LinearLayout with a title "Custom Mudita Launcher" and
  four black-outlined white buttons: Change messaging app, Change clock app,
  Hide / customize apps, Turn ON/OFF reading mode.
- `PickerActivity`: fires `ACTION_PICK_ACTIVITY`, saves the picked package under
  a `pref_key` passed via intent extra (`sms_override` or `clock_override`), and
  immediately launches the pick.

### Gestures
- Long-press anywhere on the home screen (500ms hold, <30px drift) opens the Hub.
  Implemented via an override of `MainActivity.dispatchTouchEvent` that schedules
  a `HubRunnable` on `ACTION_DOWN` and cancels it on MOVE/UP.
- Double-tap on the apps icon also opens the Hub. Implemented in `LG2/a;->j`
  (the icon-tap dispatcher) using a static `LAST_APPS_TAP_MS` timestamp.

### "Reading mode" feature (gated by toggle)
- `MeinkHelper.init()` reflects into the hidden `android.meink.IMeinkService`
  via `ServiceManager.getService("meink")` and calls `setMode(READING=1)` (transaction
  5, interface token `android.meink.IMeinkService`).
- Gated by a SharedPref `refresh_on_home` (named for historical reasons; the Hub
  UI label reads "Turn ON/OFF reading mode"). Called:
  - once in `LauncherApplication.onCreate` if pref is on at process start
  - once when the Hub toggle flips ON
  - once in `MainActivity.onResume`, but only when `WAS_IN_BACKGROUND` is true
    (set by `onPause`), posted with 100ms `postDelayed` so the call happens after
    the UI has settled
- The pre-existing flash overlay code (`FlashRunnable`) is left in place but no
  longer invoked; can be stripped if APK size matters.

## How to apply

Starting from a freshly apktool-decoded copy of `KompaktLauncher.apk`:

```bash
apktool d -f -o decoded KompaktLauncher.apk
cd <parent-of-decoded>
patch -p0 < kompakt_launcher_mods_clean.patch
```

Then rebuild:

```bash
apktool b -f --use-aapt2 -o rebuilt.apk decoded
zipalign -f -p 4 rebuilt.apk rebuilt-aligned.apk
apksigner sign --ks <your-keystore> --out rebuilt-signed.apk rebuilt-aligned.apk
```

And install alongside the stock launcher:

```bash
adb install rebuilt-signed.apk
adb shell pm grant com.mudita.launcher.custom android.permission.READ_SMS
adb shell pm grant com.mudita.launcher.custom android.permission.READ_CONTACTS
adb shell pm grant com.mudita.launcher.custom android.permission.READ_CALL_LOG
adb shell pm grant com.mudita.launcher.custom android.permission.WRITE_CALL_LOG
```

Then press the home button on the device; Android will show a launcher chooser
and you can pick either the stock Mudita launcher or `Launcher` (the custom one).

## Files touched (13 total)

- `AndroidManifest.xml`
- `res/values/styles.xml`
- `smali/G2/a.1.smali`
- `smali/T1/a.smali`
- `smali/com/mudita/launcher/LauncherApplication.smali`
- `smali/com/mudita/launcher/MainActivity.smali`
- `smali/v2.1/a.smali`
- New: `smali/com/mudita/launcher/FlashRunnable.smali` (dead code)
- New: `smali/com/mudita/launcher/HubActivity.smali`
- New: `smali/com/mudita/launcher/HubRunnable.smali`
- New: `smali/com/mudita/launcher/MeinkHelper.smali`
- New: `smali/com/mudita/launcher/MeinkInitRunnable.smali`
- New: `smali/com/mudita/launcher/PickerActivity.smali`

## Notes

- Patch was generated against apktool 2.10.0's decoding of the v1.3.1 APK.
  Future versions of the launcher may have different obfuscated class names
  (e.g., `LG2/a`, `LT1/a`, `Lv2/a`) and require remapping.
- The `meink` service is specific to Mudita Kompakt. On other devices the
  reflection call returns null and `MeinkHelper.init()` is a no-op.
- `kompakt_overrides` SharedPrefs keys used:
  - `sms_override` (String) - package to launch instead of `com.mudita.messages`
  - `clock_override` (String) - package to launch instead of `com.mudita.alarm`
  - `refresh_on_home` (Boolean) - gates the meink reading-mode feature
