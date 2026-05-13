# Minecraft PE 0.6.1 — XP / Win2K fork

A Windows-side fork of the Minecraft Pocket Edition 0.6.1 alpha source. The
upstream codebase (based on Kolyah35's port) builds a desktop GLFW client; this
tree keeps that working and layers on bug fixes, gameplay features, and
compatibility work for older Windows.

The default build target is a **single 32-bit `MinecraftPE.exe` that runs on
Windows XP SP2/SP3**, and (with the bundled Mesa3D DLL) on Windows 2000 with
the extended kernel.

## What's actually in here

Compared to upstream, this fork ships:

### Redstone
A functional, mostly vanilla-faithful redstone subsystem built from the
scaffolding upstream left behind (`Tile::isSignalSource`, `Level::getSignal`,
`hasNeighborSignal`, `hasDirectSignal`). Includes:

- **Redstone dust** — 0..15 strength stored in `auxValue`, decays one per hop,
  cuts properly when the source is removed. Grayscale cross sprite at terrain
  `(4, 10)` is tinted red at render time via `getColor` — bright when powered,
  dim when off so you can see the wire layout.
- **Lever** — wall-mountable on any solid face (and floor). `auxValue` bit
  `0x8` is on/off, bits `0..2` are mount face (1..5, same convention as torch).
  No proper lever sprite exists in this build's atlas, so the body renders as
  a small wooden cube that flips to cobblestone when thrown.
- **Button** — wall-mountable like the lever, with a timed press: 20 ticks for
  stone, 30 for wood. `setShape` flattens the AABB against whichever face
  it's mounted on.
- **Pressure plate** — random-ticks itself to scan for entities in a small
  AABB above the plate, sets aux 0/1, fires neighbor updates.
- **Redstone torch** — two tile IDs (`notGate_off` = 75, `notGate_on` = 76)
  sharing a class with a `_lit` flag. Acts as a NOT-gate: schedules a 2-tick
  state-check whenever neighbours change, swaps variant in `tick`. Always
  strongly powers the block directly above (regardless of mount), so
  `torch → dirt → door` opens the door.
- **Redstone lamp** — two-variant tile (123 off, 124 on) that lights when any
  signal touches it. Off form uses a baked "opaque glass" texture (the glass
  speckle pattern composited onto a warm-white backing, written into the
  previously-unused atlas cell at `(4, 11)`); on form uses the glowstone
  sprite with full light emission.
- **Doors / TNT** react to weak power via the existing `hasNeighborSignal`
  path — nothing else needed there. Wires also do an "indirect neighbour
  update" (neighbours-of-neighbours) on every strength change so torches and
  components attached to wire-adjacent blocks notice the change.

### `/time` command
`/time set day|noon|night|midnight` and `/time set <ticks>` now work
correctly. This build's day cycle is 19200 ticks (vs Java's 24000), so the
symbol table is anchored to this cycle (`day`/`noon` → `TICKS_PER_DAY/4`,
`night`/`midnight` → `3*TICKS_PER_DAY/4`), and numeric input is rescaled from
vanilla's 0..23999 range so `/time set 6000` still means noon.

### HUD
Hearts and armor moved to their Java positions on top of the hotbar. Hearts
sit at the left edge of the hotbar; armor bar is right-anchored to the right
edge of the hotbar with a hard gap so the two rows don't touch.

### Save / load
`LevelData::setTagData` / `getTagData` now serialize the current `Dimension`.
Previously, saving and quitting from the Nether reloaded the world in the
overworld at the Nether coordinates — instant death if those were below
sea level or inside a wall. Dimension is now preserved across reloads.

### Buckets
- Empty bucket raycast now ignores liquid-skipping mode, so right-clicking
  water actually hits the water surface instead of the sand block underneath.
- If the raycast lands on a solid block adjacent to water, the bucket falls
  back to the face-neighbour cell and picks up the liquid there.
- Bucket placement places the *dynamic* fluid id (water 8, lava 10) and
  schedules a tick so it actually starts flowing.

### Start screens
Removed the Kolyah35 credit text, GitHub icon blit, and clickable URL handler
from both `StartMenuScreen` and `TouchStartMenuScreen`.

### Windows 2000 / XP compatibility
The OpenGL context request was lowered from 2.1 to 1.1 in `main_glfw.h` so
Mesa3D's software rasteriser (which is what Win2K's extended-kernel VMs end
up using) can fulfill it. A 32-bit, self-contained Mesa3D 17.3.7
`opengl32.dll` is checked in under `win2k/`; copy it next to `MinecraftPE.exe`
on systems whose native OpenGL driver doesn't make the cut. Newer Mesa
branches (20.x, 26.x) won't load on Win2K with the extended kernel — 17.3.7
is the last release `pal1000` built explicitly against the older Windows
ABI.

## Building

The supported build is **Windows + Visual Studio 2017 Build Tools** with the
`v141_xp` toolset. The PowerShell script handles vcvars, picks the right
Ninja, and writes the binary to `build-xp\MinecraftPE.exe`.

```powershell
# from the repo root
powershell.exe -ExecutionPolicy Bypass -File .\build-xp.ps1

# clean rebuild
.\build-xp.ps1 -Clean
```

Requirements:

- Visual Studio 2017 Build Tools with the `v141_xp` (Windows XP) toolset
  installed. The script pins `-vcvars_ver=14.16` so newer toolsets won't
  silently take over.
- CMake 3.21 or newer.
- Ninja on `PATH` (or update the `$NinjaDir` parameter in `build-xp.ps1`).
- Git (some CMake `FetchContent` deps clone at configure time).

Output is `build-xp\MinecraftPE.exe`. The `data\` directory next to the exe
holds the game assets (terrain.png, items.png, the language file, etc.) —
these are copied into `build-xp\data\` as part of the build.

### Running on Windows 2000 (extended kernel)

1. Build as above.
2. Copy `build-xp\MinecraftPE.exe` and the adjacent `data\` directory to the
   target machine.
3. Copy `win2k\opengl32.dll` (~24 MB) into the same folder as
   `MinecraftPE.exe`. Don't overwrite `C:\WINNT\system32\opengl32.dll` —
   keeping the Mesa one next to the exe scopes it to just this game.
4. Optional: `set GALLIUM_DRIVER=softpipe` before launch to force the
   pure-C reference renderer if `llvmpipe` (the default, JIT, needs SSE2)
   has trouble.

### Other platforms

The upstream CMake build for Linux / desktop Windows / Android / Web / iOS
is still in the tree (`CMakeLists.txt`, `build.ps1`, `build.sh`, the
`project/` subdirectories). None of the changes above are tied to the XP
build specifically — they all live in shared `src/` code — but the XP path
is the one I actually test against.

## Layout

- `src/` — the game code. Notable subdirs: `world/level/tile` (the redstone
  family lives here), `world/item`, `client/gui/screens` (console + HUD),
  `client/renderer`, `network`.
- `data/` — game assets; `data/images/terrain.png` is the tile atlas
  (256×256, 16×16 grid). `terrain.png.bak` is the original; the live file
  has the lamp's opaque-glass cell baked into `(4, 11)`.
- `build-xp/` — CMake out-of-source build directory for the XP target.
- `win2k/` — the Mesa3D `opengl32.dll` and a short README.
- `glad/` — vendored GLAD loader.
- `Minecraft PE 0.6.1 Win32 port/` — an older snapshot of upstream; not used
  by the active build.

## Texture atlas notes

MCPE 0.6.1's `terrain.png` predates redstone, so several of the textures
we'd want (lever, redstone wire when colored, lamp on/off) don't exist as
ready-made sprites. The workarounds in this tree:

- Redstone wire: the grayscale cross at `(4, 10)` is tinted red at render
  time via `Tile::getColor`.
- Lever: planks for off, cobblestone for on. No real lever sprite.
- Redstone torch: the existing torch sprite at `(0, 5)` is reused; the
  rendering is the standard `SHAPE_TORCH`. Off variant points at `(3, 7)`.
- Redstone lamp off: baked into the previously-empty cell `(4, 11)` —
  glass speckle on a warm-white opaque backing.
- Redstone lamp on: glowstone sprite at `(9, 6)` with `setLightEmission(1.0f)`.

If you want to restore the original `terrain.png`, the script that bakes the
lamp cell first writes a `data/images/terrain.png.bak` backup; just copy it
back over `terrain.png`.
