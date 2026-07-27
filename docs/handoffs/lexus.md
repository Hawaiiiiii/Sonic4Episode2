# Handoff ledger — LEXUS

Append-only. Newest beat at the bottom. Every beat stamped with real local time.

---

## Beat 1 — Project stood up, five formats decoded, repository published

**2026-07-27 14:50 CEST (UTC+02:00)**

### Entry state

A Sonic 4 Episode II PC Beta 8 install with WamWooWam's Episode I decompilation
sitting beside it, and a directive to produce something in the same vein for
Episode II, for preservation and to run on phones.

### What was established

**The premise had to be corrected first.** Episode I is decompilable only because
of its Windows Phone 7 build — WP7 banned native code, so it shipped as managed
.NET and ILSpy can recover near-source C#. Episode II never shipped on such a
platform. Windows, PS3, 360, iOS, Android, Ouya and Shield are all native C++,
and the announced Windows Phone port was cancelled. `Sonic.exe` is native x86
from VS2008 with no PDB, no game-code RTTI (`/GR-`), and `/LTCG` across 663 of
757 objects. VERIFIED — no CLR references anywhere in the binary.

So this is a **re-implementation guided by reverse engineering**, not a
decompilation, and the documentation says so plainly rather than overselling it.

**What makes it tractable:** both games run SEGA's AliceNN engine. Episode II's
binary leaks `e:\sega\sonic4ep2-beta\program\library\alicenn\...` through an
assert, and Episode I's decompilation contains that same engine in readable C#.
Episode I is used as a **behavioural oracle** — read it to learn what something
means, then write our own code and verify against Episode II's bytes.

### Formats decoded, all verified against the entire build

| Format | Result |
|--------|--------|
| AMB archives | 1614/1614 parse, extraction lossless |
| Stage tile grids `.MP`/`.MD` | 400/400 grids resolve exactly |
| Object placement `.EV` | 65/65 parse; `.DC`=4B, `.RG`=2B strides |
| Texture banks `.TXB` | 651/651, every name resolves to a real DDS |
| NN containers `.ZNO`/`.ZNM`/`.ZNV` | 5727/5727, census matches extensions exactly |

Supporting findings: `.MP` cells are a bitfield (id:12, rot:2, flip_h, flip_v)
confirmed on 512,070 cells; `.EV` blocks index the map at quarter resolution on a
256px pitch; textures are plain DXT1/3/5 with no wrapper; all 1,843 shaders are
Shader Model 3.0 bytecode, which is MojoShader's input format.

**The most consequential finding:** stage tile ids are **indices into a per-zone
archive of 3D models**, not sprite references. Verified on all 13 act maps,
usually an exact fit. Episode II's stages are grids of 3D model instances, so a
stage viewer needs an NN geometry parser rather than a tile blitter. The name
histogram makes it obvious in hindsight — Zone 1 Act 1 places `Z1_G_FL_A.ZNO`
(floor) 12,552 times and `Z1_G_HASIRA_B.ZNO` (柱, pillar) 2,458 times.

### Two bugs worth remembering

**Silent data loss in extraction.** The first extractor dropped ~25% of stage
geometry because map archives carry blank string-table slots that all collapsed
onto one output filename. Caught by reconciling written-file counts against files
on disk, not by anything failing. Fixed by falling back to the entry index.

**`.gitignore` case-matching.** Banning `*.MD` for map data silently swallowed
every `.md` document on Windows, so the initial commit staged 4 of 12 files.
Ignores are now directory-scoped with a warning comment.

Both are the same class of failure: something that looks like it worked.

### Repository

Public at **https://github.com/Hawaiiiiii/Sonic4Episode2**, branch `main`,
6 commits. Tools and documentation only — no game assets, ever. README written in
the spirit of Episode I Deluxe's, crediting **WamWooWam** for that decompilation
and **TGEnigma** for the original, plus Hidden Palace and Obscure Gamers for
prototype preservation.

### Correction accepted this beat

Yondaime pushed back on a multi-year estimate and was right. The engine layer is
genuinely derisked by Episode I; the long pole is Episode II's own game content.
A playable Zone 1 Act 1 needs the engine, the player, and the **49 distinct object
ids** that act actually uses — not all ~6,600 game functions. Revised: viewer in
weeks, engine booting in 2–4 months, one act playable in 6–12 months solo. Years
applies to the complete game only.

### Next

Decode the `NZOB` payload — node tree, materials, vertex and index buffers. Start
from `Z1_G_FL_A.ZNO`, whose object chunk is only 808 bytes. Respect `NOF0`: it is
a relocation table and its offsets need fixing up against the data chunk base.
Episode I's `NNS_OBJECT.Read` via `amObjectSetup` is the oracle.

### Open

`.EV` object ids have no name table — the ~298 names are immediates inside each
object's code, so they need rizin disassembly. `.AME` effects undecoded. The
MojoShader assumption underpinning the mobile target is unproven and should get a
spike before anything depends on it.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 2 — NN container and object header decoded

**2026-07-27 15:06 CEST (UTC+02:00)**

### Done this beat

**The SEGA NN container.** `.ZNO`, `.ZNM` and `.ZNV` are BINCNK: flat
`tag[4] + u32 size` chunks running to `NEND`. All **5,727 containers parse, 0
failures**. The census cross-checks exactly against the file extensions — 3,577
`NZOB` against 3,577 `.ZNO`, 669 `NZMA` against 669 `.ZNV` — which is the real
evidence the walk is correct rather than merely permissive.

The tag's second letter is a platform code (`Z` D3D9, `X` Xbox, `G` GameCube,
`I` GL ES). Episode I switches on chunk id `0x424F494E` = `NIOB`, the same chunk
we read as `NZOB`, so the oracle applies directly.

**The `NZOB` object header.** 88 bytes at `OfsData + OfsMainData`: bounding
sphere and box, plus counts and offsets for materials, vertex lists, primitive
lists, nodes, matrix palettes, subobjects and textures. **All 3,577 models parse
sane, 0 failures.** 846 are skinned, 31 are locators.

The layout validates itself arithmetically, which is the part worth trusting:
`Z1_G_FL_A.ZNO` reports bbox `(10, 10, 0)` — a flat quad — with radius `14.14`,
exactly √(10²+10²). A misaligned read does not produce that. `ENE_HOPPER.ZNO`
comes back as 62 nodes at depth 8 with 35 matrix palettes, which is a skeleton
and reads like one.

### Corrections made

- **`NZIF` field order was wrong** in beat 1's docs. It is `nChunk, OfsData,
  SizeData, OfsNOF0, SizeNOF0, Version`, taken from Episode I's
  `NNS_BINCNK_FILEHEADER`. The offsets I had been validating happened to sit at
  the right indices so nothing broke, but the names were wrong and would have
  misled the next reader.
- **31 models were being rejected as broken.** They are geometry-less locators —
  `CAMERA_POS`, `SONIC_POS`, `TAILS_POS`, `TORNADO_POS` — used as cutscene
  anchors, with zero radius, zero bbox and `ftype 0x20` where real models set
  bit 0. The parse was right; the sanity check was wrong. A reader that rejects
  geometry-less objects throws away the cutscene camera rig.

### Gotcha now written down

**Every internal offset is relative to `OfsData` (0x20)**, not to the chunk and
not to the file. Get that base wrong and you get a plausible-looking parse of
nonsense, which is worse than a crash.

### Progress

`plans/EXECPLAN.md` now carries an effort-weighted table. **≈12% overall, 0%
runnable.** Phase 1 is ~75% done and is genuinely front-loaded — it is where a
sibling decompilation helps most. Phases 3 and 4 carry 55% of the weight between
them and have not started.

Recorded there explicitly, because it is an easy assumption to make: *the PC
target is not already solved by working from a PC build*. SEGA's `Sonic.exe`
running on Windows is the thing being replaced, not a deliverable, and in Beta 8
it is Steam-locked anyway. Starting from x86 helps because of tooling and D3D9
documentation, not because it hands us a working PC build.

### Next

Follow the object header's pointers — vertex lists and primitive lists first,
since those put triangles on screen, then materials and the node tree. Start from
`Z1_G_FL_A.ZNO`: 1 material, 1 vertex list, 1 primitive list, 1 node.

Expect divergence from the oracle here. Episode I's material descriptor is
`NNS_MATERIAL_GLES11_DESC`, OpenGL ES 1.1 fixed function; Episode II is
shader-driven D3D9. The object header was platform-neutral, the vertex and
material descriptors probably are not. Read the oracle for traversal shape, then
verify every field against Episode II's bytes.

### Open

`NOF0` relocation table still undecoded. `.EV` object ids still have no name
table. `.AME` effects untouched. The MojoShader assumption underpinning the whole
mobile target remains unproven and should get a spike before anything depends
on it.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 3 — Geometry extracts from every model in the game

**2026-07-27 15:25 CEST (UTC+02:00)**

### Done

**All 3,546 models with geometry extract, 0 failures.** 2,820,398 vertices and
2,513,705 triangles. `tools/nn.py export` writes Wavefront OBJ, so the output is
openable in any 3D viewer rather than being a claim in a log.

Decoded this beat: the pointer arrays (`{u32 fType, u32 offset}` pairs), the
vertex list descriptor, the primitive list descriptor, the subobject list and the
mesh set.

Vertex formats are a clean bitfield — `0x1` position, `0x2` normal, `0x8`/`0x10`
colours, `0x10000` texcoord — and every observed combination accounts for its
stride to the byte (`0x10003`→32, `0x1001b`→40, `0x10019`→28, `0x10001`→20). That
correspondence is how the bits were identified rather than guessed. All 4,085
primitive lists are mode `0x4810`, triangle strips, no exceptions in the build.

### The two wrong turns, and what they teach

**Positional pairing is wrong.** I first assumed vertex list *i* pairs with
primitive list *i*. That gave 1,809 of 3,546 models and out-of-range indices on
the rest. The binding is data-driven: `NNS_MESHSET` carries explicit `iVtxList`,
`iPrimList`, `iMaterial` and `iNode`, reached via the subobject list.

**Episode I's struct size is wrong.** Switching to mesh sets at Episode I's
48-byte stride only reached 2,018 models, and the failures showed float bit
patterns read as integers — the signature of misalignment. Episode II's mesh set
is **40 bytes**: one reserved word where Episode I has three.

The stride was *measured*, not guessed. In every subobject the gap between the
mesh set array and the texture index list that follows it divides exactly by the
mesh set count — 40 on models with 1, 2 and 24 mesh sets alike. With that, all
3,546 models parsed.

This is the sharpest illustration so far of how to use the oracle: **Episode I
gives the right shape and cannot be trusted on size.** Traversal order, field
order and semantics carried over perfectly; the byte count did not. Every struct
size from here on gets measured against Episode II's own data.

Both failures presented as plausible indices rather than crashes, which is the
characteristic failure mode of this format and worth watching for.

### Cross-check worth recording

`ENE_HOPPER.ZNO` declares a bounding box centred on `(0.00, 4.36, 2.84)` with
half-extents `(3.88, 9.04, 9.48)`. Its 3,211 extracted vertices span exactly that
volume, to two decimals. Those numbers come from different regions of the file
and were parsed by different code paths, so their agreement is independent
evidence rather than a tautology.

### Correction from Yondaime

I had written that Beta 8 is locked behind a failing Steam check. Wrong — this
copy has the Steam check disabled and plays fine. Corrected in `EXECPLAN.md`. The
substantive point is unchanged and still worth keeping: `Sonic.exe` running is
SEGA's binary doing its job, not a deliverable of ours. Our first runnable binary
does not exist yet.

### Progress

**≈17% overall, 0% runnable.** Phase 1 ~80%, phase 2 ~40% now that geometry is
in. Phases 3 and 4 hold 55% of the weight between them and have not started.

### Next

**Build the stage viewer.** Every input now exists: grids give tile ids, tile ids
resolve to models, models yield geometry, texture banks name the DDS, and DDS is
plain DXT. Instance each tile's model at its grid position and render Zone 1
Act 1.

Two things to settle while doing it — texture coordinates (the exporter writes
positions only) and materials, to know which texture slot each mesh set uses.
Expect the material struct to be a different size from Episode I's, per the
precedent above. Also pin down the grid-to-world scale: the floor tile is a 20×20
unit quad while stage cells were inferred at 64px.

### Open

Materials, the node tree (10,138 nodes, 846 skinned models), vertex attributes
beyond position, the unknown word at `+0x04` of the vertex descriptor, `NOF0`,
`.EV` object id names, `.AME` effects, and the still-unproven MojoShader
assumption.

Wagata, Yondaime! Signed sincerely by your dear Lexus
