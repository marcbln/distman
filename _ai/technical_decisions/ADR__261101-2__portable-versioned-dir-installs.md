---
title: Portable tar.zst payloads installed into versioned directories
status: Accepted
date: 2026-11-01
deciders: Marc
tags: [distribution, packaging, install, rollback]
adrId: 261101-2
---

# Portable tar.zst payloads installed into versioned directories

## Context

The apps being distributed (docdeck, repoman, …) are desktop programs built as
Tauri or native binaries. Staff machines are macOS, Linux, and Windows. We need
install, update, and rollback without elevation prompts, OS installer plumbing,
or three different install code paths. The reference launchers we studied either
drive native installers (MSI/DMG/deb) or manage their own versioned trees.

## Decision

distman packs the **launchable payload** of each app into one archive per target:
`<app>-<version>-<os>-<arch>.tar.zst`, containing the built `.exe` (Windows), the
`.app` bundle (macOS), or the bare executable (Linux). The manifest's `entry`
field names what to execute inside the unpacked tree.

The toolbox unpacks into `<data>/apps/<app>/<version>/`, records the installation
in `installed.json` (the source of truth — no `current` symlink), keeps the
previous version by default, and launches the active version directly. Rollback
is a pointer change in `installed.json`. No MSI, DMG, deb, rpm, or AppImage is
used; WebKitGTK (Linux) and WebView2 (Windows 10+) are assumed present.

Packing is deterministic (sorted entries, zeroed mtimes, fixed zstd level) so the
same build produces the same sha256 regardless of the packing machine.

## Consequences

- One install/update/rollback flow for all three operating systems, no
  administrator rights, no FUSE, no registry, no uninstall entries.
- Rollback is instant and offline; pruning is deleting a directory.
- Deterministic archives make republication and verification trivial.
- No OS integration beyond what the toolbox creates (Start menu/dock entries,
  file associations, default-app registration are absent).
- macOS Gatekeeper complains about unsigned builds; Windows SmartScreen and
  endpoint protection may flag freshly built binaries.
- Apps must be self-contained: installer side effects (services, drivers, PATH
  changes, prerequisites) are not supported by this model.

## Alternatives Considered

- **Drive native installers** — elevation, per-OS quirks, uninstall state, and
  rollback becomes a reinstall. Rejected.
- **AppImage on Linux, MSI/DMG elsewhere** — three code paths and FUSE
  dependency; portable archives cover the same ground uniformly. Rejected.
- **Tauri's built-in updater** — each app would update itself and the toolbox
  would lose visibility and rollback control. Rejected.

## Related Decisions

`ADR__261101-1__static-per-target-release-manifest.md`
