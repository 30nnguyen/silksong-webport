# Troubleshooting, Browser Setup & Bug Reporting Guide

If something looks wrong, sounds silent, stutters, or crashes while playing on **[d1qh2ou2210k6v.cloudfront.net](https://d1qh2ou2210k6v.cloudfront.net/)**, check the tips below and open a GitHub Issue so we can fix it.

---

## 1. Recommended Browsers & Hardware

| Environment | Status | Notes |
| --- | --- | --- |
| **Chrome / Edge (Desktop, 64-bit) — WebGPU** | **Best (Primary Target)** | Full shader parity, native 60 FPS cap, lowest CPU draw-call overhead, verified across all 15+ tested biome families. |
| **Chrome / Edge / Firefox (Desktop, 64-bit) — WebGL 2** | **Supported** | Full GLES3 shader parity (including fog/masking, blur, refraction, Film Grain, and Darkness Sprite variants). |
| **Safari / WebKit** | Experimental | WebGPU/WebAudio/IDBFS quirks may vary by macOS/iOS version. Desktop Chromium is strongly recommended. |
| **Mobile Browsers** | Not Recommended | Browser tab memory caps on mobile OSes are often below the ~2–3 GB combined WASM + JS/GPU footprint required by the full-resolution game. |

- **System RAM**: 8 GB minimum recommended (16 GB+ comfortable for multitasking alongside the browser tab).
- **Disk Space (Optional Offline Play)**: ~2.45 GB free browser storage quota if you enable **Offline Play → Cache Full Game**.

---

## 2. Opening Settings, Saves, Rewind & Diagnostics

- **Open Settings Drawer**: Hover your mouse over the **bottom-right corner** of the screen to reveal the corner HUD pill and click it, or press **`Ctrl + Shift + S`** at any time.
- **Settings Controls**:
  - **Save Slots (`Slot 1–4`)**:
    - **Backup `.dat`**: Downloads the encrypted native-compatible `.dat` save file for that slot.
    - **Import file / Drag & Drop**: Imports a `.dat` (or compatible `.json`) save directly into the selected slot and syncs it to IndexedDB (`IDBFS`).
  - **Native In-Game Rewind (`↺`)**:
    - Every committed bench/save write automatically creates a rolling native restore point (up to 20 per slot, preserving `NODEL*` milestone points).
    - Click the **`↺` (Rewind)** icon directly beneath any save slot on the title menu's Save Select screen to browse and restore `AUTO SAVES`.
  - **Offline Play**:
    - **Cache Full Game**: Downloads and SHA-256 verifies all 210 game files (shell, WASM engine, 165 Addressables bundles, and 26 cutscenes) into browser `CacheStorage` for offline play.
    - **Verify**: Audits cached file integrity against the release manifests.
  - **Memory & Display**:
    - **Scale**: Adjusts canvas render resolution (`0.5×`, `0.75×`, `0.85×`, or `1.0×` native DPR, keeping full-resolution source textures intact). Lowering to `0.75×` or `0.5×` helps integrated/low-end GPUs maintain 60 FPS.
    - **HUD**: Toggle between **Hidden** (invisible at rest; reveals on bottom-right corner hover), **Warnings**, or **Always Visible**.
    - **Fullscreen**: Enters fullscreen on `document.documentElement` so the Settings drawer remains usable in fullscreen.
    - **Diagnostics**: Opens crash/memory telemetry (live WASM heap, free headroom, allocator layout status, recent errors) so you can copy-paste it into a bug report.

---

## 3. Common Issues & Quick Fixes

### Audio doesn't play immediately on boot
Browsers block Web Audio until the first user gesture. Press any keyboard key, click the canvas, or press any gamepad button — the audio bridge automatically unlocks and resumes queued cues.

### Game freezes when switching tabs or minimizing
This is intentional (**Pause-on-Blur**). The engine suspends ticks and Web Audio while unfocused so you don't take damage or burn CPU/battery in the background. Click back into the window to resume instantly.

### Controller / Gamepad not detected
Press any button on your controller (Xbox, PlayStation DualShock 4 / DualSense, Nintendo Switch Pro) after the page loads so the browser's Gamepad API exposes the device to the runtime. Native WebGL platform detection is enabled so standard controller layouts map properly.

### Stuck on an old cached version after an update
Hard-refresh the tab (**`Ctrl + Shift + R`** on Windows/Linux, **`Cmd + Shift + R`** on macOS). The Service Worker pins active tabs to consistent releases and automatically promotes updated verified shells on fresh navigation.

### Low FPS on integrated GPU / laptop
1. Open Settings (`Ctrl + Shift + S`) → set **Scale** to **`0.75×`** or **`0.5×`**.
2. Ensure browser **Hardware Acceleration** is enabled in Chrome/Edge settings (`chrome://settings/system`).
3. Close other heavy GPU/WebGL tabs so the browser isn't competing for VRAM.

---

## 4. How to Report a Bug

Found something broken, missing, silent, or visually off compared to the native Windows release? **Please let us know!**

1. Go to the **[Issues tab](https://github.com/30nnguyen/silksong-webport/issues/new/choose)** and pick either:
   - **Bug Report (Gameplay / Crash / Save / Memory)**, or
   - **Visual / Shader / Audio / Cutscene Bug**.
2. Include:
   - The **exact room / boss / menu / cue** where it occurred.
   - Your **browser, OS, GPU, and backend (WebGPU vs WebGL 2)**.
   - **Steps to reproduce** (especially whether it happens on first room entry or after leaving & re-entering).
   - A **screenshot** (for visual glitches) and, if possible, a **`.dat` backup** (Settings → **Backup `.dat`**) right before the bug so we can reproduce the exact frame in an isolated test.
   - Any **Diagnostics** output (`Ctrl + Shift + S` → **Diagnostics**) or red errors from DevTools Console (`F12`).
