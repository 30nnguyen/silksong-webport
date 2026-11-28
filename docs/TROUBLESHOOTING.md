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
    - **Import file / Drag & Drop**: Imports a `.dat` (or compatible `.json`) save directly into the selected slot, syncs it to IndexedDB (`IDBFS`), and refreshes the save slot live with **no browser tab reload** (if overwriting your currently active slot in gameplay, an inline **Return to Title & Load Save** button lets you switch cleanly without reloading).
  - **Native In-Game Rewind (`↺`)**:
    - Every committed bench/save write automatically creates a rolling native restore point (up to 20 per slot, preserving `NODEL*` milestone points).
    - Click the **`↺` (Rewind)** icon directly beneath any save slot on the title menu's Save Select screen to browse and restore `AUTO SAVES`.
  - **Offline Play**:
    - **Cache Full Game**: Downloads and SHA-256 verifies all 210 game files (shell, WASM engine, 165 Addressables bundles, and 26 cutscenes) into browser `CacheStorage` for offline play. *(Requires opening https://d1qh2ou2210k6v.cloudfront.net/ directly in a tab — browsers block Service Workers inside local `file://` iframes.)*
    - **Verify**: Audits cached file integrity against the release manifests.
    - **↻ Refresh Files**: Fetches fresh manifests and updates cached files in-place without reloading the tab.
    - **Clear Cache** (red, 2-click confirm): Deletes cached game files to free disk quota while keeping all your saves untouched.
  - **Memory & Display**:
    - **✦ Free Memory**: Reclaims unused Unity assets, runs garbage collection, and evicts unreferenced cached bundles in-place without reloading the tab.
    - **Scale**: Adjusts canvas render resolution live (`0.5×`, `0.75×`, `0.85×`, or `1.0×` native DPR, no reload required) while keeping full-resolution source textures intact. Lowering to `0.75×` or `0.5×` helps integrated/low-end GPUs maintain 60 FPS.
    - **HUD**: Toggle between **Compact**, **Detailed**, **Warnings**, or **Hidden** (invisible at rest; reveals on bottom-right corner hover).
    - **Fullscreen**: Enters fullscreen on `document.documentElement` so the Settings drawer remains visible, and locks **`Escape`** (Chromium Keyboard Lock API) so pressing `Escape` pauses the game instead of exiting fullscreen *(hold `Escape` for ~2s to exit fullscreen)*.
    - **Diagnostics**: Opens crash and performance telemetry (auto-classified crash verdict, current/recent room trail, WebGPU/WebGL GPU adapter & device-lost tracking, WASM + MEMFS + Data total memory breakdown, failed asset fetches, and engine errors) with 1-click **Report on GitHub (Auto-Filled)**, **Copy Report**, **Download `.txt` / `.json`**, and a **Last Crash** tab so crash logs aren't lost after reloading.

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

### Low FPS or crashes on Chromebooks / 4–8 GB RAM laptops
1. Open Settings (`Ctrl + Shift + S`) → set **Scale** to **`0.75×`** or **`0.5×`**.
2. Close other browser tabs/apps — Chromebooks and laptops with integrated GPUs share system RAM between the OS, WASM heap, cached bundles, and GPU VRAM.
3. Click **✦ Free Memory** in Settings before entering major new areas or after boss fights.
4. Ensure browser **Hardware Acceleration** is enabled (`chrome://settings/system`).

---

## 4. How to Report a Bug

Found something broken, missing, silent, or crashing compared to the native Windows release? **Please let us know!**

1. **Fastest (1-click auto-filled)**:
   - If the game crashed, click **Report on GitHub (Auto-Filled)** directly on the **SESSION CRASH REPORT** popup (or press `Ctrl + Shift + S` → **Report Bug** / **Diagnostics** → **Last Crash**).
   - This automatically fills in your room/scene, graphics backend, OS/GPU/RAM, automatic root-cause verdict, memory breakdown, and recent errors — AND copies the full log to your clipboard.
2. **Manual**: Go to the **[Issues tab](https://github.com/30nnguyen/silksong-webport/issues/new/choose)** and pick either:
   - **Bug Report (Gameplay / Crash / Save / Memory)**, or
   - **Visual / Shader / Audio / Cutscene Bug**.
3. If possible, also drag & drop a **screenshot** (for visual glitches) or a **`.dat` backup** (Settings → **Backup `.dat`**) right before the bug so we can reproduce the exact frame in an isolated test.
