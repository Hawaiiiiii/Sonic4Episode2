# RESUME HERE — live state

Single source of truth for where this project stands. Update at the end of every
working session. If chat history is gone, start from this file.

## Where things stand

Phase 1 (asset containers) is largely done. Three formats are decoded and
verified against the whole build: the **AMB** container, the **stage tile grids**
(`.MP`/`.MD`), and the **object placement tables** (`.EV`). The binary has been
surveyed and scoped. No engine or game code has been written yet.

## Paths

| What | Where |
|------|-------|
| Game root (Beta 8) | `C:\Users\DavidErikGarciaArena\Downloads\Sonic 4 - Episode 2 (Beta 8)\Sonic 4 - Episode 2 (Beta 8)` |
| This project | `<game root>\Sonic4Episode2` |
| Episode I decompilation (the Rosetta stone) | `<game root>\Sonic4Episode1-master` |
| AliceNN engine already in C# | `<game root>\Sonic4Episode1-master\Sonic4Episode1\AppMain\Am\` |
| AMB parser Episode I side | `Sonic4Episode1\AppMain\Am\AmFs.cs:153-221` (`readAmbHeader`) |

## Verified this session

- `Sonic.exe` is native x86 (VS2008, D3D9 + D3DX9_43, DINPUT8, DSOUND, XINPUT1_3,
  steam_api) with **no CLR references**. Episode I's IL-decompilation approach
  cannot be reused. `Launcher.exe` is a managed settings dialog only.
- Episode II runs the same **AliceNN** engine as Episode I — proven by the
  embedded source path `...\alicenn\source\library\amMalloc.h` and by Episode I's
  decompiled `AppMain/Am/*` implementing that same library.
- Shared data conventions: Episode I's source names the very directories this
  build ships (`G_COM`, `G_ZONE1-4`, `DEMO`, ...).
- **AMB format decoded and verified**: 1,614/1,614 archives in this build parse
  cleanly. Spec in `docs/FORMAT-AMB.md`, tool at `tools/amb.py`.
- Extraction is lossless. An earlier version silently lost ~25% of stage geometry
  because map archives carry blank string-table slots that all collapsed onto one
  filename; fixed by falling back to the entry index, then confirmed by
  reconciling 1,464 written against 1,464 on disk for `G_ZONE1`.
- Content inventory across all archives: 3,577 `.ZNO` models, 2,853 `.DDS`,
  1,431 `.ZNM` motions, 925 `.AME`, 922 `.PSH` + 921 `.VSH` shaders, 669 `.ZNV`,
  651 `.TXB`, 540 nested `.AMB`, 336 `.MP`, 258 `.MD`, 256 `.AMA`, 79 `.EV`.
- `.ZNO` models are SEGA NN chunked data, Direct3D 9 variant — first chunk magic
  `NZIF`, then `NZTL`.
- **Stage tile grids decoded** (`docs/FORMAT-STAGEMAP.md`, `tools/stagemap.py`):
  `u16 width, u16 height`, then `w*h` cells — `u16` for `.MP`, `u8` for `.MD`.
  All 400 grids in the build resolve exactly. Rendered previews of Zone 1 Act 1
  show real platformer terrain, and `_ATTR_B` mirrors `_B`'s silhouette, which
  confirms the attribute layers are parallel to the tile layers. Independently
  corroborated afterwards by Episode I's `readMPFile`/`readMDFile`
  (`AppMain/Gm/GmMap.cs:113,140`), which match field for field.
- **Object placement decoded** (`docs/FORMAT-EVENTS.md`): `.EV` files index the
  stage with a block grid at *quarter* map resolution (rounded up) holding `u32`
  offsets; each block is a `u16` count plus that many **12-byte** records. All 65
  `.EV` files parse. Field meanings inside a record are still inferred, not proven.
- `.DC` and `.RG` share the `.EV` block-grid header but use a different record
  size — still unknown, and Episode I cannot help because its `readDCFile`,
  `readRGFile` and `readEVFile` are unimplemented stubs.
- **Binary surveyed** (`docs/ENGINE.md`): ~8,000 functions, of which ~6,600
  (~2.5 MB) are game code and ~17% is CRI/NN/TinyXML middleware to be replaced
  rather than reversed. No PDB. No game-code RTTI — built `/GR-`, so the class
  hierarchy needs vtable-from-constructor analysis. `/LTCG` on 663 of 757 C++
  objects. ~82% of functions are FPO-optimised.
- **Shaders are Shader Model 3.0 bytecode** — 922 `ps_3_0` + 921 `vs_3_0`, all
  with `CTAB` constant tables. This is MojoShader's input format, which is how
  FNA already runs shaders on OpenGL, so the mobile path is a translation step
  rather than re-authoring 1,843 shaders. Prove this with a spike early.
- Audio is CRI ADX2 statically linked (`.CSB` cue sheets, one `.CPK`), output via
  DirectSound 8. Steam is a hard dependency with fatal-error paths, to be stubbed.
- Data footprint 1.2 GB, including ~151 MB of Episode I content under `G_EP1COM`
  and `G_EP1ZONE1-4` for Episode Metal.
- Episode I decompilation is Unlicense/public domain, so its engine code can be
  reused freely.

## Environment

- Python 3.14.5 and git are available.
- **No .NET SDK is installed** (`dotnet --list-sdks` is empty). Any C# phase
  requires installing one first. All current tooling is deliberately Python.

- **Texture banks decoded** (`docs/FORMAT-TXB.md`, `tools/txb.py`): `.TXB` is
  **big-endian** (unlike AMB) — `#TXB`, `u32` count at `0x10`, `u32` table offset
  at `0x14`, then 20-byte entries each pointing at a NUL-terminated name. All
  **651 banks** parse and every texture name resolves to a `.DDS` in the same
  archive. This is the tile-to-graphics link needed for a stage viewer.

- **Textures are standard DDS.** Sampled 530 across zones 1-3: 465 DXT1, 37 DXT5,
  22 DXT3, 6 uncompressed, all power-of-two (256x256 most common). No custom
  wrapper, no swizzling — decoding is off-the-shelf, and the only mobile concern
  is transcoding DXT to ETC2/ASTC where a device lacks S3TC.

## Tooling note

**rizin is installed** and is the RE tool for this project — no Ghidra or IDA on
this machine. It handled the binary survey (8,007 functions via `afl`).

- **Tile ids are 3D model indices** — the single most consequential finding so
  far. A geometry-layer tile id indexes `ZONE<zone>[<tileset>]_M.AMB`, which holds
  `.ZNO` models. Verified on all 13 act maps, usually an exact fit (Zone 1: 298
  models, top id 297). So the stages are grids of 3D model instances, **not**
  sprite tilemaps, and a stage viewer requires an NN geometry parser rather than
  a tile blitter. `_ATTR_*` layers use a separate id space (collision, ids > 2700).

## Repository

Public at **https://github.com/Hawaiiiiii/Sonic4Episode2**, branch `main`.
Tools and docs only — the `.gitignore` is directory-scoped on purpose. Do **not**
add bare extension globs like `*.MD` or `*.MP` to it: Windows matches ignore
patterns case-insensitively, so `*.MD` silently swallows every `.md` file in
`docs/`. That bug already cost one broken initial commit.

## Next step

**Parse `.ZNO` geometry.** This is now the critical path — it gates the stage
viewer, and the viewer is the first artifact worth showing anyone. The format is
SEGA NN chunked data, Direct3D 9 variant: first chunk `NZIF`, then `NZTL`. Start
from `G_ZONE1/MAP/ZONE1_M.AMB` and the handful of ids that dominate Zone 1 Act 1
(`Z1_G_FL_A`, `Z1_G_HASIRA_B`, `Z1_G_WL_A`) — getting one floor tile on screen is
worth more than a complete parser.

Episode I's `amObjectSetup` (`AppMain/Am/AmObject.cs`) reads the same chunked NN
format for its own platform variant and is the oracle to read first.

After that, in rough order:

1. **`.EV` object ids to names** — needs disassembly with rizin, since the ~298
   names are immediates inside each object's code rather than a table.
2. **MojoShader spike** — take one extracted `ps_3_0` shader and confirm it
   yields usable GLSL ES. The whole mobile target rests on this assumption.
3. **`.AME` effects** and the remaining minor formats.

The Episode Metal content (`G_EP1ZONE1-4`) remains the best calibration set for
any decoder, since Episode I's decompilation independently establishes what those
stages contain.
