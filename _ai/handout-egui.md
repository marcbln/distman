# Production Handout: Building a Tray/Toolbox App with `egui`

This reference guide covers best practices and architectural patterns for building a production-ready, low-resource, cross-platform system tray utility using **`egui`** and **`eframe`**.

---

### 1. Threading & Architecture (The Golden Rule)

`egui` runs on the main thread and re-evaluates the UI synchronously. **Never block this thread with file I/O, network requests, or archive decompression.**

* **Pattern:** Run a background multi-threaded runtime (like `tokio`) alongside your UI thread.
* **Communication:** Use thread-safe channels (`crossbeam-channel` or `std::sync::mpsc`) to pass events between your background workers and the UI loop.

```rust
pub struct ToolboxApp {
    rx: std::sync::mpsc::Receiver<WorkerEvent>,
    tx: std::sync::mpsc::Sender<UiCommand>,
    download_progress: f32,
    is_visible: bool,
}

impl eframe::App for ToolboxApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        // 1. Drain incoming messages without blocking
        while let Ok(event) = self.rx.try_recv() {
            match event {
                WorkerEvent::Progress(p) => self.download_progress = p,
                WorkerEvent::DownloadComplete => { /* update list */ }
            }
        }

        // 2. Render UI
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.add(egui::ProgressBar::new(self.download_progress));
        });
    }
}
```

---

### 2. Battery & CPU Optimization (Reactive Mode)

To ensure your app runs at **0.0% CPU** while idling in the system tray:

1. **Only Repaint on Data Arrival:** When a background download updates its progress, call `ctx.request_repaint()` from the worker thread only when new data is sent over the channel.
2. **Throttle High-Frequency Streams:** Do not repaint for every 4KB network packet. Throttle UI updates to a max rate of 30–60 Hz using a simple timer/debounce in your downloader.
3. **Guard Repaint Calls with Visibility:** If the window is hidden in the tray, **do not trigger repaints**:
   ```rust
   // In your worker / message processor:
   if self.is_visible {
       ctx.request_repaint();
   }
   ```
4. **Beware of Indeterminate Spinners:** If you use `egui::Spinner`, it forces a repaint every frame (~60 FPS). Hide any active spinners when the window is closed to the tray.

---

### 3. Window Lifecycle & Tray Integration

A Toolbox-like app requires a frameless, floating popover window that hides when dismissed.

* **Use Modern Viewports (`eframe` Viewport API):**
  Configure the window to be frameless, transparent, and dismissed on lost focus:
  ```rust
  let native_options = eframe::NativeOptions {
      viewport: egui::ViewportBuilder::default()
          .with_decorations(false)         // No titlebar or native OS borders
          .with_transparent(true)          // Allows rounded window corners
          .with_always_on_top()            // Stays above other apps when open
          .with_inner_size([360.0, 520.0]) // Fixed toolbox dimensions
          .with_resizable(false)
          .with_visible(false),            // Start hidden until tray clicked
      ..Default::default()
  };
  ```
* **Dismiss on Click-Away:**
  In your `update` loop, listen for loss of focus to automatically hide the window back to the tray:
  ```rust
  if ctx.input(|i| i.viewport().focused == Some(false)) {
      self.hide_to_tray(ctx);
  }
  ```
* **Tray Library:** Use the standalone **`tray-icon`** crate (maintained by the Tauri team). It handles native system trays across Windows, macOS, and Linux (via D-Bus/StatusNotifierItem) independently of WebViews.

---

### 4. Overcoming the "Game Debugger" Aesthetic

By default, `egui` has high-contrast colors, harsh borders, and compact spacing. You can transform it into a modern desktop app with three adjustments:

#### A. Soften Visuals & Add Large Corner Radii
```rust
let mut visuals = egui::Visuals::dark();
visuals.window_rounding = egui::Rounding::same(10.0);
visuals.widgets.noninteractive.rounding = egui::Rounding::same(6.0);
visuals.widgets.inactive.rounding = egui::Rounding::same(6.0);
visuals.widgets.hovered.rounding = egui::Rounding::same(6.0);

// Use modern, flat dark-mode backgrounds (e.g. JetBrains #1E1F22 palette)
visuals.window_fill = egui::Color32::from_rgb(30, 31, 34);
visuals.panel_fill = egui::Color32::from_rgb(30, 31, 34);
visuals.window_stroke = egui::Stroke::new(1.0, egui::Color32::from_rgb(45, 47, 51));

ctx.set_visuals(visuals);
```

#### B. Generous Spacing and Margins
Increase default padding so layouts do not feel cramped:
```rust
let mut style = (*ctx.style()).clone();
style.spacing.item_spacing = egui::vec2(10.0, 10.0);
style.spacing.window_margin = egui::Margin::same(14.0);
style.spacing.button_padding = egui::vec2(12.0, 8.0);
ctx.set_style(style);
```

#### C. Card Panels for Tool Entries
Group installed items into cards using `egui::Frame`:
```rust
egui::Frame::none()
    .fill(egui::Color32::from_rgb(43, 45, 48))
    .rounding(egui::Rounding::same(8.0))
    .inner_margin(egui::Margin::same(10.0))
    .show(ui, |ui| {
        ui.horizontal(|ui| {
            ui.label(egui::RichText::new("RustRover").strong());
            ui.with_layout(egui::Layout::right_to_left(egui::Align::Center), |ui| {
                if ui.button("Open").clicked() { /* spawn */ }
            });
        });
    });
```

---

### 5. Internationalization (Preventing Box `▯` Errors)

`egui` only ships with a lightweight Latin font. If a user has a folder path, tool name, or OS locale containing Japanese, Chinese, Cyrillic, or emoji, `egui` will render blank boxes unless fallback fonts are registered.

1. Download a font family like **Noto Sans** or **Inter** (plus **Noto Sans CJK** for East Asian scripts).
2. Bundle the `.ttf`/`.otf` files into your binary using `include_bytes!`:
   ```rust
   let mut fonts = egui::FontDefinitions::default();

   fonts.font_data.insert(
       "noto_sans".to_owned(),
       egui::FontData::from_static(include_bytes!("../assets/NotoSans-Regular.ttf")),
   );
   fonts.font_data.insert(
       "noto_cjk".to_owned(),
       egui::FontData::from_static(include_bytes!("../assets/NotoSansCJK-Regular.ttc")),
   );

   // Configure fallback order
   fonts.families.get_mut(&egui::FontFamily::Proportional)
       .unwrap()
       .insert(0, "noto_sans".to_owned());
   fonts.families.get_mut(&egui::FontFamily::Proportional)
       .unwrap()
       .push("noto_cjk".to_owned());

   ctx.set_fonts(fonts);
   ```

---

### 6. Graphics Backend: `glow` vs `wgpu`

When adding `eframe` to `Cargo.toml`, you must choose a rendering backend:

* **Use `glow` (OpenGL) if:** Your highest priority is supporting legacy hardware, budget Linux VMs, or older Mesa drivers. `glow` has the smallest binary overhead and the highest chance of working on low-end Linux setups.
* **Use `wgpu` (Metal / Vulkan / DX12) if:** You want the most modern, future-proof graphics pipeline with smooth tearing-free VSync on Windows and macOS, and robust multi-monitor DPI scaling.

---

### 7. Deployment & OS Quirks Checklist

* **Hide Windows Console Window:** By default, Rust desktop apps launch a command prompt alongside the window. Add this line to the very top of your `main.rs`:
  ```rust
  #![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]
  ```
* **Wayland Floating Position:** Under Wayland on Linux, security boundaries prevent client apps from freely setting their absolute screen coordinates (unlike X11 or Windows). Your floating popover will rely on the window manager or user cursor position for placement.
* **Packaging:** Distribute as:
  * **Windows:** A single `.exe` (or wrapped via InnoSetup/NSIS for autostart shortcuts).
  * **macOS:** A standard `.app` bundle with `LSUIElement = true` in `Info.plist` (this tells macOS the app is an agent/tray utility and hides it from the Dock).
  * **Linux:** An **AppImage** or **Flatpak** (ensures graphics/font libraries remain bundled and isolated from distro differences).

