---
title: egui/eframe + tray-icon for the staff toolbox
status: Accepted
date: 2026-11-01
deciders: Marc
tags: [toolbox, ui, rust, egui, performance]
adrId: 261101-3
---

# egui/eframe + tray-icon for the staff toolbox

## Context

The toolbox is a tray-resident desktop utility that must run on macOS, Linux, and
Windows, mostly idle, on modest hardware. It must stay light (memory and CPU),
start fast, and require no web runtime. Both reference suite managers are egui
apps; the team's existing apps (docdeck, repoman) are Tauri.

## Decision

- `eframe`/`egui` desktop app with the `tray-icon` crate for the system tray.
- A `tokio` background runtime owns all network, download, extraction, and hashing
  work. The UI thread never blocks: workers send events over a channel, and
  `update()` drains it non-blockingly.
- Repaint discipline: `request_repaint()` only on new data and only when the
  window is visible; high-frequency progress is throttled; spinners are hidden
  while in the tray.
- Frameless, always-on-top popover that hides on lost focus on macOS/Windows; a
  normal resizable window on Linux where Wayland restricts positioning.
- `glow` renderer (OpenGL) for the widest support on older Linux/Mesa hardware;
  revisit `wgpu` if VSync/DPI issues appear.
- Bundled Noto Sans (+ CJK fallback) fonts so paths and names never render as
  boxes.
- Platform packaging: single `.exe` on Windows with `windows_subsystem = "windows"`,
  `.app` with `LSUIElement = true` on macOS, ELF/AppImage on Linux; autostart
  entries per OS.

## Consequences

- Low memory footprint, single binary, no webview, no JavaScript toolchain.
- Works on old hardware and minimal Linux installations.
- UI code is more verbose than HTML/CSS, and accessibility is limited.
- Wayland does not allow clients to position windows freely; the Linux layout
  must not depend on absolute placement.
- Two UI idioms in the company (Tauri for product apps, egui for the toolbox).

## Alternatives Considered

- **Tauri** — reuses the team's existing stack but pays for a webview and a web
  build pipeline; the toolbox is a simple list, not a document UI. Rejected.
- **A local web page with a shell script backend** — cannot install or launch
  natively and needs a separate supervisor. Rejected.

## Related Decisions

`ADR__261101-2__portable-versioned-dir-installs.md`
