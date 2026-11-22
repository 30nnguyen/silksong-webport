# Hollow Knight: Silksong — **FULLY FIXED AND WORKING** Web Port (.html file · https://30nnguyen.github.io/silksong-webport/)

> **Play the FULLY FIXED AND WORKING Port Now (No Download Required):** **[https://30nnguyen.github.io/silksong-webport/](https://30nnguyen.github.io/silksong-webport/)**
>
> **Prefer a single `.html` file?** Download **[`silksong.html`](https://raw.githubusercontent.com/30nnguyen/silksong-webport/main/silksong.html)** (right-click → *Save Link As*) and double-click it to open in Chrome or Edge — no unzipping, no extra files, always up to date.
>
> **Found a bug, missing sound, weird visual, or crash?** **[Open a Bug Report](https://github.com/30nnguyen/silksong-webport/issues/new/choose)** and let us know so we can fix it!

Play the **FULLY FIXED AND WORKING** *Hollow Knight: Silksong* right in your browser (WebGPU & WebGL 2) with **original HD graphics**, **complete audio & 1080p cutscenes**, **working saves + built-in Rewind (`↺`)**, **offline play**, **controller support**, **zero ads**, and **huge crash/memory fixes** compared to older broken Silksong web ports.

> [!IMPORTANT]
> **ENJOYING THE PORT? PLEASE DROP A STAR ON THIS REPO AT THE TOP RIGHT — IT REALLY HELPS OTHER PLAYERS FIND A WORKING VERSION!**

---

## What We Fixed & Improved (In Plain English)

Older Silksong browser ports suffered from frequent out-of-memory crashes, missing Silk HUD, silent needle swings, broken saves, black gaps in rooms, pink/red screen bugs, softlocks, and a 30-second black screen on boot. Here is everything we fixed:

### 1. Way Less Crashing & Out-of-Memory (Full HD, No Blurry Downgrades)
- **Massive free memory jump (`21% → 65–80%` free)**: Older web ports ran right up against the browser's 4 GB limit and crashed after ~15 minutes or while saving. We moved the 9 biggest game packs and the 574-room world archive out of the tight WebAssembly memory wall so you keep **65–67% free headroom during gameplay** (**~80% on the title screen**) with **zero** memory crashes across long sessions.
- **No more 2 GB in-browser unzip freeze**: Assets stream cleanly on demand instead of decompressing a giant multi-part zip inside browser RAM.
- **Fixed a hidden room-change memory leak**: Every time you walked between rooms, the game previously leaked **89 invisible mesh objects** until it ran out of RAM. Now it leaks **0** even across repeated room transitions and Quits to Menu.
- **310 MB smaller download & cache**: Removed **213.7 MB** of unused duplicate video data tucked inside the cinematics pack and compressed the huge uncompressed UI pack (**114 MB → 16.6 MB**) without losing a single pixel or sound.
- **Smoother frame pacing & fewer hitches**: Fixed background pauses from memory checks, cache hashing (**140.9 ms → 18.7 ms** worst frame), and save writing (**886.5 ms → 20.8 ms** worst hitch).

### 2. Full Original HD Graphics — No Missing Walls, Weird Colors, or Visual Glitches
- **Full original resolution restored**: Fixed **294 blurry/downscaled textures** and **773 sprites** so every single one of the **2,808 textures** and **26 cutscenes** matches the original full-resolution game. *(You can still optionally lower Render Scale to `0.75×` or `0.5×` in Settings if you're on an older/low-end laptop.)*
- **Fixed missing black walls & bright ceiling/bridge gaps (549 rooms)**: In rooms like Moss Grotto, Bone Bottom's bridge, and the Seamstress room, leaving and coming back used to delete the black walls/ceilings and show bright blue/teal voids. Fixed across **5,970 tile meshes in 549 rooms**.
- **Fixed the Whispering Vaults cyan cutout box**: Fixed a shader math bug across 4 shared fog/mask shaders that flooded dark areas outside rooms with bright teal fog (especially noticeable in the Whispering Vaults east shaft).
- **Fixed the white square at Mount Fay's bench & gray box in Shakra's shop**: Restored real ice refraction across **117 objects in 7 rooms**.
- **Fixed the whole-screen red tint**: Fixed a loading race that turned entire rooms red as you moved around (affected **48 color zones across 45 rooms**, including Underworks and Mount Fay).
- **Fixed sharp green fog edges in Underworks**: Fixed swapped fog alpha/color channels across **34 rooms**.
- **Fixed wrong lighting, dark areas & Black Thread visuals**: Repaired lighting math and missing shader variants across **600+ materials** (including `Sprites/Lit`, `Default-ColorFlash`, `Darkness Sprite`, Surface area lighting, stencil masking in 34 rooms, and the Hunter meter arc).
- **Fixed Pause Blur, Bloom, UI Blend Modes & Real Film Grain**: Restored background blur behind menus, fixed Bloom glow and 3 WebGL render-target glitches, and replaced the broken Film Grain option with real 3-pass Film Grain (`Options → Video → Noise` works without crashing!).
- **Fixed the pink/coral screen bug**: Equipping red tools and closing the inventory used to leave a persistent pink `#ec837a` tint over the whole world. Fixed cleanly at the source.

### 3. Complete Sound Effects, Music & 1080p Cutscenes
- **Over 14,300 silent sound effects restored**: The original web export accidentally blanked out thousands of sound links — including **Hornet's needle swings**, room ambience, enemy sounds, and **77 Trobbio / Tormented Trobbio boss cues** (drums, tension, smoke, tornado). We restored **14,270+ missing audio links across 552 rooms, 10 standalone scenes, and 6 shared packs**.
- **Vaultkeeper's Melody & all 7 Psalm Cylinders work**: Fixed broken audio IDs so every Psalm Cylinder and the Sacred Cylinder / Needolin sequence play properly and save your Vaultkeeper's Melody progress.
- **Last 5 Windows-only sounds added**: Converted and added the 5 missing native-only sounds (boss stun, boss stun loop, silk chunk regen loop, poison damage tag, shadelord fill). Every Web audio source is now wired and audited.
- **Fixed audio timing, looping & silent cues**: Fixed streamed music/ambience reporting `0` length, fixed loops jumping to the wrong spot, fixed queued sounds staying silent, and made audio automatically unlock on your first keypress, click, or controller button press.
- **All 26 cutscenes verified at 1080p**: Fixed video autoplay/WebGPU texture crashes and verified **Act 3's intro** (playing smoothly into `Song_Tower_Destroyed`), **Mr. Mushroom**, and **all 5 endings** at `1920×1080`.

### 4. Gameplay Softlocks, Missing HUD, Boss Gates & Controls Fixed
- **Silk Spool HUD & stuck animations fixed**: Fixed dropped animation-finish callbacks in WebGL that caused the **Silk Spool HUD to stay invisible** and softlocked **crest binds, arena walls, Widow, and Grand Mother Silk**.
- **No more boss rooms bleeding into neighboring rooms**: Fixed **Bell Beast** spawning in Moss Grotto and **High Halls Arena's gate** blocking the hallway when leaving the room — hardened across **all 23 boss & additive subscenes in all 574 rooms** (even when rapidly bouncing in and out of doorways).
- **Fixed switching Crests mid-close**: Closing the Tools/Crest menu right in the middle of switching crests no longer freezes or breaks the crest UI.
- **Native PlayStation, Xbox & Switch Pro controller support**: Fixed WebGL platform detection so PS4, PS5 DualSense, Xbox, and Nintendo Switch Pro controllers map naturally without extra tools.
- **Fixed mouse trapping, Fullscreen & stuck keys**:
  - Disabled broken 2D Pointer Lock so you can freely click UI buttons while keeping the in-game cursor clean.
  - Fullscreen keeps the Settings drawer visible on top.
  - Opening the Settings menu while holding a movement key no longer makes Hornet keep running on her own.
- **Smooth 60 FPS cap & Auto-Pause on Tab Switch**: Locks frame pacing cleanly to 60 FPS on 120–240 Hz monitors and automatically pauses the game & audio when you switch tabs or minimize the window.

### 5. Saves That Actually Save, Working In-Game Rewind (`↺`) & Offline Play
- **No more "50/50 chance of losing your save"**: Bench rests and Save & Quit now flush straight to browser storage (`IndexedDB`) automatically — even if you close the tab without hitting Quit to Menu. Global settings (`PlayerPrefs` and `shared.dat`) persist too.
- **Multiple tabs won't overwrite your progress**: Added a multi-tab save journal so having an older tab open can never silently roll back or wipe a newer save.
- **Built-in In-Game Rewind (`↺`) actually works**: Clicking the **`↺`** icon under a save slot used to spin forever on WebGL because browsers don't support background C# threads. We fixed the single-threaded WebGL freeze AND added an automatic checkpoint writer so every save creates a rolling native restore point (up to 20 per slot) right inside Silksong's native `AUTO SAVES` menu!
- **PC-compatible `.dat` Save Backup & Import**: Open Settings (`Ctrl + Shift + S` or bottom-right corner) to back up or drag-and-drop `.dat` saves across Slots 1–4.
- **True Offline Play**: Click **Offline Play → Cache Full Game** in Settings to download and SHA-256 verify all 210 game files (~2.4 GB) into your browser so you can play completely offline with zero internet.

### 6. Clean Dark Interface, Smooth Boot Screen & Zero Ads
- **No more 30-second black screen or stuck `84%` bar**: Replaced the broken loader with a smooth, accurate circular loading ring that goes straight into Language Select and the Title Screen.
- **All ads & third-party watermarks removed**: Purged every ad splash screen and third-party site watermark from the startup screen and UI, restoring the authentic Team Cherry title screen while keeping all existing browser saves compatible.
- **Minimal, out-of-the-way Settings UI**: Clean flat dark monospace drawer (`Ctrl + Shift + S` or hover the **bottom-right corner**). When HUD is set to Hidden, the corner pill is completely invisible while you play and fades in smoothly only when you move your mouse to the bottom-right corner.

---

## How to Play

1. **Direct Browser Link (Recommended)**: Open **[https://30nnguyen.github.io/silksong-webport/](https://30nnguyen.github.io/silksong-webport/)** in desktop Chrome or Edge (64-bit).
2. **Single Standalone HTML File**: Download **[`silksong.html`](https://raw.githubusercontent.com/30nnguyen/silksong-webport/main/silksong.html)** from this repository and open it directly in your browser. It fetches the latest verified game files automatically without needing any other local files or source code.

### Controls Quick Reference

| Action | Default Keyboard | Controller |
| --- | --- | --- |
| Move | `Arrow Keys` | Left Stick / D-Pad |
| Jump | `Z` | `A` / `Cross` / `B` |
| Attack | `X` | `X` / `Square` / `Y` |
| Dash / Sprint | `C` | `RT` / `R2` |
| Bind / Heal / Cancel | `A` | `B` / `Circle` / `L1` |
| Needolin / Switch Crest (at bench) | `D` | `Y` / `Triangle` |
| Quick Map | `Tab` | `LB` / `L1` |
| Inventory / Tools & Crests | `I` / `Q` | Select / Touchpad |
| Pause Menu | `Escape` | Start / Options |
| **Web Port Settings, Saves & Offline Cache** | **`Ctrl + Shift + S`** *(or hover bottom-right corner)* | — |

*(You can rebind any in-game control inside Silksong's native Options → Keyboard / Controller menu.)*

---

## Found a Bug? Let Us Know!

*Hollow Knight: Silksong* is huge, and while we've tested across 15+ biomes, multiple bosses, Act 3, and all endings on both WebGPU and WebGL 2, there can still be rare edge cases on specific GPUs, browsers, or late-game rooms.

**If anything looks off, sounds silent, freezes, or crashes, please report it here:**
- **[Report a Gameplay / Crash / Save / Controls Bug](https://github.com/30nnguyen/silksong-webport/issues/new?template=bug_report.yml)**
- **[Report a Visual / Shader / Audio / Cutscene Bug](https://github.com/30nnguyen/silksong-webport/issues/new?template=visual_audio_bug.yml)**

Let us know the **room/boss name**, your **browser & GPU**, a **screenshot** if visual, and (optionally) copy the **Diagnostics** info from Settings (`Ctrl + Shift + S` → **Diagnostics**) or attach a `.dat` save backup.

---

## Want the Technical Details?

- **[Technical Deep Dive](docs/TECHNICAL_DEEP_DIVE.md)** — Exact root causes, shader/DXBC parity details, WASM allocator analysis, audio/video bridge engineering, and verification numbers.
- **[Chronological Engineering Changelog](docs/CHANGELOG.md)** — Full timeline of every milestone from July 2026 to the latest release.
- **[Troubleshooting & Browser Tips](docs/TROUBLESHOOTING.md)** — WebGPU vs. WebGL 2, laptop Render Scale tips, offline caching, and save backup/import help.

> [!NOTE]
> This public repository hosts the project documentation, standalone [`silksong.html`](silksong.html) launcher, and public issue tracker. Source code is kept private to prevent unauthorized re-hosting/ad-wrapping; the playable build deploys automatically to **[https://30nnguyen.github.io/silksong-webport/](https://30nnguyen.github.io/silksong-webport/)**.

---

## Credits

- *Hollow Knight: Silksong* is created and owned by **Team Cherry** — please support the official game!
- Browser porting, bug fixes, memory/shader/audio parity, save/rewind systems, and UI by **[@30nnguyen](https://github.com/30nnguyen)**.
