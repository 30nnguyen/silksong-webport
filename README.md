# Hollow Knight: Silksong — Browser Web Port (WebGPU / WebGL 2)

> **Play Now:** **[https://30nnguyen.github.io/silksong-webport/](https://30nnguyen.github.io/silksong-webport/)**
>
> **Found a bug, visual glitch, silent cue, or softlock?** **[Open a Bug Report](https://github.com/30nnguyen/silksong-webport/issues/new/choose)** — please let us know so we can reproduce and fix it!

A deeply engineered, full-resolution **WebGPU & WebGL 2** browser port of *Hollow Knight: Silksong* running on Unity WebAssembly (`wasm32`) with native-parity shaders, complete audio & 1080p cutscenes, **65.5–80.0% free WASM OOM headroom**, native in-game **Rewind (`↺`)**, SHA-256-verified **Offline Play**, and a clean, ad-free dark monospace interface.

> [!NOTE]
> This repository serves as the **public documentation hub, engineering changelog, and bug tracker** for the web port. To prevent unauthorized re-hosting or tampering, source code and game binaries are intentionally omitted from this repository; the playable build is deployed automatically to GitHub Pages at **[30nnguyen.github.io/silksong-webport](https://30nnguyen.github.io/silksong-webport/)**.

---

## Highlights at a Glance

| Pillar | What Was Achieved |
| --- | --- |
| **Memory & 32-bit WASM OOM** | Raised gameplay free WASM headroom from **21.2%** (`3,467.5 / 4,080 MiB`, 3 heap grows) to **65.5–67.2%** across 15 biomes (**80.0%** at Title, **`0` WASM heap grows**), eliminated a linear **`+89 Mesh / room-transition`** native leak (`+0` across 8 round trips), cut WASM `MemoryFile` residency by **-75.0%**, and trimmed **310.42 MiB** of dead/uncompressed bundle bloat. |
| **Whole-Corpus Graphics & Shaders** | Full-resolution textures (**2,808/2,808** verified payloads, restoring **294** reduced textures & **773** sprites). Repaired HLSLcc `-0.0` fog/mask inversion, `Sprites/Lit` & `Default-ColorFlash` math/variants, `Darkness Sprite`, `UIBlur`, `UI/BlendModes/Multiply` & 28 `GrabPass`es, native 3-pass Film Grain, Mount Fay ice refraction, camera blur/darkness target feedback, positional red-wash grade markers, and **5,970** baked black-tile meshes across **549** rooms (**`0/255` pixel delta** vs. Windows DXBC on WebGPU & WebGL 2). |
| **Complete Audio & 1080p Video** | Restored **14,270** erased `AudioSource` clip references across **552** rooms, **10** standalone scenes & **6** prefab bundles, **77** Trobbio stage cues, **7** Vaultkeeper Psalm Cylinder catalog keys, and the final **5** native-only FSB5 recordings (decoded & transcoded to AAC). Rebuilt Web Audio clock/MP4 duration/loop scheduling and hardened WebGPU/WebGL `<video>` playback across all **26** full-res `1920×1080` cutscenes (including Act 3 & all endings). |
| **Scene Lifecycle & Gameplay** | Audited all **190,363** WASM functions to close direct-call hook bypasses; fixed Bell Beast & High Halls Arena stranded gates across all **23** additive subscenes in **574** rooms; fixed single-threaded WebGL `Task.Run` deadlock blocking native **Rewind (`↺`)**; fixed crest-switch-close corruption, 2D Pointer Lock trapping, Fullscreen overlays, stuck keys, native PS4/PS5/Switch/Xbox WebGL gamepad detection, and 60 FPS frame pacing. |
| **Saves, Native Rewind & Offline Play** | Multi-tab-safe `IDBFS` mutation journal (prevents silent cross-tab save rollback), `shared.dat` & `PlayerPrefs` persistence, off-main-thread Web Worker save codec (**886.5 ms → 20.8 ms** worst main-thread stall), automatic per-slot native `Restore_Points` Rewind (`↺`) checkpoints, and a SHA-256-verified Service Worker offline cache with per-tab release pinning and chunked hashing (**140.9 ms → 18.7 ms** worst frame). |
| **Minimalist UI & Zero-Black-Screen Boot** | Unified flat dark monospace UI (`Ctrl+Shift+S` or bottom-right corner hover pill), zero-black-screen single-source-of-truth linear ring-locked boot loader (eliminating the ~30s black screen and 84% hard-refresh stall), complete third-party ad/splash purge, and authentic Team Cherry title restoration. |

---

## Everything We Built & Fixed

For exhaustive root-cause forensics and exact numbers, read the **[Technical Deep Dive](docs/TECHNICAL_DEEP_DIVE.md)** and the chronological **[Engineering Changelog](docs/CHANGELOG.md)**.

### 1. Memory Architecture & 32-bit WASM OOM Breakthrough
Browser WebAssembly (`wasm32`) has a hard **4,080 MiB** linear address space ceiling (`4,278,190,080` bytes). Early builds ran at **~3.58 GiB** and crashed on save serialization (`256 MiB` contiguous allocation failure). Rather than sacrificing visual fidelity by downscaling textures, we redesigned asset residency and fixed native leaks:
- **File-Backed MEMFS Streaming for Persistent Bundles**: Moved the full **574-scene archive** (`483 MiB`) and the **Top 9 largest persistent `_GameCameras` AssetBundles** (`1,302.67 MiB`) out of WASM `MemoryFile` blocks into zero-copy JS-owned MEMFS buffers materialized on demand with deduplicated concurrent fetching and retry logic — cutting WASM `MemoryFile` residency by **-75.0%** (`2,032.5 MiB → 507.4 MiB`).
- **-310.42 MiB Bundle Compaction**: Stripped **213.7 MiB** of unreachable embedded `VideoClip` stream bytes from `packed-cinematics` (`217.70 MiB → 4.94 MiB`, keeping all 17 `AudioClip`s bit-exact and all 26 `VideoClip` metadata headers intact since videos stream as `.webm`), and LZ4HC-recompressed the sole uncompressed UI sprite bundle (`114.27 MiB → 16.61 MiB`, internal serialized files SHA-256 bit-exact).
- **`+89 Mesh / Round-Trip` Native Leak Eliminated**: Fixed a linear native `Mesh` leak across all 8 `tk2d*Sprite` `Awake`/`Build` paths where overwritten meshes kept `HideFlags.DontUnloadUnusedAsset` (`52 → 20`). Across 8 consecutive `Hang_04 ↔ Hang_06` room transitions, live `Mesh` count stayed flat (`1,182 → 1,182 → 1,182`, **`+0` leak**) and returned to the exact Title baseline (`38`) on Quit to Menu.
- **Allocator & Fetch Hygiene**: Capped Emscripten's speculative `_emscripten_resize_heap` over-grow (`+512 MiB → +128 MiB`) with exact page-aligned fallback, reused the `82.57 MB` decoded WASM buffer at startup, eliminated a `2.6 GiB` `res.clone()` response-body stream tee during boot progress tracking, and corrected an off-by-one `dlmalloc` `_gm_.top` telemetry pointer + bounded treebins free-block accounting in `crashtrack.js`.

### 2. Whole-Corpus Graphics & Shader Parity (WebGPU & WebGL 2)
Every visual discrepancy reported against the native Windows release was traced to its first divergent GPU draw, shader instruction, or asset lifetime boundary:
- **Baked Black-Tile Cover Lifetime (549 Rooms / 5,970 Meshes)**: Room re-entry previously exposed blue/cyan ceiling and bridge openings because the Web export stored baked 2D tile meshes in shared assets that `tk2dTileMap.OnDestroy` permanently destroyed via `DestroyImmediate`. Recording baked source meshes at `Awake`, detaching references before teardown, and unloading through `Resources.UnloadAsset` once the last owner leaves restored **2,914,887 opaque black quads** across **549 rooms**.
- **Whole-Corpus Shader Parity (`0/255` Pixel Delta vs. Windows DXBC)**:
  - Fixed the HLSLcc `itof(-lt)` `-0.0` sign-inversion bug in GLES3 & WGSL across all 4 affected game shaders (`Custom/Tiled Scrolling Fog`, `Sprites/Tiled Scrolling Masked`, `Sprites/Tiled Masked`, `Custom/LavaBase`), eliminating the Whispering Vaults (`Library_09`) cyan-teal exterior void cutout box on both WebGPU and WebGL 2.
  - Built a bit-exact Unity 6000 shader parameter-blob and GLES3/WGSL subprogram codec to restore stripped variants and math parity across **`Sprites/Lit`** (201 materials: ambient scale `×2.0`, unclamped `_SaturationLerp`, `BLACKTHREAD` ordering & dithering), **`Sprites/Default-ColorFlash`** (406 materials: `_StencilRef`/`_StencilComp`, 7 missing `BLACKTHREAD` multi-keyword combos, `RECOLOUR` across all 16 combos), **`Sprites/Darkness Sprite`** (4 non-dithering variants), **`UI/Blur/UIBlur`** (4 dropped properties, `ZWrite Off`, `USE_MASK`, `BLUR_PLANE`, uniform buffer sizing), **`UI/BlendModes/Multiply`** (clean 2-pass `GrabPass` + `Multiply`), **28** `UI/BlendModes/Optimized/*` `GrabPass` passes + 75 pass states, **`Sprites/Stencil-Write`**, and **6** sibling dithering shaders. All **86/86** GLES3 and **86/86** Dawn WebGPU pipelines compile with zero errors.
- **Native Post-Processing, Refraction & Lighting Fixes**:
  - Restored native **3-pass Film Grain** (replacing a passthrough stub; `Options → Video → Noise` matches Windows within ±1/255 on both backends), native **Bloom** weights/orientation/stack cleanup, **Camera Light Blur** shader & `Shader.Find` registration, and **Camera Target Lifecycle** synchronization (eliminating 3 WebGL `_DarknessCutout` / world-blur render-target feedback conflicts).
  - Fixed **Mount Fay ice refraction** (`_SpriteRefractionGrab` named grab pass across 117 objects in 7 rooms, removing the white square at Shakra's bench and gray box behind shop menus).
  - Fixed **Underworks sharp green fog edges** (`ETC1_EXTERNAL_ALPHA` channel swizzle across 222 renderers in 34 rooms), **positional whole-screen red wash** (`GradeMarker.Update` deferred until async zone profile loads across 96 markers in 86 rooms), **stencil filtering** (`Custom/ScrollTexture`, 447 references in 34 rooms), **Hunter arc fade** (`Custom/RadialFill`), **fog dithering** (20 programs restored), and **Surface lighting profile ordering** (`Cradle_Destroyed_Challenge_01`).
- **Full-Resolution Asset Restoration**: Audited all **2,808** serialized textures across all 165 bundles + 37 bootstrap textures and restored **294** downscaled textures together with **773** coupled `Sprite` records and all **26** full-resolution video clips.

### 3. Complete Audio & 1080p Cutscene Parity
- **14,270 Restored `AudioSource` Bindings**: Discovered that the original WebGL export silently erased both `m_audioClip` and `m_Resource` pointers across thousands of `AudioSource` components. Restored **77** Trobbio / Tormented Trobbio stage cues plus **14,270** additional sources (**13,560** in **552** gameplay rooms, **182** in **10** standalone scenes, **528** in **6** prefab bundles) mapped to **1,188** shipped recordings.
- **Vaultkeeper's Melody Progression Fix**: Repaired 7 regenerated audio GUID aliases in `catalog.bin` so all Psalm Cylinders and the Sacred Cylinder / Needolin sequence load, play, grant **Vaultkeeper's Melody** (`HasMelodyLibrarian`), and persist cleanly.
- **Final 5 Native-Only FSB5 Clips**: Decoded, AAC-transcoded, and injected the 5 recordings absent from the Web export (`boss_stun_ss_loop`, `ui_silk_chunk_regenerating_loop_2d`, `gg_shadelord_screen_fill`, `boss_stun`, `poison_damage_tag`), and completed a 100% audit of the 131 Windows-only `AudioSource` owners (confirming all 131 belong strictly to 4 post-Web-export Windows patch additions; zero silent Web `AudioSource` defects remain).
- **Web Audio & Video Runtime Bridges**: Added early MP4 header duration parsing (eliminating zero-length reads before decode/metadata), accurate media playhead tracking, decoded loop bounds, Web Audio clock scheduling, `keydown` + gamepad audio unlock, and 9 bridge stability fixes. Hardened `_JS_WebGPU_ImportVideoTexture` & `_JS_Video_*` across all **26** cutscenes (`1920×1080`, including Act 3 intro into `Song_Tower_Destroyed`, Mr. Mushroom, and all 5 endings).

### 4. Scene Transitions, Gameplay & Input Hardening
- **High Halls Arena, Bell Beast & All 23 Additive Subscenes**: Fixed stranded boss subscenes (such as Bell Beast's `Bone_05_boss` leaking into Moss Grotto or High Halls' `Hang_04_boss` `Battle Gate` blocking `Hang_06`'s right exit) by deferring Addressables memory release from PreFetch to PostActivation, removing an out-of-bounds IL2CPP heap write on `SceneAdditiveLoadConditional.Unload`, hooking `OnDisable` & `LoadRoutine::MoveNext` for late-completion orphan sweeps across all **23** additive subscenes in **574** rooms, and sweeping orphans on Quit to Menu.
- **Direct-WASM-Call Bypass Audit (190,363 Functions)**: Audited every defined function in `w-pt.wasm` against runtime hook slots and migrated bypassed private slots to live virtual/indirect roots for tk2d/PlayMaker completion parity (`ROOT A`, fixing animation-callback softlocks like crests, spools, arena walls, Widow, and Grand Mother Silk) and scene-departure reconciliation.
- **Crest-Switch-Close Guard**: Intercepts closing the Tools pane mid-crest-switch to stop active transition coroutines, revert unconfirmed crest selection, and reset visual fade groups cleanly (eliminating the historic coral `#ec837a` world-overlay corruption).
- **Native In-Game Rewind (`↺`) Deadlock Fix**: Resolved an infinite spinner when clicking Rewind (`↺`) on WebGL caused by `FetchDataRequest<T>` dispatching file decode onto `Task.Run` (unavailable in single-threaded WebGL) by executing the decode synchronously on the main thread.
- **Controls, Gamepads, Fullscreen & 60 FPS Cap**: Disabled 2D Pointer Lock trapping (keeping hardware cursor hidden while allowing direct clicks on UI), promoted `document.documentElement` in fullscreen so the Settings drawer renders in the top layer, stopped swallowing key/pointer release events when panels open (eliminating stuck-key movement drift), enabled native WebGL gamepad detection (PS4, PS5, Switch Pro, Xbox), added a 60 FPS frame-pacing cap on high-refresh displays, and added Pause-on-Blur suspension.

### 5. Save Durability, Native Rewind (`↺`) & Offline-First Play
- **Multi-Tab Safe `IDBFS` Persistence**: Built a local mutation journal in `IDBFS` so stale browser tabs can never silently roll back or wipe another tab's progress, extended checkpoint persistence to `shared.dat`, `shared.dat.bak*`, `PlayerPrefs`, and `.del` markers, and hooked completed bench/rename writes so progress survives abrupt tab closure even without Quitting to Menu.
- **Off-Main-Thread Save Codec & Native Rewind Writer**: Moved AES-256-ECB save encoding/decoding into a reusable Web Worker with stall recovery (**886.5 ms → 20.8 ms** worst main-thread frame hitch). Every committed `user{1..4}.dat` write automatically produces a rolling native `Restore_Points{slot}/restoreData{N}.dat` checkpoint (up to 20 per slot, preserving `NODEL*` milestones) accessible from the game's built-in **Rewind (`↺`)** button.
- **SHA-256 Verified Offline Service Worker**: Per-tab shell & content version pinning across SW restarts, online-probed legacy cache self-healing, retired-version quota reclamation, cooperative time-sliced SHA-256 hashing (**140.9 ms → 18.7 ms** worst frame during background caching, zero >50 ms frames), and coalesced progress reporting.

### 6. Minimalist UI, Linear Boot Loader & Ad Purge
- **Zero-Black-Screen, Ring-Locked Boot Loader**: Hooked engine lifecycle milestones (`StartManager`, scene load state, `UIManager`) and replaced jumpy progress bars with a single monotonic `requestAnimationFrame` controller that keeps the circular ring and numeric percentage locked on the same frame, handles Unity's two-pass preload cleanly, avoids the ~30s black screen and `84%` hard-refresh pause, and never obscures Language Select or the Team Cherry intro.
- **Flat Dark Monospace UI & Single Corner-Reveal Pill**: Unified boot overlay, Settings drawer (`Slot 1–4`, `Backup .dat`, `Import file`, `Offline Play`, `Scale`, `HUD`, `Fullscreen`, `Diagnostics`), pause veil, and crash diagnostics under one clean flat dark monospace design. Replaced redundant floating buttons with a single bottom-right pill (`opacity: 0; pointer-events: none` at rest when HUD is Hidden; smoothly reveals on bottom-right corner hover or `Ctrl+Shift+S`).
- **Ad Purge & Authentic Attribution**: Removed all third-party ad splashes and placeholder domains from Unity startup data and UI sprite bundles while preserving `/idbfs/` save directory paths, and restored authentic Team Cherry title assets cleanly.

---

## Controls Quick Reference

| Action | Default Keyboard | Gamepad |
| --- | --- | --- |
| Move | `Arrow Keys` | Left Stick / D-Pad |
| Jump | `Z` | South (`A` / `Cross` / `B`) |
| Attack | `X` | West (`X` / `Square` / `Y`) |
| Dash / Sprint | `C` | Right Trigger / `R2` |
| Bind / Heal / Cancel | `A` | Circle / `B` / `L1` |
| Needolin / Crest Switch (at bench) | `D` | `Y` / `Triangle` |
| Quick Map | `Tab` | `LB` / `L1` |
| Inventory / Crest & Tools Pane | `I` / `Q` | Select / Touchpad |
| Pause Menu | `Escape` | Start / Options |
| **Web Port Settings / Saves / Diagnostics** | **`Ctrl + Shift + S`** (or hover **bottom-right corner**) | — |

*(All in-game keybindings can also be customized inside Silksong's native Options → Keyboard / Controller menu.)*

---

## Verification & Quality Bar

Before shipping, every major milestone was validated through automated headless Chromium (WebGPU + WebGL 2) and unit-contract suites:
- **15-Biome Visual & Memory Sweep**: 12 WebGPU biomes (`Hang_04 → Hang_06`, `Library_09`, `Song_10`, `Song_18`, `Ward_01`, `Cog_04`, `Peak_04`, `Under_04`, `Bone_05`, `Bonetown`, `Cradle_03`, `Abyss_Cocoon`) + 3 WebGL 2 biomes — **15/15 passed** at **65.7–67.2% OOM Left**, **`0` WASM grows**, **`0.0000%` magenta pixels**, **`0` GPU validation errors**, **`0` page errors**.
- **Endurance & Leak Check**: 8 consecutive `Hang_04 ↔ Hang_06` room round trips + Quit to Menu — **`0` WASM grows**, **`+0` Mesh leak** (`1,182 → 1,182 → 38`), `layoutVerified: true` at every checkpoint.
- **Progression, Rewind & Cutscenes**: Verified Moss Mother, Bell Beast, and Fourth Chorus combat, Cocoon break & Silk Bind heal, Vaultkeeper's Melody + Save/Quit, mid-crest-switch pane close, native Rewind (`↺`) `AUTO SAVES` loading, no-quit bench cold-reload persistence, Act 3 intro (`Act_03_Intro.webm` `1920×1080` → `Song_Tower_Destroyed`), and all 7 ending/secret `.webm` clips at `1920×1080`.

---

## Found a Bug? Let Us Know!

Despite extensive automated testing across 15+ biomes and 574 scenes, *Hollow Knight: Silksong* is enormous — some late-game bosses, rare transitions, specific hardware/driver combinations, or individual room cues may still have edge cases.

**If you spot anything broken, visually off, silent, or crashing, please report it:**
1. Open an issue via **[New Issue](https://github.com/30nnguyen/silksong-webport/issues/new/choose)**.
2. Choose **Bug Report** (crashes, OOM, softlocks, saves, controls) or **Visual / Audio Bug** (rendering, lighting, tiles, sound, cutscenes).
3. Include the **room/boss name**, your **browser & GPU**, a **screenshot** (if visual), and optionally the **Diagnostics** text (`Ctrl + Shift + S` → **Diagnostics**) or a **`.dat` save backup** right before the issue.

See **[docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)** for browser recommendations, offline cache tips, and save management help.

---

## Documentation Index

- **[Technical Deep Dive](docs/TECHNICAL_DEEP_DIVE.md)** — Full engineering breakdown of memory, shaders, audio/video, gameplay/additive scenes, saves/rewind/offline caching, and UI/loader architecture.
- **[Chronological Changelog](docs/CHANGELOG.md)** — Complete milestone-by-milestone history of every fix from initial port foundation to the latest release.
- **[Troubleshooting & Bug Reporting Guide](docs/TROUBLESHOOTING.md)** — Browser support, settings drawer reference, common fixes, and how to collect diagnostics.

---

## Disclaimer & Credits

- *Hollow Knight: Silksong* is created and owned by **Team Cherry**. Please support the official release!
- Browser porting, runtime engineering, shader/audio parity, memory architecture, save/rewind systems, and UI maintained by **[@30nnguyen](https://github.com/30nnguyen)**.
