# Technical Deep Dive — Silksong Browser Web Port

This document explains the root causes, engineering solutions, and measured verification results across all seven major engineering pillars of the *Hollow Knight: Silksong* WebGPU / WebGL 2 browser port.

---

## 1. Memory Architecture & 32-bit WASM OOM Breakthrough

### The Fundamental Constraint
Unity WebGL targets 32-bit WebAssembly (`wasm32`), which caps linear memory at **4,080 MiB** (`4,278,190,080` bytes). The full-resolution game ships **165 Addressables AssetBundles** (574 gameplay scenes, scenery/prop atlases, Hornet animations, music/voices/SFX, prefabs, fonts) plus **26 cutscenes**. Earlier browser ports either suffered frequent out-of-memory crashes near `~3.58 GiB` (especially when Unity allocated a `256 MiB` contiguous buffer during save serialization) or aggressively downscaled textures to `0.25×` and stripped audio to stay under the ceiling.

Our goal was **full native resolution AND full audio** with generous long-session OOM headroom.

### Root Causes & Architectural Fixes

| Problem | Root Cause | Solution | Measured Impact |
| --- | --- | --- | --- |
| **WASM `MemoryFile` exhaustion** | Persistent `_GameCameras` AssetBundles and the `483 MiB` 574-scene archive were loaded through `UnityWebRequest` into WASM linear memory (`csize=8216` `MemoryFile` blocks consuming `2,032.5 MiB` inside the 4 GiB wall). | Moved the 574-scene archive and the **Top 9 largest persistent `_GameCameras` AssetBundles** (`1,302.67 MiB`: `packed-collections`, `packed-sprites-scenery`, `packed-audio-hornetmusic`, `packed-prefabs`, `packed-animations-props`, `packed-audio-hornetvoices`, `packed-audio-sfx`, `packed-animations-hornet`, `packed-fonts`) to file-backed JS-owned MEMFS buffers materialized on demand (`canOwn=true`, deduplicated concurrent fetch with retries, `UnityFS` header validation). | Cut WASM `MemoryFile` residency by **-75.0%** (`2,032.5 MiB → 507.4 MiB`), balancing WASM (`~1.35 GiB / 4.08 GiB`) and JS (`~1.96 GiB / 4.19 GiB`) well below 50% of both ceilings. |
| **Linear native `Mesh` leak (`+89 / transition`)** | All 8 `tk2d*Sprite` `Awake`/`Build` code paths created procedural meshes stamped with `HideFlags.DontUnloadUnusedAsset` (`52`) and overwrote `this.mesh` without destroying the previous mesh, leaking **+89 native `Mesh` objects per room round-trip** (`+965` surviving Quit to Menu). | Intercepted all 8 `tk2d*Sprite` `Awake`/`Build` entry points to destroy overwritten meshes via `Object::Destroy` and clear `DontUnloadUnusedAsset` (`52 → 20`) via `Object::set_hideFlags`. | Across 8 consecutive `Hang_04 ↔ Hang_06` transitions, `Mesh` count stayed flat (`1,182 → 1,182 → 1,182`, **`+0` leak**) and returned to exact baseline (`38`) on Quit to Menu. |
| **310.42 MiB dead & uncompressed bundle bloat** | `packed-cinematics` retained `213.7 MiB` of unreachable embedded `VideoClip` stream bytes even though WebGL streams all 26 cutscenes from external `.webm` files; `packed-sprites-ui` was shipped completely uncompressed (`114.27 MiB`). | Stripped unreachable `VideoClip` stream payloads from `packed-cinematics` (`217.70 MiB → 4.94 MiB`, preserving all 17 `AudioClip`s bit-exact & all 26 `VideoClip` metadata headers) and LZ4HC-recompressed `packed-sprites-ui` (`114.27 MiB → 16.61 MiB`, bit-exact internal `SerializedFile`). | **-310.42 MiB** saved in download transfer, browser offline cache, and archive footprint. |
| **Speculative heap over-grow & startup clone** | Emscripten's `_emscripten_resize_heap` speculatively requested up to `+512 MiB` extra per grow, fragmenting browser address space; startup also duplicated the `82.57 MB` decoded WASM buffer and boot progress tracking called `res.clone()` across `2.6 GiB` of Addressables streams. | Capped speculative grow headroom to `+128 MiB` with an exact page-aligned grow fallback; reused the decoded WASM buffer at instantiation; switched boot progress observation to `Content-Length` headers only (zero response-body cloning). | **`0` WASM heap grows** across all 15 tested biomes and 8-round-trip endurance tests (flat `2,048.0 MiB` committed WASM). |
| **Allocator telemetry bug & 1 Hz stall** | `crashtrack.js` originally ran a full native heap walk every second (`60–83 ms` normally, `267–423 ms` under 4× CPU throttling) and later read `_gm_.dv` (`+20`) instead of `_gm_.top` (`+24`), causing `layoutVerified` to turn `false` after large frees. | Switched periodic telemetry to validated `dlmalloc` globals with corrected `_gm_.top` offset and bounded treebins free-block traversal (≤ 256 nodes), keeping exact heap walks available on demand in Diagnostics. | Fast HUD tracks exact heap walk within **0.2 MiB**, `layoutVerified: true` at every checkpoint, and 4×-throttled frame p95 dropped from **350 ms → 83.3 ms**. |

**End Result:** Gameplay WASM free headroom rose from **21.2%** (`3,467.5 / 4,080 MiB`, 3 grows) to **65.5–67.2%** across 15 WebGPU/WebGL 2 biomes and **80.0%** at the Title Screen (`0` grows).

---

## 2. Whole-Corpus Graphics & Shader Parity (WebGPU + WebGL 2)

Every rendering bug was investigated by comparing decoded Windows DXBC shaders, serialized Unity materials/meshes, and live WebGPU/WebGL 2 draw traces rather than applying fragile visual hacks.

### Baked Black-Tile Cover Lifetime (549 Rooms / 5,970 Meshes)
- **Symptom**: Rooms like Moss Grotto (`Tut_01`) and Bone Bottom (`Bone_05`) rendered cleanly on initial entry, then dropped their opaque black ceiling/bridge cover meshes after leaving and re-entering the room — revealing blue/teal background gaps with zero GPU validation errors.
- **Root Cause**: The Web export serialized baked `tk2dTileMap` source meshes as shared AssetBundle assets (whereas native Windows scenes owned local scene instances). On room exit, `tk2dTileMap.OnDestroy → DestroyGameData → DestroyImmediate` permanently destroyed the shared asset pointers, leaving subsequent `Awake()` calls with null meshes.
- **Fix**: Tracked baked 2D tile source meshes at `Awake`, detached references prior to native teardown, and released them safely through `Resources.UnloadAsset` once the last tracked owner exits. Static auditing confirmed coverage across **5,970 meshes** (**2,914,887 opaque black quads**) in **549 rooms** (the remaining 41 packed scene entries are menus, cutscenes, or additive pieces).

### Whole-Corpus Shader Audit & Unity 6000 Parameter-Blob Codec
We built a bit-exact Unity 6000 shader parameter-blob and GLES3/WGSL subprogram codec (`458/458` parameter blobs and `516/516` subprograms verified bit-exact) to audit all 165 WebGL bundles + 69 built-in shaders against all 148 native Windows shaders:

- **HLSLcc `itof(-lt)` `-0.0` Sign-Inversion Bug**: Across 4 shared game shaders (`Custom/Tiled Scrolling Fog`, `Sprites/Tiled Scrolling Masked`, `Sprites/Tiled Masked`, `Custom/LavaBase`), HLSLcc miscompiled DXBC conditional sign-mask idioms as `-0.0` instead of `1.0` in GLES3 and WGSL. This caused `Remasker` exterior voids (notably Whispering Vaults `Library_09`) to flood with bright cyan-teal fog (`[6.48, 21.66, 22.66]`), creating an abrupt rectangular cutout around the playable shaft. Fixing all 4 shaders dropped exterior void pixels to `[0.00, 0.20, 0.00]` (WebGPU) / `[0.00, 0.74, 0.45]` (WebGL 2).
- **`Sprites/Lit` (201 materials)**: Fixed doubled ambient light scaling (`glstate_lightmodel_ambient * 4.0 → * 2.0`), unclamped `_SaturationLerp` ordering before ambient, `BLACKTHREAD` before `COLOR_FLASH`, and restored stripped `BLACKTHREAD` & `DITHERING_NOISE` variants.
- **`Sprites/Default-ColorFlash` (406 materials)**: Restored `_StencilRef` / `_StencilComp`, 7 missing `BLACKTHREAD` multi-keyword combinations, and `RECOLOUR` across all 16 variant combinations.
- **`Sprites/Darkness Sprite`**: Synthesized 4 missing non-dithering shader variants (`12/12` GLES3 + `12/12` Dawn WebGPU pipelines verified live with `DITHERING_NOISE` toggled off).
- **`UI/Blur/UIBlur` & Camera Light Blur**: Restored 4 dropped `m_PropInfo` properties, `ZWrite Off` on passes `1, 3, 5, 7, 9`, `USE_MASK` & `BLUR_PLANE` variants, WebGPU uniform buffer alignment, `BlurManager` / `LightBlurredBackground` components, and `Shader.Find` `ScriptMapper` registration.
- **`UI/BlendModes/Multiply` & Optimized UI Blend Modes**: Rebuilt `UI/BlendModes/Multiply` as a clean 2-pass `GrabPass` + `Multiply` shader and restored **28** `GrabPass { "_GUIBlendingSharedGT" }` passes + **75** pass states across `packed-guiblendmodes` plus `_StencilRef` in `Sprites/Stencil-Write`.
- **Dithering Parity**: Restored `DITHERING_NOISE` variants across 2 scrolling fog shaders (20 programs) and 6 sibling shaders (`Sprites/Cherry-Default`, `Sprites/Screen`, `Sprites/Tiled Masked`, `Sprites/Default Mix Channels`, `UI/BlendModes/LinearLight`, `UI/BlendModes/Screen`).
- **Pixel Verification**: All **86/86** GLES3 and **86/86** Dawn WebGPU `GPURenderPipeline`s compile with 0 errors, and all **8/8** Windows DXBC pixel controls match at **`maxDiff = 0/255`** on both backends.

### Additional Verified Rendering Fixes
- **Native 3-Pass Film Grain**: Replaced a 1-pass passthrough stub with 3 native-equivalent GLES3/WGSL passes (`Options → Video → Noise` matches Windows DXBC within ±1/255 across all 24 pixel controls).
- **Native Bloom & Uber Post-Processing**: Repaired camera-local Uber effect collection, brightness preview, WebGPU 208-byte uniform buffer declaration, Bloom texel-orientation predicate, near/far Sgx weights, numeric vs. string property overloads, temporary texture format handling, dual-camera isolation, and 48-byte native stack cleanup.
- **WebGL Render-Target Feedback Conflicts**: Initialized `_DarknessCutout` immediately after `Awake` and synchronized the world blur producer with its main-camera consumer, eliminating all 3 self-sampling render-target feedback errors during startup and loading.
- **Mount Fay Refraction (`GrabPassBatcher`)**: Restored the named `_SpriteRefractionGrab` grab pass across **117** refraction objects in **7** rooms, removing the white square at Shakra's bench and the gray box behind shop menus.
- **Underworks Sharp Green Fog Mask**: Fixed `ETC1_EXTERNAL_ALPHA` channel swizzle in `Sprites/Tiled Scrolling Masked` (opacity from alpha instead of red; RGB preserved) across **222** renderer references in **34** rooms.
- **Positional Whole-Room Red Wash (`GradeMarker`)**: In WebGL, `GradeMarker.Start` yielded on async zone profile loading while `Update` already published red placeholder curves into active keyframe pairs. Deferring `Update` until the zone profile resolves fixed **96** active zone markers across **86** rooms (**48** red-placeholder markers in **45** rooms).
- **Stencil Filtering, Hunter Arc Fade & Surface Lighting**: Bound `_StencilRef` in `Custom/ScrollTexture` (**447** references in **34** rooms), fixed angular fade & premultiplied color in `Custom/RadialFill`, and restored the native Surface lighting profile order (`Cradle_Destroyed_Challenge_01` selects `0.83` saturation instead of `0.70`).
- **Full-Resolution Texture Restoration**: Audited all **2,808** image payloads across all 165 bundles + 37 bootstrap textures and restored **294** reduced textures alongside **773** coupled `Sprite` records and all **26** original WebM videos.

---

## 3. Complete Audio & 1080p Cutscene Parity

Audio cues were traced end-to-end across the full chain: `authored owner → serialized pointer / catalog key → clip bytes → decode & clock → Web Audio mixer → gameplay event`.

- **14,270 Restored Serialized `AudioSource` References**: The original WebGL export dropped both `m_audioClip` and `m_Resource` references on thousands of `AudioSource` components while keeping the underlying audio clips inside shipped bundles. Starting from Trobbio's silent stage cues (**77** restored references across Trobbio & Tormented Trobbio), a whole-corpus cross-reference against Windows restored **14,270** additional sources (**13,560** across **552** gameplay rooms, **182** across **10** standalone scenes, and **528** across **6** reusable prefab bundles, including 19 Hornet attack sources) mapped cleanly to **1,188** shipped clips with channel-count and duration checks, updating external CAB references and catalog dependencies while keeping 609 unedited archive nodes byte-identical.
- **Vaultkeeper's Melody Progression Fix**: Seven audio clips reimported from WAV to OGG received regenerated GUIDs while the relic board retained the original keys. Aliasing all 7 original keys in `catalog.bin` (preserving all 41,966 existing keys) unblocked Psalm Cylinder playback, the Sacred Cylinder + Needolin sequence, and persistent `HasMelodyLibrarian` completion.
- **Final 5 Windows-Only FSB5 Clips**: Decoded the last 5 recordings missing from the Web export (`boss_stun_ss_loop`, `ui_silk_chunk_regenerating_loop_2d`, `gg_shadelord_screen_fill`, `boss_stun`, `poison_damage_tag`) from native FSB5 banks, transcoded them to AAC MP4, injected them into `packed-prefabs` with copy-on-write `m_PreloadTable` relocation, and verified non-silent Chromium WebAudio decoding on all 5.
- **131-Owner Windows Audio Audit**: Audited all 154 unmatched Windows `AudioSource` entries: 23 non-absent entries were already 100% wired in Web matching Windows 1-to-1, and all 131 absent entries belong strictly to 4 post-Web-export Windows patch additions (110 under `Swamp Barnacle/Black Mosquito Ball Attack` across 7 Aqueduct rooms, 10 `Caravan Lech Wounded`, 10 `Under_17 song_golem_hanging`, 1 `Coral_39 Sand Centipede Attacker`). Zero silent Web `AudioSource` bugs remain.
- **Web Audio Bridge Overhaul**:
  - Parses MP4 header duration immediately from raw bytes before decode/metadata resolves (fixing zero-length reads on short AAC clips and streamed ambience — e.g. Seamstress trace dropped from 29 invalid duration reads to 0).
  - Reads real `<audio>` playhead positions, preserves decoded loop bounds, schedules queued audio accurately on the Web Audio clock, avoids resurrecting late-decoded released clips, suppresses compressed-sound `getFrequency()` warning spam, guards zero-pitch/pause/mute/null states, unlocks AudioContext on `keydown` AND gamepad button presses, and recovers cleanly from `'interrupted'` AudioContext states.
- **WebGPU & WebGL Video Bridge (`1920×1080`)**:
  - Guarded `_JS_WebGPU_ImportVideoTexture` (null, `readyState < 2`, `requestVideoFrameCallback` fallback, zero-dimension guard, `try/catch`), added `keydown` gesture unlock, clamped volume to `[0, 1]`, prevented stale autoplay rejections from reviving paused/destroyed players, fixed CORS attribute ordering, safe destroy without `src=""`, GL state save/restore, and detached failed-media pause cleanup.
  - Verified all **26** WebM cutscenes, including Act 3 intro (`Act_03_Intro.webm` `1920×1080`, `16.866s` → `Song_Tower_Destroyed`), Mr. Mushroom, and all 5 endings.

---

## 4. Scene Transitions, Gameplay & Engine Lifecycle

- **Bell Beast, High Halls Arena & All 23 Additive Subscenes**:
  - *Symptom*: Rapidly exiting boss rooms could strand additive boss subscenes in adjacent rooms (e.g. Bell Beast's `Bone_05_boss` appearing in Moss Grotto, or High Halls' `Hang_04_boss` `Battle Gate` spawning at `(13.29, 6.29)` and blocking `Hang_06`'s right exit walkway).
  - *Root Causes*: (1) Addressables memory release fired during PreFetch instead of PostActivation; (2) an out-of-bounds IL2CPP heap write occurred on static `SceneAdditiveLoadConditional.Unload`; (3) slow/async additive subscene loads completing after room departure escaped teardown.
  - *Fix*: Deferred release to PostActivation, removed the OOB heap write (properly pruning destroyed entries & bumping `List._version`), hooked `SceneAdditiveLoadConditional::OnDisable` (guarded against double-unloads) and `LoadRoutine::MoveNext` for deterministic late-completion orphan sweeps across all **23** audited additive subscenes in **574** scenes, pre-seeded subscene suffix registries, and added orphan cleanup on Quit to Menu.
- **Whole-WASM Direct-Call Bypass Audit (`190,363` Functions)**:
  - Audited every defined function in `w-pt.wasm` to find runtime table hooks silently bypassed by direct WASM `call` instructions.
  - Migrated 6 bypassed private `DoPlay`/`DoWatch` slots in `ROOT A` (tk2d/PlayMaker completion parity) to their zero-direct-caller virtual `OnEnter` + `preGetSprite` pairs plus `PlayMakerFSM::Preprocess`, and added 15 indirect-root hooks covering all 5 bypassed methods in the scene-departure reconciler (`AddBlueHealthQueued`, `ClearEffects`, `ClearEffectsLite`, `MaxHealth`, `ResetAllCrestStateMinimal`).
  - Also fixed `ROOT A`'s `readStringVars` offset (`+36 → +32`), `Il2CppArray.bounds` check, and preserved non-BellBeast `Memory_*` scenes (such as `Memory_LaceTower`).
- **Crest-Switch-Close Guard & Historic Tools-Pink Closure**:
  - Early builds suffered a persistent coral `#ec837a` world overlay after equipping red tools and closing the inventory while `ModeSwitchSequence` coroutines remained in flight. Combined serialized close-teardown fixes with a live runtime guard on `InventoryToolCrestList` pane-end callbacks that stops active switch/move/scroll coroutines, reverts unconfirmed crest changes, resets state flags, and restores crest group fades cleanly.
- **Native In-Game Rewind (`↺`) WebGL Deadlock Fix**:
  - Pressing Rewind (`↺`) on WebGL previously spun forever because `FetchDataRequest<T>` dispatched restore-point deserialization to `Task.Run` (which never executes on single-threaded WebGL). Hooking both generic request slots to execute synchronously on the main thread allows Silksong's native `AUTO SAVES` list to populate immediately.
- **Input, Gamepads, Fullscreen & 60 FPS Pacing**:
  - Disabled 2D Pointer Lock trapping so hardware cursor stays hidden via CSS `cursor: none` while allowing direct clicks on UI controls.
  - Promoted `document.documentElement` in both fullscreen entry points so Settings and Diagnostics stay visible in the browser top layer.
  - Stopped swallowing `keyup` / `pointerup` / `mouseup` / `pointercancel` inside the Settings panel (reducing stuck-key Hornet cloak drift from crossing the entire room to `1.37 px`).
  - Scoped `Application.get_platform` inside `InputHandler` gamepad initialization so PS4, PS5, Switch Pro, and Xbox controllers map natively on WebGL.
  - Added a 60 FPS frame-pacing cap on high-refresh-rate displays (`60–240 Hz` verified at 600 iterations / 10 s) and Pause-on-Blur suspension when unfocused.

---

## 5. Save Durability, Native Rewind (`↺`) & Offline-First Architecture

- **Multi-Tab Safe `IDBFS` Mutation Journal**:
  - Standard Emscripten `IDBFS` overwrites IndexedDB with a tab's stale MEMFS snapshot if two tabs are open, silently rolling back or erasing another tab's progress.
  - Added a local `IDBFS` mutation journal with synchronous `getLocalSet` snapshotting, direct-MEMFS modification detection, scoped hydration, cross-tab non-empty directory protection, and generation-checked cleanup.
  - Extended checkpoint persistence to cover `shared.dat`, `shared.dat.bak*`, `PlayerPrefs`, and `.del` deletion markers, and hooked final save-file renames/closes so bench saves commit to IndexedDB even if the user closes the tab without Quitting to Menu.
- **Off-Main-Thread Save Codec & Native Rewind (`↺`) Writer**:
  - Moved AES-256-ECB save encoding/decoding and JSON parsing into a reusable Web Worker with a 4-second stall watchdog & single automatic worker replacement — slashing the worst save-commit main-thread stall from **886.5 ms → 20.8 ms**.
  - Intercepts committed `user{1..4}.dat` IndexedDB writes to asynchronously generate native-format `Restore_Points{slot}/restoreData{N}.dat` files (`yyyy/MM/dd`, rolling 20-entry cap while preserving `NODEL*` milestones) synced back into `/idbfs/`, seamlessly integrating with Silksong's native in-game **Rewind (`↺`)** UI.
- **SHA-256 Verified Offline Service Worker (`sw.js` + `offline-core.js`)**:
  - Verifies every cached bootstrap file (14 files) and content asset (196 files, ~2.40 GiB) by exact byte length and SHA-256 digest before accepting into `CacheStorage`.
  - Maintains durable per-tab shell and content release pins across Service Worker restarts, falls back to verified cache during transient 5xx outages, reclaims retired asset versions under storage quota pressure without evicting active pins, and self-heals legacy caches only after confirming online reachability.
  - Replaced typed-array SHA-256 iteration with scalar reads (**279.6 ms → 109.2 ms** per 8 MiB) and added cooperative time-sliced hashing on the main thread (reducing worst frame during background caching from **140.9 ms → 18.7 ms**, zero frames > 50 ms).

---

## 6. Minimalist UI, Zero-Black-Screen Boot Loader & Ad Purge

- **Zero-Black-Screen, Linear, Ring-Locked Boot Loader**:
  - Hooked engine lifecycle milestones (`StartManager::Start`, `Platform::SetSceneLoadState`, `UIManager::Start`) so the boot screen spans cleanly from initial fetch through `CoreManagers` and `Menu_Title` activation — eliminating the ~30-second black screen between Unity preload and the title menu without ever obscuring Language Select or Team Cherry.
  - Replaced CSS-lagging progress bars with a single-source-of-truth monotonic `requestAnimationFrame` controller that updates both the circular progress ring and numeric percentage on the same tick, detects Unity's two-pass preload reset without jumping backwards, tracks Addressables streaming via `Content-Length` headers only (zero RAM cloning), and uses continuous asymptotic CPU-work creep to eliminate the `84%` hard-refresh stall (`maxDiff = 0.50%`, `0` backward jumps across cold load & hard refresh).
- **Unified Flat Dark Monospace UI**:
  - Redesigned every webport surface (boot overlay, Settings drawer, HUD pill, window-blur pause veil, crash diagnostics modal, and warning banner) into one crisp flat dark monospace theme (`#0b0d11` / `#151a22` surfaces, `#e6edf3` / `#8b949e` text, `#58d5c9` cyan / `#3fb950` ok / `#e3b341` warn / `#f85149` error accents).
  - Consolidated duplicate floating controls into a single bottom-right pill (`opacity: 0; pointer-events: none` at rest when HUD is Hidden; smoothly reveals on bottom-right corner hover or `Ctrl+Shift+S`, with zero native hover tooltip popups).
- **Third-Party Ad Purge & Authentic Attribution**:
  - Removed all third-party ad splashes and placeholder domains from Unity startup data and UI sprite bundles while preserving `PlayerSettings.companyName` so existing `/idbfs/` save paths remain 100% compatible.
  - Restored authentic Team Cherry sprites and centered the version readout cleanly beneath the Team Cherry logo (`+0.5 px` horizontal delta at `1920×1080`).

---

## 7. Whole-Port Integrity & Multi-Biome Verification

- **Binary Parity Audit**: Audited **548,733** `MonoBehaviour` TypeTrees, **18,046** PlayMaker FSMs (**1,071,499** action calls), and **10** localized font glyph tables — **0** TypeTree mismatches, **0** stripped PlayMaker actions, **0** missing glyphs.
- **Catalog Closure**: Verified **0** missing `m_Externals` across all **22,703** catalog locations (`_GameCameras` complete at 121 dependencies) and repaired stale `m_BundleSize` records.
- **15-Biome Visual & Memory Matrix**: Automated Chromium runs across 12 WebGPU + 3 WebGL 2 biomes verified **65.7–67.2% OOM Left**, **0** WASM grows, **9/9** file-backed bundles, **0** leaked `hide52` meshes, **0.0000%** magenta pixels, **0** GPU validation errors, and **0** page errors.
