# Install on the Pixel 6 (oriole) from a release

Current release: **[oriole-23.2-20261119-build11](https://github.com/gramarye-os/aohp-lineage/releases/tag/oriole-23.2-20261119-build11)**, the first one signed
with our own keys. [Signed builds](#signed-builds-2026-11-19-and-later) says what that changes.

This is the sequence that was actually run on 2026-10-04 with
[build 3](https://github.com/gramarye-os/aohp-lineage/releases/tag/oriole-23.2-20261004-build3) (two attempts, logs `REPORT-flash-1/2.md` on chex), following
the [LineageOS install wiki for oriole](https://wiki.lineageos.org/devices/oriole/install/) with the deviations noted.
File names and version strings below are from build 3: use the ones of the release you downloaded.

On 2026-11-19 the signed build went onto a phone that already ran one of our releases. Only the recovery part was done
that day (format data, Apply from ADB, sideload: 2 min 53 s, first boot 54 s later); the fastboot steps were not repeated.

## Prerequisites

- **Unlocked bootloader** (*Settings → Developer options → OEM unlocking*, then `fastboot flashing unlock` — wipes the
  phone; the wiki has the details). The test phone had been unlocked long before; `ro.boot.verifiedbootstate=orange`,
  `flash.locked=0`.
- **Stock Android 16 firmware level**: LineageOS 23.2 requires the vendor/bootloader/radio from an Android 16 stock
  build (the wiki lists the minimum). The test phone had vendor fingerprint `google/oriole/oriole:16/BP4A.251205.006/…`,
  bootloader `slider-17.0-15681480` (slot b reported `slider-16.4-14097577` — both fine), baseband
  `g5123b-170612-250829-B-14077263`. If needed, flash the matching
  [factory image](https://developers.google.com/android/images#oriole) first (`flash-all.sh`, or just bootloader +
  radio), boot it once.
- Host: `adb` and `fastboot` (platform-tools 35+; the Lineage tree's `out/host/linux-x86/bin/fastboot` 36.0.1 was used),
  USB cable. All commands below used `-s 1A071FDF6008UR` (the phone's serial) — drop it if only one device is attached.
- Download `lineage-23.2-20261004-UNOFFICIAL-oriole.zip`, `boot.img`, `dtbo.img`, `vendor_boot.img`, `SHA256SUMS`
  and verify: `sha256sum -c SHA256SUMS` (ignore the missing `vbmeta.img` line if you did not download it).

**Everything on the phone is wiped** (the factory-reset step is mandatory when coming from stock or another ROM).

## Flash

```bash
# 1. to the bootloader
adb -s 1A071FDF6008UR reboot bootloader
fastboot -s 1A071FDF6008UR devices            # phone listed; 'fastboot getvar current-slot' shows a or b

# 2. boot images for the current slot (the wiki's order; the reboot between dtbo and vendor_boot is required)
fastboot -s 1A071FDF6008UR flash boot boot.img
fastboot -s 1A071FDF6008UR flash dtbo dtbo.img
fastboot -s 1A071FDF6008UR reboot bootloader
fastboot -s 1A071FDF6008UR flash vendor_boot vendor_boot.img

# 3. factory reset — done from the bootloader instead of the recovery menu (no key presses needed)
fastboot -s 1A071FDF6008UR -w                 # erases userdata + metadata

# 4. into Lineage recovery (it lives in the vendor_boot you just flashed)
fastboot -s 1A071FDF6008UR reboot recovery
```

The phone shows the LineageOS recovery. From here two paths:

**Headless (builds before 2026-10-06 only, build 5 and older; their recovery accepts adb without authorization):**
```bash
adb -s 1A071FDF6008UR devices                 # "recovery" (not "unauthorized")
adb -s 1A071FDF6008UR reboot sideload         # state becomes "sideload"
adb -s 1A071FDF6008UR sideload lineage-23.2-20261004-UNOFFICIAL-oriole.zip
```
**On the phone (the way since build 6, whose recovery shows the host as `unauthorized`):**
on the recovery screen pick **Apply update → Apply from ADB** (volume keys to move, power to select), then run the
`adb sideload …` line above. `adb devices` shows `unauthorized` or `sideload`; the sideload works either way.

The host's progress counter stops around **47 %** and then prints `Total xfer: 1.00x` — that is success (the second
half is the on-device install). Took ~3 min. The recovery screen says the install is done; then:

Pick **Reboot system now** on the phone. (`adb -s 1A071FDF6008UR reboot` also works on builds before 2026-10-06.)
Do not reboot before the sideload: the phone would boot the old system on the freshly reset data.

## First boot

- Boots to the other slot (A/B OTA; the payload carries boot/dtbo/vendor_boot/vbmeta for that slot, so the images you
  flashed by hand are overwritten with identical copies). `sys.boot_completed=1` ~45 s after the reboot.
- The bootloader shows the usual **orange state / "Your device software can't be checked for corruption"** warning for
  5 s on every boot (unlocked bootloader) — expected.
- **adb is on immediately** (`persist.sys.usb.config=adb`). Since build 6 it is key-authorized (`ro.adb.secure=1`): our
  build host is let in by a key in the image, any other host gets the "Allow USB debugging?" dialog.
  Check: `adb -s 1A071FDF6008UR shell getprop ro.lineage.version` → `23.2-20261004-UNOFFICIAL-oriole`,
  `getenforce` → `Enforcing`, `pm list packages | grep -E 'aohp|openclaw'` → `org.aohp.driver`, `ai.openclaw.app`,
  `ps -A | grep aohp-containerd` running as root, and `ss -ltnp` shows `127.0.0.1:6666` owned by `org.aohp.driver`
  (the Driver's bridge comes up at boot without opening the app).
- The Lineage setup wizard runs on the screen (language, Wi-Fi, lock screen). No GApps are included; the vendor
  `com.google.euiccpixel` crash-loops in logcat without gservices (harmless Lineage-without-GApps noise).
- `adb root` says *"ADB Root access is disabled by system setting"* until *Developer options → Rooted debugging* is
  enabled. Not needed for the agent container (`aohp sandbox exec` is root inside it).

Continue with [provisioning.md](provisioning.md).

## Signed builds (2026-11-19 and later)

From build 11 on, the releases are signed with our own keys and are marked `release-keys`. Earlier releases used the
public AOSP test keys.

- **Coming from an earlier release of ours** (build 10 or older): the data has to be wiped, because a phone only
  accepts updates signed with the keys of the build it runs. The phone already has a Lineage recovery, so the fastboot
  steps can be skipped: reboot to recovery, **Factory reset → Format data / factory reset**, then **Apply update →
  Apply from ADB**, `adb sideload` the new zip, **Reboot system now**. Do not skip the format: without it the phone
  boots, but keeps data and app updates made under the old keys.
- **adb** is as on every build since build 6: key-authorized in the booted system, with the build host's key in the
  image (other hosts get the usual dialog), and `unauthorized` in the recovery.

## Updating to a later build

On a signed build, later builds arrive through **Settings → System → Updater**. It reads the list of builds from
`https://gramarye.mcast.network`, installs the new one to the other slot in the background and asks for a reboot. Data
and the AOHP environments are kept. This was tested on the OnePlus 13 on 2026-11-19, not yet on a Pixel 6.

## Deviations from the wiki (summary)

- Factory reset via `fastboot -w` from the bootloader instead of *Factory reset → Format data* in the recovery menu.
- Builds before 2026-10-06: `adb reboot sideload` instead of the menu's *Apply from ADB* (their recovery had
  `ro.adb.secure=0`). Since build 6 the recovery shows the host as `unauthorized`, so the menu is the way in.
- No GApps, no `vbmeta --disable-verification` step (vbmeta.img is in the release for completeness; the install
  did not flash it).

## Back to stock

Google factory images for oriole: https://developers.google.com/android/images#oriole — unzip, `flash-all.sh`
(includes `-w`). Re-lock only after stock boots (`fastboot flashing lock`), if at all.
