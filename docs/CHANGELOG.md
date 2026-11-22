# Engineering Changelog — Silksong Browser Web Port

Complete chronological record of engineering milestones from the initial browser port through the latest verified release.

---

## November 22, 2026 — Minimalist UI, Linear Boot Loader, Native Rewind (`↺`) & Ad Purge
- **Minimalistic Dark Monospace UI**: Unified boot screen, Settings drawer, single corner-reveal HUD pill, blur pause veil, crash diagnostics modal, and warning banner under one flat dark monospace design. Removed redundant floating pills, native hover tooltips, and legacy backup-vault/clipboard clutter.
- **Zero-Black-Screen, Ring-Locked Boot Loader**: Hooked `StartManager`, scene load state, and `UIManager` lifecycle transitions; built a single-source-of-truth monotonic rAF progress controller locking ring + percentage on the exact same tick, handling Unity's two-pass preload reset seamlessly, eliminating the ~30s black screen and `84%` hard-refresh pause (`0` backward jumps, `maxDiff = 0.50%`).
- **Native In-Game Rewind (`↺`) Integration**: Replaced browser-only backup lists with automatic off-main-thread native `Restore_Points{slot}/restoreData{N}.dat` checkpoint generation (20-entry rolling cap, preserving `NODEL*` milestones) and fixed WebGL's single-threaded `Task.Run` deadlock on `FetchDataRequest<T>` so Silksong's native **Rewind (`↺`)** button populates `AUTO SAVES` immediately.
- **Ad Purge & Authentic Team Cherry Restoration**: Purged all third-party ad splashes and placeholder domains from startup data and UI sprites while preserving `/idbfs/` save paths; restored authentic Team Cherry title assets and centered the version readout cleanly beneath the Team Cherry logo.

## November 22, 2026 — Stage 1004: Memory & OOM Breakthrough (`65.5–80.0%` OOM Left)
- **File-Backed MEMFS Streaming (`-75.0%` WASM `MemoryFile` Residency)**: Redirected the Top 9 largest persistent `_GameCameras` AssetBundles (`1,302.67 MiB`) to on-demand zero-copy MEMFS buffers (`canOwn=true`, deduplicated concurrent fetch with retries), lifting gameplay WASM headroom from **21.2% → 65.5–67.2%** (`80.0%` at Title) with **`0` WASM heap grows**.
- **`-310.42 MiB` Bundle Compaction**: Stripped `213.7 MiB` of unreachable embedded `VideoClip` stream bytes from `packed-cinematics` (`217.70 → 4.94 MiB`, 17 `AudioClip`s bit-exact & WebAudio-verified) and LZ4HC-recompressed `packed-sprites-ui` (`114.27 → 16.61 MiB`, bit-exact `SerializedFile`).
- **Eliminated `+89 Mesh / Round-Trip` Leak**: Fixed `tk2d*Sprite` `HideFlags.DontUnloadUnusedAsset` (`52 → 20`) and overwritten-mesh destruction across all 8 `Awake`/`Build` paths (`1,182 → 1,182 → 1,182` across 8 round trips; `38` on Quit to Menu).
- **Allocator & Telemetry Polish**: Capped `_emscripten_resize_heap` over-grow (`+512 → +128 MiB`), removed `2.6 GiB` `res.clone()` boot fetch teeing, and fixed `crashtrack.js`'s 4-byte `_gm_.top` offset bug + bounded treebins free accounting (`layoutVerified: true`, within `0.2 MiB` of exact walk).
- **Crest-Switch-Close Guard**: Hardened `InventoryToolCrestList` pane-end teardown to stop in-flight switch coroutines and revert unconfirmed crest state cleanly.
- **131-Owner Audio Closure & 15-Biome Sweep**: Closed all 154 `unmatched-detail` Windows audio entries (`0` silent Web `AudioSource` bugs remain); passed 12 WebGPU + 3 WebGL 2 biomes, no-quit bench reload, Act 3 intro (`1920×1080` → `Song_Tower_Destroyed`), and all 7 ending/secret clips.

## November 22, 2026 — Stage 1003: Direct-WASM Audit, High Halls Arena & Input Hardening
- **Direct-WASM-Call Bypass Repair**: Audited all **190,363** WASM functions against runtime hooks; migrated 6 bypassed `ROOT A` slots to zero-direct-caller virtual `OnEnter` + `preGetSprite` pairs + `PlayMakerFSM::Preprocess`, and added 15 indirect roots for the scene-departure reconciler.
- **High Halls Arena (`Hang_04` / `Hang_04_boss`) & All 23 Additive Subscenes**: Identified the right-exit blocking gate as stranded `Hang_04_boss` `Battle Gate` overlapping `Hang_06`; hardened all 23 additive subscenes across 574 scenes via `OnDisable`, `LoadRoutine::MoveNext` late-completion orphan sweeps, pre-seeding, and Quit-to-Menu cleanup.
- **Pointer Lock, Fullscreen, Stuck Keys & Gamepads**: Disabled 2D Pointer Lock trapping, promoted `document.documentElement` in fullscreen, unblocked key/pointer release events in Settings (`1.37 px` cloak settle), enabled native PS4/PS5/Switch/Xbox WebGL gamepad detection + gamepad WebAudio unlock, hardened `_JS_Video_*` helpers, and added `.del` markers to `IDBFS`.
- **`Sprites/Darkness Sprite` & `_GameCameras` Closure**: Synthesized 4 non-dithering `Darkness Sprite` variants (`12/12` GLES3 + `12/12` WGSL) and closed the final 2 transitive catalog dependencies (`0` missing across `22,703` locations).

## November 21, 2026 — Stage 1002: Whole-Corpus Shader Parity, Multi-Tab `IDBFS` & WebGPU Video
- **Shader Codec & DXBC Math Parity**: Built bit-exact Unity 6000 parameter-blob & GLES3/WGSL codec (`458/458` blobs, `516/516` subprograms). Fixed `Sprites/Lit` (201 materials), `Sprites/Default-ColorFlash` (406 materials), `UI/Blur/UIBlur`, rebuilt 2-pass `UI/BlendModes/Multiply`, and restored `DITHERING_NOISE` on 6 sibling shaders (`86/86` GLES3 & `86/86` Dawn WebGPU pipelines; `8/8` Windows DXBC controls at `0/255` delta).
- **Multi-Tab Safe `IDBFS` Journal**: Prevented stale tabs from silently overwriting or deleting saves in another tab; persisted `shared.dat`, `shared.dat.bak*`, and `PlayerPrefs`; fixed empty-slot import deadlock.
- **WebGPU Video & Loader Integrity**: Hardened `_JS_WebGPU_ImportVideoTexture` (null/readyState/rVFC/dimensions/volume/`keydown` unlock), fixed `GM_SCENE_LOAD` struct offset & release deduplication, and repaired stale catalog `m_BundleSize` entries.

## November 21, 2026 — Stage 1001: Vaults Masking, Bell Beast Subscene & Complete Audio Parity
- **HLSLcc `-0.0` Sign-Inversion Fix**: Fixed all 4 affected game shaders (`Custom/Tiled Scrolling Fog`, `Sprites/Tiled Scrolling Masked`, `Sprites/Tiled Masked`, `Custom/LavaBase`), eliminating the Whispering Vaults (`Library_09`) cyan-teal exterior void cutout box; restored `_StencilRef`/`_StencilComp` in `Default-ColorFlash`, 28 `GrabPass`es + 75 pass states in `packed-guiblendmodes`, `_StencilRef` in `Stencil-Write`, and `ZWrite Off` in `UIBlur`.
- **Bell Beast (`Bone_05_boss`) Additive-Scene Lifecycle**: Deferred Addressables release from PreFetch to PostActivation, removed OOB IL2CPP heap write on `SceneAdditiveLoadConditional.Unload`, and added post-transition orphan sweeping.
- **Final 5 FSB5 Clips & 9 Web Audio Bridge Fixes**: Decoded, AAC-transcoded, and injected the 5 missing native-only recordings into `packed-prefabs` and fixed 9 Web Audio runtime bugs (including `keydown` unlock, pending sample rate, deferred play, and warning suppression).

## November 20–21, 2026 — Shared Audio Repair (14,270 Cues), Vaultkeeper's Melody & Mount Fay Refraction
- **Whole-Port Audio Reference Restoration**: Restored **77** Trobbio stage cues and **14,270** erased `AudioSource` clip bindings across **552** rooms, **10** standalone scenes, and **6** prefab bundles, plus early AAC MP4 header duration parsing.
- **Vaultkeeper's Melody**: Repaired 7 regenerated Psalm Cylinder GUID aliases in `catalog.bin`, unblocking Vaultkeeper's Melody progression and persistent save flags.
- **Mount Fay Ice Refraction**: Restored `_SpriteRefractionGrab` across **117** refraction objects in **7** rooms, removing the white square at Shakra's bench and gray box behind shop menus.

## November 18–19, 2026 — Lighting, Fog, 60 FPS Cap, Streamed Audio & Offline Resilience
- **Streamed Audio Timing**: Early MP4 header duration parsing, live media playhead tracking, decoded loop bounds, and Web Audio clock scheduling.
- **60 FPS Frame Cap & Late-Game Cutscenes**: Added a 60 FPS loop ceiling for high-refresh displays, prevented stale autoplay rejections from reviving paused/destroyed videos, and verified all 5 endings, Act 3 intro, Mr. Mushroom, `Cradle_03`, and `Abyss_Cocoon`.
- **Startup Memory & Offline Optimization**: Reused decoded `82.57 MB` WASM buffer, added exact page-aligned grow fallback, bounded transition history, coalesced offline progress updates, added retired-cache quota reclamation, per-tab release pins across SW restarts, Web Lock crash-tab ownership (eliminating false multi-tab crash warnings), and save-worker stall recovery.
- **Lighting & Fog Parity**: Fixed positional whole-room red wash (`GradeMarker` deferred profile wait across 96 markers in 86 rooms), Underworks sharp green fog edges (`ETC1_EXTERNAL_ALPHA` across 222 renderers in 34 rooms), `Custom/ScrollTexture` stencil refs (447 refs in 34 rooms), `Custom/RadialFill` Hunter arc fade, fog dithering (20 programs), and Surface lighting profile order.

## November 15–17, 2026 — Full-Resolution Restoration, Baked Tile Lifetimes & Stutter Elimination
- **Full-Resolution Restoration**: Audited all **2,808** textures and restored **294** reduced textures + **773** coupled `Sprite` records + all **26** original WebM videos.
- **Baked Black-Tile Lifetime Fix**: Fixed room-re-entry black-tile loss across **5,970** meshes (**2,914,887** quads) in **549** rooms.
- **Native Post-Processing**: Restored native 3-pass Film Grain, native Bloom weights/calls/stack cleanup, Camera Light Blur + `Shader.Find` registration, WebGPU uniform buffer alignment, and WebGL darkness/blur camera target lifecycles.
- **Measured Stutter & Memory Fixes**: Moved the `483 MiB` 574-scene archive to file-backed MEMFS (`+484.9 MiB` free WASM), moved save decode/parsing to a Web Worker (`886.5 ms → 20.8 ms`), replaced 1 Hz full heap walks with fast allocator globals (`350 ms → 83.3 ms` throttled p95), and switched to scalar time-sliced SHA-256 hashing (`140.9 ms → 18.7 ms` worst frame).
- **No-Quit Checkpoint Persistence**: Hooked completed engine save writes/renames to flush straight through `IDBFS`.

## October – November 13, 2026 — Foundation, Offline Sync, `ROOT A` Parity & Boss Verification (Stages 547–990)
- **Tools-Pink Coral Overlay Resolution (`CORAL37`)**: Traced persistent `#ec837a` world corruption after red-tool equip/close to stranded crest-switch transitions and implemented close-teardown protection.
- **Sprite Bounds & Colour Parity (`SpriteBounds547/577`, `Colour693`)**: Repaired downscaled logical sprite bounds and native-vs-Web color grading LUT/saturation ordering.
- **`ROOT A` tk2d / PlayMaker Parity & `OOM` Transition Reclaimer (Stages 851–954)**: Built unified runtime WASM hooking for Addressables scene-unload release, tk2d animation-completion parity, departure reconciliation, and WebGPU PostFX restoration.
- **Offline Sync & Closed-Loop Gameplay Verification (Stage 990)**: Fixed Service Worker bypass streaming (`0` `Failed to fetch` errors across 210 files), audited 548,733 TypeTrees / 18,046 FSMs / 10 font tables (`0` mismatches), and verified live 1080p combat against Moss Mother, Bell Beast, and Fourth Chorus, Cocoon recovery, and native Silk Bind healing.

## July – September 2026 — Initial Browser Port & Diagnostic Tooling
- Established the WebGPU/WebGL 2 Unity WebAssembly runtime shell, Addressables catalog CRC pipeline, Pause-on-Blur suspension, AES-256-ECB save backup/import helpers, crash/OOM telemetry (`crashtrack.js`), and the initial offline `CacheStorage` architecture.
