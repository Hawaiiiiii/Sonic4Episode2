# RESUME HERE — live state

Single source of truth for where this project stands. Update at the end of every
working session. If chat history is gone, start from this file.

## Where things stand

Phase 1 (asset containers) is nearly finished and **model geometry now extracts
completely**. Decoded and verified against the whole build: the **AMB** container,
the **stage tile grids** (`.MP`/`.MD`), the **object placement tables**
(`.EV`/`.DC`/`.RG`), the **texture banks** (`.TXB`), the **NN container**
(`.ZNO`/`.ZNM`/`.ZNV`), the **NZOB object header**, and **vertex, index and mesh
set data** — 2.8M vertices and 2.5M triangles out of 3,546 models with zero
failures. The binary has been surveyed and scoped.

**No engine or game code has been written yet — nothing here runs the game on any
platform.** Overall progress against the full goal is roughly **27%**, and the
runnable figure is **0%**. See the weighted table in `plans/EXECPLAN.md`.

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

- **NN container decoded** (`docs/FORMAT-NN.md`, `tools/nn.py`): `.ZNO`/`.ZNM`/
  `.ZNV` are SEGA BINCNK — flat `tag[4] + u32 size` chunks to `NEND`. All **5,727
  containers parse, 0 failures**, and the census cross-checks exactly: 3,577
  `NZOB` = the `.ZNO` count, 669 `NZMA` = the `.ZNV` count. The tag's second
  letter is the platform code (`Z`=D3D9, `X`=Xbox, `G`=GameCube, `I`=GL ES),
  which is what proves Episode I's `NIOB`/`NITL` are the same chunks.
- `NFN0` preserves the **original authored filename with its real casing** —
  `Z1_G_hasira_B.zno` where the AMB says `Z1_G_HASIRA_B.ZNO`.
- The 50 `.XNM` files contain `NZMO`, i.e. Direct3D motions with a stale Xbox
  extension. Not console leftovers, no separate decoder needed.
- **`NZOB` object header decoded** — 88 bytes at `OfsData + OfsMainData`, giving
  bounding volume plus counts and offsets for materials, vertex lists, primitive
  lists, nodes, matrix palettes, subobjects and textures. **All 3,577 models
  parse sane, 0 failures**; 846 are skinned, 31 are geometry-less cutscene
  locators (`CAMERA_POS`, `SONIC_POS`). Self-validating: the floor tile reports
  bbox (10,10,0) with radius 14.14 = sqrt(200).
- **All internal offsets are relative to `OfsData`** (0x20), not to the chunk or
  file. Getting that base wrong yields a plausible-looking parse of nonsense.
- **GEOMETRY EXTRACTS.** All **3,546 models with geometry, 0 failures** —
  **2,820,398 vertices and 2,513,705 triangles**. `tools/nn.py export` writes
  Wavefront OBJ. Vertex format flags decode by bit (0x1 position, 0x2 normal,
  0x8/0x10 colours, 0x10000 texcoord) and every combination accounts for its
  stride exactly. All 4,085 primitive lists are mode `0x4810`, triangle strips.
- **Mesh sets are the binding, and they are 40 bytes not Episode I's 48.**
  Vertex and primitive lists are NOT positionally paired: `NNS_MESHSET` carries
  explicit `iVtxList`/`iPrimList`/`iMaterial`/`iNode`. Assuming positional
  pairing fails on ~half the corpus; using Episode I's 48-byte stride fails on
  more. Both fail with plausible-looking indices rather than obvious errors.
  Stride was measured from the gap between each mesh set array and the texture
  list after it, which divides exactly by the mesh set count.
- Independent cross-check: `ENE_HOPPER.ZNO`'s extracted vertex bounds reproduce
  its declared bounding box centre and half-extents to two decimals, from a
  different region of the file.
- **Node tree decoded — 144 bytes**, where Episode I's is 112. Second size
  divergence after the mesh set. Verified on all **846 multi-node models**: links
  in range, exactly one root each, finite non-zero scales. Strides 136 and 152
  fail on 846 and 845. Found by dumping the array and spotting the repeat after
  a brute-force sweep found nothing universal.
- **Texture coordinates and normals extract.** Attributes pack in fixed order
  with no padding (position, normal, diffuse, specular, texcoord), so an
  attribute's offset is the sum of the present ones before it. OBJ export now
  writes positions, UVs and normals; UV needs its V axis flipped for OBJ.
- **Texture list (`NZTL`) decoded** — 20-byte entries, the one struct matching
  Episode I's size. **9,665 of 9,815 references (98.5%) resolve to a real DDS.**
  The 150 that do not are effect/cutscene textures living in separately loaded
  archives. Names keep authored casing; `_dif`/`_spe`/`_env` suffixes.
- Mesh-to-texture binding is **partially** solved: a subobject lists `s32`
  indices into `NZTL`, but `Z1_G_HASIRA_B` has 3 materials against 2 textures, so
  the final selector sits in the material. Single-texture models are unambiguous,
  which covers most stage tiles.
- **Materials are variable-size** — flag-driven optional blocks, gaps of 196 and
  200 bytes within one model. Unlike every other struct here, there is no single
  stride. One block is an RGBA colour. Still need whichever field selects the
  texture bank slot.

- **STAGES ASSEMBLE** (`tools/stageview.py`). Zone 1 Act 1 instances **17,526
  tiles** into 3.7M vertices / 1.6M triangles, and the orthographic projection
  reproduces the silhouette the tile-grid render predicted — two independent
  pipelines agreeing. Writes OBJ and PNG.
- **A grid cell is 20 world units.** Dominant tile bbox in `ZONE1_M.AMB` is
  exactly 20x20, with multi-cell pieces at 40 and 60.
- **Models carry a fixed authored origin unrelated to placement.** Tile 32 at
  cells (98,0)..(98,5) reports an identical centre every time — the tileset was
  laid out side by side in an authoring scene. `stageview.py` re-centres each
  model on its bounding box before instancing. This reconstructs the engine's
  transform rather than reproducing it; correct silhouette, not proven exact.
- Rendering note: lambert shading is useless here because nearly every face
  points at the camera in a side-scroller. Colour by tile id instead.

- **SHADERS ARE TRANSLATABLE** (`docs/FORMAT-SHADER.md`, `tools/shader.py`).
  All **1,843 parse cleanly, 0 failures** — 922 `ps_3_0`, 921 `vs_3_0`, every one
  carrying `CTAB`, 98,672 instructions across **26 distinct opcodes all from the
  documented SM1-3 set**. `rep`/`endrep` balance exactly at 373 each, which is a
  free self-check on the token walk. This is MojoShader's input format, so the
  mobile shader path is off-the-shelf. Remaining risk is output quality and ES 2.0
  fallbacks, not feasibility — a large downgrade from where the plan started.

- **`NOF0` DECODED** - `u32 count`, `u32 reserved`, then `count` base-relative
  byte offsets. **3,577 models, 0 failures, 134,372 entries.** Read out of the
  loader at `Sonic.exe:0x006c6c33`: `offset >> 2` indexes u32s and the base is
  added in place. **So the file layout IS the in-memory layout** - Episode II
  relocates rather than re-parsing. `NOF0` doubles as a map of which words are
  pointers, which is the tool that finally cracked materials open.
- The same function independently confirms the chunk walk (compares `NZOB`,
  `NEND`, `NZTL`) and the **20-byte texture entry** (loop steps by `0x14` at
  `0x006c6cce`), and shows the engine uppercases texture names in place.
- **Materials partially recovered** via the relocation map: `u32 flags`,
  `u32 reserved`, pointer to a **colour block** at `+0x08` (count then RGBA
  floats), pointer to a **render-state block** at `+0x0C` (leading int then
  packed u16 pairs). Pointer `fType` `0x30000000` adds exactly 4 bytes over
  `0x10000000`. **Texture selector hypothesis**: `Z1_G_HASIRA_B`'s materials 1
  and 2 are identical except that leading int (16 vs 24) and their meshes use
  different textures - unproven, needs the draw path.

- **MATERIALS DECODED AND THE TEXTURE CHAIN CLOSED.** The binding sits at
  material `+0x18` -> texture map block `+0x04` = index into `NZTL`. **9,431 of
  9,431 verified in range, 0 out of range**; 336 materials are untextured.
  `Z1_G_HASIRA_B` - the model that defeated the earlier subobject approach with 3
  materials against 2 textures - resolves correctly: material 1 to
  `Z1_1_block_06_dif.dds`, material 2 to `Z1_1_block_21_dif.dds`.
  Full chain: `mesh set -> i_material -> material -> +0x18 -> index -> NZTL -> .DDS`.
  OBJ export now writes a matching `.mtl` with `map_Kd`.
- This also explains the `fType` size correlation: `0x30000000` materials are 4
  bytes larger than `0x10000000` ones precisely because that flag carries the
  optional texture-map pointer.

- **TEXTURED STAGE RENDERING WORKS.** `tools/dds.py` decodes **2,853/2,853
  textures, 0 failed** (DXT1 1273, DXT5 832, DXT3 662, RAW32 78, RAW16 5, RAW8 3)
  with no third-party dependency. `stageview.py` samples them per triangle:
  a 1,851-tile region of Zone 1 Act 1 renders with **240,041 of 240,041 textured
  triangles resolved**, coming out as recognisable sandstone brickwork, water and
  foliage.
- Full asset chain verified end to end:
  `AMB -> grid -> tile id -> model -> mesh set -> material -> texture index ->
  NZTL -> DDS -> pixels -> UV-mapped geometry`.
- DDS notes: uncompressed formats are handled mask-driven rather than by depth,
  since the build ships L8 luminance and X1R5G5B5 alongside B8G8R8A8. The six
  `NULL.DDS` files decode to fully transparent and that is **correct** - they are
  8x8 blank placeholders, so do not treat transparency as a decode failure.
- `stageview.py --region x,y,w,h` limits the assembly to a cell rectangle, which
  is what makes a textured render finish in reasonable time in pure Python.

## Repository

Public at **https://github.com/Hawaiiiiii/Sonic4Episode2**, branch `main`.
Tools and docs only — the `.gitignore` is directory-scoped on purpose. Do **not**
add bare extension globs like `*.MD` or `*.MP` to it: Windows matches ignore
patterns case-insensitively, so `*.MD` silently swallows every `.md` file in
`docs/`. That bug already cost one broken initial commit.

## Next step

**Build the stage viewer.** Everything it needs now exists: stage grids give tile
ids, tile ids resolve to models, models yield geometry, texture banks name the
DDS, and DDS is standard DXT. Assemble Zone 1 Act 1 by instancing each tile's
model at its grid position and render it. That is the first artifact worth
showing anyone.

Two things to settle while doing it:

1. **Vertex attributes beyond position** — the exporter writes positions only.
   Texture coordinates are at a known stride and are needed for a textured view.
2. **Materials**, to know which texture bank slot each mesh set uses. Expect a
   different size from Episode I's `NNS_MATERIAL_GLES11_DESC`, per the mesh set
   precedent.

Scale note worth confirming: the floor tile is a 20x20 unit quad and stage cells
were inferred at 64px, so the grid-to-world scale needs pinning down before
instancing.

After that, in rough order:

1. **`.EV` object ids to names** — needs disassembly with rizin, since the ~298
   names are immediates inside each object's code rather than a table.
2. **MojoShader spike** — take one extracted `ps_3_0` shader and confirm it
   yields usable GLSL ES. The whole mobile target rests on this assumption.
3. **`.AME` effects** and the remaining minor formats.

The Episode Metal content (`G_EP1ZONE1-4`) remains the best calibration set for
any decoder, since Episode I's decompilation independently establishes what those
stages contain.
