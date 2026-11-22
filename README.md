# POCO F3 (alioth) — Clean Flash + Root Guide

A complete, tested walkthrough for moving a POCO F3 off an abandoned custom
ROM (or stock MIUI) onto a current ROM with root — without losing personal
data, and with working banking apps.

This is not a generic flashing guide. It was written from a real install on a
real device and documents the things that actually go wrong: the recovery that
won't boot, the boot image that's secretly truncated, the A/B slot confusion
that makes a healthy phone look bricked — and, most importantly, why **Magisk
failed against banking apps while kernel-integrated KernelSU passed**.

**Tested configuration (final, verified)**

| Item | Value |
|---|---|
| Device | POCO F3, model M2012K11AG, codename `alioth`, arm64 |
| Starting ROM | AncientOS 12.1 v6.4 (Sept 2022), bootloader already unlocked |
| ROM | crDroid 13.1, Android 17 (`crDroidAndroid-17.0-20261115-alioth-v13.1.zip`) |
| GApps | NikGapps full arm64-17 (`NikGapps-full-arm64-17-20261021.zip`) |
| Firmware | Global `fw_alioth_OS1.0.2.0.TKHMIXM.zip` |
| Kernel | `4.19.325-InfiniR_Alioth_v13.1_KSU_raystef66` — **KernelSU built in** |
| Root | KernelSU (backslashxx fork) via kernel driver — **no Magisk** |
| Manager | `KernelSU_v3.3.1-55_32736` from `backslashxx/KernelSU` releases |
| Recovery used | crDroid recovery / TWRP — both work |
| Result | Android 17, rooted, **both banking apps open normally, before and after reboot** |

> The PixelOS + Magisk path also works as an install (see the alternative path
> section below), but on
> this device **Magisk could not be hidden well enough** for a
> Promon/RASP-protected bank app. KernelSU root lives entirely inside the
> kernel — there is no userspace daemon, no `su` binary on disk, no socket —
> so there is nothing for the app's shield to find.

## Contents

- [The honest answer about "keeping your data"](#the-honest-answer-about-keeping-your-data)
- [Requirements](#requirements)
- [Step 0 — Identify the device](#step-0--identify-the-device-and-state)
- [Step 1 — Back up internal storage](#step-1--back-up-internal-storage)
- [Step 2 — Download the flash files](#step-2--download-the-flash-files)
- [Step 3 — Recovery strategy](#step-3--recovery-strategy)
- [Step 4 — Reboot to fastboot, verify unlock](#step-4--reboot-to-fastboot-and-verify-the-unlock)
- [Step 5 — Recovery: format data](#step-5--recovery-and-format-data)
- [Step 6 — Push files and flash](#step-6--push-files-and-flash)
- [Step 7 — Root with the built-in KernelSU](#step-7--root-with-the-built-in-kernelsu)
- [Step 8 — Banking apps: what worked and why](#step-8--banking-apps--what-worked-and-why)
- [Step 9 — Restore your data](#step-9--restore-your-data)
- [Step 10 — Camera and tools](#step-10--camera-gcam-and-recommended-tools)
- [Step 11 — Optimization](#step-11--post-install-optimization)
- [Alternative: PixelOS + Magisk](#alternative-pixelos--magisk-flash-path)
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
- A working recovery on the device (see [Recovery strategy](#step-3--recovery-strategy)).

Commands below are **bash syntax** (`for` loops, `export`, `head`). On Windows,
run them from Git Bash or WSL. If you use PowerShell/cmd, run each `adb pull` /
`adb push` line by hand — there are only a dozen.

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
| crDroid ROM zip | [crdroid.net/alioth](https://crdroid.net/alioth/13) — take the newest v13.x build | ~1.9 GB |
| GApps | [NikGapps](https://nikgapps.com/downloads) — `NikGapps-full-arm64-17-*` | ~1.2 GB |
| Global firmware | `fw_alioth` `OS1.0.2.0.TKHMIXM` from [XiaomiFirmwareUpdater](https://xmfirmwareupdater.com/firmware/alioth/stable/OS1.0.2.0.TKHMIXM/) or its GitHub releases mirror | ~140 MB |
| KernelSU manager | `KernelSU_v3.3.1-55_*-release.apk` from [backslashxx/KernelSU releases](https://github.com/backslashxx/KernelSU/releases) — **this exact fork**, see Step 7 | ~12 MB |

> **Firmware is region-specific.** `TKHMIXM` (MIXM) is the **Global** branch —
> right for a Global-model phone. If your device shipped EEA (`EUXM`) or India
> (`INXM`) firmware, grab the matching `fw_alioth` for your branch instead, or
> you can end up with broken modem/band behavior. Check your original ROM's
> version string before wiping if unsure.

**Do not download Magisk** — you will not need it. crDroid's bundled InfiniR
kernel has KernelSU compiled in (`CONFIG_KSU=y`); root comes from the kernel
itself.

Sanity-check the ROM zip before you wipe:

```bash
unzip -l crDroidAndroid-*.zip | head    # should list payload.bin or image files
```

## Step 3 — Recovery strategy

crDroid ships its own recovery images (`boot`, `vendor_boot`, `dtbo`, `vbmeta`)
next to the ROM zip on the download page. The official install path is:

```bash
fastboot flash dtbo        dtbo.img
fastboot flash vendor_boot vendor_boot.img
fastboot flash boot        boot.img
fastboot --set-active=a    # match your active slot (getvar current-slot)
fastboot reboot recovery
```

On the test unit, an existing TWRP 3.6.2 was used instead and worked fine —
ADB is live in TWRP's main menu, which makes pushing files easy. If your
official recovery sits at the POCO logo forever (confirmed on this device
with PixelOS recovery), use a known-good `alioth` TWRP instead:

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
- `fastboot` commands hanging while `fastboot devices` lists the serial means
  the phone's fastboot state is wedged: hold **Power ~10 s** to force-reboot,
  then re-enter fastboot with **Power + Volume Down**.

**If the phone hangs at the POCO logo at any point:** hold **Power + Volume
Down** for 10–15 seconds. It forces a reboot straight into fastboot even when
Power alone does nothing. This combo is your undo button — remember it.

## Step 5 — Recovery and format data

```bash
fastboot reboot recovery        # or Power+VolUp at boot
```

Boots the wrong recovery (or back to the POCO logo)? Your recovery probably
lives on the **other** slot — drop back to fastboot and switch:

```bash
fastboot getvar current-slot
fastboot --set-active=a         # or b — whichever slot has recovery
fastboot reboot recovery
```

In recovery:

1. **Wipe → Format Data** → type `yes` → confirm.
   This is the point of no return — everything on internal storage is erased.
2. Back to the main menu. Do **not** reboot.

## Step 6 — Push files and flash

ADB is live in recovery's main menu. Push everything:

```bash
adb shell mkdir -p /sdcard/flash
adb push flash/crDroidAndroid-*.zip  /sdcard/flash/
adb push flash/NikGapps-*.zip        /sdcard/flash/
adb push flash/fw_alioth_*.zip       /sdcard/flash/
```

Then on the phone, **Install** from `/sdcard/flash/` **in this order**:

1. `crDroidAndroid-*.zip` — the ROM.
2. `NikGapps-*.zip` — GApps (same recovery session, before first boot).
3. `fw_alioth_*.zip` — the firmware, if not already on it.

Then **Wipe cache/dalvik** if offered, and **Reboot → System**.

> Do **not** flash any root zip or Magisk APK here. Root comes from the
> kernel that ships inside crDroid's boot image — there is nothing to patch.

## Step 7 — Root with the built-in KernelSU

crDroid's bundled kernel (`InfiniR_Alioth_..._KSU`) has KernelSU compiled in.
The kernel driver is a **backslashxx fork** — on the test unit it reports
version `32700-6`, i.e. **uapi 6**. The manager APK must speak the same uapi,
or the manager will show "Working" but silently skip the `install()` step
that deploys the `ksud` binary — and nothing will ever get root.

| Manager | Result on this device |
|---|---|
| Official KernelSU `v3.3.1` (tiann) | Shows "Working", but **uapi 5** → reduced single-page UI, no bottom nav, `install()` skipped. Useless for granting root |
| KernelSU-Next (any version) | "KernelSU Next v2 signature not found" — wrong driver family |
| SukiSU Nova | "Not installed" — wrong driver family |
| **backslashxx `KernelSU_v3.3.1-55`** | **"Working 🍉", uapi 6, full UI (Home / Superuser / Module / Settings)** — the correct manager |

Steps:

1. Install the manager:

   ```bash
   adb install flash/KernelSU_v3.3.1-55_*.apk
   ```

2. Open the **KernelSU** app once. It should show **"Working"** with the
   kernel version string and a bottom bar with **Home | Superuser | Module |
   Settings**. If you only see one page with no bottom bar, you have the
   wrong manager — go back to step 1.

3. Grant root to apps you trust. There is **no popup dialog** like Magisk —
   grants are made in advance: **Superuser** tab → find the app → toggle
   **Superuser** on its App Profile page.

4. For root over adb, grant **Shell** (`com.android.shell`, UID 2000):
   Superuser tab → search `shell` → tap Shell → turn on **Superuser**.

5. Verify:

   ```bash
   adb shell /system/bin/su -c id
   # expect: uid=0(root) gid=0(root) context=u:r:ksu:s0
   ```

### How the `su` command works here — read this before looking for a binary

`/system/bin/su` **does not exist as a file** on this setup. The kernel hooks
the `execve` syscall for that exact path and, if the calling UID is on the
KernelSU allowlist, substitutes execution of `/data/adb/ksud`, which spawns
your root shell. Consequences:

- `which su` / `ls /system/bin/su` will say *not found* — that is normal.
- Copying a shell to `/data/local/tmp/su` and running it does **nothing** —
  only the literal path `/system/bin/su` is intercepted.
- Because there is no file, no daemon, and no socket, package-scan and
  filesystem-scan root detection find nothing. That is the point.

`adb root` also does not work (user build) and is unrelated — don't rely on it.

### Root persistence

The allowlist lives in `/data/adb/ksu/` and root survives reboots — verified
on the test unit (`uid=0` immediately after reboot, no manager interaction
needed). If you ever wipe `/data` or reflash the ROM clean, reinstall the
manager and re-grant; `/data/adb/ksud` is redeployed when the manager's
`install()` runs.

## Step 8 — Banking apps: what worked and why

This was the hard part of the whole project, and the reason this guide
recommends kernel-integrated KernelSU over Magisk.

**Failed approach (Magisk):** Magisk 31.1 + Zygisk + DenyList + Zygisk
Assistant hid mounts, maps, and environment variables well enough for *some*
apps — but a Promon-style protected bank app still detected it. Traces that
leak with Magisk even when hiding is configured:

- `magiskd` abstract socket visible in `/proc/net/unix`
- `/debug_ramdisk/.magisk` mount points
- `ro.dalvik.vm.native.bridge=libzygisk.so` readable by any app
- `magisk`/`magisk_file` strings inside the world-readable SELinux policy
- `magisk.db`, patched-boot artifacts, module mounts
- The Magisk manager package itself (uninstalling it helps, but the daemon
  traces remain)

Each of these was individually confirmed readable by an unprivileged app on
the test device. After eliminating every measurable trace, the hardened app
still flagged root — the shield likely probes deeper runtime/kernel state.

**Working approach (KernelSU in-kernel):** the stock crDroid boot image with
the built-in KernelSU driver. No Magisk, no Zygisk, no DenyList, no hiding
modules. Both banking apps — including the one that defeated Magisk — open
normally, and still do after a reboot. The kernel grants root internally;
from an app's perspective the device looks like an ordinary unlocked-but-
unrooted phone.

**Rules to keep it working:**

- **Do not install Magisk.** Ever. It reintroduces every trace above.
- KSU modules are supported (Module tab, Magisk-format module zips), but
  avoid Zygisk-dependent modules — there is no Zygisk in this setup and none
  is needed for the banks to work.
- Grant root sparingly — the default app profile gives full root.
- The `me.weishu.kernelsu` manager package is itself detectable by name; in
  practice the tested bank apps still pass with it installed. If a future app
  gets stricter, you can uninstall the manager — the allowlist and root keep
  working, you just can't change grants until you reinstall it.

If you ever need to go back to fully unrooted: flash the unmodified
`boot_a`/`boot_b` image for your crDroid build — the KernelSU driver then has
no allowlist entries and nothing to grant (though the driver itself remains
compiled into the kernel).

## Step 9 — Restore your data

```bash
for d in Documents Download Books Music Movies YMusic \
         backups SwiftBackup Pic Pic14; do
  adb push "backup/$(basename "$d")/" "/sdcard/$d/"
done
adb shell mkdir -p "/sdcard/Android/media/com.whatsapp/WhatsApp"
adb push "backup/Media/" "/sdcard/Android/media/com.whatsapp/WhatsApp/Media/"
```

> WhatsApp media moved on Android 11+: a fresh install reads
> `Android/media/com.whatsapp/WhatsApp/Media`, not the legacy `/sdcard/WhatsApp/`
> path. If you restore to the old location your photos/videos/files sit there
> but WhatsApp won't see them.

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

Recommended tools (all verified working on this setup; grant each one in the
KernelSU Superuser tab as needed):

| App | Purpose | Source |
|---|---|---|
| AdAway | System-wide adblock (root hosts) | F-Droid |
| AppManager | App ops, trackers, debloat | F-Droid / GitHub |
| AFWall+ | Per-app firewall | F-Droid |
| SmartPack KM | Kernel settings viewer/tweaker | GitHub |
| Shizuku | ADB-level API bridge for tools | GitHub |
| Termux | Terminal (`/system/bin/su` gives root inside it) | F-Droid |
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
adb shell /system/bin/su -c 'fstrim -v /data'

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

## Alternative: PixelOS + Magisk flash path

Kept for reference — this was the first tested path. It works fine as a ROM
install; the only problem is banking-app compatibility (see Step 8).

Files used: `PixelOS_alioth-17.0-20261112-0826.zip` (~3 GB), firmware as
above, `Magisk-v31.1.apk`, and the **~192 MB** `boot-alioth_*.img` from the
PixelOS recovery-images page (`download.pixelos.net/seventeen/alioth/recovery/`).

> **Verify the boot image size before flashing.** Mirror sites (SourceForge in
> particular) have served **truncated** boot images for this device — ~23 MB
> instead of ~192 MB. Magisk then fails with `Unable to unpack the boot image`
> and the recovery may refuse to boot. The real `boot-alioth_20261112_0826.img`
> is **201,326,592 bytes**. Check `head -c 8 boot-*.img` → `ANDROID!`.

- Recovery: the official PixelOS recovery sat at the POCO logo forever on the
  test unit (both slots, even with `dtbo` flashed). Use TWRP instead.
- Do **not** flash `Magisk.zip` in TWRP — it errors with
  `Cannot mount /system` on this A/B device. Instead: boot the ROM, install
  the Magisk APK, **Install → Select and Patch a File** on the stock
  `boot-alioth_*.img`, then `fastboot flash boot_<active slot>` the
  `magisk_patched-*.img` from `Download/`.
- Root check after boot: `adb shell su -c id` → `uid=0` once you've approved
  the prompt in the Magisk app.
- TWRP flashes A/B ROMs to the **inactive** slot — reboot into that slot, and
  flash the patched boot image to the slot the system is actually running on.

## Troubleshooting — things that actually happened

| Symptom | Cause | Fix |
|---|---|---|
| `adb devices` → `offline` | Pending USB-debugging auth | Unlock phone, accept dialog, replug cable |
| `fastboot` sees nothing, phone at POCO logo | Hung boot | Hold **Power+VolDown** 10–15 s → fastboot |
| `fastboot devices` lists serial but commands hang | Wedged fastboot state | Replug cable / new port, hold Power 10 s, re-enter fastboot |
| `Fetching super ... FAILED ... 'unknown command'` | Harmless fastbootd quirk | Ignore — check the actual `Writing ... OKAY` lines |
| TWRP: `Cannot mount /system`, zip error 1 | TWRP can't flash Magisk on A/B | Patch boot.img in the Magisk app instead |
| Magisk: `Unable to unpack the boot image` | Truncated boot.img (~23 MB vs 192 MB) | Re-download from the official PixelOS host |
| Phone boots to logo after flash | Wrong slot | `fastboot getvar current-slot`, flash matching `boot_a`/`boot_b` |
| Git Bash: `failed to stat remote object 'C:/Program Files/Git/sdcard'` | MSYS path mangling | `export MSYS_NO_PATHCONV=1` |
| KSU manager shows "Working" but no bottom nav / can't grant | Manager uapi older than driver | Use **backslashxx KernelSU v3.3.1-55** (uapi 6), not tiann v3.3.1 (uapi 5) |
| KSU-Next: "v2 signature not found" | Driver isn't KernelSU-Next | Use the backslashxx manager |
| SukiSU Nova: "Not installed" | Driver isn't SukiSU | Use the backslashxx manager |
| `su` → file not found | `su` isn't a file on this setup | Run `/system/bin/su` literally — the kernel intercepts the path |
| `su` runs but still `uid=2000` | Shell not allowlisted | KernelSU app → Superuser → Shell → toggle on |
| `adb root` refused | User build; unrelated to KernelSU | Grant Shell in the manager, use `/system/bin/su` |
| Bank app screenshot is black | `FLAG_SECURE`, not a crash | Judge by the physical screen, not `screencap` |
| Hardened bank app still detects Magisk | Userspace traces (sockets, props, mounts) | KernelSU path — see Step 8 |
| No "load config" in LMC8.4 | It's a hidden gesture | Double-tap next to the shutter button |

## Sanity checklist before you wipe

- [ ] `backup/` on the PC contains everything you care about (open it and look)
- [ ] SMS/call log exported if wanted
- [ ] Swift Backup run + pulled (if you want app data)
- [ ] 2FA/authenticator migrated or codes saved — these do **not** come back
- [ ] WhatsApp Google Drive backup current (if chats matter)
- [ ] All flash files downloaded (ROM, GApps, firmware, KernelSU manager)
- [ ] Battery above ~50%
- [ ] You know the Power+VolDown forced-fastboot combo

## Warnings

- Everything here requires an unlocked bootloader, voids warranty, and can
  brick the device if you flash images meant for another model. Only files
  built for `alioth` belong anywhere near this phone.
- Flashing the wrong slot or an image for a different device is the main way
  people brick these phones. When in doubt, `fastboot getvar` everything.
- KernelSU grants are silent and total — only allowlist apps you trust.
- Keep your PC-side backup until you've lived on the new ROM for a week or two.
- Third-party APK sources carry real risk. Prefer F-Droid, official GitHub
  releases, and the ROM's own site over random mirrors.
- Root-hiding is a moving target. The configuration here is what was verified
  on this device on this date — a bank-app update or ROM update can change it.

## License

MIT — see [LICENSE](LICENSE). Device names and ROMs belong to their respective
projects; this guide is an independent walkthrough with no affiliation.
