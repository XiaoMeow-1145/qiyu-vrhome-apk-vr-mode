# Qiyu VRHome APK VR Mode

## Rule

Produce a patched APK file. Do not require adb, a connected headset, `adb install`, `am start`, logcat, screenshots, or on-device validation unless the user explicitly asks for device testing.

The primary manifest signal is application-level metadata:

```xml
<application ...>
    <meta-data android:name="qiyu-sdk-flag" android:value="true" />
</application>
```

On QIYU Mix / VR700-style firmware, Qiyu Home asks `PMExt.isVRApplication(packageName)`. The PackageManager implementation returns true when either:

- `ApplicationInfo.metaData.getBoolean("qiyu-sdk-flag")` is true.
- `ApplicationInfo.vrType == 2`.

An offline APK patch should therefore add `qiyu-sdk-flag=true`. Do not treat `com.qti.intent.category.SNAPDRAGON_VR` as the decisive Qiyu Home VR-app signal. It can be added only as a compatibility hint when requested.

Do not add QiyuHome's own launcher action `iqiyi.3d.vr.launcher` to third-party apps. That action belongs to `com.qiyu.vraio.home/.QiyuHomeActivity`, not to patched app manifests.

## Optional Legacy Category

For compatibility with Qualcomm/SXR-style launch patterns, the script can also add this category to the app's `MAIN + LAUNCHER` activity:

```xml
<category android:name="com.qti.intent.category.SNAPDRAGON_VR" />
```

Use this only when the user asks for the legacy category too:

```bash
python3 ~/.codex/skills/qiyu-vrhome-apk-vr-mode/scripts/patch_qiyu_vrmode_apk.py \
  /path/to/input.apk \
  --add-qti-category \
  --output /path/to/input-qiyu-vrmode.apk
```

If the APK has more than one launcher activity and a category must be added, specify `--activity`.

## Quick Start

Use the bundled script first:

```bash
python3 ~/.codex/skills/qiyu-vrhome-apk-vr-mode/scripts/patch_qiyu_vrmode_apk.py \
  /path/to/input.apk \
  --output /path/to/input-qiyu-vrmode.apk
```

This default path adds only `qiyu-sdk-flag=true`, rebuilds, aligns when possible, and signs the output APK.

If you also need the optional QTI category and the APK has more than one launcher activity, specify the target explicitly:

```bash
python3 ~/.codex/skills/qiyu-vrhome-apk-vr-mode/scripts/patch_qiyu_vrmode_apk.py \
  /path/to/input.apk \
  --add-qti-category \
  --activity com.example.game/.MainActivity \
  --output /path/to/input-qiyu-vrmode.apk
```

If the user only wants the patched file generated, stop after the script writes the output APK. Do not ask for a device.

## Required Tools

The script requires:

- `apktool`
- one signer: `apksigner` from Android SDK build-tools, or `jarsigner`
- `keytool` if the script must create a debug signing key
- `zipalign` is preferred, but the script can continue without it

If a required tool is missing, report the exact missing tool and stop. Do not invent a binary APK patch by editing zip bytes directly.

## Script Behavior

The script performs these deterministic steps:

1. Decode the APK with `apktool`.
2. Parse decoded `AndroidManifest.xml`.
3. Locate the manifest `<application>`.
4. Add or update `<meta-data android:name="qiyu-sdk-flag" android:value="true" />`.
5. If `--add-qti-category` or `--category` is used, locate the target `activity` or `activity-alias` and patch its `MAIN + LAUNCHER` `intent-filter`.
6. Rebuild the APK with `apktool`.
7. Align with `zipalign` when available.
8. Sign the generated APK.
9. Print the package, metadata result, optional activity/categories, and output path.

The default signing key is a generated debug key under the work directory. This is enough to produce an installable APK, but it will not update over an already-installed app signed with a different certificate. If the user needs update-in-place, ask for the original signing keystore or tell them the existing package must be uninstalled before installing the debug-signed patched APK.

## Manual Fallback

Only use this fallback if the script cannot be used but the required external tools are available.

Decode:

```bash
apktool d -f /path/to/input.apk -o /tmp/qiyu-vrmode-decoded
```

Patch `/tmp/qiyu-vrmode-decoded/AndroidManifest.xml` so the `<application>` element has:

```xml
<meta-data android:name="qiyu-sdk-flag" android:value="true" />
```

If also adding the optional legacy category, patch the main launcher activity so it has:

```xml
<intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.LAUNCHER" />
    <category android:name="com.qti.intent.category.SNAPDRAGON_VR" />
</intent-filter>
```

Rebuild:

```bash
apktool b /tmp/qiyu-vrmode-decoded -o /tmp/qiyu-vrmode-unsigned.apk
```

Align if `zipalign` exists:

```bash
zipalign -p -f 4 /tmp/qiyu-vrmode-unsigned.apk /tmp/qiyu-vrmode-aligned.apk
```

Sign with `apksigner`:

```bash
apksigner sign \
  --ks /path/to/debug.keystore \
  --ks-key-alias qiyuvrmode \
  --ks-pass pass:android \
  --key-pass pass:android \
  --out /path/to/output-qiyu-vrmode.apk \
  /tmp/qiyu-vrmode-aligned.apk
```

If using `jarsigner`, sign the aligned APK into the output path:

```bash
jarsigner \
  -keystore /path/to/debug.keystore \
  -storepass android \
  -keypass android \
  -signedjar /path/to/output-qiyu-vrmode.apk \
  /tmp/qiyu-vrmode-aligned.apk \
  qiyuvrmode
```

## Static Checks

Static checks are allowed and do not require adb:

```bash
apktool d -f /path/to/output-qiyu-vrmode.apk -o /tmp/qiyu-vrmode-check
rg -n "qiyu-sdk-flag|com.qti.intent.category.SNAPDRAGON_VR|android.intent.action.MAIN|android.intent.category.LAUNCHER" \
  /tmp/qiyu-vrmode-check/AndroidManifest.xml
```

If `aapt` or `aapt2` is available, a manifest dump is also acceptable. Do not fail the task just because no adb device exists.

Optional device-side verification, only when explicitly requested by the user:

```bash
adb shell service call package 211 s16 <package.name>
```

`00000001` in the result indicates the firmware-side `PMExt.isVRApplication(packageName)` check returned true. Do not run this by default.

## Reference

Read `references/qiyu-vrhome-vr-signals.md` when you need the historical rationale, the Qiyu Home windowed-launch evidence, known category patterns, or what not to add.

