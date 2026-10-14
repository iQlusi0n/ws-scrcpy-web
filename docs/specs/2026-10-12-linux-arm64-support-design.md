# Linux arm64 (Raspberry Pi OS) Support — Design

**Status:** approved 2026-10-12 (scope and the Docker adb source ruled by the user, see §4).
**Plan:** `docs/plans/2026-10-12-linux-arm64-support.md`.

"Raspberry Pi OS support" is, concretely, **linux-arm64 support**: the Pi 5 is an
ordinary aarch64 glibc host and nothing in this app is Pi-specific. The name of the
work is arm64; the Pi is the reference device.

## 1. What is already arm64-correct (measured 2026-10-12)

| Piece | Evidence |
|---|---|
| velopack native addon | `velopack@1.2.161` ships `lib/native/velopack_nodeffi_linux_arm64_gnu.node`; needs glibc ≥ 2.39 (`Dockerfile:14-22`). Raspberry Pi OS Trixie has 2.41. |
| node-pty prebuilt | `linux-arm64-glibc` is a published leg of `node-pty-prebuilds.yml:100-101`; `NodePtyResolver.ts:89` and `fetch-prebuilts.mjs:163` branch on `process.arch`. |
| Node.js updater | `DependencyDefinitions.ts:320` builds `node-v<ver>-linux-${arch}.tar.gz`; nodejs.org publishes arm64 with the same signed SHASUMS. |
| mkcert | `DependencyDefinitions.ts:41-44` picks `linux-arm64`. |
| scrcpy-server | A JAR pushed to the device; host arch is irrelevant. |
| Build-time native deps | `package-lock.json` carries `@swc/core-linux-arm64-gnu`, `@typescript/typescript-linux-arm64`, `@biomejs/cli-linux-arm64`, `@esbuild/linux-arm64`, `@rolldown/binding-linux-arm64-gnu`. |
| Rust sources | No `target_arch`/`x86_64` in `launcher/`, `common/`, `tray/`; the only arch is the build triple named in `release.yml` and `stage-publish.mjs`. `ksni` is `target_os`-gated. |
| Docker Node interpreter | The runtime stage symlinks the image's own `node` as the seed (`Dockerfile:157-158`), so the interpreter is arch-correct by construction; `fetch-node.mjs` is deliberately not run there. |
| vpk (Velopack CLI) | Picks the AppImage runtime from the main exe's ELF machine (`LinuxPackCommandRunner.cs:90-92`, `AppImageTool.cs:15-18`), vendors `appimagekit-runtime-aarch64` and `UpdateNix_arm64`, and uses the system `mksquashfs` on Linux. |

## 2. What is x64-only today

| Piece | Where | Fix |
|---|---|---|
| Seed (bootstrap) Node for the AppImage | `scripts/fetch-node.mjs:51-54` hardcodes `linux-x64` with one sha256 | `process.arch` branch + a second pinned sha256 |
| Launcher build target | `release.yml:203,229,232`; `scripts/stage-publish.mjs:72-82` | `aarch64-unknown-linux-musl` on an arm64 runner; triple chosen by arch |
| AppImage static runtime | `scripts/swap-appimage-runtime.mjs:51-60` pins `runtime-x86_64` | pick `runtime-aarch64` by arch; sha256 `00cbdfcf917cc6c0ff6d3347d59e0ca1f7f45a6df1a428a0d6d8a78664d87444` (type2-runtime `20251108`, downloaded and hashed 2026-10-12; the x86_64 hash from the same release matches the existing pin) |
| Velopack channel / asset / feed names | `package-linux.mjs:122` (`linux-<channel>`), `UpdateService.resolveExplicitChannel` (`:414-417`), `linuxAppImageAssetName` (`linuxUpdateAssets.ts:7-9`), `release.yml` publish globs | arm64 gets its own channel `linux-arm64-<channel>`; x64 names are unchanged so existing installs keep updating |
| Docker platforms | `docker-publish.yml` `platforms: linux/amd64` (both the Scout-gate build and the push) | `linux/amd64,linux/arm64` |
| `start.sh` Node probe | Only `dependencies/node/node` and `seed/node/node` | unchanged — the seed becomes arch-correct via `fetch-node.mjs` |
| **adb** | Google publishes no arm64 Linux `platform-tools` (see §3) | §4 |

## 3. adb: the one real gap

Re-verified 2026-10-12 against Google's own index and CDN:

- `repository2-{1,2,3}.xml` list exactly three `platform-tools` archives (`-linux.zip`,
  `-darwin.zip`, `-win.zip`). The index knows `host-arch=aarch64` — for `emulator`,
  `ndk`, `cmake`, `cmdline-tools`, `build-tools` — but only with `host-os=macosx`.
- Six plausible arm64 names under `dl.google.com/android/repository/` all 404.
- `platform-tools_r37.0.1-linux.zip` (SHA-1 matches the index) contains
  `ELF 64-bit LSB pie executable, x86-64`. Not fat.

So the app's managed download (`DependencyDefinitions.ts:324-345`,
`DependencyManager.installAdb`) can never produce a working binary on linux-arm64:
it downloads, verifies, installs, and dies with `exec format error` on the first
spawn. The only arm64 Linux adb binaries are third-party builds from the
Apache-2.0 AOSP source.

Debian's `adb` package (`android-platform-tools` source, `1:34.0.5-12` in Trixie,
which Raspberry Pi OS mirrors) was downloaded and run on 2026-10-12:

- `adb version` prints `Version 34.0.5-debian` → the app's `/Version ([\d.]+)/`
  parser reads `34.0.5`.
- `pair`, `connect`, `shell`, `forward`, `push` are compiled in and work.
- **mDNS is compiled out** (`debian/system/libadb.mk` comments out
  `client/transport_mdns.cpp`, `mdnsresponder_client.cpp`, `client/openscreen/*`).
  `adb mdns check` prints nothing; `adb mdns services` prints its header and no
  rows, exit 0.

Consequence, accepted by the user for this work ("don't worry about the openscreen
problem"): on a system adb without mDNS, **QR pairing** (`PairingService.ts:207,223,253`)
and the **Quick Scan mDNS channel** (`DeviceDiscoveryApi.ts:52`, `index.ts:240`) find
nothing. Manual pairing by code, `adb connect ip:port`, the TCP-5555 sweep and
everything downstream work. Getting mDNS back (a DNS-SD browser in Node, or a
`nmeum/android-tools ≥ 37.0.0p1` build with `-DANDROID_TOOLS_ADB_ENABLE_MDNS=ON`) is
**out of scope** and tracked separately.

## 4. Decisions

### D1 — On linux-arm64 the app uses a system adb, found by absolute-path allowlist

`resolveAdbPath` gains a third source, `system`, taken only when
`platform === 'linux' && arch === 'arm64'` and no `adbPath` override is set. It
returns the first existing path from a fixed list:

```
/usr/bin/adb
/usr/lib/android-sdk/platform-tools/adb   (Debian's real location; /usr/bin/adb symlinks here)
/usr/local/bin/adb
```

and, when none exists, `/usr/bin/adb` (so the missing binary surfaces exactly the way a
not-yet-downloaded managed adb does today: `AdbExecError('spawn')`, and the
Dependencies card says what to install).

This is **not** a `PATH` fallback. It is the same absolute-candidate mechanism
`src/server/service/systemTools.ts` already uses for `systemctl`/`pkexec`, applied
to the one platform where the managed path is impossible. `Config.ts:178-181`'s rule
("no system-PATH fallback by design") stands on every other platform, and a
`config.json` `adbPath` override still wins everywhere.

### D2 — An adb the app did not install is "externally managed" in the Dependencies panel

Whenever `adbResolution.source !== 'bundled'` (the `system` source above, or a user's
`adbPath` override on any platform), the adb dependency:

- reports the version of the binary actually in use (`checkInstalled` runs
  `Config.adbPath`, not `<deps>/adb/adb`);
- never looks up Google's index, never auto-installs, never offers **update**;
- shows a `managed by: system` / `managed by: config.json` badge, and, when the binary
  is missing, an install hint (`sudo apt install adb`) instead of an **install** button.

This fixes a pre-existing wart on every platform: today an `adbPath` override still
downloads Google's zip into `<deps>/adb/` and the card describes that file, not the
one in use.

### D3 — arm64 AppImage is a separate Velopack channel

`linux-arm64-beta` / `linux-arm64-stable` → assets `WsScrcpyWeb-linux-arm64-<channel>.AppImage`,
feeds `releases.linux-arm64-<channel>.json`. x64 keeps `linux-<channel>` and its
asset names, so the updater of every existing install is untouched. The server picks
the channel from `process.arch` (`UpdateService.resolveExplicitChannel`,
`linuxAppImageAssetName`). An arm64 box can therefore never download an x64 AppImage
and vice versa.

### D4 — arm64 release leg builds natively on `ubuntu-24.04-arm`

Not cross-compiled: every staging script keys on `process.arch` (prebuilts, seed
Node, launcher triple, runtime swap), vpk needs a native `dotnet`, and the
node-pty workflow already uses this runner. `build-linux` becomes a two-leg matrix.

### D5 — Docker goes multi-arch; the arm64 leg installs Debian's adb, version-pinned (ruled 2026-10-12)

This **supersedes the 2026-09-02 ruling** recorded in
`docs/specs/2026-06-09-sp4-docker-image-design.md` §16.1 (`linux/amd64` only) and
amends §9 for the arm64 leg only: `apt-get install adb=<exact version>` from the same
digest-pinned Trixie base the image already trusts for its libc. The pin is exact and
the build fails when Debian moves it, so a bump is a deliberate commit — the same
posture as every other pinned dependency. The amd64 leg is unchanged: it keeps
downloading and verifying Google's platform-tools at first boot.

In the container, `resolveAdbPath` lands on `/usr/bin/adb` through D1 with no
container-specific code.

The alternatives considered and not chosen for this round, in case the ruling is
revisited: building `nmeum/android-tools ≥ 37.0.0p1` from a sha256-pinned tarball
(Apache-2.0, optional mDNS via the new Rust `adbmdns` backend; more to build and
maintain), or running Google's x86-64 adb under `qemu-x86_64-static` with a ~4 MB
x86 glibc sysroot (verified working under emulation on x86; keeps Google's bits and
mDNS; ~18 MB heavier).

### D6 — What stays x64-only

Windows arm64, Intel macOS, and 32-bit ARM (`armv7`, Pi OS 32-bit) are out of scope.
`getArch()` keeps collapsing everything that is not `arm64` to `x64`.

## 5. Host prerequisites (documented, not vendored)

Raspberry Pi OS **Trixie, 64-bit** (glibc 2.41 ≥ the addon's 2.39; Bookworm's 2.36
exits at import). `sudo apt install adb`. For a source checkout, Node ≥ 24 for arm64
(`engines`); apt's `nodejs` is 20.x.

## 6. Verification surface

- Unit: `resolveAdbPath` `system` branch; `getDependencyDefinitions` external-adb
  mode; `resolveExplicitChannel`/`linuxAppImageAssetName` with `arch`;
  `fetch-node.mjs`/`swap-appimage-runtime.mjs` arch selection.
- CI: a native `ubuntu-24.04-arm` job runs `cargo test`/`clippy` for the aarch64 musl
  target and `npm test`; the Docker CI build adds `linux/arm64`.
- Manual smoke on a Pi 5 (new rows in `docs/smoke-tests/smoke-test.md`, tag `[RPi]`):
  AppImage launches, system adb detected, pair by code, connect, stream, service
  install (user and system scope), in-app update to the next arm64 build, Docker
  arm64 image first boot + stream.
