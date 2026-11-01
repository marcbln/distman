---
filename: "_ai/backlog/active/261101_2346__DESIGN_HANDOFF__topdata-toolbox-v0.1.md"
title: "Design Handoff: Topdata toolbox v0.1 (distman + staff launcher)"
createdAt: 2026-11-01 23:46
updatedAt: 2026-11-01 23:46
status: validated
tags: [distman, toolbox, distribution, rust, egui, release-engineering, tailscale]
documentType: DESIGN_HANDOFF
---

# Design Handoff: Topdata toolbox v0.1 (distman + staff launcher)

## Context

Two audiences, one pipeline: devs build internal desktop apps with **distman** and
publish them to a static fileserver reachable only inside the Tailscale network;
**Topdata staff** use a tray-resident **toolbox** to install, launch, and update
those apps, and to open a curated list of internal cloud web UIs. First apps in
scope: `docdeck` (Tauri v2, `com.marc.docdeck`) and `repoman` (native binary,
Tauri-style data dirs). Staff machines: macOS, Linux, Windows.

Reference implementations studied (registered in git-digest): `smyrgeorge/artcraft-toolbox`,
`storytold/craft-launcher`, plus `axodotdev/cargo-dist`, `axodotdev/axoupdater`,
`tauri-apps/plugins-workspace`, `velopack/velopack`, `AprilNEA/gpui-updater`,
`EricVogt93/artcraft-launcher`, `Project-Colony/Colony`, `sparkle-project/Sparkle`.

Durable decisions were promoted to ADRs (see Related below); this file records the
decisions that only matter for this effort, the rejected options, and what remains open.

## Decisions

- **Build/publish topology.** distman is a dev CLI + per-app recipe files
  (`apps/<id>.toml`: build command, payload path per target, entry). No
  cross-compilation: each target is built and published on a machine running that
  OS. Rule out: a central CI coordinator for v1.
- **Two-document static contract.** `/v1/catalog.json` (apps metadata + curated
  links + MOTD, `revision`, ETag-cached) and `/v1/latest-<os>-<arch>.json`
  (app → version, publishedAt, notes, entry, archive path/size/sha256). Rule out:
  per-app feed documents and directory listings.
- **Per-target manifests, no shared writer.** A Mac build only rewrites
  `latest-macos-universal.json`; Windows only `latest-windows-x64.json`. Rule out:
  merge logic and a global publish lock. Accepted cost: platform version skew.
- **Crash-safe publish order.** Upload artifact to a temp name → rename → verify
  size/sha256 on the server → write manifest atomically (`.tmp` + rename), keep
  the previous manifest. Refuse a version that exists with a different sha256;
  identical re-publish is idempotent. Rule out: any manifest pointing at a
  missing artifact.
- **Portable payload archives.** `<app>-<version>-<os>-<arch>.tar.zst` containing
  the `.exe` (Windows), `.app` (macOS), or bare ELF (Linux); `entry` names what to
  exec. Deterministic packing (sorted entries, zeroed mtimes, fixed zstd level).
  Rule out: MSI/DMG/deb/rpm/AppImage and per-OS installer code.
- **Versioned-directory installs.** Toolbox unpacks to `<data>/apps/<app>/<version>/`;
  `installed.json` is the truth (no `current` symlink); keep previous version
  (0–2 configurable); rollback = pointer change. Rule out: symlinks (Windows
  privileges), registry, uninstall entries, FUSE.
- **Toolbox UI stack.** egui/eframe + `tray-icon`; tokio workers own network,
  download, extraction, hashing; UI drains a channel in `update()` and never
  blocks; `request_repaint()` only on new data and only when visible; throttled
  progress; spinners hidden in tray; glow (OpenGL) renderer; bundled Noto Sans +
  CJK fallback; frameless popover that hides on focus loss on macOS/Windows,
  normal window + tray on Linux; autostart per OS; macOS `.app` with
  `LSUIElement=true`; Windows `windows_subsystem="windows"`. Rule out: webview,
  JavaScript build pipeline, and absolute window placement on Wayland.
- **Tray-first states.** Icon reflects health: green (all good), amber (any
  endpoint/link unreachable or toolbox update available), grey (off tailnet),
  red (feed unreachable but network up / stale manifest). Menu: Open toolbox,
  Check now, Quit.
- **Links are data.** `catalog.json.links[]`: id, name, url, optional icon,
  optional health URL, `groups` (unused in v1). Opened with the OS browser
  (`open`/`xdg-open`/`start`) — no webview. Health probes only when the window is
  visible and the tailnet is reachable; failures show a dot, not an error dialog.
- **Update visibility.** On start, every 6 h (configurable), and manual: two
  conditional GETs, semver compare against `installed.json`, emit
  `UpdateAvailable`. New versions notify once; `autoUpdate` is off by default in
  v0.1 (manual Install).
- **Failure handling.** Download to `<data>/downloads/<file>.part` with HTTP Range
  resume; verify size+sha256; extract to `<version>.tmp` with path-traversal,
  symlink, entry-count, and size limits; rename into place; never execute an
  unverified archive; clean `.part`/`.tmp` on start; free-space check (~2× archive).
  A fresh version that fails to start leaves the previous version available with
  a one-click Roll back.
- **Off-network behavior.** Cached catalog still renders; installed apps still
  launch; update checks report "off network"; health dots grey; no blocking error
  UI.
- **CLI surface for distman.** `build`, `publish`, `catalog sync`, `fetch`,
  `serve --dir`, `prune --keep N`. `catalog sync` is the only global writer.
- **Testing seam.** Unit tests for schema round-trip, target aliases, semver
  ordering, deterministic packing, extraction limits; e2e via `distman serve`
  against a tiny fixture app and a toolbox headless `--check --json` mode, run on
  all three OSes.

## Rejected Alternatives

- **Single global manifest for all targets** — needs a coordinator and merge
  logic; concurrent dev publishes race.
- **API/database server** — operational surface and state for data that changes
  only at publish time.
- **Client reads server directory listings** — no checksums, notes, or atomic
  publish semantics; server-specific parsing in every client.
- **One manifest per app (per app+target)** — more requests, no simpler client;
  the per-target file already answers "what do I run?".
- **Native installers driven by the toolbox (MSI/DMG/deb)** — elevation prompts,
  rollback becomes a reinstall, three install code paths.
- **AppImage on Linux** — FUSE dependency; several launchers end up extracting
  anyway. Bare ELF is assumed to have WebKitGTK from the distro.
- **Tauri's built-in updater for the apps** — apps update themselves; toolbox
  loses visibility and rollback control.
- **Tauri for the toolbox UI** — webview memory/footprint and a second web stack
  for a list UI.
- **Client-side group/permission checks** — policy would be duplicated in every
  client and drift from IAM later.
- **Application-level SSO now** — scope not justified while the tailnet already
  gates access.
- **Signature verification in v1** — sha256 + tailnet-only reachability is the v1
  bar; a `signature` field is an additive change later.
- **Adopting externally installed apps in v0.1** — Tauri binaries do not
  uniformly expose `--version`, so reliable version probes come later.

## Constraints Discovered

- The fileserver is static and only reachable inside the tailnet; no server-side
  logic may be assumed in v1.
- Staff OS mix is macOS, Linux, Windows — three payload shapes, one install flow.
- `docdeck` is Tauri v2 with `bundle.targets = "all"` but no CI and no updater;
  `repoman` is an x86-64 ELF binary with config under `~/.config/repoman/`.
- Cross-compilation is unreliable (cargo-dist's own docs treat per-platform
  runners as the norm), so publishing is per-OS by design.
- Windows 10+ has WebView2 preinstalled; Linux is assumed to provide WebKitGTK;
  macOS unsigned `.app` bundles trigger Gatekeeper, and fresh Windows binaries
  may trip SmartScreen/AV.
- `tar.zst` requires the `zstd`/`tar` crates in the toolbox; humans unpacking
  artifacts by hand need `zstd` or `distman fetch`.
- The egui production handout (`_ai/handout-egui.md`) is binding for UI code:
  no blocking on the UI thread, visibility-gated repaints, `tray-icon`,
  `LSUIElement`, `windows_subsystem`, bundled fonts, Wayland placement caveats.
- `installed.json` is the only source of truth for install state; the filesystem
  is scanned only to repair missing/corrupt records.
- All manifest paths must be relative to a configurable base URL (tailnet address
  now, IAM-gated host later).

## Open Questions

- **Transport for `publish`/`catalog sync`** — rsync over SSH, a mounted share, or
  an S3-compatible store (garage/minio)? Owner: Marc.
- **Single-writer process for `catalog.json`** — who runs `catalog sync`, and is a
  small merge step needed if two devs edit apps/links concurrently? Owner: Marc.
- **Artifact retention** — default `--keep` value and whether the server or
  `prune` enforces it. Owner: Marc.
- **Recipe coverage** — exact build/payload mapping for docdeck (Tauri: `.app`,
  `.exe`, bare binary) and repoman (binary + data dirs); does anything besides the
  executable need to ship? Owner: Marc.
- **Version provenance** — read from each app's manifest (Cargo.toml/package.json)
  or always passed to `distman publish`? Owner: Marc.
- **Icons and screenshots** — publish into `catalog.json`/artifact tree, or fetch
  from the app repo? Owner: Marc.
- **MOTD edit path** — hand-edited `catalog.json` or a distman subcommand? Owner: Marc.
- **Auto-update default** — confirm manual-install-only for v0.1, with a settings
  toggle for later. Owner: Marc.
- **Health probe definition** — HEAD vs GET, interval, timeout, and whether probe
  results are ever cached server-side. Owner: Marc.
- **Group model for IAM** — group names and which apps/links each sees; needed
  before server-side filtering. Owner: Marc (+ IAM owner).

## Scope Boundary

**In v0.1:** static contract + per-target manifests; distman `build`/`publish`/
`catalog sync`/`fetch`/`serve`/`prune`; portable tar.zst payloads; versioned-dir
install/update/uninstall/rollback; update checks with notifications; curated links
with health dots; tray + window; autostart; offline degradation; bootstrap of the
toolbox via manual install, then self-update through its own catalog entry;
docdeck and repoman onboarded.

**Out of v0.1:** IAM and per-user/per-group visibility; signature verification;
adoption of externally installed apps; automatic rollback/quarantine after a bad
launch; config push/injection at launch; delta updates; staged rollout channels;
Windows Start-menu integration, file associations, and OS uninstall entries;
web/WASM build; mobile; a server component of any kind.
