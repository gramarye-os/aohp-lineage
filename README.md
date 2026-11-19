# LineageOS 23.2 + AOHP

**LineageOS 23.2 (Android 16) with the [AOHP](https://github.com/aohp-os/aohp) agent harness built in**, for the
**Pixel 6 (`oriole`)**, the **OnePlus 13 (`dodge`, CPH2655)** and the **Solana Saga (`ingot`)**. The ROM ships `aohp-containerd` + the AOHP framework
services, the **AOHP Driver** app (host console + agent bridge), the **OpenClaw Android** app (chat client) and a Debian
agent-container template, so a phone runs an OpenClaw gateway *on the device* inside a container, and the OpenClaw app
pairs with it over loopback. The first builds were flashed and verified on real phones on 2026-10-04 (SELinux enforcing,
gateway live, app paired, test message answered). The current releases, of 2026-11-19, were installed on all
three phones that day.

This repository is the **documentation + ROM release host**. Code lives in the repos listed below.

## What runs inside what

![LineageOS + AOHP: phone → ROM → aohp-containerd → Debian env → OpenClaw gateway; the OpenClaw app and the AOHP Driver app talk to it over loopback](docs/architecture.svg)

Read it bottom-up: a stock Lineage kernel; the Lineage/Android 16 system with AOHP's framework services patched in;
`aohp-containerd`, a root daemon that chroots a Debian rootfs under `/data/aohp/envs/<name>` and supervises its units; and
inside that env the OpenClaw gateway on `127.0.0.1:18789`. Two ordinary Android apps sit on top: the **AOHP Driver** (the
console that creates envs, holds the API key in the Keystore and hosts the `ws://127.0.0.1:6666` bridge the `aohp` CLI
talks to) and the **OpenClaw Android** app (the chat client, paired to the on-phone gateway). Everything is loopback;
a dev host is only needed for flashing and optional headless provisioning over `adb forward`.

### Why a ROM at all?

The same gateway could run on a stock phone with root, or inside a PRoot "Linux in an app" (the only route that can
ship in the Play Store), or be rewritten for the platform. Each gives something up: root breaks attestation and
sepolicy; PRoot puts an app-uid process under ptrace, Doze and the low-memory killer; a rewrite stops being OpenClaw
the day upstream moves. A ROM is the one option where the gateway is a real, supervised, SELinux-confined system
service and upstream merges land unchanged — at the price of a flash. The trade-offs, and how this differs from the
app-operating agents Google, Honor and Xiaomi ship, are in [docs/linux-tool-use.md](docs/linux-tool-use.md).

| | |
|---|---|
| **ROM releases** | [Releases page](https://github.com/gramarye-os/aohp-lineage/releases) - tags are `oriole-<lineage>-<date>-build<N>` (Pixel 6) and `dodge-<lineage>-<date>-build<N>` (OnePlus 13) and `ingot-<lineage>-<date>-build<N>` (Solana Saga); the newest tag per device is the current build, and each release note says what changed and which build it supersedes. |
| Build from source | [docs/build.md](docs/build.md) |
| Install from a release | [docs/install-oriole.md](docs/install-oriole.md) · [docs/install-dodge.md](docs/install-dodge.md) · [docs/install-ingot.md](docs/install-ingot.md) |
| First boot, provisioning the agent container, pairing the app | [docs/provisioning.md](docs/provisioning.md) |
| Units: systemd-style services, timers, `systemctl` shim | [docs/units.md](docs/units.md) |
| Known issues | [docs/known-issues.md](docs/known-issues.md) |

> These are **development builds**: `userdebug`. Since oriole build-6 / dodge build-4 (2026-10-06) adb is key-authorized
> (`ro.adb.secure=1`; the build host's key is baked in, other hosts get the RSA prompt) and `adb root` works on request
> (userdebug adbd; Lineage's *Rooted debugging* toggle keeps it root across boots). Earlier builds used `WITH_ADB_INSECURE=true`
> (no authorization at all, also in recovery). Do not use them on a phone that
> holds anything you care about. The dodge zip contains OnePlus proprietary vendor blobs (TheMuppets).
>
> **Signed builds:** since 2026-11-19 (oriole build 11, dodge build 26, ingot build 5) the releases are signed with our
> own keys (`release-keys`) instead of the public AOSP test keys, and later builds arrive through *Settings → System →
> Updater*. A phone on an earlier release has to be wiped once to move over; the install pages say how.

## What is in the ROM

| component | where | from |
|---|---|---|
| AOHP framework: `aohp_container`, `aohp_virtual_display`, `aohp_agent_view`, `aohp_event_stream`, `aohp_file_bridge`, `aohp_security_bridge`, `aohp_ad`, `aohp_vault`, `aohp_taint`, … services; `ro.aohp.virtual_display_policy=true` | system_server, product build.prop | frameworks/base + build/make patches (aohp-os upstream squash + injinj commits) |
| `aohp-containerd` (chroot container runtime, host netns; since build-5/build-3 also the **unit manager**: systemd-subset units from the container's `/etc/aohp/system`, restart policy, ordering, timers, `UNIT` socket op; since dodge build-8/15 also **`HostExec=` units** that run on the Android side, [docs/units.md](docs/units.md)) + `aohp-containerd.rc`, `/system/etc/aohp/cgroup.conf` | `/system/bin`, `/system/etc/init` | system/core patches (`aohp-containerd/`) |
| sepolicy: `aohp_container_daemon`, `aohp_container_socket` (private policy, see build notes), `aohp_agent_app.te`; since build-4/build-2: `netlink_audit_socket` (sshd/PAM/sudo), `proc_*` reads, io_uring, FICLONE/FS_IOC ioctls, `sysfs_dm` reads; since dodge build-15: GPU (`gpu_device`, dmabuf heap) for Mesa in the env and the `app_process` + Binder/fd rules that let a `HostExec=` unit run the Termux:X11 server — [docs/gpu.md](docs/gpu.md), GPU-composited Vulkan via patched gamescope: [docs/gamescope.md](docs/gamescope.md) | plat_sepolicy.cil | system/sepolicy patches |
| **AOHP Driver** `org.aohp.driver` 0.9.0 — Runtime / Harness / Terminal / Web / Keys tabs, hosts the agent bridge (`ws://127.0.0.1:6666` JSON-RPC for the `aohp` CLI, incl. `sandbox.unit`, `appfn.*` for Android AppFunctions and `webapp.*` for pinned web apps), Keystore-backed secrets, **Units card** per env, boot autostart = **env-start** (enabled units in dependency order; 0.3.0-style replay of recorded services for envs without units), first-run wizard | `/system/priv-app/AOHPDriver`, platform-signed, privileged since 0.7.0 (allow-list `privapp-permissions-aohp-driver.xml`) | [gramarye-os/aohp-driver](https://github.com/gramarye-os/aohp-driver) — [release v0.9.0-cred.26](https://github.com/gramarye-os/aohp-driver/releases/tag/v0.9.0-cred.26) (built by GitHub Actions) |
| **OpenClaw Android** `ai.openclaw.app` 2026.8.2 (thirdParty release) | `/system/app/OpenClawAndroid`, presigned with the injinj `aohp-apps` key | [injinj/openclaw release android-aohp-2026.8.2](https://github.com/injinj/openclaw/releases/tag/android-aohp-2026.8.2) |
| Debian trixie agent template (Node 24, OpenClaw, `aohp` CLI with `unit`/`timer`, gcc, python, gh, age, aohp-agents layer; wireguard-tools + openssh-server installed but inactive, `/opt/aohp-agents/net/` kit; since templates-20261005c: unit files in `/etc/aohp/system` (`openclaw-gateway.service` enabled) + `systemctl`/`journalctl` shims) | `/system/etc/aohp/rootfs-templates/debian.tar.gz` | [gramarye-os/aohp-agents templates-20261119a](https://github.com/gramarye-os/aohp-agents/releases/tag/templates-20261119a) (Dockerfiles in `template/`). dodge and ingot carry the `debian-gpu` variant (GL and X client libraries, Mesa for Adreno GPUs), oriole the lean one |
| **Tessera** `com.daryan.tessera` (launcher for Linux GUI apps running in the env) | installed into `/data` at first boot from `/system/etc/aohp/preinstall/` (it cannot be a `/system` app), presigned with the `aohp-apps` key | [gramarye-os/tessera-linux release v0.7.0-gramarye.3](https://github.com/gramarye-os/tessera-linux/releases/tag/v0.7.0-gramarye.3) |
| **F-Droid** 1.23.2 + Privileged Extension 0.2.13 (installs and updates without a prompt per app) | `/system/app/FDroid`, `/system/priv-app/FDroidPrivilegedExtension`, presigned by F-Droid | f-droid.org, pinned by version and sha256 in `vendor/aohp/fetch-prebuilts.sh` |
| Update channel: `lineage.updater.uri` points the stock LineageOS Updater at `https://gramarye.mcast.network` (since 2026-11-19) | product build.prop | `vendor/aohp/aohp.mk` |
| `initial-package-stopped-states-aohp.xml` — the Driver is not "stopped" on first boot, so it receives `BOOT_COMPLETED` and the bridge + gateway come up before anyone opens the app | `/system/etc/sysconfig` | aohp-driver `aosp/` |

## Repository map

All platform/device branches are called `lineage-23.2-aohp` = LineageOS `lineage-23.2` + the aohp-os upstream squash + injinj commits. The table lists the main repos; `aohp.xml` has all of them (kernels, vendor blobs, a few patched apps).

| repo | role |
|---|---|
| [gramarye-os/local_manifests](https://github.com/gramarye-os/local_manifests/tree/lineage-23.2-aohp) (`aohp.xml`, `README-lineage.md`) | the local manifest that turns a stock `lineage-23.2` `repo init` into this tree |
| [gramarye-os/android_build](https://github.com/gramarye-os/android_build/tree/lineage-23.2-aohp) | build/make — AOHP product bits (replaces `LineageOS/android_build`) |
| [gramarye-os/platform_system_core](https://github.com/gramarye-os/platform_system_core/tree/lineage-23.2-aohp) | system/core — `aohp-containerd` (branch on the existing aohp-os fork: GitHub allows one fork per network) |
| [gramarye-os/android_system_sepolicy](https://github.com/gramarye-os/android_system_sepolicy/tree/lineage-23.2-aohp) | system/sepolicy — AOHP policy; `aohp_container_socket` moved to private policy for `sepolicy_freeze_test` |
| [gramarye-os/android_frameworks_base](https://github.com/gramarye-os/android_frameworks_base/tree/lineage-23.2-aohp) | frameworks/base — AOHP services, AIDL in `core/java/com/android/internal/aohp/` |
| [gramarye-os/android_device_google_raviole](https://github.com/gramarye-os/android_device_google_raviole/tree/lineage-23.2-aohp) | Pixel 6 family: `lineage_oriole.mk` inherits `vendor/aohp/aohp.mk` (+3 lines) |
| [gramarye-os/android_device_oneplus_dodge](https://github.com/gramarye-os/android_device_oneplus_dodge/tree/lineage-23.2-aohp) | OnePlus 13: `lineage_dodge.mk` inherits `vendor/aohp/aohp.mk` (+3 lines) |
| [gramarye-os/android_device_osom_ingot](https://github.com/gramarye-os/android_device_osom_ingot/tree/lineage-23.2-aohp) | Solana Saga: `lineage_ingot.mk` inherits `vendor/aohp/aohp.mk`, plus kernel config additions the container needs |
| [gramarye-os/android_vendor_aohp](https://github.com/gramarye-os/android_vendor_aohp) (private) | `vendor/aohp/aohp.mk` (PRODUCT_PACKAGES, allow-list, props) and `fetch-prebuilts.sh` (templates + APKs, sha256-pinned) |
| [gramarye-os/AOHPAgentDriver](https://github.com/gramarye-os/AOHPAgentDriver) (`injinj-main`) | `packages/apps/AOHPAgentDriver`: Android.bp for the `aohp-rootfs-*` template modules (the stock demo app itself is **not** shipped) |
| [gramarye-os/aohp-driver](https://github.com/gramarye-os/aohp-driver) | the AOHP Driver app (Kotlin/Compose), its `aosp/` Android.bp files, `.github/workflows/release.yml` |
| [gramarye-os/aohp-agents](https://github.com/gramarye-os/aohp-agents) | container templates (`template/`), `bootstrap.sh` / `aohp-bootstrap`, `aohp-secrets`, installers |
| [gramarye-os/tessera-linux](https://github.com/gramarye-os/tessera-linux) | Tessera, the launcher for Linux GUI apps; hosts its APK release |
| [injinj/openclaw](https://github.com/injinj/openclaw) | OpenClaw fork; hosts the signed Android APK release |
| [gramarye-os/aohp](https://github.com/gramarye-os/aohp) (`feat/units`, on top of `feat/cli-secret`) | the `aohp` CLI with `secret get/set/list` and `unit`/`timer` (upstream PR pending) |

## Quick start (install a release)

1. Unlock the bootloader and bring the phone to the stock firmware level the LineageOS wiki requires
   ([oriole](https://wiki.lineageos.org/devices/oriole/install/), [dodge](https://wiki.lineageos.org/devices/dodge/install/),
   [ingot](https://wiki.lineageos.org/devices/ingot/install/) — the Saga must be on stock Android 14 before unlocking).
2. Download the device's release assets, `sha256sum -c SHA256SUMS` (dodge: `cat …zip.part0 …zip.part1 > …zip` first).
3. Follow [docs/install-oriole.md](docs/install-oriole.md), [docs/install-dodge.md](docs/install-dodge.md) or
   [docs/install-ingot.md](docs/install-ingot.md): fastboot the
   boot images → Lineage recovery → factory reset → *Apply update → Apply from ADB* → `adb sideload <zip>` →
   *Reboot system now*.
4. First boot: adb is on (our build host is authorized by a key in the image; any other host gets the usual dialog).
   `aohp-containerd` and the Driver bridge are already running. Continue with
   [docs/provisioning.md](docs/provisioning.md) to create the agent container, provision OpenClaw and pair the app.
5. Later builds: *Settings → System → Updater* on the phone. Coming from a release older than 2026-11-19, start again
   at step 2 instead: the signing keys changed, so that move needs the factory reset.

## Credits and licenses

- [LineageOS](https://lineageos.org/) — the ROM, device trees and kernels (Apache-2.0 / GPL-2.0 as per project).
- [AOHP — Android Open Harness Project](https://github.com/aohp-os) (`aohp-os/aohp`, `platform_frameworks_base`,
  `platform_system_core`, `platform_system_sepolicy`, `platform_build`) — the agent harness (Apache-2.0). The injinj
  branches carry its patch set; fixes are being upstreamed as PRs.
- [TheMuppets](https://github.com/TheMuppets) — proprietary vendor blobs for oriole, dodge and ingot (device vendors' licenses).
- [F-Droid](https://f-droid.org/) — the bundled client and Privileged Extension (their own licenses).
- [OpenClaw](https://github.com/openclaw/openclaw) — gateway and Android app (see upstream LICENSE).
- Debian / Node.js / the packages inside the container template — their respective licenses.
- Nothing here is affiliated with or endorsed by LineageOS, Google, OnePlus or the AOHP project.
