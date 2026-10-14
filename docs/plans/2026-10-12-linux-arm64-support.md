# Linux arm64 (Raspberry Pi OS) Support — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** ws-scrcpy-web runs on a 64-bit linux-arm64 host (reference: Raspberry Pi 5, Raspberry Pi OS Trixie) three ways — from a source checkout, as an arm64 AppImage with the in-app updater, and as a `linux/arm64` Docker image — using the distro's `adb` where Google ships none.

**Architecture:** Every arch-sensitive seam already keys on `process.arch` except four build scripts, two name strings, and adb. Phase 1 teaches adb resolution a `system` source for linux-arm64 and makes any non-app-installed adb "externally managed" in the Dependencies panel. Phase 2 gives the four scripts an arm64 branch, gives arm64 its own Velopack channel (`linux-arm64-<channel>`), and adds a native `ubuntu-24.04-arm` release leg. Phase 3 widens the Docker build to `linux/arm64` with Debian's `adb` pinned in that leg only. Phase 4 is the Pi smoke module and the ruling amendments.

**Tech Stack:** TypeScript (Node server + browser, vitest) · Node build scripts (`scripts/*.mjs`) · Rust launcher (`aarch64-unknown-linux-musl`) · Velopack `vpk` · GitHub Actions (`ubuntu-24.04-arm`) · Docker buildx (QEMU leg).

**Design spec:** `docs/specs/2026-10-12-linux-arm64-support-design.md` (decisions D1–D6, measured facts, pinned hashes). Read it first; this plan does not repeat the rationale.

**Out of scope (by ruling, 2026-10-12):** mDNS on a system adb (QR pairing / Quick Scan mDNS stay empty on Debian's build — see spec §3); Windows arm64; Intel macOS; 32-bit ARM.

**Branch:** `feat/linux-arm64-support`.

---

## File structure

**Modify:**
- `src/server/Config.ts` — `resolveAdbPath`: `system` source for linux-arm64 (D1); `AdbSource` type exported.
- `src/server/DependencyDefinitions.ts` — `getDependencyDefinitions(depsPath, { inContainer, adb })`: the adb definition becomes externally managed when `adb.source !== 'bundled'` (D2).
- `src/server/DependencyManager.ts` — thread `adb` into definitions; skip latest-lookup / auto-install / `update()` for an externally managed dependency; expose `managedBy` + `installHint` on `DependencyInfo`.
- `src/common/DependencyTypes.ts` — `DependencyInfo.managedBy?: 'app' | 'system' | 'config'`, `installHint?: string`.
- `src/app/client/DependencyPanel.ts` — badge + hint rendering, no action button for externally managed.
- `src/server/index.ts` — pass `config.adbSource` to the manager (both construction sites, `:132-136` and `:360`).
- `scripts/fetch-node.mjs` — `process.arch` branch with the arm64 pin.
- `scripts/swap-appimage-runtime.mjs` — runtime chosen by arch, second pinned hash.
- `scripts/stage-publish.mjs` — launcher triple chosen by arch.
- `scripts/package-linux.mjs` — channel `linux-arm64-<channel>` on arm64.
- `src/server/linuxUpdateAssets.ts`, `src/server/UpdateService.ts` — arch in channel + asset name (D3).
- `.github/workflows/release.yml` — `build-linux-arm64` job (separate job, not a matrix — see ci.yml:270-275 for why); publish globs.
- `.github/workflows/ci.yml` — `arm64-linux-checks` job; Docker CI build adds `linux/arm64`.
- `.github/workflows/docker-publish.yml` — `linux/amd64,linux/arm64`, QEMU, Scout gate per platform.
- `Dockerfile` — `ARG TARGETARCH`; arm64 leg installs `adb=<pin>`.
- `README.md`, `docs/TECHNICAL_GUIDE.md` (§13, §22.5, §26), `docs/RELEASING.md`, `docs/specs/2026-06-09-sp4-docker-image-design.md` (§16.1 amendment), `docs/smoke-tests/smoke-test.md`, `CHANGELOG.md`.

**Create:**
- `src/server/__tests__/config.adbPath.system.test.ts`
- `src/server/__tests__/dependencyDefinitions.externalAdb.test.ts`
- `scripts/__tests__/fetch-node.arch.test.mjs`, `scripts/__tests__/swap-appimage-runtime.arch.test.mjs` (the scripts are untested today; test the exported selector functions only)

**Pins introduced** (all verified 2026-10-12; spec §2):
```
node-v24.19.0-linux-arm64.tar.xz   01443c1e1a29e531ccad5a46fefa6df490d2189c49f7955904aecdbb0fe86fdc
type2-runtime 20251108 runtime-aarch64   00cbdfcf917cc6c0ff6d3347d59e0ca1f7f45a6df1a428a0d6d8a78664d87444
Debian trixie adb                   1:34.0.5-12   (re-check at implementation time: `apt-cache policy adb` in the arm64 base)
```

---

## Build / test commands

- Unit: `npm test` (vitest). Single file: `npx vitest run src/server/__tests__/config.adbPath.system.test.ts`.
- Lint/types: `npm run lint && npx tsc --noEmit`.
- Rust (Linux host): `cargo test --workspace && cargo clippy --workspace --all-targets -- -D warnings`. arm64 musl build on an x86 host needs `cross` (`cross build --target aarch64-unknown-linux-musl --release --workspace`); CI's arm64 job builds natively.
- Scripts: `node scripts/fetch-node.mjs` (downloads ~30 MB; cache under `dependencies/node-bootstrap/`).
- Docker: `docker buildx build --platform linux/arm64 --load -t ws-scrcpy-web:arm64 .` (needs `docker/setup-qemu` or binfmt on the host).

**Verification checkpoints on a real Pi 5 (Trixie 64-bit):** after Task 1.5 (source run), after Task 2.9 (AppImage + in-app update), after Task 3.4 (Docker image). Each is a numbered smoke row in Task 4.1; record results there.

---

## Phase 1 — Host: system adb on linux-arm64, externally managed adb everywhere

### Task 1.1: `resolveAdbPath` gains the `system` source

**Files:** `src/server/Config.ts:162-184`; test `src/server/__tests__/config.adbPath.system.test.ts`; existing `config.adbPath.test.ts` keeps passing unchanged.

- [ ] **Step 1: Failing tests.** Signature becomes
  ```ts
  export type AdbSource = 'config' | 'bundled' | 'system';
  export const SYSTEM_ADB_CANDIDATES = ['/usr/bin/adb', '/usr/lib/android-sdk/platform-tools/adb', '/usr/local/bin/adb'] as const;
  export function resolveAdbPath(
      fileConfig: FlatConfig,
      dependenciesPath: string,
      platform: NodeJS.Platform = process.platform,
      arch: string = process.arch,
      exists: (p: string) => boolean = fs.existsSync,
  ): { path: string; source: AdbSource }
  ```
  Cases: linux+arm64+no override → first existing candidate, `source: 'system'`; none exist → `/usr/bin/adb`, `source: 'system'`; linux+x64 → bundled (unchanged); win32+arm64 → bundled (D6: only linux-arm64 takes the branch); override wins on linux+arm64 → `source: 'config'`; `exists` is called only for the candidates, in order, and never for the bundled path.
- [ ] **Step 2: Implement.** Keep the function pure; `exists` is injected. Update the doc comment's priority chain (`:162-171`) to three entries and say explicitly why this is not a PATH fallback (spec D1).
- [ ] **Step 3: Caller.** `Config.ts:1004-1012`: store the resolution as `Config.adbSource` (new getter) next to `adbPath`; the log line already prints `source=`.
- [ ] **Step 4:** `npx vitest run src/server/__tests__/config.adbPath*.test.ts` green.

### Task 1.2: `DependencyInfo` carries `managedBy` and `installHint`

**Files:** `src/common/DependencyTypes.ts:18-50`.

- [ ] Add `managedBy?: 'app' | 'system' | 'config'` (absent ⇒ `'app'`, so every existing consumer and fixture is unchanged) and `installHint?: string`. JSDoc each: `managedBy` says whether the app may download/update this dependency; `installHint` is a one-line, platform-specific instruction shown when an externally managed dependency is missing.

### Task 1.3: External-adb mode in definitions and manager

**Files:** `src/server/DependencyDefinitions.ts:324-345,460-465`; `src/server/DependencyManager.ts:175-220,246-330,527-601,400-480`; `src/server/index.ts:132-136,360`; test `src/server/__tests__/dependencyDefinitions.externalAdb.test.ts`; extend `dependencyManager.*.test.ts` where the auto-install loop is covered.

- [ ] **Step 1: Failing tests.**
  - `getDependencyDefinitions(deps, { inContainer: false, adb: { path: '/usr/bin/adb', source: 'system' } })`: the adb definition has `managedBy: 'system'`, `installHint: 'Install the distribution package: sudo apt install adb'`, `checkInstalled` runs `/usr/bin/adb --version` (not `<deps>/adb/adb`), `checkLatest` resolves `null` **without any fetch** (assert `fetch` is not called), `getDownloadUrl` throws.
  - Same with `source: 'config'` → `managedBy: 'config'`, hint names `config.json adbPath`.
  - `source: 'bundled'` → today's definition, byte-for-byte behavior (`managedBy` absent).
  - Manager: with `managedBy: 'system'` and `installedVersion === null`, `autoInstallMissing()` does not call `update('adb')`; `update('adb')` returns `{ success: false, errorMessage: 'adb is managed by the system; the app does not install it' }` and never touches the network; `checkAll` leaves status `UpToDate` when installed and `NotInstalled` (not `Error`) when missing, with `latestVersion: null` and `latestLookup` marked not-attempted (reuse the deferred/refused machinery at `:261-330` rather than adding a fourth state — read it first).
- [ ] **Step 2: Implement.** New opts field on `getDependencyDefinitions`; `DependencyDefinition` gains `managedBy?` and `installHint?`; `DependencyManager` constructor copies both onto the info (next to `deferInstall`, `:218`), `autoInstallMissing` skips `managedBy !== 'app'` beside the `deferInstall` skip (`:582-584`), `update()` refuses early, `checkLatest` short-circuits. `index.ts` passes `{ path: config.adbPath, source: config.adbSource }` at both construction sites.
- [ ] **Step 3:** The `/api/dependencies` payload now carries the two fields — confirm `src/server/api/*` serializes `DependencyInfo` whole (it does today: grep `JSON.stringify(` of the info map) and no allow-list filters them out.
- [ ] **Step 4:** `npm test` green; `npx tsc --noEmit` clean.

### Task 1.4: Dependencies panel renders externally managed adb

**Files:** `src/app/client/DependencyPanel.ts:27-29,255-340`; its test file `src/app/client/DependencyPanel.test.ts`.

- [ ] **Step 1: Failing test.** A row with `managedBy: 'system'`, `installedVersion: '34.0.5'`, `status: 'up-to-date'` renders the version, a `dep-badge dep-managed-system` badge reading `system`, and **no** button; the latest cell reads `—` with a title "Managed by the system; the app does not update it". A row with `managedBy: 'system'`, `installedVersion: null`, `status: 'not-installed'` renders the `installHint` text in the action cell and no `data-install` button. `managedBy: 'config'` → badge `config.json`. Rows without `managedBy` render exactly as before (snapshot the existing fixtures).
- [ ] **Step 2: Implement** in `offersInstall`, `latestCell`, `statusLabel`, `actionButton`. One CSS rule for the new badge in `src/style/` next to `dep-not-installed`.
- [ ] **Step 3:** `npm test` green. Visual check in the browser against a locally forced `adbPath` override (any platform): card shows `config.json` badge and no update offer — this is the pre-existing wart from spec D2 closed.

### Task 1.5: Docs for Phase 1 + Pi checkpoint

- [ ] `docs/TECHNICAL_GUIDE.md` §13.1: a short "Externally managed adb" subsection: the three sources, the linux-arm64 rule, the candidate list, what the panel shows, why mDNS is missing on Debian's adb (link spec §3).
- [ ] `README.md` config table (`:349`, `adbPath` row): mention that on linux-arm64 the app uses the system `adb` by default and name `sudo apt install adb`.
- [ ] `CHANGELOG.md` `[Unreleased]` → `### Added`: "linux-arm64 hosts use the system `adb` (Raspberry Pi OS: `sudo apt install adb`); an adb the app did not install is shown as externally managed and never overwritten."
- [ ] **Checkpoint P1 (Pi 5):** `sudo apt install adb`, Node 24 arm64, `npm ci --ignore-scripts && node scripts/fetch-prebuilts.mjs && npm run build && node dist/index.js`. Log shows `adbPath=/usr/bin/adb (source=system)`; no velopack warning; card shows `34.0.5 · system`; pair by code → connect → stream; ADB shell opens (node-pty arm64 prebuilt). Record in Task 4.1 rows 22.1–22.4.

---

## Phase 2 — arm64 AppImage: build scripts, channel, release leg, updater

### Task 2.1: `fetch-node.mjs` picks the archive by arch

**Files:** `scripts/fetch-node.mjs:40-54,117-139`; new `scripts/__tests__/fetch-node.arch.test.mjs`.

- [ ] **Step 1:** Refactor `platformConfig()` into an exported pure `selectArchive(platform, arch)` returning `{ archiveName, archiveSha256, innerRel, seedNodeName }`; keep `main()` behavior identical. Add the Linux table:
  ```js
  const LINUX = {
      x64:   { name: `node-${NODE_VERSION}-linux-x64.tar.xz`,   sha256: '14b342e71204f811bde6153be8e04b62aef63c236fef92b55f9c83154b409647' },
      arm64: { name: `node-${NODE_VERSION}-linux-arm64.tar.xz`, sha256: '01443c1e1a29e531ccad5a46fefa6df490d2189c49f7955904aecdbb0fe86fdc' },
  };
  ```
  Unsupported arch on linux (e.g. `arm`) throws, naming the two supported ones. Honour `FETCH_NODE_ARCH` env as an override (lets the x86 CI leg smoke-fetch the arm64 archive if ever needed; document it in the header comment).
- [ ] **Step 2: Tests** for `selectArchive` across `linux/x64`, `linux/arm64`, `linux/arm` (throws), `win32/x64`, `win32/arm64` (throws — D6). Register the file in `vitest.config.ts` if `scripts/__tests__` globs need it (check `include`).
- [ ] **Step 3:** Update the comment at `Dockerfile:50-54` — it describes the old single-pin behavior; it should now say the script *is* arch-aware but the image still prefers its own interpreter.

### Task 2.2: `swap-appimage-runtime.mjs` picks the runtime by arch

**Files:** `scripts/swap-appimage-runtime.mjs:45-77`; new `scripts/__tests__/swap-appimage-runtime.arch.test.mjs`.

- [ ] Replace `RUNTIME_URL`/`RUNTIME_SHA256` with an exported `RUNTIMES = { x64: { url: …runtime-x86_64, sha256: '2fca8b44…' }, arm64: { url: …runtime-aarch64, sha256: '00cbdfcf…' } }` and an exported `selectRuntime(arch = process.arch)`; `verifyRuntimeBytes` default comes from the selected entry. Also verify the downloaded ELF's `e_machine` matches the arch (`0x3E` x86-64, `0xB7` aarch64) before composing — a wrong-arch runtime is the first code to run on launch.
- [ ] Tests: selection per arch; `e_machine` check rejects a cross-arch buffer; existing `verifyRuntimeBytes` tests unchanged.
- [ ] Keep the header comment's pin-bump instructions and extend them to "bump both hashes together".

### Task 2.3: `stage-publish.mjs` picks the launcher triple by arch

**Files:** `scripts/stage-publish.mjs:70-82`.

- [ ] `const LINUX_TRIPLE = { x64: 'x86_64-unknown-linux-musl', arm64: 'aarch64-unknown-linux-musl' }[process.arch]`; throw for anything else. Update the comment (`:72-74`) to name both triples.
- [ ] After copying, assert the launcher ELF's `e_machine` matches `process.arch` (same helper as Task 2.2 — put it in `scripts/elf-machine.mjs` and import from both).

### Task 2.4: `package-linux.mjs` emits the arm64 channel

**Files:** `scripts/package-linux.mjs:115-145`.

- [ ] `const ARCH_SEGMENT = process.arch === 'arm64' ? 'arm64-' : ''; const CHANNEL = \`linux-${ARCH_SEGMENT}${channelBase}\`;` — x64 output names are byte-identical to today. Log the channel. Update the comment at `:115`.
- [ ] Confirm by reading vpk's naming that the AppImage comes out as `WsScrcpyWeb-linux-arm64-<channel>.AppImage` and the feed as `releases.linux-arm64-<channel>.json` (the x64 leg's names prove the `<packId>-<channel>` pattern; if vpk differs, `linuxAppImageAssetName` in Task 2.5 must match whatever it emits — the asset name is the contract).

### Task 2.5: Updater resolves the arm64 channel and asset

**Files:** `src/server/linuxUpdateAssets.ts:6-9`; `src/server/UpdateService.ts:101-140,389-392,414-417,1096`; `src/common/ConfigEvents.ts:18` (unchanged — arch is a separate dimension); tests `src/server/__tests__/linuxUpdateAssets.test.ts`, `UpdateService.test.ts`.

- [ ] **Step 1: Failing tests.** `linuxAppImageAssetName('beta', 'arm64') === 'WsScrcpyWeb-linux-arm64-beta.AppImage'`; `('beta', 'x64')` and `('beta')` keep returning `WsScrcpyWeb-linux-beta.AppImage`. `UpdateService` with `platform: 'linux', arch: 'arm64'` resolves feed channels `['linux-arm64-beta', 'linux-arm64-stable']` for `beta` and downloads the arm64 asset in the apply path; with `arch: 'x64'` nothing changes (existing tests are the proof).
- [ ] **Step 2: Implement.** `UpdateServiceOptions.arch?: string` (default `process.arch`, mirrors the `platform` seam at `:117-122`); `resolveExplicitChannel` inserts `arm64-` on linux+arm64; the apply call at `:1096` passes `this.arch`. Also the `releases.linux-…json` reader in `src/server/updateFeedResolver.ts` — it takes the explicit channel string from `resolveExplicitChannel`, so verify by test that it requests `releases.linux-arm64-beta.json`.
- [ ] **Step 3:** `UpdateService.ts:80-93` doc comment: add the arch sentence.

### Task 2.6: Release workflow — `build-linux-arm64`

**Files:** `.github/workflows/release.yml:185-272,273-350`.

- [ ] Duplicate `build-linux` as `build-linux-arm64` (a **separate job**, not a matrix: `ci.yml:270-275` explains the required-check naming hazard; the same caution applies to anything a ruleset may pin). Differences only:
  - `runs-on: ubuntu-24.04-arm`
  - rust target `aarch64-unknown-linux-musl`; the `musl-tools` apt package provides `musl-gcc` for the runner's native arch; keep the Google-Chrome-repo `rm` guard.
  - add `sudo apt-get install -y squashfs-tools` (vpk shells out to `mksquashfs`; do not assume the arm image preinstalls it — the x64 job gets it from the ubuntu-latest image). Add the same line to `build-linux` for symmetry and to stop relying on the image.
  - verify step: `file "$BIN"` must print `ARM aarch64` and `statically linked`.
  - artifact name `linux-arm64-final`, feed file `releases.linux-arm64-${{ needs.prepare.outputs.channel }}.json`.
  - attest provenance for the arm64 AppImage.
- [ ] `publish`: `needs: [prepare, build-windows, build-linux, build-linux-arm64]`; add the four `artifacts/linux-arm64-final/…` lines to `files:`. `SHA256SUMS` already globs every `*.AppImage`/`*.nupkg` under `artifacts/` — basenames stay unique because the channel is in the name; add a `test` that no two entries share a basename, since `parseSha256Sums` matches by basename (`linuxUpdateAssets.ts:68-72`).
- [ ] `scripts/assert-version-sync.mjs` / `bump-version.mjs` / `extract-changelog.mjs`: grep for `linux-` and `AppImage` and extend any list of expected release assets.

### Task 2.7: CI — `arm64-linux-checks`

**Files:** `.github/workflows/ci.yml` (append after `windows-rust-checks`).

- [ ] New job on `ubuntu-24.04-arm` (separate job; comment block copied from `:255-278` and adapted): rust toolchain with `targets: aarch64-unknown-linux-musl` + `components: clippy`, `musl-tools`, then `cargo build --release --workspace --target aarch64-unknown-linux-musl`, `cargo test --workspace`, `cargo clippy --workspace --all-targets -- -D warnings`, `npm ci`, `npm test`, `node scripts/fetch-prebuilts.mjs` (proves the arm64 node-pty prebuilt resolves), `node scripts/fetch-node.mjs` (proves the pin). No e2e here.
- [ ] `build-and-test`'s Docker step (`:185`) stays amd64; Phase 3 adds the arm64 build.

### Task 2.8: Docs for Phase 2

- [ ] `README.md:293-302` Linux install: list the arm64 assets (`WsScrcpyWeb-linux-arm64-stable.AppImage` / `-beta`), say which Pi OS (Trixie 64-bit) and that `adb` comes from apt. Keep x64 lines as they are.
- [ ] `docs/RELEASING.md:52,116-119`: the arm64 leg and its artifacts; the per-arch feed files; the runner.
- [ ] `docs/TECHNICAL_GUIDE.md` §22 (`:2224`) and §22.5 (`:2280`): channel/asset naming now has an arch segment on arm64; table of the four Linux channel strings.
- [ ] `CHANGELOG.md`: "arm64 AppImage (Raspberry Pi 5 / Raspberry Pi OS 64-bit) on its own update channel."

### Task 2.9: Checkpoint P2 (Pi 5)

- [ ] Download the arm64 AppImage from a release built on this branch (or a workflow_dispatch run), `chmod +x`, launch: no `libfuse2`, tray/no-tray per desktop, card shows system adb, stream works. Install user-scope service; reboot; still up. Install system-scope (`pkexec`) — exercises `linux_service.rs` on arm64. Publish a second build; in-app update applies and relaunches (`linux_apply.rs`), and the updater fetched `…linux-arm64-beta.AppImage`, never the x64 one (check `ws-scrcpy-web.log`). Record rows 22.5–22.9.

---

## Phase 3 — Docker `linux/arm64`

### Task 3.1: Dockerfile arm64 leg installs Debian's adb

**Files:** `Dockerfile:70-100,157-170`.

- [ ] In the `runtime` stage, after the `apt-get upgrade` RUN: `ARG TARGETARCH` and
  ```dockerfile
  RUN if [ "$TARGETARCH" = "arm64" ]; then \
        apt-get update && apt-get install -y --no-install-recommends adb=1:34.0.5-12 \
        && rm -rf /var/lib/apt/lists/* \
        && test "$(/usr/bin/adb version | sed -n 's/^Version \([0-9.]*\).*/\1/p')" = "34.0.5"; \
      fi
  ```
  with a comment block in the file's own style: why (no Google arm64 adb), the ruling and date, that the pin is exact and a Debian bump fails the build on purpose, that mDNS is absent in this build (QR pairing / mDNS scan inert; TCP sweep works), and that amd64 is untouched. Verify the exact version string against the pinned base at implementation time (`apt-cache policy adb`).
- [ ] Nothing else in the image changes: `resolveAdbPath` finds `/usr/bin/adb` via D1; `DEPS_PATH` stays `/data/dependencies`; adb never lands on the volume on arm64.
- [ ] `HEALTHCHECK` comment (`:180-182`): note that on arm64 first boot has no adb download, so the window is only for amd64.

### Task 3.2: `docker-publish.yml` multi-arch

**Files:** `.github/workflows/docker-publish.yml:36-40,125-140`.

- [ ] Add `docker/setup-qemu-action` (SHA-pinned, repo convention) before buildx; `platforms: linux/amd64,linux/arm64` on the push step. The Scout gate currently builds and scans one amd64 image: build the arm64 image too (`--platform linux/arm64 --load`, separate tag) and run the same Scout gate on it, so a fixable CVE in the arm64 leg (e.g. in the `adb` package chain) blocks publish like any other.
- [ ] Rewrite the two comment blocks that cite the 2026-09-02 ruling (`:36-38`, `:132-136`) to cite the 2026-10-12 ruling and spec D5.
- [ ] Build-time risk to measure on the first run: the build stage runs webpack/swc/TS7 under QEMU for the arm64 leg. If it exceeds ~40 min, switch to a native-runner matrix (`ubuntu-24.04-arm` builds + `docker buildx imagetools create` manifest merge) — plan that as a follow-up commit, not a redesign.

### Task 3.3: CI builds the arm64 image

**Files:** `.github/workflows/ci.yml:185`.

- [ ] Add `docker buildx build --platform linux/arm64 -t ws-scrcpy-web:ci-arm64 .` (needs `setup-qemu`) as a build-only check; the Docker e2e suite stays on the amd64 image.
- [ ] `qa-manifest.json`: leave amd64-only; add a line to `docs/smoke-tests/automation-coverage.md` stating the arm64 image is covered by manual rows 22.10–22.12 only.

### Task 3.4: Docs + ruling amendment, Checkpoint P3

- [ ] `docs/specs/2026-06-09-sp4-docker-image-design.md` §16: append **16.6 (2026-10-12)** recording that §16.1's amd64-only ruling is lifted for arm64 with Debian's pinned `adb`, cross-referencing the new spec; amend §9's wording to "amd64 leg".
- [ ] `README.md:426`: drop "`linux/amd64` only for now"; say both arches are published, arm64 uses Debian's adb (no mDNS → Quick Scan's mDNS channel and QR pairing inert there; pair by code / connect by `ip:port` / subnet sweep work).
- [ ] `docs/TECHNICAL_GUIDE.md` §26 (`:2841`): the arm64 leg paragraph.
- [ ] `CHANGELOG.md`: "Docker image is published for `linux/arm64`."
- [ ] **Checkpoint P3 (Pi 5):** `docker run … bilbospocketses/ws-scrcpy-web:<tag>` (or the CI-built image): healthy within the window, card shows `34.0.5 · system`, `adb connect ip:port` device streams, `docker stop` exits 0 with `adb kill-server` in the log. Record rows 22.10–22.12.

---

## Phase 4 — Smoke module and wrap-up

### Task 4.1: Smoke rows

**Files:** `docs/smoke-tests/smoke-test.md:17-21,97-110` + a new module.

- [ ] Add platform tag `[RPi]` (Raspberry Pi OS Trixie 64-bit, Pi 5) to the legend; add **Module 22 — linux-arm64 (Raspberry Pi)** to the index with rows `22.1`–`22.12` (21 is the highest module today) as enumerated in the three checkpoints above, each anchored `<a id="t-22-N"></a>` in the file's row format.
- [ ] `scripts/check-smoke-coverage.mjs`: run it; extend if it validates module numbering.

### Task 4.2: Final pass

- [ ] `npm run lint && npx tsc --noEmit && npm test`; `cargo test --workspace`; `node scripts/assert-version-sync.mjs`.
- [ ] Every doc line that still says "Linux (x86_64 only)" or equivalent: `grep -rn "x86_64\|amd64 only\|x64 only" README.md docs/ --include=*.md` and resolve each hit (fix or justify).
- [ ] PR against `main` from `feat/linux-arm64-support`, with the three checkpoint results pasted in the description.

---

## Risks and how each is caught

| Risk | Caught by |
|---|---|
| vpk's arm64 path untested in this repo (runtime pick, `UpdateNix_arm64`, nupkg naming) | Task 2.4 name assertion + 2.9 checkpoint; the asset name is the only contract the updater needs |
| `ksni`/D-Bus transitive crates failing on `aarch64-unknown-linux-musl` | Task 2.7 CI build; fallback is `aarch64-unknown-linux-gnu` for the launcher (it runs inside the AppImage on glibc hosts anyway) — decide only if the musl build fails |
| QEMU-emulated Docker build too slow | Task 3.2 measurement; native-runner matrix is the documented fallback |
| Debian bumps `adb` and breaks the image build | By design (exact pin); the fix is a one-line bump commit |
| A user on Pi OS **Bookworm** (glibc 2.36) | velopack addon refuses to load; README states Trixie. Not mitigated in code — same rule as `Dockerfile:14-22` |
| Pi OS 32-bit | Out of scope (D6); `getArch()` maps `arm` to `x64`, which fails loudly at the first spawn. Consider a startup refusal with a clear message if a report ever comes in |
