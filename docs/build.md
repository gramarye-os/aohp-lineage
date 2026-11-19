# Building LineageOS 23.2 + AOHP from source

Everything below was run on `chex` (Fedora, 32 threads, 123 GB RAM) on 2026-10-03/04; the commands are the ones from
the build scripts, not a transcription of the wiki. Disk: ~65 GB `.repo`, ~91 GB working tree, 43 GB `out/` for dodge
(incl. the in-tree kernel), ~15 GB for oriole; plus the kernel tree `out-kernel/` for oriole (22 GB).

Since 2026-11-04 the release builds are made on a second host (`gru`) by a GitHub Actions workflow that runs the same
steps: sync, `fetch-prebuilts.sh`, `breakfast`, `m bacon`, then it stages the artifacts and publishes the release.
Since 2026-11-19 those builds are signed with our own keys, see [Signing](#signing).

## 1. Sync the tree

```bash
mkdir -p ~/lineage/android && cd ~/lineage/android
repo init -u https://github.com/LineageOS/android.git -b lineage-23.2 --git-lfs --no-clone-bundle --partial-clone --clone-filter=blob:limit=10M
mkdir -p .repo/local_manifests
curl -fsSL https://raw.githubusercontent.com/gramarye-os/local_manifests/lineage-23.2-aohp/aohp.xml -o .repo/local_manifests/aohp.xml
repo sync -c -j8 --no-tags --no-clone-bundle --optimized-fetch --prune --retry-fetches=3
```

- The partial-clone flags work with Lineage's manifest (61 min for the base tree on a fast link; the 5.5 GB
  `android_frameworks_base` pack dominates). The dodge additions synced in 6 min.
- `vendor/aohp` is a **private** repo in the gramarye-os organisation: `repo sync` needs a GitHub credential with access
  to it (`gh auth login` + `gh auth setup-git`, or a `~/.git-credentials` entry). Everything else is public.
- The manifest replaces a handful of LineageOS projects (among them `build/make`, `system/core`, `system/sepolicy`,
  `frameworks/base`) with our `lineage-23.2-aohp` branches and adds the Pixel 6, OnePlus 13 and Solana Saga device trees,
  kernels and vendor repos. Its remote is still named `injinj`; it points at `github.com/gramarye-os`. No `roomservice`
  additions are needed (`breakfast` found everything present).

## 2. Fetch the prebuilts that are not in git

```bash
vendor/aohp/fetch-prebuilts.sh            # Debian template + AOHPDriver.apk + OpenClawAndroid.apk + Tessera + F-Droid + stock prebuilts, all sha256-verified; the pinned versions are at the top of the script
# vendor/aohp/fetch-prebuilts.sh debian-gpu apks stock fdroid   # what the dodge and ingot release builds use: the Debian template with Mesa for Adreno GPUs
# vendor/aohp/fetch-prebuilts.sh fedora arch   # optional extra templates; aohp.mk enables them when the file exists
```

This writes `packages/apps/AOHPAgentDriver/rootfs/debian.tar.gz` (from the newest
[aohp-agents `templates-*` release](https://github.com/gramarye-os/aohp-agents/releases)),
`packages/apps/AOHPDriver/{Android.bp,AOHPDriver.apk,initial-package-stopped-states-aohp.xml}` and
`packages/apps/OpenClawAndroid/{Android.bp,OpenClawAndroid.apk}` (from the
[aohp-driver](https://github.com/gramarye-os/aohp-driver/releases) and
[openclaw](https://github.com/injinj/openclaw/releases/tag/android-aohp-2026.8.2) releases pinned at the top of the
script). The two `packages/apps/*` dirs are not repo projects, so the script creates them.

The `stock` part fetches `packages/apps/AOHPAgentDriver/AOHPAgentDriver.apk` and `rootfs/alpine.tar.gz` from the
[gramarye-os/AOHPAgentDriver `prebuilts-20261005` release](https://github.com/gramarye-os/AOHPAgentDriver/releases/tag/prebuilts-20261005).
Neither is installed on the phones (only `aohp-rootfs-debian` is in `PRODUCT_PACKAGES`; `AOHPAgentDriver` /
`aohp-rootfs-alpine` belong to the Cuttlefish product), but Soong resolves every module's `apk`/`src` at analysis time,
so a checkout without the files fails at `m`. Every fetched file is sha256-pinned (templates/stock via the release's
`SHA256SUMS`, APKs via variables at the top of the script); a differing existing file is kept as `<file>.bak-<date>`.
Tags recorded in `rootfs/.templates-release`, `rootfs/.stock-release`, `AOHPDriver/.apks-release`.

## 3. Build

```bash
cd ~/lineage/android
source build/envsetup.sh

# Pixel 6
breakfast oriole                   # = lunch lineage_oriole-bp4a-userdebug + build_kernel: repo-inits/syncs
                                   #   android_kernel_google_gs-6.1_manifest (lineage-23.2) into out-kernel/google/gs-6.1
                                   #   and runs the Kleaf build_raviole.sh (~10 min; the GKI core is prebuilt there)
m -j10 bacon                       # -> out/target/product/oriole/lineage-23.2-<date>-UNOFFICIAL-oriole.zip, boot.img, dtbo.img, vendor_boot.img, vbmeta.img

# OnePlus 13
breakfast dodge                    # lineage_dodge-bp4a-userdebug; kernel 6.6 built in-tree (INLINE_KERNEL_BUILDING,
                                   #   kernel/oneplus/sm8750 + ~50 TARGET_KERNEL_EXT_MODULES from sm8750-modules)
m -j10 bacon                       # -> out/target/product/dodge/lineage-23.2-<date>-UNOFFICIAL-dodge.zip, boot/dtbo/init_boot/vbmeta/vendor_boot/recovery.img
```

`brunch oriole` / `brunch dodge` is the one-step equivalent of `breakfast` + `m bacon`.

**adb.** Since oriole build 6 / dodge build 4 (2026-10-06) this is a normal Lineage userdebug build: `ro.adb.secure=1`,
`ro.debuggable=0`. `vendor/aohp/aohp.mk` turns adb on at boot (`persist.sys.usb.config=adb`) and puts the adb public
key(s) in `vendor/aohp/adb_keys` into the image (`PRODUCT_ADB_KEYS`), so those hosts get adb from first boot without the
dialog and every other host gets the usual authorization prompt. The file holds our build host's key: **replace it
with your own `~/.android/adbkey.pub` before building.** The Lineage recovery of these builds shows the host as
`unauthorized`; a sideload is started from its menu (see the install pages and [known-issues.md](known-issues.md)).
`adb root` additionally needs *Developer options → Rooted debugging* on the phone (Lineage's `adbroot_service`), which
is not needed for AOHP work (`aohp sandbox exec` is root inside the env).

The first builds (oriole ≤ build 5, dodge ≤ build 3) were made with Lineage's `WITH_ADB_INSECURE=true` instead
(`ro.adb.secure=0`: no authorization at all, also in the recovery). That knob has to be in the environment before
`breakfast`; it is no longer used.

### Build times on chex (`-j10`, systemd `CPUQuota=1000%`, no ccache)

| | time | notes |
|---|---|---|
| oriole, cold | 2 h 31 min to the sepolicy failure at 92 % (first attempt), then 11 min incremental | 219k ninja actions |
| oriole, prop change (build 3) | 5 min | |
| dodge, cold incl. kernel + DLKMs | **1 h 53 min** | 133k actions, peak 49.6 GB RAM, `KERNEL_OBJ` 2.5 GB |
| `m -j10 systemimage` after an app/template change | 1–2 min | |

Run long builds detached (`systemd-run --user --unit=lineage-build --collect -p CPUQuota=1000% -p Nice=10 bash -c '… > build.log 2>&1'`)
and poll the log; a terminal session that dies takes the build with it.

### Things that bit

- **`sepolicy_freeze_test`**: Lineage 23.2 builds with `TARGET_RELEASE=bp4a` and a frozen vendor API, so any new type in
  `system/sepolicy/public/` fails the build ("The following public types were added: aohp_container_socket"). The AOHP
  GSI used `trunk_staging` where the test is a no-op. Fix in the sepolicy branch: `aohp_container_socket` lives in
  `private/file.te`. No other sepolicy change was needed for either phone.
- **`policycap functionfs_seclabel`**: the OnePlus GSI needed it dropped (its stock 6.6 kernel rejected it). Lineage's
  own `lineage-23.2` sepolicy carries the policycap and its dodge kernel (6.6.142, built here) accepts it — so the patch
  is intentionally **not** applied; verified by the dodge first boot (enforcing, no init loop).
- **`manifest_check`** fails on any `<uses-library>` mismatch between an imported APK and its `Android.bp`: the Driver
  declares none (→ no `optional_uses_libs`); the OpenClaw app declares `androidx.window.extensions` /
  `androidx.window.sidecar` as not-required (→ listed in `optional_uses_libs`). Check with
  `aapt dump badging <apk> | grep uses-library` after any APK bump.
- A new system app is *stopped* until first launch and gets no `BOOT_COMPLETED`; the Driver is exempted by
  `initial-package-stopped-states-aohp.xml` (sysconfig). Verified with a fresh secondary user.
- Soong resolves `prebuilt_etc.src` for every module at analysis time, so an optional template module whose tarball is
  absent breaks *every* build — the fedora/arch modules are `soong_config`-gated and disabled unless `aohp.mk` sees the file.
- The Driver is signed with the platform key of the build. Without your own keys that is Lineage's default dev key
  (`build/make/target/product/security/platform.*`, byte-identical to the AOHP tree's); with
  `vendor/lineage-priv/keys` it is yours. See [Signing](#signing).

### Signing

A tree without `vendor/lineage-priv/keys` builds with the public AOSP test keys: the result is marked `test-keys`, and a
phone running it accepts any update signed with those public keys. Our releases since 2026-11-19 are signed with our own
keys instead. LineageOS has this built in:

```bash
cp -r lineage/scripts/lineage-priv-template vendor/lineage-priv/keys
cd vendor/lineage-priv/keys && ./keys.sh        # 89 key pairs, no passwords; edit the subject in make_key.sh first if you want your own name in them
```

`vendor/lineage/config/common.mk` includes `vendor/lineage-priv/keys/keys.mk` when it exists. A normal `m bacon` then
signs the apps, the APEXes and the OTA zip with these keys and marks the build `release-keys`; the platform-signed
Driver gets the new platform key with no change in `vendor/aohp`.

- Keep the directory out of git and keep a backup. A phone only accepts updates signed with the keys of the build it
  runs, so losing the keys means a factory reset for every phone on those builds.
- For the same reason a phone cannot move between a test-key build, one of our releases and a build signed with your
  own keys without a factory reset.
- Our keys are not published. If you sign your own builds, change `lineage.updater.uri` in `vendor/aohp/aohp.mk` as
  well: it points the Updater at our server, and the builds there are signed with our keys, not yours.

## 4. Verify the artifacts

`sha256sum` the zip and images and keep the list (the releases' `SHA256SUMS` were produced this way). Useful checks on
the zip: `unzip -p <zip> META-INF/com/android/metadata` (`ota-type=AB`, `pre-device`, `post-build`), and
`unzip -l <zip> | grep -E 'aohp|AOHPDriver|OpenClawAndroid'` is not possible for a payload OTA — inspect
`out/target/product/<device>/system/` instead (`system/priv-app/AOHPDriver`, `system/app/OpenClawAndroid`,
`system/bin/aohp-containerd`, `system/etc/aohp/`, `system/etc/sysconfig/initial-package-stopped-states-aohp.xml`),
`system_ext/etc/build.prop` for `ro.adb.secure=1`, `system/build.prop` for `ro.build.tags` (`test-keys` or
`release-keys`) and `product/etc/build.prop` for `lineage.updater.uri`.

The out directory collects one `lineage-23.2-<date>-UNOFFICIAL-<device>.zip` name per build day, and all of them are
hard links to the newest zip. Take the name that matches `ro.lineage.version` in `product/etc/build.prop`.

## 5. Publishing a build

- Zip ≤ 2 GiB (oriole, 1.98 GB): upload as-is. Zip > 2 GiB (dodge, 3.76 GB): `split -b 1900M -d -a 1 <zip> <zip>.part`
  and publish the parts + a `SHA256SUMS` that lists the whole zip and each part (see the dodge release).
- `gh release create <tag> --repo gramarye-os/aohp-lineage --target main --notes-file notes.md SHA256SUMS *.img <zip or parts>`,
  detached (a 3.8 GB upload takes a while); verify with `gh release view <tag> --json assets` that the sizes match.

### Offering a build through the Updater

The images set `lineage.updater.uri` to `https://gramarye.mcast.network/api/v2/devices/{device}/builds`, so the stock
LineageOS Updater asks that URL for a list of builds. What the server has to do:

- Answer over HTTPS, and serve the list itself without a redirect (the Updater does not follow one for the list).
- Support range requests for the zip: the Updater streams the payload straight out of it.
- Return a JSON array with one entry per build:

  ```json
  [{
    "datetime": 1795113695,
    "type": "UNOFFICIAL",
    "version": "23.2",
    "files": [{
      "filename": "lineage-23.2-20261119-UNOFFICIAL-dodge.zip",
      "url": "https://gramarye.mcast.network/dodge/<build>/lineage-23.2-20261119-UNOFFICIAL-dodge.zip",
      "size": 3749884583,
      "sha256": "95aee9e2d42ea1e07c5a31eda2afb4884947e38720198754c36ca8ea84e18841",
      "os_patch_level": "2026-11-01",
      "os_sdk_level": 36,
      "ota_property_files": "<the ota-property-files line of META-INF/com/android/metadata in the zip>"
    }]
  }]
  ```

  `datetime` is `post-timestamp` from the same metadata file (= `ro.build.date.utc` of the build).

A phone is offered a build when its `datetime` is newer than the installed build and `type` and `version` match
`ro.lineage.releasetype` and `ro.lineage.build.version`. It installs it only if the zip is signed with the keys of the
build it runs. Our server is a small static file server on the build host; the list is rewritten by the build pipeline
when a build is to be offered.
