# POCO F3 (alioth) — Clean Flash to PixelOS + Root

A complete, tested walkthrough for moving a POCO F3 off an abandoned custom ROM
(or stock MIUI) onto a current PixelOS build with Magisk root — without losing
your personal data.

This is not a generic flashing guide. It was written from a real install on a
real device and documents the things that actually go wrong: the recovery that
won't boot, the boot image that's secretly truncated, the Magisk zip that
TWRP can't flash, and the A/B slot confusion that makes a healthy phone look
bricked.

**Tested configuration**

| Item | Value |
|---|---|
| Device | POCO F3, model M2012K11AG, codename `alioth`, arm64 |
| Starting ROM | AncientOS 12.1 v6.4 (Sept 2022), bootloader already unlocked |
| Target ROM | PixelOS 17.0 (`PixelOS_alioth-17.0-20261112-0826.zip`) |
| Firmware | Global `fw_alioth_OS1.0.2.0.TKHMIXM.zip` |
| Root | Magisk 31.1 via patched boot image |
| Recovery used | TWRP 3.6.2 (already installed on slot A) |
| Result | Android 17, rooted, Zygisk + Play Integrity, ~22 GB of user data restored |

## Contents

- [The honest answer about "keeping your data"](#the-honest-answer-about-keeping-your-data)
- [Requirements](#requirements)
- [Step 0 — Identify the device](#step-0--identify-the-device-and-state)
- [Step 1 — Back up internal storage](#step-1--back-up-internal-storage)
- [Step 2 — Download the flash files](#step-2--download-the-flash-files)
- [Step 3 — Recovery strategy](#step-3--recovery-strategy)
- [Step 4 — Reboot to fastboot, verify unlock](#step-4--reboot-to-fastboot-and-verify-the-unlock)
- [Step 5 — Boot TWRP, format data](#step-5--boot-twrp-and-format-data)
- [Step 6 — Push files and flash](#step-6--push-files-and-flash)
- [Step 7 — Boot, patch, root](#step-7--boot-patch-root)
- [Step 8 — Root hardening](#step-8--root-hardening-banking-apps-play-integrity)
- [Step 9 — Restore your data](#step-9--restore-your-data)
- [Step 10 — Camera and tools](#step-10--camera-gcam-and-recommended-tools)
- [Step 11 — Optimization](#step-11--post-install-optimization)
- [Troubleshooting](#troubleshooting--things-that-actually-happened)
- [Pre-wipe checklist](#sanity-checklist-before-you-wipe)
- [Warnings](#warnings)

---

## The honest answer about "keeping your data"

There is **no** upgrade path that keeps data on the phone when switching ROM
families. Different ROM = different encryption = mandatory `format data`. What
you *can* do is get the data off the phone first and put it back after. That's
what this guide does.

What survives if you follow the steps:

- Documents, scans, downloads, books, music, movies
- WhatsApp received media
- Anything else in internal storage you choose to pull
- Old Swift Backup archives (if you have them)

What does not survive unless you back it up separately:

- App data: logins, chats, authenticator/2FA codes, game saves
- SMS and call log (export first — commands below)
- Photos/videos (assume Google Photos has them, or pull them too)

> **Do not skip the backup.** `format data` is irreversible and wipes internal
> storage including `/sdcard`.

---

## Requirements

- Windows PC (adapt paths for Linux/macOS — most commands are identical)
- USB cable connected to a **USB 2.0 Type-A port directly on the PC**. This
  phone's fastboot is unreliable on USB 3.x ports and hubs on some machines.
- [Android Platform Tools](https://developer.android.com/tools/releases/platform-tools) —
  extract to `platform-tools/` and use that `adb`/`fastboot` for everything
- ~30 GB free disk space for the backup + files
- Phone charged above ~50%
- Existing TWRP on the device (see [Recovery strategy](#step-3-recovery-strategy)).
  If you're coming from a custom ROM you probably already have it.

### Git Bash users (Windows)

MSYS/Git Bash rewrites `/sdcard/...` into `C:/Program Files/Git/sdcard/...` and
breaks every `adb pull`/`adb push`. Export this once per shell:

```bash
export MSYS_NO_PATHCONV=1
```

Or use PowerShell / cmd instead.

---

## Step 0 — Identify the device and state

Connect the phone, authorize USB debugging, then:

```bash
adb devices                                          # expect: <serial> device
adb shell getprop ro.product.device                  # expect: alioth
adb shell getprop ro.product.model                   # M2012K11AG
adb shell getprop ro.build.display.id                # what ROM you're on
adb shell getprop ro.build.version.security_patch    # how stale it is
adb shell df -h /sdcard                              # how much data to back up
```

If `adb devices` shows `offline`: unlock the phone, watch for the
"Allow USB debugging?" dialog, check "Always allow" and accept.

## Step 1 — Back up internal storage

Pull whatever matters to the PC. Example (what was backed up on the test
device — adjust to your folders):

```bash
mkdir backup
for d in Documents Download Books Music Movies YMusic \
         "WhatsApp/Media" "Android/data/com.intsig.camscanner" \
         backups SwiftBackup Pic Pic14; do
  adb pull "/sdcard/$d/" "backup/$(basename "$d")/"
done
```

Also export SMS and call log (needs an app like "SMS Backup & Restore", or your
existing backup tool), and export your installed package list:

```bash
adb shell pm list packages -3 > backup/user-apps.txt
```

**Optional but strongly recommended:** run a fresh [Swift Backup](https://swiftapps.org/)
to local storage *before* wiping, then pull `SwiftBackup/` again. Without it,
all app data — logins, chats, authenticator codes — is gone.

Skip `DCIM/Camera` if your photos are in Google Photos (that's ~60+ GB you
don't need to move).

## Step 2 — Download the flash files

Create a `flash/` folder. You need four things:

| File | Where | Approx. size |
|---|---|---|
| PixelOS ROM zip | [pixelos.net/download/alioth](https://pixelos.net/download/alioth) — take the newest build | ~3 GB |
| Global firmware | `fw_alioth` `OS1.0.2.0.TKHMIXM` from [XiaomiFirmwareUpdater](https://xmfirmwareupdater.com/firmware/alioth/stable/OS1.0.2.0.TKHMIXM/) or its GitHub releases mirror | ~140 MB |
| Magisk APK | [github.com/topjohnwu/Magisk/releases](https://github.com/topjohnwu/Magisk/releases) | ~10 MB |
| Boot image | `boot-alioth_*.img` from the PixelOS download page's recovery images, or extracted from the ROM zip's `payload.bin` | **~192 MB** |

`vendor_boot`/`dtbo` images are only needed if you take the official
recovery-based route (Step 3). The TWRP route this guide follows doesn't use
them.

### Verify the boot image — this is the #1 failure point

Mirror sites (SourceForge in particular) have served **truncated** boot images
for this device. A truncated image still "downloads successfully" but Magisk
will fail with `Unable to unpack the boot image` and the PixelOS recovery may
refuse to boot entirely.

For the `20261112` build the real `boot-alioth_20261112_0826.img` is
**201,326,592 bytes (~192 MB)**. If yours is ~23 MB, it's broken — re-download
from `download.pixelos.net`, not a mirror.

Check the Android magic bytes too:

```bash
head -c 8 boot-alioth_*.img   # expect: ANDROID!
```

## Step 3 — Recovery strategy

The official PixelOS instructions say to flash their `vendor_boot` + `boot`
images, `fastboot reboot recovery`, then sideload from their recovery.

**On this device that recovery may never boot** — the phone sits on the POCO
logo indefinitely (confirmed on the test unit, across both slots, even with
`dtbo` flashed). If you already have a working TWRP, skip the official recovery
entirely and flash from TWRP. That is the route this guide documents; it is
also more flexible because ADB works in TWRP's main menu.

If you don't have TWRP yet and PixelOS recovery won't boot, flash a known-good
`alioth` TWRP image first:

```bash
fastboot flash recovery_a twrp-alioth.img
fastboot --set-active=a
fastboot reboot recovery
```

## Step 4 — Reboot to fastboot and verify the unlock

```bash
adb reboot bootloader
fastboot devices
fastboot getvar product         # alioth
fastboot getvar unlocked        # yes — REQUIRED
fastboot getvar current-slot    # note which slot is active
```

- If `unlocked: no`, you cannot proceed without `fastboot oem unlock`, which
  itself wipes the phone. (This guide assumes an already-unlocked bootloader.)
- `error: no devices/emulators found` usually means Windows bound the wrong
  driver, or a USB 3.x port. Try a USB 2.0 port and check Device Manager for
  "Android Bootloader Interface".

**If the phone hangs at the POCO logo at any point:** hold **Power + Volume
Down** for 10–15 seconds. It forces a reboot straight into fastboot even when
Power alone does nothing. This combo is your undo button — remember it.

## Step 5 — Boot TWRP and format data

```bash
fastboot reboot recovery        # or Power+VolUp at boot
```

In TWRP:

1. **Wipe → Format Data** → type `yes` → confirm.
   This is the point of no return — everything on internal storage is erased.
2. Back to the main menu. Do **not** reboot.

## Step 6 — Push files and flash

ADB is live in TWRP's main menu. Push everything:

```bash
adb shell mkdir -p /sdcard/flash
adb push flash/PixelOS_alioth-*.zip      /sdcard/flash/
adb push flash/fw_alioth_*.zip           /sdcard/flash/
adb push flash/Magisk-v*.apk             /sdcard/flash/
adb push flash/boot-alioth_*.img         /sdcard/flash/
```

Then on the phone, **Install** from `/sdcard/flash/` **in this order**:

1. `PixelOS_alioth-*.zip` — takes several minutes; it's a ~3 GB payload.
   When it finishes, tap **Wipe cache/dalvik** when offered.
2. `fw_alioth_*.zip` — the firmware.
3. **Skip the Magisk zip** — see the warning below.

> **Do not flash Magisk.zip in TWRP.** On this device TWRP fails with
> `mount: can't find /system in /etc/fstab` / `Cannot mount /system` and the
> flash errors out. It's harmless — ROM and firmware still install — but you
> root properly in Step 7 by patching the boot image instead, which is also
> the cleaner method.

Note which slot the ROM installer targeted (TWRP flashes A/B ROMs to the
**inactive** slot — e.g., it installed to slot **B** on the test unit).

## Step 7 — Boot, patch, root

1. TWRP → **Reboot → System**, and if it asks which slot, pick the slot the ROM
   was installed to (e.g., **B**). Decline the TWRP-app install prompt.
2. First boot takes a few minutes. Complete setup, enable USB debugging again
   (Developer options: tap Build number 7×).
3. Install Magisk:

   ```bash
   adb install flash/Magisk-v*.apk
   ```

4. In the Magisk app: **Install → Select and Patch a File** → choose
   `boot-alioth_*.img` (you pushed it in Step 6; also in `Download/` if you
   put it there). It writes `magisk_patched-*.img` to `Download/`.
5. Pull it and flash it to the **slot the system is running on**:

   ```bash
   adb pull /sdcard/Download/magisk_patched-*.img flash/
   adb reboot bootloader
   fastboot getvar current-slot            # confirm: b on the test unit
   fastboot flash boot_b flash/magisk_patched-*.img   # or boot_a — match the slot!
   fastboot reboot
   ```

   Flashing the wrong slot is the classic way to end up staring at a POCO logo.
6. Verify root:

   ```bash
   adb shell su -c id        # expect: uid=0(root) ... context=u:r:magisk:s0
   adb shell su -c magisk -v # e.g. 31.1:MAGISK:R
   ```

   Open the Magisk app once first if `su` isn't responding — the daemon needs
   its first run, and a superuser dialog may need approval.

## Step 8 — Root hardening (banking apps, Play Integrity)

In the Magisk app:

1. Settings → enable **Zygisk**
2. Settings → enable **Enforce DenyList**, then **Configure DenyList** and tick
   Google Wallet, Google Play services, and your banking apps as you reinstall
   them.
3. Install [PlayIntegrityFork](https://github.com/osm0sis/PlayIntegrityFork/releases)
   as a Magisk module (push the zip, or Magisk → Modules → Install from storage).
4. Reboot, then check integrity with the Play Integrity API checker or just try
   your banking apps.

No guarantee on banking apps — this improves your odds; results vary per app
and per ROM build.

## Step 9 — Restore your data

```bash
for d in Documents Download Books Music Movies YMusic \
         "WhatsApp/Media" backups SwiftBackup Pic Pic14; do
  adb push "backup/$(basename "$d")/" "/sdcard/$d/"
done
```

CamScanner data goes back to `Android/data/com.intsig.camscanner` — note that
on Android 11+ you may need to reinstall the app first, then copy via a file
manager, since direct `adb push` into `Android/data` is restricted on newer
versions. Alternative: copy the scans into `Documents/` where the app can
import them.

Then clean up the staging folder on the phone:

```bash
adb shell rm -rf /sdcard/flash
```

## Step 10 — Camera (GCam) and recommended tools

Stock custom-ROM cameras are weak on this sensor. The community-tested option
for `alioth` is an LMC8.4 GCam build with a POCO F3 config:

1. Get an `alioth`-compatible build from
   [celsoazevedo.com/files/android/google-camera](https://www.celsoazevedo.com/files/android/google-camera/) —
   `LMC8.4_R18*` works well.
2. Download a POCO F3 config (e.g. `Tempest-LMC8.4-F3-v5.2.xml`, `f3-leica.xml`
   from the same site) into `/sdcard/LMC8.4/`.
3. Loading the config is a hidden gesture: in the camera viewfinder,
   **double-tap the empty area next to the shutter button** → choose the XML.
   (There is no "Load config" menu item.)

Recommended tools (all verified working on this setup):

| App | Purpose | Source |
|---|---|---|
| AdAway | System-wide adblock (root hosts) | F-Droid |
| AppManager | App ops, trackers, debloat | F-Droid / GitHub |
| AFWall+ | Per-app firewall | F-Droid |
| SmartPack KM | Kernel settings viewer/tweaker | GitHub |
| Shizuku | ADB-level API bridge for tools | GitHub |
| Termux | Terminal | F-Droid |
| F-Droid | FOSS app store | f-droid.org |
| Aurora Store | Play Store apps anonymously | F-Droid (`com.aurora.store`) |
| SD Maid SE | Storage cleaner | F-Droid / GitHub |
| AccA | Charge limit (80%) — battery longevity | GitHub `MatteCarra/AccA` |
| Lawnchair + Lawnicons | Launcher + themed icons | GitHub |

MiXplorer (excellent file manager) has no direct APK link — get it via the
Mobilism app or XDA.

## Step 11 — Post-install optimization

```bash
# AOT-compile all apps (runs in background, ~10-30 min, phone runs warm)
adb shell pm bg-dexopt-job

# TRIM the flash storage
adb shell su -c fstrim -v /data

# Snappier UI — halve animation durations
adb shell settings put global window_animation_scale 0.5
adb shell settings put global transition_animation_scale 0.5
adb shell settings put global animator_duration_scale 0.5

# Confirm 120 Hz is active (alioth has a 120 Hz panel)
adb shell dumpsys display | grep -i refresh
```

AccA/charge limiting depends on kernel support — open the app and let it check;
if it can't find a control node, the limit simply can't be applied on that
kernel. Don't force it.

---

## Troubleshooting — things that actually happened

| Symptom | Cause | Fix |
|---|---|---|
| `adb devices` → `offline` | Pending USB-debugging auth | Unlock phone, accept dialog, replug cable |
| `fastboot` sees nothing, phone at POCO logo | Hung boot | Hold **Power+VolDown** 10–15 s → fastboot |
| PixelOS recovery never boots (POCO logo forever) | Recovery image incompatible with old firmware | Use existing TWRP (other slot), or flash TWRP |
| `Fetching super ... FAILED ... 'unknown command'` | Harmless fastbootd quirk | Ignore — check the actual `Writing ... OKAY` lines |
| TWRP: `Cannot mount /system`, zip error 1 | TWRP can't flash Magisk on A/B | Patch boot.img in the Magisk app instead |
| Magisk: `Unable to unpack the boot image` | Truncated boot.img (~23 MB vs 192 MB) | Re-download from the official PixelOS host |
| Phone boots to logo after flash | Magisk/flashing hit wrong slot | `fastboot getvar current-slot`, flash the matching `boot_a`/`boot_b` |
| Git Bash: `failed to stat remote object 'C:/Program Files/Git/sdcard'` | MSYS path mangling | `export MSYS_NO_PATHCONV=1` |
| `su` → Permission denied | Magisk daemon not initialized / unapproved | Open Magisk app once, approve the dialog, retry |
| No "load config" in LMC8.4 | It's a hidden gesture | Double-tap next to the shutter button |

## Sanity checklist before you wipe

- [ ] `backup/` on the PC contains everything you care about (open it and look)
- [ ] SMS/call log exported if wanted
- [ ] Swift Backup run + pulled (if you want app data)
- [ ] 2FA/authenticator migrated or codes saved — these do **not** come back
- [ ] WhatsApp Google Drive backup current (if chats matter)
- [ ] All four flash files downloaded, boot.img size verified (~192 MB)
- [ ] Battery above ~50%
- [ ] You know the Power+VolDown forced-fastboot combo

## Warnings

- Everything here requires an unlocked bootloader, voids warranty, and can
  brick the device if you flash images meant for another model. Only files
  built for `alioth` belong anywhere near this phone.
- Flashing the wrong slot or an image for a different device is the main way
  people brick these phones. When in doubt, `fastboot getvar` everything.
- Keep your PC-side backup until you've lived on the new ROM for a week or two.
- Third-party APK sources carry real risk. Prefer F-Droid, official GitHub
  releases, and the ROM's own site over random mirrors.

## License

MIT — see [LICENSE](LICENSE). Device names and ROMs belong to their respective
projects; this guide is an independent walkthrough with no affiliation.
