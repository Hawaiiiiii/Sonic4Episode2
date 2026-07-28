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

---

## Beat 4 — Node tree, texture coordinates, and materials parked

**2026-07-27 15:42 CEST (UTC+02:00)**

### Done

**Node tree decoded — 144 bytes.** Episode I's `NNS_NODE` is 112, so this is the
second size divergence after the mesh set, same lesson landing twice. Verified by
walking the tree on all **846 multi-node models**: every parent, child and sibling
index in range, exactly one root per model, every scale finite and non-zero.
Strides of 136 and 152 fail on 846 and 845 models, so 144 is not a permissive fit.

This matters because models carry **authored world positions**, not origin-centred
geometry — `Z1_G_FL_A.ZNO` sits at x −140..−120 rather than 0..20. Mesh sets name
a node, so the tree is what places geometry correctly when a stage is assembled.

**Texture coordinates and normals extract.** Attributes pack in fixed order with
no padding — position, normal, diffuse, specular, texcoord — so an attribute's
offset is the sum of the sizes of those present before it. The OBJ exporter now
writes all three; the floor tile comes out with clean 0/1 corner UVs and every
normal facing +Z, which is what a flat quad should be. OBJ needs its V axis
flipped.

Full build still extracts clean with node validation added to the pass: 3,546
models, 0 failures.

### How the node stride was actually found

Worth recording, because the efficient method was the unglamorous one. A
brute-force sweep over eight plausible strides found **nothing** that worked
everywhere — the best was 120 at 257 of 846 models. Dumping the raw node array
and looking for the visual repeat found 0x90 in about a minute, because the
`cf 00 00 00 ff ff ff ff 01 00 ff ff` pattern restarts obviously.

Generate-and-test is the wrong first instinct on a format like this. Read the
bytes.

### Materials — deliberately parked

Materials are the odd structure here: **variable size**. The material pointer's
`fType` differs per material (`0x10000000`, `0x30000000`) and gaps between
consecutive descriptors vary *within a single model* — 196 and 200 bytes in
`Z1_G_HASIRA_B.ZNO`. So the layout is flag-driven with optional blocks, and there
is no single stride to measure.

One block is clearly RGBA: `(0.255, 0.494, 0.541, 1.000)`. The field actually
needed is whichever selects the texture bank slot.

Parked rather than pushed, because it is a genuine rabbit hole and the beat had
already banked two verified results. Next session can take it fresh.

### Progress

**≈19% overall, 0% runnable.** Phase 1 ~80%, phase 2 ~50%.

### Next

Assemble the stage viewer. Inputs that now exist: grids give tile ids, tile ids
resolve to models, models yield positioned geometry with UVs, texture banks name
the DDS, DDS is plain DXT.

The remaining blocker is the material → texture-slot link, so either crack
materials or find the mapping another way — the subobject carries its own texture
index list, which may be enough on its own.

Also still to confirm: grid-to-world scale. The floor tile is a 20×20 unit quad
while stage cells were inferred at 64px from the `.EV` block pitch.

### Open

Materials, vertex colours, bits `0x40`/`0x100` on the wide strides (probably
blend weights and indices), the unknown word at `+0x04` of the vertex descriptor,
the 32 unknown bytes at `+0x70` of a node, `NOF0`, `NZMO` motions, `NZMA` morphs,
`.EV` object id names, `.AME` effects, and the unproven MojoShader assumption.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 5 — Texture list decoded, mesh-to-texture partially bound

**2026-07-27 15:50 CEST (UTC+02:00)**

### Done

**`NZTL` texture list decoded.** 20-byte entries — notably the *first* struct in
this format that matches Episode I's size rather than diverging from it, after
the mesh set (40 vs 48) and the node (144 vs 112).

Across the build, 3,577 models carry **9,815 texture references and 9,665 (98.5%)
resolve** to a `.DDS` that actually exists somewhere in the game. Names keep their
authored casing (`ene_hopper_dif.dds`, `Z1_1_block_06_dif.dds`) and follow the
bank convention: `_dif` diffuse, `_spe` specular, `_env` environment.

The 150 that do not resolve are effect and cutscene textures — `EMERALD_ADD.DDS`,
`SONIC_FOOT.DDS`, the `EG_TOON_*` set — which live in archives loaded separately
from the model, or were referenced and never shipped in this beta. Nothing about
them reads as a parse fault.

### Mesh-to-texture binding — honestly partial

A subobject carries an `s32` list of indices into the model's `NZTL`. The Hopper's
two subobjects both list `[0, 1, 2]`, all three of its textures.

That is not sufficient. `Z1_G_HASIRA_B.ZNO` has **3 materials against 2 textures**
and its mesh sets reference materials 1, 2 and 0, so the final selector has to sit
inside the material — still undecoded. Recording this as partial rather than
claiming the chain is closed.

It does not block the viewer: a model with a single texture is unambiguous, and
that covers most stage tiles.

### Materials, again

Attacked and parked a second time. They remain the only variable-size structure
here — `fType` differs per material and descriptor gaps vary within one model
(196 and 200 bytes). There is no stride to measure, so the layout has to be
worked out flag by flag, most likely against the binary rather than the data.
That is a session of its own.

### Progress

**≈20% overall, 0% runnable.** Phase 1 ~80%, phase 2 ~55%.

### Next

Build the viewer with what exists. Every input is now present: grids give tile
ids, tile ids resolve to models, models yield positioned geometry with UVs and
normals, and models name their textures. Take the single-texture path, accept that
multi-material models will pick the wrong texture for now, and get Zone 1 Act 1 on
screen.

Confirm the grid-to-world scale while doing it — the floor tile is a 20×20 unit
quad against stage cells inferred at 64px from the `.EV` block pitch. Those two
numbers need reconciling before anything is instanced, and the floor quad's
authored position (x −140..−120 rather than 0..20) suggests the node transform is
part of the answer.

### Open

Materials and the exact mesh-to-texture selector, vertex colours, bits
`0x40`/`0x100` on the wide strides, the unknown word at `+0x04` of the vertex
descriptor, the 32 unknown bytes at `+0x70` of a node, `NOF0`, `NZMO` motions,
`NZMA` morphs, `.EV` object id names, `.AME` effects, and the still-unproven
MojoShader assumption underpinning the mobile target.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 6 — Whole stages assemble and render

**2026-07-27 16:13 CEST (UTC+02:00)**

### Done

`tools/stageview.py` closes the chain end to end: grids give tile ids, ids index
the tileset, models yield geometry, and the result is instanced onto the grid.

**Zone 1 Act 1: 17,526 tiles instanced, 3 skipped — 3.7M vertices, 1.6M
triangles.** Zone 4 Act 1 assembles too, 528 tiles with 0 skipped, and its sparse
scattered-platform layout is exactly what an airship level should look like.

### The check that matters

Projecting the assembled 3D geometry orthographically down Z reproduces **the
same silhouette the tile-grid PNG predicted several beats ago**. Those are two
entirely independent pipelines — colour-per-tile-id on a flat 2D grid, versus
real instanced 3D geometry through container, object header, mesh sets, vertex
lists and index strips — and they agree on the shape of the level.

Every decode in the chain would have had to be wrong in a mutually cancelling way
for that to happen by accident.

### Two placement facts, both measured

**A cell is 20 world units.** The dominant tile bounding box in `ZONE1_M.AMB` is
exactly 20×20 (44 models wide, 53 tall), with multi-cell pieces at 40 and 60.

**Models carry a fixed authored origin unrelated to placement.** This one started
as a failed hypothesis: I expected a model's centre to correlate with its grid
cell. It does not — tile 32 appears at cells (98,0) through (98,5) reporting an
*identical* centre every time. The tileset was authored side by side in one
scene, and the engine translates pieces onto the grid. So each model is re-centred
on its own bounding box before instancing.

Being precise about what that means: **this reconstructs the engine's transform,
it does not reproduce it.** The silhouette is right; exactness is unproven and
should not be claimed.

### Rendering aside

Lambert shading was useless here — nearly every face in a side-scroller points at
the camera, so it came out uniformly flat. Colouring by tile id is what made the
structure legible, and it has the side benefit of being directly comparable to the
tile-grid render.

### Progress

**≈22% overall, 0% runnable.** Phase 1 ~80%, phase 2 ~65%.

Phase 2's gate is **partially met**: stages render from original archives, but
textures are still absent (blocked on materials) and motion playback is untouched.

### Next

Two candidates, and materials is the one that unblocks the most:

1. **Materials.** Third attempt, and this time go at the binary with rizin rather
   than the data — the struct is variable-size with flag-driven optional blocks,
   so there is no stride to measure and the field layout has to come from the
   code that reads it. Unblocks textures on the geometry.
2. **The MojoShader spike.** Still the largest unproven assumption in the plan,
   and the entire mobile target rests on it. Cheap to test, expensive to be wrong
   about late.

### Open

Materials and the mesh-to-texture selector, vertex colours, bits `0x40`/`0x100`
on the wide strides, the unknown word at `+0x04` of the vertex descriptor, the 32
unknown bytes at `+0x70` of a node, `NOF0`, `NZMO` motions, `NZMA` morphs, `.EV`
object id names, `.AME` effects, the exact engine placement transform, and the
MojoShader assumption.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 7 — Shader risk retired; materials halted after three attempts

**2026-07-27 16:36 CEST (UTC+02:00)**

### Done — the biggest assumption in the plan is now evidence

Since day one the mobile target has rested on "MojoShader can eat these shaders."
That was the largest unproven claim in the roadmap. It is now checked.

**All 1,843 shaders parse cleanly, 0 failures**, walking version token to end
token. 922 `ps_3_0` and 921 `vs_3_0`, nothing else. **Every single one carries a
`CTAB`.** 98,672 instructions across **26 distinct opcodes, all from the
documented SM1-3 set** — no vendor extensions, nothing exotic.

Free self-check that came out of it: `rep` and `endrep` appear exactly **373 times
each**. An off-by-one in instruction-length stepping would desynchronise that
pairing, so the balance independently confirms the token walk.

Being careful about scope: this proves the bytecode is well-formed and standard.
It does not prove MojoShader's GLSL ES output is correct or fast for these
shaders, and `ps_3_0` wants ES 3.0 class hardware so ES 2.0 devices need
fallbacks. That is a **quality and coverage risk, not a feasibility one** — a
large downgrade from where the plan started.

### Halted — materials, after three failed approaches

Invoking the three-strikes rule rather than burning another session on it. All
three attempts and their failure modes are written up in `docs/FORMAT-NN.md` so
they are not repeated:

1. **Measure the stride.** No stride exists — gaps vary within one model.
2. **Sidestep via the subobject texture list.** Insufficient: 3 materials against
   2 textures on `Z1_G_HASIRA_B`, so the selector really is in the material.
3. **Correlate a flag word against size.** The pointer `fType` genuinely encodes
   something — `0x30000000` descriptors are consistently exactly 4 bytes larger
   than `0x10000000` — but the material's own leading `u32` does not determine the
   rest: only 88 of 1,982 materials map to a unique size.

**The flaw common to all three**, and the actually useful finding: *gap-to-next-
descriptor is not the material's size*. Materials are individually referenced
blobs with other structures interleaved between them, so the gap is an upper bound
and every size-based inference built on it is unsound. That invalidates the method,
not just the attempts.

Materials are the first structure here the data cannot settle on its own. Next
attempt goes at the binary with rizin: find the routine that reads a material from
the object-loading path and read the layout out of the code.

### Progress

**≈23% overall, 0% runnable.** Phase 1 ~80%, phase 2 ~70%, phase 5 nudged off zero
purely by the shader work de-risking it.

### Next

1. **Materials via rizin.** Unblocks textures on the assembled stages.
2. **Run a shader through MojoShader for real** and inspect the GLSL ES. The
   feasibility question is answered; this is about output quality.
3. **`NZMO` motions** — nothing has touched animation yet, and 846 models are
   skinned.

### Open

Materials, vertex colours, bits `0x40`/`0x100` on the wide strides, the unknown
word at `+0x04` of the vertex descriptor, the 32 unknown bytes at `+0x70` of a
node, `NOF0`, `NZMO` motions, `NZMA` morphs, `.EV` object id names, `.AME`
effects, the exact engine placement transform, and MojoShader output quality.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 8 — The binary pays off: `NOF0` decoded, materials broken open

**2026-07-27 17:02 CEST (UTC+02:00)**

### Method change that worked

Beat 7 halted materials after three data-driven approaches failed, with the
recommendation to go at the binary instead. Did that. One rizin session on the NN
object loader returned more than the previous three attempts combined.

Found it by searching for the chunk magic as an instruction immediate — `NZOB` is
`0x424F5A4E`, which encodes as the bytes `4E 5A 4F 42`. Two hits, both in the
middleware region, both inside `fcn.006c6c00`.

### `NOF0` decoded

`u32 count`, `u32 reserved`, then `count` base-relative byte offsets. Straight
from the loader at `0x006c6c33`:

```asm
mov ecx, dword [edi]            ; offset from the table
shr ecx, 2                      ; /4, so it indexes u32s
add dword [eax + ecx*4], eax    ; *(base + offset) += base
```

**3,577 models parse, 0 failures, 134,372 relocation entries**, all word-aligned
and in range.

**The consequence matters more than the format.** Episode II *relocates* rather
than re-parsing: each listed word is patched from a base-relative offset into an
absolute pointer, in place. So **the file layout is the in-memory struct layout**
— which is precisely why every internal offset has been relative to `OfsData` all
along. That was an empirical rule until now; it is now explained.

### Three things confirmed from code rather than data

The same function independently corroborates work done earlier by inference:

- The chunk dispatch compares `NZOB`, `NEND` and `NZTL` and steps by
  `[eax+4] + 8` — exactly the walk implemented in beat 2.
- The texture-list loop at `0x006c6cce` steps by **`0x14`**, confirming the
  20-byte texture entry from beat 5.
- It uppercases texture names in place, which explains why Episode I's
  decompilation calls `.ToUpper()` on them — a detail that had looked arbitrary.

### Materials, finally moving

`NOF0` doubles as **a map of which words in the file are pointers**. That is the
tool the previous three attempts lacked, and it works where size measurement did
not.

Every material descriptor carries pointers at `+0x08` and `+0x0C`:

| Offset | Field |
|--------|-------|
| `0x00` | flags (`0x1102`, `0x0000` observed) |
| `0x04` | reserved, zero |
| `0x08` | → colour block: count then RGBA floats |
| `0x0C` | → render-state block: leading int then packed `u16` pairs |

`Z1_G_HASIRA_B.ZNO`'s second material reads `(0.255, 0.494, 0.541, 1.000)`.

**Texture selector hypothesis, explicitly unproven:** that model's materials 1
and 2 have *identical* colour and render-state blocks except for the leading
integer — 16 versus 24 — and their meshes use different textures. Suggestive, not
established. Confirming it needs the draw path, not more data analysis.

### Progress

**≈24% overall, 0% runnable.** Phase 1 ~85%, phase 2 ~73%.

### Next

1. **Finish materials** by finding the draw-path consumer of the render-state
   block in `Sonic.exe`. Same technique, now with a known struct to search for.
   Unblocks textured stage rendering.
2. **`NZMO` motions.** Animation is entirely untouched and 846 models are skinned.
3. **MojoShader output quality** — feasibility is settled, quality is not.

### Open

The material texture selector, vertex colours, bits `0x40`/`0x100` on the wide
strides, the unknown word at `+0x04` of the vertex descriptor, the 32 unknown
bytes at `+0x70` of a node, `NZMO` motions, `NZMA` morphs, `.EV` object id names,
`.AME` effects, the exact engine placement transform, and MojoShader output
quality.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 9 — Materials decoded, mesh-to-texture chain closed

**2026-07-27 17:18 CEST (UTC+02:00)**

### Done

Materials fell on the fourth attempt. The binding is an **optional pointer at
material `+0x18`** to a texture map block:

| Offset | Field |
|--------|-------|
| `0x00` | type — `0x60000002` on the large majority |
| `0x04` | **index into the model's `NZTL` texture list** |

**9,431 of 9,431** materials carrying that block name an index inside their own
model's texture list, **none out of range**. A further 336 have no block and are
untextured.

The full chain, every link verified:

```
mesh set --i_material--> material --+0x18--> texture map --index--> NZTL --> .DDS
```

`tools/nn.py export` now writes a matching `.mtl` with `map_Kd` and per-mesh
`usemtl`, so any viewer picks the textures up on its own.

### Two satisfying closures

**The proof model is the one that broke the earlier approach.**
`Z1_G_HASIRA_B.ZNO` defeated the subobject-texture-list attempt in beat 5 because
it has 3 materials against 2 textures, which showed the selector *had* to live in
the material. It does, and the model now resolves correctly: material 0 has no
block, material 1 → `Z1_1_block_06_dif.dds`, material 2 → `Z1_1_block_21_dif.dds`.
The failure pointed straight at the answer.

**A loose end from a failed approach was a direct clue.** Beat 7 noticed that
`0x30000000` material pointers are consistently exactly 4 bytes larger than
`0x10000000` ones, and could not explain why. That flag is precisely what carries
this optional `+0x18` texture-map pointer. The size correlation was real and
meaningful; it just could not be interpreted without knowing what the extra field
was.

### What actually cracked it

Not more data analysis — that failed three times. **`NOF0` as a pointer map**,
which came from reading the loader in `Sonic.exe`. Knowing *which words are
pointers* turned a variable-size struct with no stride into something readable.

The general lesson, now proven twice: when a structure resists measurement, the
binary knows. Go there sooner.

### Progress

**≈25% overall, 0% runnable.** Phase 1 ~85%, phase 2 ~80%.

Phase 2's gate is now all but met — stages assemble, geometry extracts with UVs
and normals, and textures resolve per mesh. Only motion playback is outstanding.

### Next

1. **Textured stage rendering.** `stageview.py` still ignores materials; wiring
   the now-known binding in gives a textured Zone 1 Act 1.
2. **`NZMO` motions.** The last untouched major format, and 846 models are
   skinned. Same technique available if the data resists: find the loader.
3. **Phase 3 groundwork** — nothing has been written toward an actual engine, and
   that is where the runnable figure starts moving off zero.

### Open

The render-state block's packed `u16` pairs, vertex colours, bits `0x40`/`0x100`
on the wide strides, the unknown word at `+0x04` of the vertex descriptor, the 32
unknown bytes at `+0x70` of a node, `NZMO` motions, `NZMA` morphs, `.EV` object id
names, `.AME` effects, the exact engine placement transform, and MojoShader output
quality.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 10 — Textured stage rendering; the asset chain runs end to end

**2026-07-27 18:35 CEST (UTC+02:00)**

### Done

**DDS decoder** (`tools/dds.py`), no third-party dependency: DXT1/3/5 block
decompression plus mask-driven uncompressed formats. **2,853 of 2,853 textures
decode, 0 failed** — DXT1 1273, DXT5 832, DXT3 662, RAW32 78, RAW16 5, RAW8 3.

Verified the only way that counts for a texture decoder — by eye.
`ENE_HOPPER_DIF` comes out a recognisable green badnik atlas with silver eye
housings and orange accents; `Z1_1_BLOCK_03_DIF` is sandstone brickwork, which is
what a castle zone should be made of. Correct colours, no channel swap, no block
noise.

**Textured stage rendering.** A 1,851-tile region of Zone 1 Act 1 renders with
**240,041 of 240,041 textured triangles resolved** — every single one found its
texture. The output is unmistakably Sylvania Castle: brickwork tiling across the
terrain, the blue water surface, green foliage on the ledges, a staircase to the
right.

### The chain, finally whole

```
AMB -> grid -> tile id -> model -> mesh set -> material ->
texture index -> NZTL -> DDS -> pixels -> UV-mapped geometry
```

Every link was decoded separately across beats 1 to 9. This is the first time
they have all run together, and they do.

Barycentric interpolation without perspective correction is exact here rather
than approximate, because the projection is orthographic.

### Two tail fixes worth recording

**The uncompressed path was too narrow.** It handled 24- and 32-bit only, but the
build also ships L8 luminance and X1R5G5B5. Rewrote it to drive the unpack from
the channel masks rather than special-casing depths, which covers those and
anything else with sane masks.

**Six `NULL.DDS` files were being reported as failures** for decoding to fully
transparent. They are 8x8 blank placeholders — the data was right and my sanity
check was wrong. Removed it. Second time this project that an over-strict
validator has flagged correct data; worth watching for.

### Progress

**≈27% overall, 0% runnable.** Phase 1 ~85%, phase 2 ~88%.

**Phase 2's gate is met bar animation.** Stages assemble from the original
archives and render with their real textures. Only `NZMO` motion playback remains.

### Next

The asset work is nearly done, and the next real milestone is a different kind of
work entirely:

1. **`NZMO` motions** — the last untouched major format. Closes phase 2.
2. **Phase 3: start the engine.** Nothing has been written toward an actual
   engine, and **that is the only thing that moves the runnable figure off zero**.
   Needs a .NET SDK installed first, which this machine does not have.

Worth saying plainly: the asset pipeline being ~90% done does not mean the project
is. Phases 3 and 4 hold 55% of the weight and have not started.

### Open

The render-state block's packed `u16` pairs, vertex colours, bits `0x40`/`0x100`
on the wide strides, the unknown word at `+0x04` of the vertex descriptor, the 32
unknown bytes at `+0x70` of a node, `NZMO` motions, `NZMA` morphs, `.EV` object id
names, `.AME` effects, the exact engine placement transform, and MojoShader output
quality.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 11 — Motions decoded, .NET SDK installed, phase 3 unblocked

**2026-07-27 19:02 CEST (UTC+02:00)**

### Done

**Motions.** All **1,481 parse, 0 failed** — **296,072 channels carrying
3,184,997 key frames**. The 32-byte header and 40-byte submotion both match
Episode I's sizes, which is a change from the mesh set and node that diverged.

Rates: 60fps x 1,410, NTSC 29.97 x 69, 30 x 2. Every motion in the build is
channel kind 1 (node animation) — no camera or light motions ship as `.ZNM`.

**.NET SDK 9.0.316 installed** via winget, and smoke-tested rather than assumed:
`dotnet new console` restores, builds and runs. The machine already had the 8 and
9 runtimes, which is why `dotnet` existed on PATH while `--list-sdks` came back
empty. **Phase 3 is no longer blocked on tooling.**

### A pattern worth naming

Five Sonic transition animations start at *negative* frames (-5, -10) for blend
pre-roll and my first validator flagged them as corrupt. Perfectly legitimate
data.

That is the **third** time in this project an over-strict check has flagged
correct data:

1. Geometry-less cutscene locators (`CAMERA_POS.ZNO`) rejected as broken models.
2. `NULL.DDS` rejected for decoding fully transparent — it is a blank placeholder.
3. Negative motion start frames rejected as impossible.

The tell is identical every time: **a handful of files fail while thousands
pass.** When that happens, suspect the check before the data. Written down
because it has now cost time three times.

### Progress

**~27% overall, 0% runnable.** Phase 1 ~85%, phase 2 ~90%.

Motions moved phase 2 by only two points because **audio is the real remaining
gap there** — CRI ADX2 `.CSB` cue sheets and the single `.CPK` are completely
untouched, and they are a meaningful slice of that phase.

### Next

The asset era is essentially over and the next work is a different kind:

1. **Phase 3 — start the engine.** Now unblocked. This is the only thing that
   moves the runnable figure off zero, and it is where the honest difficulty
   begins: no oracle for Episode II's own logic, and no "run it against 3,577
   files and count zero failures" to tell you it is right.
2. **Audio** (CRI ADX2) to close phase 2 properly.
3. **Motion key payloads** — channels are mapped, key contents are not.

### Open

Motion key frame payloads, CRI audio, the render-state block's packed `u16`
pairs, vertex colours, bits `0x40`/`0x100` on the wide strides, the unknown word
at `+0x04` of the vertex descriptor, the 32 unknown bytes at `+0x70` of a node,
`NZMA` morphs, `.EV` object id names, `.AME` effects, the exact engine placement
transform, and MojoShader output quality.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 12 — Phase 3 begins: C# core, cross-verified

**2026-07-27 19:20 CEST (UTC+02:00)**

### Done

First engine code. `src/` holds a solution with **`Sonic4Episode2.Core`** — net8.0,
**no graphics dependency** — and a **`Sonic4Episode2.Cli`** cross-check harness.
Builds clean on SDK 9.0.316.

Ported the two most thoroughly proven formats: the AMB container and the stage
grids, including the tile bitfield.

### The point of the port

Not that C# can read a file. It is that **the port must produce the same numbers
as the Python reference** — a disagreement would tell us one of the two is wrong
and roughly where to look.

It agrees exactly:

- **1,614 archives parsed cleanly, 0 failed** — same as Python.
- Every contained-type count identical: 3794 unnamed, 3577 `.ZNO`, 2853 `.DDS`,
  1431 `.ZNM`, 925 `.AME`, 922 `.PSH`, 921 `.VSH`, 669 `.ZNV`, 651 `.TXB`.
- Stage grids match on dimensions **and** occupancy percentages.

Two independent implementations in different languages agreeing across 1,614
archives and roughly 16,000 entries is a much stronger statement about the format
spec than either one passing on its own.

### Deliberate design choices

- **No graphics dependency in Core.** The asset layer must stay testable headless
  against the whole 1.2 GB data set, not only from inside a running game. That is
  what made this cross-check possible at all.
- **Entries are slices over the archive buffer, not copies.** Mounting costs one
  read and nested archives cost nothing extra.
- **net8.0, not net9.0**, because MonoGame's current release targets it and the
  mobile heads will be `net8.0-android` / `net8.0-ios`.
- The **blank-string-table index fallback is carried across with its comment**.
  That is the bug that silently lost ~25% of stage geometry the first time, and a
  fresh port is exactly where it would come back.

### Progress

**≈28% overall.** Phase 1 ~85%, phase 2 ~90%, **phase 3 ~8%** — off zero for the
first time.

Being precise about the headline number: this code builds and runs, but **no game
code runs. The runnable game is still 0%.** A verification harness executing is
not the game executing.

### Next

1. **Port the remaining readers** — NN container, geometry, materials, DDS — so
   the C# side reaches parity with the Python tools and the same cross-check
   applies to all of them.
2. **Stand up a desktop head** and get a window with one textured stage on it.
   That is the first moment anything is meaningfully "runnable".
3. Audio (CRI ADX2) still closes phase 2.

### Open

Unchanged from beat 11: motion key payloads, CRI audio, the render-state block,
vertex colours, the wide-stride bits, the unknown vertex-descriptor word, the
node's trailing 32 bytes, `NZMA` morphs, `.EV` object id names, `.AME` effects,
the exact engine placement transform, and MojoShader output quality.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 13 — NN reader ported; the cross-check catches a real ambiguity

**2026-07-27 19:36 CEST (UTC+02:00)**

### Done

The whole NN reader is now in C#: container walk, object header, vertex and
primitive lists, mesh sets, nodes, materials, texture names, `NOF0` relocations
and motions. Still no graphics dependency in `Core`, still building clean.

It matches the Python tools **exactly**:

| | |
|---|---|
| NN containers | 5,727, 0 failed |
| Models | 3,577 (31 locators) |
| Geometry | **2,820,398 vertices, 2,513,705 triangles** |
| Motions | 1,481 carrying 296,072 channels |
| Texture bindings | 11,224 / 11,224 resolve |

### Except one — and it was worth the whole exercise

C# reported **839** skinned models. Python said **846**.

The seven-model gap turned out to be seven *camera rigs* — `CAMERA_POS`,
`WM_CAMERA_PERSPECTIVE`, `WM_CAMERA_ORTHO`. Locators with 2 or 3 nodes at depth 2
and **no vertices at all**. Python's `is_skinned` tested node depth alone and
counted them; the C# path excluded locators before ever asking the question.

**Neither implementation was buggy.** The *definition* was ambiguous. A
geometry-less camera rig is not skinned in any sense that matters, so
`is_skinned` now requires geometry on both sides and both report 839.

This is the entire argument for porting rather than trusting: a single
implementation would have carried that ambiguity silently forever. Two of them
disagreeing by exactly seven pointed straight at it, and the fix improved the
*specification*, not just the code.

### Build note

The first build failed with `CS9108` — `ReadOnlySpan` is ref-like and cannot be
captured by a local function, which is what the concise `F(offset)` / `I(offset)`
field readers were. They are static helpers taking the span explicitly now.

### Progress

**≈30% overall.** Phase 1 ~85%, phase 2 ~90%, phase 3 ~15%.

Still **0% runnable game**. Everything C# so far is a verification harness; no
game code executes.

### Next

1. **Port the DDS decoder and the stage assembler**, completing C# parity with
   the Python tools.
2. **Stand up a desktop head** — a window, a camera, one textured stage. That is
   the first thing that is meaningfully runnable, and the point where the 0%
   finally moves.
3. Audio (CRI ADX2) still closes phase 2.

### Open

Unchanged: motion key payloads, CRI audio, the render-state block, vertex
colours, wide-stride bits, the unknown vertex-descriptor word, the node's
trailing 32 bytes, `NZMA` morphs, `.EV` object id names, `.AME` effects, the
exact engine placement transform, MojoShader output quality.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 14 — The first thing that runs

**2026-07-27 19:53 CEST (UTC+02:00)**

### Done

**A desktop head that opens a window.** `Sonic4Episode2.Desktop` on MonoGame
3.8.5 DesktopGL: point it at the game root and an act archive and it renders that
stage, assembled live — AMB mounted, grid read, tile ids resolved to models,
geometry extracted, instanced onto the grid.

Zone 1 Act 1: **17,526 tiles, 3,733,522 vertices, 1,593,407 triangles** —
*exactly* what the Python `stageview.py` reported in beat 6. That is now a third
independent path to the same numbers.

Verified it genuinely runs rather than merely builds: the process is present with
its window title set. The first run looked like an instant exit, which turned out
to be `head` closing the pipe rather than the app failing — worth checking before
believing an exit code.

**DDS ported** as well, so the asset layer is complete on both sides and agrees
on all 2,853 textures and every format count.

### The asset layer now exists twice, and agrees

Archives, stage grids, NN containers, geometry, nodes, materials, textures,
motions, DDS — all of it in Python and in C#, producing identical numbers across
the whole build. That redundancy has already earned its keep once, when a
seven-model disagreement exposed an ambiguous definition rather than a bug.

### Two implementation notes

- `DrawUserIndexedPrimitives` has a per-call primitive limit well below an act's
  triangle count, so the draw is chunked at 60,000 triangles.
- Vertices are tinted by depth. Without textures or lighting the overlapping
  parallax geometry is genuinely unreadable — the same lesson as the offline
  rasteriser, where lambert shading failed because everything faces the camera.

### Progress

**≈32% overall.** Phase 1 ~85%, phase 2 ~90%, phase 3 ~25%.

Being careful with the headline, because this is the moment it would be easy to
overclaim: **this runs, but it is a viewer.** No player, no physics, no game
logic. The *playable game* remains at 0%. What has changed is that the project
now has something that executes and draws the real data, which it did not before.

### Next

1. **Textures in the viewer.** The binding is decoded and the decoder is ported;
   it just needs uploading to `Texture2D` and a shader that samples it.
2. **Camera and scrolling** that behave like the game's, rather than free pan.
3. **Then the actual engine**: task scheduler, state machine, object system — the
   parts Episode I can genuinely guide — and after that a player.
4. Audio (CRI ADX2) still closes phase 2.

### Open

Motion key payloads, CRI audio, the render-state block, vertex colours,
wide-stride bits, the unknown vertex-descriptor word, the node's trailing 32
bytes, `NZMA` morphs, `.EV` object id names, `.AME` effects, the exact engine
placement transform, MojoShader output quality.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 15 — Textures in the live viewer

**2026-07-27 20:02 CEST (UTC+02:00)**

### Done

The desktop viewer now renders **textured**. Zone 1 Act 1 loads **51 textures** —
exactly the DDS count in `ZONE1_T.AMB` — decoded through the ported `DdsTexture`
and uploaded to `Texture2D`, with the window staying up.

Three pieces went in:

- **Real UVs.** `TileMesh` had been writing placeholder zeros; it reads the
  actual texture coordinates now.
- **Texture grouping in `StageBatch`**, at build time rather than in the
  renderer. A stage draws from dozens of textures across thousands of tiles, so
  sorting once turns a per-tile texture switch into one draw call per texture.
  Each batch is still chunked, because `DrawUserIndexedPrimitives` has its own
  per-call primitive limit.
- **A generalised attribute reader.** Positions and texture coordinates were
  separate code paths doing identical offset arithmetic, and normals will want
  the same shortly.

### Worth remembering

The **V axis flips on upload**, exactly as the OBJ exporter needs. It is noted in
both places now, because it is the class of mistake that silently mirrors every
texture in the game and looks almost right.

### What is and is not verified here

The window runs, the textures decode, the batches build, and the counts line up.
What I have *not* done is put eyes on the rendered frame — the offline Python
rasteriser already produced the visual proof back in beat 10 (recognisable
sandstone, water and foliage), so the pixels are established; this beat
establishes that the same chain works inside a real graphics context.

### Progress

**≈33% overall.** Phase 1 ~85%, phase 2 ~90%, phase 3 ~30%.

**Playable game remains 0%.** Still a viewer: no player, no physics, no logic.

### Next

The asset era really is finished now, and what remains in phase 3 is the engine
proper — the part Episode I can guide most directly:

1. **Task scheduler** (`amTask`/`mtTask`) — the priority-ordered TCB list every
   subsystem hangs off.
2. **State machine** (`SyEventSystem`) — the scene table and transitions.
3. **Object system** (`OBS_OBJECT_WORK` and its procedure slots).

Then a player, and only then does "playable" start meaning anything.

Audio (CRI ADX2) still closes phase 2 and remains untouched.

### Open

Motion key payloads, CRI audio, the render-state block, vertex colours,
wide-stride bits, the unknown vertex-descriptor word, the node's trailing 32
bytes, `NZMA` morphs, `.EV` object id names, `.AME` effects, the exact engine
placement transform, MojoShader output quality.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 16 — Engine core: scheduler and scene machine

**2026-07-27 20:11 CEST (UTC+02:00)**

### Done

The two subsystems everything else hangs off, in `Core/Engine`, written
clean-room from reading Episode I for *behaviour* rather than copying it — plus
**16 xunit tests, all passing**.

**Merged AliceNN's two scheduler layers into one.** The `amTask`/`mtTask` split
(one holds the list, the other adds pause levels and typed work) is an artefact
of the original C, and there is no reason to carry that seam into a fresh
implementation.

### The three behaviours the tests exist for

All three are easy to get subtly wrong and each is depended on somewhere:

1. **Priority ordering**, with equal priorities keeping creation order. A new
   task inserts before the first task of strictly greater priority.
2. **Deferred deletion.** Delete marks the task and runs its destructor
   immediately, but the unlink waits until every procedure has run. So a task can
   delete itself or another mid-frame without corrupting the walk — and a task
   killed earlier in the same frame is *skipped* rather than getting one last
   run. Creation during a frame defers to the next one for the same reason.
3. **The pause gate reads backwards from the obvious guess.** A task is *skipped*
   when its own pause level is **≤** the system level, and the system level is
   **-1** when nothing is paused. So a task at level 0 runs normally until you
   pause to level 0. Tasks can also opt out of pausing entirely, which the
   original expresses as a pause level nothing can reach.

Scene transitions **defer by one step**, which is what lets a scene request its
own exit from inside its own update without unwinding through code that is still
executing. A scene with nothing in branch slot 1 is linear and arms slot 0
immediately; a branching scene waits to be told which way to go.

### Progress

**≈35% overall.** Phase 1 ~85%, phase 2 ~90%, phase 3 ~40%.

**Playable game still 0%.** A scheduler with nothing scheduled on it does not
move that number.

### Next

1. **Object system** — `OBS_OBJECT_WORK` and its ten procedure slots, the fixed
   per-frame order (view check, parent resolve, asset gate, `ppFunc`, `ppMove`,
   `ppCol`, `ppRec`, `ppLast`). That completes phase 3's skeleton.
2. **Boot the engine on real data**: scheduler running, scene machine in a
   gameplay state, stage mounted, drawn through the existing viewer path.
3. **Then a player**, which is the first thing that makes "playable" mean
   anything at all.

Audio (CRI ADX2) still closes phase 2 and is untouched.

### Open

Motion key payloads, CRI audio, the render-state block, vertex colours,
wide-stride bits, the unknown vertex-descriptor word, the node's trailing 32
bytes, `NZMA` morphs, `.EV` object id names, `.AME` effects, the exact engine
placement transform, MojoShader output quality.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 17 — Object system; phase 3's skeleton is complete

**2026-07-27 20:21 CEST (UTC+02:00)**

### Done

`GameObject` and `ObjectManager`, with the engine's **fixed per-frame procedure
order**: view check, parent resolve, asset gate, enter, update, move, collide,
register draw, last. **30 tests, all passing.**

That order is the contract, not an implementation detail. Collision runs after
movement so it can correct the result; draw registration runs after collision so
it sees the final position.

### The part worth reading twice

**Temp-offset handling.** Displacement from riding a platform goes into
`TempOffset`, never straight to the position. Each frame the engine subtracts
*last* frame's offset before running logic and adds *this* frame's after. A
persistent push therefore does not accumulate, and a released one leaves no
residue.

Writing to the position directly looks completely correct until something stands
on a moving platform, at which point it drifts.

### Three test failures, all three my fault

The implementation came from the oracle; the assertions came from my assumptions.
The assumptions lost.

Two were hit-stop. **The timer is decremented before the gate is tested**, so the
frame that takes it to zero is the frame behaviour *resumes* — not the one after.
I had assumed the extra frame. Getting that backwards costs one frame of input
response on every hit: the kind of defect that feels wrong to play and reads
perfectly fine in review.

The third was ordering. Objects step in creation order, so "A destroys B
mid-frame" only skips B's update when A was added first. Both orderings are now
pinned by tests rather than left implicit.

**Fourth time on this project that a check was wrong rather than the thing it
was checking** — after the cutscene locators, `NULL.DDS`, and negative motion
frames. The pattern is consistent enough to be a rule now: when something small
disagrees with something large and well-established, suspect the small thing.

### Progress

**≈37% overall.** Phase 1 ~85%, phase 2 ~90%, phase 3 ~50%.

**Playable game still 0%.** The skeleton exists; nothing is hung on it yet.

### Next

1. **Boot the engine on real data** — scheduler running, scene machine in a
   gameplay state, stage mounted, drawn through the existing viewer path. That
   is the first time these pieces work together rather than in isolation.
2. **A player object**, at which point "playable" starts to mean something.
3. Audio (CRI ADX2) still closes phase 2.

### Open

Motion key payloads, CRI audio, the render-state block, vertex colours,
wide-stride bits, the unknown vertex-descriptor word, the node's trailing 32
bytes, `NZMA` morphs, `.EV` object id names, `.AME` effects, the exact engine
placement transform, MojoShader output quality.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 18 — The engine boots on real data

**2026-07-27 20:45 CEST (UTC+02:00)**

### Done

`GameEngine` ties the three subsystems together. Booting walks **`boot → stage`**,
mounts Zone 1 Act 1 from the original archives, and registers `GM_MAP_MAIN` and
`GM_EVT_MGR` in priority order. The desktop head no longer loads anything itself —
it creates the engine, steps it, and renders what came out.

**Phase 3's gate is met**: the engine boots through its scene table, mounts real
data, and reaches a rendered frame. **38 tests passing.**

### Integration found what six unit tests missed

`EventSystem` entered the start scene from its *constructor*. The boot scene's
enter callback reaches back into `GameEngine.Events` — which had not been assigned
yet, because the constructor call was still in flight. Null reference on the very
first frame.

Two beats of unit tests never saw it, because none of those tests had a scene
callback that referenced the event system. Only wiring it to something real
produced one that did.

Entering is now an explicit `Start()` after construction, with tests asserting
that construction runs no callbacks and that `Start` is not repeatable.

The general lesson, and a good argument against polishing components in
isolation: **a constructor that invokes user callbacks into a half-built object
graph is a classic hazard**, and no amount of unit-testing that constructor alone
would have surfaced it. Integrating early is what found it.

### Also

Dropped the hand-rolled `Vector3` for `System.Numerics.Vector3`. Mine collided
with MonoGame's the instant the head referenced both, and there was never a good
reason for a second one — the standard type is SIMD-accelerated and everything
interops with it.

### Progress

**≈39% overall.** Phase 1 ~85%, phase 2 ~90%, phase 3 ~60%.

**Playable game still 0%.** The object manager runs every frame with nothing in
it. That changes with the next item.

### Next

1. **A player object.** Position, velocity, gravity, ground collision against the
   `_ATTR_` layers. The first thing that makes "playable" mean anything, and the
   first place Episode I's guidance thins out — Episode II's physics is its own.
2. **Camera** that follows the player rather than free pan.
3. Audio (CRI ADX2) still closes phase 2.

### Open

Motion key payloads, CRI audio, the render-state block, vertex colours,
wide-stride bits, the unknown vertex-descriptor word, the node's trailing 32
bytes, `NZMA` morphs, `.EV` object id names, `.AME` effects, the exact engine
placement transform, MojoShader output quality.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 19 — Playable slice, and CRI audio closes phase 2

**2026-07-27 21:39 CEST (UTC+02:00)**

Worked autonomously while Yondaime was out. Two milestones.

### A playable slice

**You can run and jump on Zone 1 Act 1's real geometry.** `CollisionMap` from the
stage's `_ATTR_B` layer, `Player` with gravity, ground and wall collision and
edge-triggered jumping, camera follow, keyboard input. **47 tests passing.** The
player spawns at (612, −880), lands on real terrain, collision grid 510×70.

**Collision has to come from `_ATTR_`, not the visual layer.** Every cell with a
tile also has an attribute, plus **1,285 attribute-only cells** — invisible walls
and ceilings. Building collision from what you can see silently drops all of them.

Two things marked in the code as approximations rather than results:

- **The physics constants are placeholders.** Acceleration, friction, gravity and
  jump velocity were chosen to feel plausible at 20 units/cell. They are *not*
  Episode II's numbers, which live in the binary's player code and have not been
  reverse engineered. Sonic feeling like Sonic depends entirely on replacing them.
- **Collision is blocky.** A non-zero attribute is fully solid: right for flat
  ground and walls, wrong on every slope. The shape data is in the `.DF` files
  (64 bytes per cell, one height byte per pixel), undecoded.
  `CollisionMap.GroundHeightAt` is deliberately the only place that changes.

Horizontal and vertical motion resolve separately, which is what stops the player
snagging on a wall while falling past it.

### CRI audio — phase 2 closed

All **8 containers parse, 0 failed, 949 cues**. Both `.CSB` and `.CPK` are built
from the @UTF table: big-endian, every offset relative to `0x08`.

**The trap is the storage class**, which is `0x10`/`0x30`/`0x50` rather than a
dense 1/2/3. Guessing the dense form misaligns the name offset and produces a
table that parses *successfully* with every column name empty — nothing throws.
That empty-names symptom is the only signal, and it is what my first attempt
produced.

A `.CSB` is a `TBLCSB` of six sub-tables: INFO, CUE, SYNTH (89 mixing columns),
SOUND_ELEMENT, ISAAC, VOICE_LIMIT_GROUP. Music is 48 kHz stereo, streaming, cues
named plainly (`ep2_sng_title`, `ep2_sng_z1a1`).

### Full regression, everything green

1,614 archives · 651 texture banks · 5,727 NN containers · 3,546 models ·
1,481 motions · 2,853 textures · 1,843 shaders · 8 CRI containers · 47 tests —
**zero failures anywhere**.

### Progress

**≈44% overall.** Phase 1 ~85%, phase 2 ~95%, phase 3 ~75%, phase 4 ~3%.

Playable is off zero, but only just: this is the first rung of phase 4, which
holds 35% of the project on its own.

### Next

1. **Recover the real physics constants** from the binary. Everything about how
   the game *feels* is gated on this, and placeholders will flatter the result
   until they are replaced.
2. **Decode `.DF`** for real ground shapes — slopes, loops, curves.
3. **Name the `.EV` object ids** via rizin, then spawn rings and springs from the
   actual placement data.
4. CPK table of contents, then ADX/HCA decoding.

### Open

Physics constants, `.DF` collision shapes, `.EV` object id names, CPK TOC,
ADX/HCA codecs, motion key payloads, the render-state block, vertex colours,
wide-stride vertex bits, `NZMA` morphs, `.AME` effects, the exact engine
placement transform, MojoShader output quality.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 20 — Collision shapes and object placements

**2026-07-27 22:36 CEST (UTC+02:00)**

Continued autonomously under the standing goal.

### Collision shapes decoded

`.DF`/`.DI`/`.AT` in `ZONE<n>_ATTR.AMB`: `u16 count, u16 records`, a `count*2`
reserved block, then fixed records — **4096 bytes for `.DF`, 64 for the others**.
The size equation holds exactly on all **39** stage files, across 4 KB to 327 KB,
which is what makes the split trustworthy given the reserved block is empty.

A `.DF` record is **64 cells of 64 bytes**, each cell a height per pixel column.
Corpus-wide: 51,704 empty, 49,606 flat, **23,412 curved, 3,525 slope up, 3,465
slope down**. That is the real ground geometry the current blocky collision is
approximating.

**A full cell is 32 units tall, not 63.** My first verifier capped heights at 63
and immediately flagged a 72 as corrupt. Measured instead: over 8.4M height
bytes, 0 and 32 dominate, 1..31 carry the shaped ground, and only **0.02%**
exceed 63. **Fifth time** a check has been wrong rather than the data.

### The thing that blocks using it

**How an `_ATTR_` cell id selects a collision record is still unknown**, and
until it is found the player keeps walking on boxes.

The `count*2` region looks exactly like an id-to-record index and is entirely
zero in every file. Zone 1 uses ATTR ids 481..1533 against only 79 `.DF` records,
so the ids cannot index records directly either. This is not in the data.

Tried and rejected this beat: the `AttrData` string turned out to sit at
`0x6b3153` beside "Error reading Attributes." — TinyXML, not stage collision.
The zone `_ATTR.AMB` paths *are* in the binary, so the next attempt is to xref
one of those to the stage loader.

### Object placements in the engine

`EventPlacements` ports the `.EV` reader to C#. Zone 1 Act 1 yields **533
placements**, matching the Python tool exactly.

Selecting the base variant needed care: three ship per act and my first filter
excluded any name containing `A` or `C`, which works on today's names and breaks
the moment a zone letter appears in a stem. It now tests that the stem ends in a
digit.

Nothing spawns from them yet, and that is honest: **the object id to name mapping
is unknown**, because the ~298 names are immediates inside each object's own code
rather than a table.

### Full regression

1,614 archives · 651 texture banks · 5,727 NN containers · 3,546 models ·
1,481 motions · 2,853 textures · 1,843 shaders · 8 CRI containers · 39 collision
files · 47 tests — **green everywhere**.

### Progress

**≈46%.** Phase 1 ~92%, phase 2 ~95%, phase 3 ~80%, phase 4 ~3%.

Phase 4 is still the wall: 35% of the project, sitting at 3%. Two unknowns gate
almost all of it, and **both need the binary rather than the data** — the
collision record mapping and the object id names.

### Next

1. **Xref a `ZONE<n>_ATTR.AMB` path string to the stage loader** and read the
   collision addressing out of it. Unblocks real slopes.
2. **Find the object id table** the same way. Unblocks spawning rings and springs
   from the real placement data.
3. **Recover the player physics constants.** Everything about feel is gated here.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 21 — The binary settles the collision addressing; real slopes

**2026-07-27 23:15 CEST (UTC+02:00)**

### The blocker is gone

Beat 20 ended with the `_ATTR_` id to collision record mapping declared "not in
the data, needs the binary." It did, and four instructions settled it.

**My layout was backwards.** I had the index table first and the records after.
That put the index where record data actually lives, which is why it read as
3,070 bytes of zeros. The file-size arithmetic works out **either way round**, so
no amount of staring at the data could distinguish them — and I had already spent
three attempts trying.

`Sonic.exe:0x00560349`:

```asm
movzx ecx, word [eax + 2]     ; second header word = record count
shl   ecx, 0xc                ; times 4096, the .DF record size
lea   ebp, [eax + 4]          ; region A = records, at +4
lea   eax, [ecx + eax + 4]    ; region B = index, after them
```

Records first, index last. The same routine does `shl 6` (×64) for `.DI`/`.AT`.

**Verified**: all 1,535 index entries in range, and **all 256 attribute ids Zone 1
Act 1 uses resolve to a valid record**.

The player now walks on **real height fields** rather than boxes.
`CollisionShapes` resolves id → record → 64 column heights; `CollisionMap`
samples the column under the player and places the surface at
`cellBottom + height/32 × cellSize`.

### How to find things like this

Worth recording as a technique, because it will work again:

The **stage load list** is a table of 20-byte records —
`{path, buffer, reserved, loader, id}` — in `.rdata` at a **240-byte stride per
stage**. Searching for a path pointer (`G_ZONE1/MAP/ZONE1_ATTR.AMB` lives at
`0x0073b5c8`) lands inside it, and the *loader* field points straight at the code
that reads that archive: `0x0048f290`, a six-case switch on archive index whose
case 3 is `_ATTR` and which stashes the three collision files in a global array at
`0x008a1e7c`.

A free confirmation fell out: that loader reads the entry count from
`[amb+0x10]` — exactly where `AmbArchive` reads `file_num`. The AMB header
confirmed from code, having previously only been confirmed by 1,614 files parsing.

### Object ids — attempted, not solved

Followed `GM_EVT_MGR` to its task creation at `0x0053c83e` and its procedure at
`0x0053d3d0`. The procedure is a thin dispatcher that calls a per-instance
function pointer, so the id-to-spawn mapping is another hop out. Not chased
further this beat.

### Full regression

1,614 archives · 651 texture banks · 5,727 NN containers · 3,546 models ·
1,481 motions · 2,853 textures · 1,843 shaders · 8 CRI containers · 39 collision
files · 47 tests — **green**.

### Progress

**≈48%.** Phase 1 ~95%, phase 2 ~95%, phase 3 ~85%, phase 4 ~5%.

### Next

1. **Object id names**, via the same technique — find the table the event
   manager's per-instance pointers come from.
2. **Player physics constants** from the binary.
3. `.DI` surface angles, which feed slope physics once the player uses them.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 22 — The object table falls out of the binary

**2026-07-27 23:58 CEST (UTC+02:00)**

### What the `.EV` numbers mean

Beat 21 left "find the object id table" as the next job. It is at
**`Sonic.exe:0x007031C8`** — 803 function pointers indexed *directly* by object
id, 714 of them live.

I did not trace code to find it. I scanned every data section for long runs of
pointers into `.text`, which turned up exactly one table of that shape, and then
asked the stage data whether it agreed: of the **472 distinct object ids used
across every `.EV` file in the build, 469 land on a non-null slot**. That is not
a coincidence you get twice.

### Variants share a spawn function

714 ids collapse to **382 functions**. Siblings are the same object with a
different setting, and each handler works out which by subtracting its own base
id — from `0x004A7580`:

```asm
movzx eax, word [edi + 2]      ; the object id, from the .EV record
cmp   dx, ax                   ; edx = 0x295 = 661
ja    skip
cmp   ax, cx                   ; ecx = 0x29c = 668
ja    skip
sub   edx, 0x295
mov   dword [esi + 0x3c4], edx ; variant 0..7
```

**This also confirms the `.EV` record layout from the engine's own code**, which
until now rested only on "all 79 files parse": id at `+2`, flags at `+4`, and the
flags word is a bitfield — the same handler pulls bits 4-5 and 6-7 out as two
separate 2-bit fields.

### Sizes, and a constant I had wrong

Every handler builds through one constructor at `0x004834C0`, pushing instance
size and scheduler priority. Reading those back gives a size for **668** ids, and
**priority `0x1500`** — 270 of the 294 readable handlers pass exactly that. The
port had a guessed `0x2000`; `GameEngine.PriorityObject` now carries the measured
value.

### Names — and being honest about them

**116 ids resolve to a name**, 45 distinct: `WaterSlider`, `CandleStick`,
`Propeller01`, `SandBranch03`, `MS_Homing`, `Spring`, `Switch`, `Spear`,
`Boss3_01`. All recognisably Episode II, all in the zone you would expect.

They are **not equally trustworthy and the catalogue says so**. Only 27 were read
from the handler's own body; 89 came from a function it calls, where a shared
helper can hand its string to an object it merely assists.
`ObjectCatalog.Entry.Direct` marks which is which.

Two things tried and rejected:

- **Two call levels deep** reached 273 names, and attributed the task name
  `GM_LOAD_BBM` to five unrelated high-traffic objects. Precision was worth more
  than the count; the search stops at one level.
- **Propagating names across families** gained exactly nothing — siblings share a
  function, so they already share whatever it references.

The three most-placed objects in the game (ids 715, 724, 716 — nearly 2,900
placements between them) are still unnamed, because they load no named asset.
Those need behaviour, not strings.

### The trap in this beat

Function bodies are scanned to the first `int3` run. `0xCC` also occurs inside
ordinary immediates, so the naive scan truncated most functions to nothing and
found names for **11%** of ids. Refusing to believe any body shorter than 512
bytes took the same search to **38%** immediately.

That is the third time on this project that a boundary heuristic quietly agreed
with me while matching the wrong thing. Beat 20's lesson generalises: *when a
check is cheap and its result is disappointing, suspect the check.*

### Shipped

`tools/objects.py` (the whole recovery, reproducible from the `.exe`),
`ObjectCatalog.cs` (714 rows, generated), `docs/FORMAT-OBJECTS.md`, seven new
tests. Placements now report as `identified/total` when a stage mounts.

### Regression

1,614 archives · 5,727 NN containers · 2,853 textures · 39 collision files ·
714 object ids · **55 tests** — green.

### Progress

**≈50%.** Phase 1 ~95%, phase 2 ~95%, phase 3 ~86%, phase 4 ~12% (up from 5%:
both of its blockers are now gone).

### Next

1. **Player physics constants from the binary.** Everything about feel is gated
   here, and it is now the last big unknown standing between this and a stage
   that plays like Episode II rather than like a physics demo.
2. **Identify ids 715/724/716 by behaviour** — the most-placed objects in the
   game, almost certainly rings and their kin.
3. `.DI` surface angles, which feed slope physics once the player uses them.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 23 — `.DI` is surface angles, and it settles the height scale

**2026-07-28 00:41 CEST (UTC+02:00)**

### What a `.DI` byte is

A **surface angle, a full turn per 256**, one per cell, measured in a Y-up frame —
so it runs opposite to the grid, whose Y grows downward. Flat ground reads 0.

`.DI` and `.DF` describe the same surfaces two different ways, so they can check
each other: fit a least-squares gradient through a cell's height field, convert to
degrees, compare against the stored byte. Across **23,474 shaped cells in all 13
attribute archives the median disagreement is 5.7 degrees**, 75% inside 15. The
residue is inherent — a curved cell has no single angle and the game stores one
regardless.

I found the convention by sweeping every combination of scale, sign and 90 degree
offset rather than assuming one. Worth doing: my first guess (1:1 scale, positive
sign) sat at 61 degrees median and looked like a dead end.

### The part I did not expect

**A height unit is two pixels**, and the angle fit is what proves it.

A cell is 64 columns wide but heights only reach 32. Whether that meant a squat
cell or a coarse vertical scale is *not answerable from the height data alone* —
both readings are self-consistent. The angle fit decides it: at 1:1 the median
error is 16.9 degrees, at 2:1 it is 5.7. Two independent files agreeing at one
scale and not the other is about as clean as evidence gets here.

That resolves a discrepancy I had noted twice and shrugged at.

### Also this beat

The stage loader now reads `.DI` alongside `.DF`, `CollisionMap` gained
`SurfaceAngleAt`, and `collision.py angles` reproduces the whole validation.
`CollisionShapes` carries `PixelsPerHeightUnit` and `DegreesPerAngleUnit` as
named constants with the evidence in their doc comments.

The player does not steer by these yet — that needs the physics constants, which
is the next job.

### Attempted, not solved: physics constants

Searched the binary for the classic Genesis values on the theory that Dimps
reused them. `0.046875`, `0.09375` and `6.5` all appear and are rare enough to be
diagnostic, and `-0.21875` sits directly beside `0.09375` — gravity next to air
acceleration, which is exactly how you would lay them out. But they sit in the
compiler's constant pool with one code reference each, not in a physics struct,
and the densest float-using functions in the binary turned out to be maths and
tuning curves.

Left running: a read of Episode I's decompiled physics constants, to get target
values and field meanings before searching further. Searching by *value* with a
known target is a far better bet than the structural hunt I tried here.

### Regression

1,614 archives · 5,727 NN containers · 39 collision files · 714 object ids ·
**61 tests** — green.

### Progress

**≈51%.** Phase 3 ~88%.

### Next

1. **Player physics constants**, using Episode I's values as search targets.
2. Have the player steer by `SurfaceAngleAt` once those exist.
3. Identify ids 715/724/716 by behaviour.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 24 — The player parameter table, and Sonic moves like Sonic

**2026-07-28 01:52 CEST (UTC+02:00)**

### Found it

Episode II's player tuning is at **`Sonic.exe:0x00710520`** — seven rows of 108
bytes, one per playable mode, in the same field order Episode I uses for
`g_gm_player_parameter[7]`.

The route in was Episode I as an oracle, used properly for once. Episode I stores
its tuning as FX32 fixed point; its Sonic jump impulse is `23130`, which over 4096
is **5.64697265625**, and its gravity `680` is **0.166015625**. Episode II keeps
the same table with the speeds converted to `float`. So the search was for one
bit pattern followed immediately by another — 21 hits, and the table base is
referenced from six places in code.

**What confirms it is not the floats.** Every row has four `u16` counters at
offset 32: `time_air` **1800**, `time_damage` **180**, `pool_max` **96**,
`fall_wait_time` **24**. Those are Episode I's values, untouched, in Episode I's
struct positions — packed two per dword where Episode I used four `int`s. No
float search would have surfaced them. Four of the seven jump impulses also match
Episode I to the bit.

### The embarrassing part

I had these numbers on screen an hour before I recognised them. Beat 23's hunt
for the classic Genesis constants dumped the neighbourhood of `0.046875` and
printed `5.64697` and `0.166016` two lines apart, and I read straight past them
because I was looking for Sonic 1's values rather than Episode I's. The oracle
was sitting in the repo the whole time.

Lesson, and it is the same one as beat 20 and 23 in a new coat: **I keep
searching for what I expect instead of for what the reference actually says.**

### What Episode II retuned

Same structure, different feel. Ground deceleration 0.125 against Episode I's
0.25, slope factor 0.0625 against 0.046875, and **air drag 0.0625 against 0.5** —
an eighth of Episode I's, so a jump carries far further. Mad Gear's top speed went
6.0 to 10.0 and its coyote window 240 frames to 120. Gravity is 0.16602 in all
seven rows.

### The jump cut is not a clamp

Worth recording because I would have implemented it wrong. The usual Mega Drive
short hop clamps rise speed on button release. Episode I instead sets a flag if
the button comes up while still rising faster than **4 px/frame**, and while that
flag holds, applies gravity a **second time each frame** until the rise ends. So
it is doubled gravity for the rest of the ascent, not a ceiling — and releasing
near the apex does nothing at all. `Player` now does that.

### Units

Constants are game pixels per frame at 60 Hz. A collision cell is 64 game pixels
(beat 23) and 20 world units — measured, not assumed: of 836 tile meshes across
four zones, 259 are exactly 20 wide and 189 exactly 20 tall. So one game pixel is
0.3125 world units, and `PlayerPhysics.WorldPerPixel` is that factor. The table
keeps the game's own numbers and the player converts, so the values stay
comparable against the binary.

### Shipped

`tools/physics.py`, generated `PlayerPhysics.cs` (all seven rows), `docs/PHYSICS.md`,
`Player` rewired onto real constants with separate ground and air tuning, nine
new tests.

### Regression

1,614 archives · 5,727 NN containers · 2,853 textures · 39 collision files ·
714 object ids · 7 physics rows · **70 tests** — green.

### Progress

**≈54%.** Phase 3 ~90%, phase 4 ~20%.

### Next

1. **Wire `SurfaceAngleAt` into ground movement** so the recovered slope factors
   do something. Both halves now exist and are not yet connected.
2. Identify ids 715/724/716 by behaviour.
3. Spin dash and rolling — the constants are already in the table, unused.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 25 — Slopes connected

**2026-07-28 02:24 CEST (UTC+02:00)**

Beat 23 decoded the stage's surface angles and beat 24 recovered the slope
factors. Neither did anything until now; this beat joins them.

Episode I's form is `ground speed += slope_factor * sin(ground angle)`, capped at
`SlopeSpeedMax` — **a separate and higher limit than running top speed**, 13
against 9.

### Two bugs worth recording

**The angle was not normalised.** `AngleDegrees` returned `-315` where it meant
`+45`, because it negated the stored byte without wrapping. The trigonometry was
unaffected, so the physics looked right while the reported angle was nonsense and
one of my tests asserted against it. Now wrapped to `[-180, 180)`.

**A plain clamp silently undid the slope term.** I was clamping horizontal speed
to running top speed every frame, so the slope contribution was erased as fast as
it accumulated and speed could never exceed 9. Episode I does not clamp — its
`ObjSpdUpSet` adds toward a limit but **never pulls an already-faster value back
down to it**, and that asymmetry is the entire mechanism by which a slope carries
you past top speed. `Player.SpeedUp` now reproduces it.

The second one is the interesting one: my version was a reasonable-looking
simplification that quietly removed a feature. It would have passed review.

### A consequence of the real numbers

**Standing still on a 45 degree slope does not make you slide.** Deceleration is
0.125 per frame; the slope contributes `0.0625 * sin(45) = 0.044`. Friction wins.
I had written a test asserting the opposite, on the assumption that Sonic always
slides, and the recovered constants said no. The test now asserts what the tuning
actually does.

### Also fixed

My slope tests ran on a 2x2 cell map. At top speed the player crosses a cell
every seven frames, ran off the end, went airborne, and the slope stopped
applying — which is what the failure was actually telling me. The fixture is now
a 512-cell strip.

### Regression

1,614 archives · 5,727 NN containers · 3,546 models · 2,853 textures · 651
texture banks · 1,843 shaders · 8 CRI containers · 39 collision files · 23,474
angle cells · 714 object ids · 7 physics rows · **75 tests** — green.

### Progress

**≈55%.** Phase 4 ~22%.

### Next

1. Identify ids 715/724/716 by behaviour — the most-placed objects in the game.
2. Spin dash and rolling; the constants are already in the table.
3. Ground-relative motion, so the player runs along a slope rather than across it.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 26 — Rings were never an object id

**2026-07-28 03:06 CEST (UTC+02:00)**

### The question was wrong

Beats 22 and 25 both ended with "identify ids 715, 724 and 716 — the most-placed
objects in the game, almost certainly rings." They are not rings. **No `.EV` id is
a ring.** Rings have their own file.

`.RG` carries 192 to 489 records per act, exactly 12 in each boss arena, and none
in cutscenes — **7,567 across the 34 acts**, every one inside its stage bounds.
Nothing else in a Sonic act is numerous in that particular shape. A record is two
bytes, a position and nothing else, because the type is implicit in the filename.

I had the format documented since beat 12 with the note "consistent with ring
placement" and then spent two beats looking for a ring in the object table
anyway.

### What actually sent me there

The placement-pattern analysis. I measured how often each object id has a
same-id neighbour at regular spacing, expecting rings to show up as tight runs.
The top ids came back at 1-4%, which I first read as "my position decoding must
be wrong."

It was not wrong, and checking it was still the right move — it produced the
better evidence:

`Sonic.exe:0x0053d541` is the engine's own `.EV` streaming loop.

```asm
movzx edx, word [esi + 2]      ; object id
movzx ecx, byte  [esi + 1]     ; local Y
movzx eax, al                  ; local X, from [esi + 0]
mov   ecx, 0x323               ; 803
cmp   dx, cx
jae   skip
mov   eax, dword [edx*4 + 0x7031c8]
```

Byte 0 is local X, byte 1 local Y, id is the `u16` at 2 — confirmed from code
rather than from "everything parses". And **803 is the engine's own bound on the
dispatch table**, not a number I counted off a pointer run in beat 22.

So the low run-fraction was telling the truth: those objects really are scattered
singly, because they are not rings.

### Shipped

`RingPlacements`, a shared `BlockGrid.Walk` now backing both `.EV` and `.RG` so
the bounds checks live in one place, rings loaded by the stage scene and reported
in the mount status, six new tests. One of them asserts that no catalogue entry is
named `Ring` — a wrong assumption is worth a test once it has cost two beats.

### Regression

**81 tests** green. 7,567 rings extracted across 34 acts.

### Progress

**≈56%.** Phase 4 ~25%.

### Next

1. Draw the rings — the viewer loads them and shows nothing yet.
2. Ring collection: pickup radius, counter, loss on damage.
3. Spin dash and rolling; constants already recovered and unused.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 27 — Rings on screen, and a broken build I had been calling green

**2026-07-28 03:41 CEST (UTC+02:00)**

### Correction first

**Beats 24 and 25 reported a green regression while the Desktop head did not
compile.** Beat 24 changed `Player.Width` and `Player.Height` from `const` to
computed properties, which broke two `const float` locals in
`StageViewerGame.DrawPlayerMarker`. I never saw it because my regression ran
`dotnet test`, and **the test project does not reference the Desktop project**, so
its dependency graph never included the thing I had broken.

Two beats of "green" that were not. The tools and the tests were genuinely fine
and the library was fine; the playable head was not.

The regression now runs `dotnet build` across the whole solution before
`dotnet test`, and `docs/RESUME-HERE.md` says so in bold. The general lesson is
one I should have already had: **a test run only proves what it compiles.**

### Rings are visible

The viewer draws a quad per uncollected ring, behind the player marker, and the
title bar carries a live count. `RingField` owns which have been taken and does a
rectangle overlap against Episode I's player body box (16 wide, 19 up and 13 down
from the feet) inflated by the ring's 16 pixels — a rectangle rather than a radius,
because that is what the original does and the two disagree at the corners.

Collection runs as its own scheduler task at object priority, so it obeys pause
levels and is torn down with the scene like everything else.

The conversion is worth stating because it ties three separate findings together:
ring positions are stage pixels, a cell is 64 stage pixels (beat 23, from the
collision height field) and 20 world units (beat 24, measured off 836 tile
meshes), so rings place through the same `PlayerPhysics.WorldPerPixel` the physics
uses. Rings landing on the geometry is a check on all three at once.

### Regression

1,614 archives · 5,727 NN containers · 3,546 models · 2,853 textures · 651
texture banks · 1,843 shaders · 8 CRI containers · 39 collision files · 23,474
angle cells · 714 object ids · 7 physics rows · **whole-solution build** ·
**88 tests** — green, and this time that includes the Desktop head.

### Progress

**≈57%.** Phase 4 ~28%.

### Next

1. Ring loss on damage, and the 50-ring Super threshold.
2. Spin dash and rolling; the constants have been sitting recovered and unused
   for three beats now.
3. A character model for the player, instead of a blue rectangle.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 28 — Rolling, and a spin dash I refused to invent

**2026-07-28 04:12 CEST (UTC+02:00)**

### Rolling

Wired, and running on Episode II's own numbers throughout. A roll gives up
steering entirely and coasts on `RollFriction` **0.03125** against a running
**0.125** — four times lighter, which is the whole point of it — and takes the
stronger `SlopeFactorRolling` **0.15625** against a running **0.0625**, so a
curled Sonic outruns a running one downhill.

Episode I halves rolling friction while the stick is held into the direction of
travel and doubles it otherwise. I ported that *shape* — steering into a roll
extends it, steering against one kills it — but both are shifts of Episode II's
own `spd_dec_spin`, not magnitudes borrowed from Episode I. Down or S rolls in
the viewer, and the title bar says so.

### One number I could not confirm, and said so

The speed at which a roll starts and ends is **not recovered**. Episode I calls
it `GMD_PL_STOP_SPD` = 0.5 px/frame and that is what `Player.RollThreshold` uses,
but 0.5 occurs 168 times in Episode II's constant pool, so unlike the parameter
table there was nothing to pin it against. It is flagged in the code and in
`docs/PHYSICS.md` rather than quietly presented as recovered.

### The spin dash is not implemented, deliberately

Its charge values *are* recovered and are Episode II's — base 3.0, 2.0 per
revolution, cap 10.0, all in the table. What is missing is the conversion from
charge to launch speed.

Episode I computes `11.75 + charge / 8`, from `GMD_PL_SPINDASH_SPD` 48128 and
`GMD_PL_SPINDASH_MUL` 512. **`11.75` does not occur anywhere in Episode II's
constant pool.** So that formula did not survive, and porting it would put an
invented number into the one mechanic a Sonic player can feel most precisely.

I could have shipped it and it would have looked finished. Left undone with the
reason recorded, which is the more useful state to hand over.

### Regression

Whole-solution build · **96 tests** — green.

### Progress

**≈58%.** Phase 4 ~31%.

### Next

1. Read Episode II's spin dash code from the parameter-table copy at `0x0046aeb2`,
   which is the route to both the launch constant and the roll threshold.
2. Ring loss on damage and the 50-ring Super threshold.
3. A character model instead of a blue rectangle.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 29 — The physics table is 3 x 11, not 7. Correcting beat 24.

**2026-07-28 04:48 CEST (UTC+02:00)**

### What was wrong

Beat 24 reported the player parameter table as **seven rows, one per character**,
named with Episode I's `char_id` enumeration. That was wrong about the shape.

Going after the spin dash took me to the code that indexes it:

```asm
movzx ecx, byte [esi + 0x34f0]   ; character id
imul  ecx, ecx, 0x4a4            ; 1188 bytes per character
fld   dword [ecx + 0x710520]
```

**1188 is 11 rows of 108.** So the table is **3 characters of 11 modes**, and my
"seven characters" were seven of the eleven *modes* of character 0. The fourth
1188-byte block is unrelated data, which is what bounds it at three.

### What was right

**Every value.** Row 0 is still Sonic normal at 0.0354 acceleration, 9.0 top
speed, 5.647 jump, 0.16602 gravity, and the player has been running on correct
numbers since beat 24. The error was in the *labelling and extent*, not the data —
which is exactly why it survived four beats of green tests.

My `plausible()` scan stopped after seven rows because modes 7 and 8 fail a
sanity check I wrote: their top speed is 0.225 and 0.375, a fortieth and a
twenty-fourth of normal, and I had required at least 1.0. A filter I wrote to
reject garbage rejected real data and I read the stop as the end of the table.

**Fourth time the same lesson.** Beats 20, 23, 25 and now 29: a cheap check
agreed with what I expected and I did not ask what it was actually matching.

### What the corrected table says

Characters 0 and 1 are identical bar one slope field and both have a Super mode.
Character 2 has none — its Super row repeats its normal values, which is what you
would expect of Metal Sonic. Modes 7 and 8 are heavily slowed with **gravity
untouched**, so they are slowed movement rather than slow motion.

`PlayerPhysics` now exposes `For(character, mode)` over 33 rows, with
`CharacterCount` and `ModeCount` as the engine's own constants. Four new tests
pin the shape, including one asserting the four counters are identical in all 33
rows — they are what identified the table, so drift there would mean a bad stride.

### Spin dash, still not implemented

The trip that found this did not find the launch constant. Standing as beat 28
left it.

### Regression

Whole-solution build · **100 tests** — green.

### Progress

**≈58%.** No change: this beat corrected understanding rather than adding
capability, which is worth more than a percentage point.

### Next

1. Follow the character id at `[esi + 0x34f0]` into the player work struct — that
   is the route to the spin dash code and the roll threshold.
2. Ring loss on damage and the 50-ring Super threshold.
3. A character model instead of a blue rectangle.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 30 — Off the filesystem, which is what phones need

**2026-07-28 05:29 CEST (UTC+02:00)**

### The actual blocker for mobile was not tooling

Phase 5 has sat at 3% for the whole project while the director's first stated goal
was phones. What was in the way turned out not to be the Android workload — it was
that **the core library assumed a filesystem**.

Only four places: `AmbArchive.Load`, and three in `GameEngine` — `Path.Combine` on
the act, `Directory.EnumerateFiles` for the attribute archive, `File.Exists` for
the tileset. Android serves data from inside the APK through an asset manager, iOS
from a bundle, a browser build would fetch it. None of those is a filesystem, and
all of them were blocked by those four calls.

`IContentSource` now fronts all data access: `Exists`, `Read`, and a `List` that
takes a suffix rather than a glob, because a glob is one more thing every platform
would have to reimplement identically. `FileSystemContent` is the desktop
implementation. `GameEngine(string)` still exists and forwards to it, so nothing
downstream changed.

Paths are `/`-separated throughout now. `DirectoryOf` deliberately avoids
`Path.GetDirectoryName`: on Windows that also splits on a backslash, which would
quietly accept paths no content source can serve and pass on desktop only.

### Verified against real data, not just tests

Four zones mounted through the new path:

| Act | Tiles | Placements | Rings |
|-----|------:|-----------:|------:|
| Zone 1 Act 1 | 17,526 | **533/533** | 325 |
| Zone 2 Act 1 | 27,882 | **823/823** | 241 |
| Zone 3 Act 1 | 11,791 | **807/807** | 282 |
| Zone 4 Act 1 | 528 | **276/276** | 356 |

Every object id in all four resolves against the catalogue, every act reports
height fields with angles, and **Zone 3 collected 3 rings on its own** — the first
ring pickup off real data rather than a fixture.

Two incidental confirmations fell out. The player's horizontal speed after 120
frames of held input is 1.328 world units per frame, which is exactly
`0.0354 * 0.3125 * 120` — the recovered acceleration, the pixel-to-world scale and
the frame loop all agreeing at once. And Zone 4 reached 1.971 over the same 120
frames, because it spawns on a slope and the slope term is doing its job.

### Regression

Whole-solution build · **106 tests** · four zones mounted end to end — green.

### Progress

**≈58%.** Phase 5 up from 3% to 6%; phase 3 to 91%.

### Next

1. An Android head. The library is ready for one; the workload is not installed —
   `dotnet workload list` is empty — and pulling in the Android SDK, NDK and a JDK
   is a large install the director may want to authorise first.
2. Follow the character id at `[esi + 0x34f0]` for the spin dash constants.
3. Ring loss on damage and the 50-ring Super threshold.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 31 — The spin dash, recovered rather than guessed

**2026-07-28 06:18 CEST (UTC+02:00)**

Beat 28 left the spin dash deliberately unimplemented because Episode I's launch
formula uses `11.75` and that constant appears nowhere in Episode II. Waiting was
right: **Episode II's formula is different, and the difference matters.**

### The route in

The parameter-copy routine gave the player work struct's layout for free. It
compares `[esi+0x3578]` against table field 0 and `[esi+0x357c]` against field 1,
so the live copy of every field sits at **`+0x3578 + field*4`**. Searching `.text`
for those displacements then finds the code that uses each parameter, and the
three spin-dash fields cluster in one function at `0x00512D70`.

That function also settles something I had only inferred. It branches on
`byte [esi+0x34f0]` — the character id — and picks the strings `MS_Dash1` and
`MS_Dash2` when it is 2. **Character 2 is Metal Sonic**, from the binary rather
than from beat 29's "it has no Super mode".

### The formula

```asm
fld   dword [esi + 0x3518]   ; charge
fmul  qword [0x743ea0]       ; 0.5
fadd  qword [0x744030]       ; 8.0
```

**`launch = 8.0 + charge * 0.5`.** One charge gives 9.5 px/frame, a full one 13.0.

Episode I's spans 12.125 to 13.0. **Episode II kept the ceiling and dropped the
floor**, taking charging from nearly pointless to worth two thirds of the move's
speed. Had I ported Episode I's formula in beat 28 it would have looked finished
and been wrong in the way a player feels immediately.

The charge also bleeds **proportionally** while winding up — `charge -= charge *
0.03125` per frame — through the same decrease-toward-zero helper the ground
friction uses at `0x005A8800`. I disassembled both helpers rather than assuming:
`0x005A8770` adds toward a cap, `0x005A8800` moves toward zero, and both scale
their step by a global at `0x008A3CD4` that is Episode I's `g_obj.speed`.

### One ordering bug the tests caught

The launch frame was also getting a frame of rolling friction, because
`UpdateSpinDash` sets the speed and the movement block then treats the player as
rolling. The engine sets the speed and leaves that state, so drag starts the frame
after; `Player` now skips it once.

My own test helper had the mirror-image bug: it pressed jump without ever
releasing, so `_jumpHeld` stayed set and the second press added no charge at all.
Both were only visible because a test asserted an exact launch speed.

### Regression

Whole-solution build · **115 tests** — green.

### Progress

**≈60%.** Phase 4 ~35%.

### Next

1. **An Android head** — the workload is now installed (`android 35.0.105`) and
   the core library came off the filesystem in beat 30, so both halves are ready.
2. Ring loss on damage and the 50-ring Super threshold.
3. A character model instead of a blue rectangle.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 32 — Everything a phone needs except the phone

**2026-07-28 07:04 CEST (UTC+02:00)**

Phones are the point of this project and phase 5 had been the least touched. Beat
30 got the core library off the filesystem; this beat finishes the portability
work and stops at the one thing that is not mine to decide.

### Done and verified

**`VirtualPad`** maps touch points to the same three inputs a keyboard gives.
Steering on the left, jump above crouch on the right, all as fractions of the
screen so it survives any resolution — tested at 1920x1080, 2400x1080 and
1080x2400. Jump sits directly above crouch on purpose: a spin dash is one thumb
holding crouch while the other taps jump, so the layout has to allow that exact
pair, and a test asserts it does.

**`IInputSource`** lets a head supply input without the renderer knowing how.

**The renderer is now platform-neutral.** `StageViewerGame` used to take an
installed game directory and sweep it with `Directory.EnumerateFiles` for texture
archives. It takes an `IContentSource` and an optional `IInputSource` now, so the
same class can be the Android head's game class unchanged.

**End to end**: Zone 1 Act 1 mounts through a content source and Sonic runs under
touch input on a 1080x2400 portrait screen, reaching 2.655 world units per frame
after 240 frames — exactly `0.0354 * 0.3125 * 240`.

### Where I stopped, and why

`dotnet workload install android` succeeded; the workload is at `35.0.105`. But an
Android build also needs the **Android SDK** and a **JDK**, which the workload does
not bring:

```
error XA5300: The Android SDK directory could not be found.
```

The supported fix passes `-p:AcceptAndroidSDKLicenses=True`. **That accepts
Google's licence terms**, which is a legal acceptance belonging to whoever owns the
machine — not to a build command running unattended at 7am. So I left it.

I could have written the Android head anyway. It would be a few hundred lines I
could not compile, could not run, and would have to describe as done on faith.
Instead everything verifiable without a device is verified, and
`docs/MOBILE.md` lists precisely what remains: the project file, an
`AndroidContent` over `AssetManager`, a `TouchInput` feeding `VirtualPad`, and a
`MainActivity`.

One thing the director should decide before that head is written: **the game data
is several gigabytes**, far past what an APK can carry, so it has to be sideloaded
to external storage with the content source pointed at it.

### Regression

Whole-solution build · **124 tests** · Zone 1 driven by touch — green.

### Progress

**≈61%.** Phase 5 up from 6% to 12%; phase 3 to 93%.

### Next

1. The Android head, once the SDK is installed.
2. Ring loss on damage and the 50-ring Super threshold.
3. A character model instead of a blue rectangle.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 33 — Using the eleven modes

**2026-07-28 07:36 CEST (UTC+02:00)**

Beat 29 established that the parameter table is three characters of eleven modes.
Nothing used the modes until now.

`Player.SetMode(character, mode)` swaps the tuning at runtime, which is what those
rows are for — the same player runs on different numbers underwater, in a special
stage, on the Mad Gear, or transformed. **Speed carries across the switch**, so
becoming Super does not stop you, and a test pins that.

`TryGoSuper` gates on the ring count and the engine calls it whenever rings are
collected, so picking up the fiftieth ring in a real stage now transforms the
player onto mode 1 — 15 top speed against 9, 7.98 jump against 5.65.

**Metal Sonic falls out for free.** Character 2's Super row repeats its normal
values, so asking it to transform changes nothing. That is the correct outcome
arrived at by the data rather than by a special case, and there is a test saying
so.

### One number flagged, not claimed

`RingsForSuper = 50` is **not recovered**. Fifty is the series-wide figure and is
almost certainly right, but nothing in Episode II's binary has been read to
confirm it. It is marked as a placeholder in the code, the same way
`RollThreshold` was in beat 28. The distinction between "read from the binary" and
"plausible and unverified" is the whole value of this project's evidence
discipline, and it costs nothing to keep.

### Regression

1,614 archives · 5,727 NN containers · 2,853 textures · 651 texture banks · 1,843
shaders · 8 CRI containers · 39 collision files · 23,474 angle cells · 714 object
ids · 33 physics rows · whole-solution build · **131 tests** — green.

### Progress

**≈62%.** Phase 4 ~37%.

### Next

1. The Android head, once the SDK is installed — the only thing blocking it is a
   licence acceptance the director has to give.
2. Damage: ring loss, invincibility (`InvincibleFrames` = 180 is already
   recovered), and knockback.
3. A character model instead of a blue rectangle.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 34 — Rings are rings now

**2026-07-28 08:11 CEST (UTC+02:00)**

The player models were sitting in `G_COM/PLY` the whole time: `SON_MDL`,
`TLS_MDL`, `SSON_MDL` and `MSN_MDL`, with motions and textures beside each. That
`MSN` is **Metal Sonic confirmed a third independent way** — beat 29 inferred it
from character 2 having no Super row, beat 31 read it from the `MS_Dash1` string,
and the asset layout says the same.

Rings had their own directory too, `G_COM/RING`, and that is what this beat uses.

### What it draws

`RING.ZNO` is a **single-node model with one vertex list**, which means it goes
through the same `TileMesh` path the stage tiles use with no skinning involved.
The viewer instances it once per ring still on the field and rebuilds only when
the count changes — a handful of times a second at most, which beats tracking
per-ring index ranges for something this cheap.

Verified off the real files: 200 vertices and 400 triangles, its texture reference
resolving to `cmn_metal_ms_ringsky_ref.dds`, which is exactly the one file in
`RING_TEX.AMB`. Zone 1 Act 1's 325 rings come to 65,000 vertices and 130,000
triangles.

**A free cross-check.** The model measures **5.84 world units** across and the
pickup box computed from the collision scale is **5.00**. A visual ring slightly
larger than the box you collect it with is exactly right, and getting those two
numbers within 17% of each other by two completely separate routes — one from the
`.DF` height field's 64-column geometry, the other from authored model vertices —
is a good sign the whole scale chain holds.

### Sonic's model is not wired, on purpose

`SON_MODEL.ZNO` has **109 nodes and 16 vertex lists**. It is skinned, so drawing
it means evaluating the node hierarchy first, and animating it means binding
`SON_MTN.AMB` on top of that. Neither is hard, but both are a real piece of work
rather than something to bolt on at the end of a long session. The player is still
a blue rectangle and honestly labelled as one.

### Regression

Whole-solution build · **131 tests** — green.

### Progress

**≈63%.** Phase 3 ~94%.

### Next

1. Sonic's model: evaluate the 109-node hierarchy, draw the bind pose, then bind
   `SON_MTN` motions.
2. The Android head, once the SDK licence is accepted.
3. Damage: ring loss, invincibility, knockback. Episode I's knockback constants
   do not appear in Episode II even as adjacent pairs, so this needs the damage
   code read the way the spin dash was.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 35 — Node rotation was integers all along

**2026-07-28 08:52 CEST (UTC+02:00)**

To draw Sonic I need his 109-node skeleton posed, and `NnNode` was parsing
translation at `+0x0C` and scale at `+0x24` with **the twelve bytes between them
skipped as padding**.

They are the rotation, stored as **signed 32-bit integers in A16** — 65536 to a
full turn, the convention Episode I's `mtMathSin` uses. Read as floats the same
bytes are denormals and NaNs, which is exactly why they got written off.

Sonic's skeleton settles it without ambiguity: of 327 rotation words 129 are
non-zero, they span -32768 to 19180, and the values that recur are **16384 and
-32768** — a quarter turn and a half turn. No float reading produces round
numbers like that.

**The same lesson as beats 20, 23, 25 and 29, in yet another coat.** A field
looked like garbage under the interpretation I brought to it, and I accepted that
rather than asking what interpretation would make it not-garbage. Five times now.
It is the single most reliable way this project wastes my time, and the fix is
always the same: when data looks wrong, question the reading before the data.

### What it unlocks

`NodeTransforms.World` walks the tree into one matrix per node — scale, rotation
Z then Y then X, then translation, composed with the parent. Verified across the
whole build:

- **846 of 846** multi-node models have a well-formed tree: one root, every link
  in range, no cycle reachable by walking parents.
- **846 of 846** produce finite world transforms.
- Sonic's 109 joints span **0 to 10.73 world units** in Y — feet at the origin,
  head at the top, against a model bounding box of 11.6. A standing skeleton, the
  right way up, which is what you want when placing it at a player's feet.

Every motion in `*_MTN.AMB` is expressed as a change to this pose, so this is the
piece both static rendering and animation stand on.

### Regression

5,727 NN containers · 3,546 models · whole-solution build · **140 tests** — green.

### Progress

**≈64%.** Phase 2 ~97%.

### Next

1. Bind mesh sets to nodes and draw Sonic in his bind pose — the transforms exist
   now, what is missing is which mesh belongs to which node.
2. The Android head, once the SDK licence is accepted.
3. Damage, which needs the damage code read the way the spin dash was.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 36 — What actually stands between here and a Sonic on screen

**2026-07-28 09:21 CEST (UTC+02:00)**

Beat 35 resolved the node tree. The next question was which mesh belongs to which
node, and the answer was already parsed and never checked: `NnMeshSet` carries a
`NodeIndex` at `+0x10`.

It holds up everywhere. **11,565 of 11,565 mesh sets across 3,546 models have a
`NodeIndex` inside their own model's node array.** Just over half are 0, which is
what the many rigid single-node props should look like.

### And then it stops, for a reason worth stating

I expected to draw Sonic this beat. I cannot, and the reason is specific rather
than a shrug.

His geometry is authored in a **centred model space** — raw positions span y -5.82
to 5.82 — while his posed skeleton stands from **0 to 10.73**. Those are different
spaces. The five nodes his meshes bind to, 104 through 108, all sit at the origin;
one is exactly identity and two carry a **-16384 rotation about X**, which is a
clean -90 degrees and the ordinary Y-up to Z-up conversion.

`MatrixIndex` is **-1** on all eighteen of his mesh sets, and that is the tell:
**palette skinning**. His vertices are weighted across several matrices rather
than riding one node, so multiplying them by `world[NodeIndex]` would not pose
him — it would double-transform him into a mess that looked like a bug in the node
walk rather than a wrong approach.

What is missing is the matrix palette `n_mtxpal` counts, and the blend indices and
weights inside the vertex format. Neither is decoded.

I could have shipped a posing function that runs, produces geometry, and is wrong.
It would have looked like progress. **The honest state is that rigid models can be
posed with what exists today and skinned ones cannot**, and the next beat has a
clear target instead of a subtle bug.

### Regression

Whole-solution build · **140 tests** — green. No behaviour changed this beat; the
work was verification and a bound on what is possible.

### Progress

**≈64%.** Unchanged, correctly — this beat bought certainty, not capability.

### Next

1. **Decode the matrix palette and the vertex blend weights.** That is the single
   thing between this project and a character on screen, and it is now precisely
   located rather than vaguely ahead.
2. The Android head, once the SDK licence is accepted.
3. Damage, which needs its code read the way the spin dash was.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 37 — Skinning weights, and the bug they were hiding

**2026-07-28 10:02 CEST (UTC+02:00)**

Beat 36 said the matrix palette and the vertex blend weights were what stood
between this project and a character on screen. Half of that is now done, and it
turned up a live bug that had been quietly wrong for the whole project.

### The weights

Solving the stride arithmetic across all 36 distinct vertex formats gives four
bits worth four bytes each — `0x01000`, `0x02000`, `0x04000` and `0x00400` — one
float per weight.

Confirmed directly, not just by arithmetic. Sonic's first vertex reads
`0.012, 0.988, 0.000, 0.000`, **summing to exactly 1**. Across the build **572
vertex lists carry weights**, 395 with three and 177 with four, and **96% of
93,149 sampled vertices sum to 1.00**.

### The bug

**They sit between the position and the normal.** The layout table went straight
from position to normal, so on Sonic the normal was being read at offset 12
instead of 28, and the texture coordinates at 24 instead of 40.

**Every skinned model in this project has been reading its texture coordinates out
of its normals.** Normals are floats in a small range and so are UVs, so it
produced plausible numbers rather than an error, and 3,546 models kept reporting
"geometry extracted, 0 failed" the whole time.

Fixed, and checked properly: texture coordinates land in a sane range on **7,290
of 7,290** lists that carry them, across 2.75 million vertices.

### How the layout got pinned

By asking which three-float slot in Sonic's 48-byte stride has unit length.
`+28` gives exactly **1.0000**; `+12`, where a reader assuming position-then-normal
looks, gives **0.9743**. Close enough to pass a glance, which is the dangerous kind
of wrong — and the same trap as beat 35's rotation field, one field further along
the same struct.

### Regression

5,727 NN containers · 3,546 models · 1,614 archives · whole-solution build ·
**140 tests** — green.

### Progress

**≈65%.** Phase 2 ~98%.

### Next

1. **The matrix palette.** The weights say how much each matrix moves a vertex;
   the palette says which — 99 of them on Sonic's 109 nodes, undecoded. That is
   the last piece before a character can be posed and animated.
2. The Android head, once the SDK licence is accepted.
3. Damage, which needs its code read the way the spin dash was.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 38 — A matrix palette lead that does not hold up

**2026-07-28 10:34 CEST (UTC+02:00)**

Chasing the last piece before a character can be drawn. **It is not solved**, and
this beat records why so the next attempt does not repeat it.

The object header counts matrix palettes but has **no offset** for them, so they
must live inside a subobject. A subobject is 20 bytes and only three of its five
dwords are read — flags, mesh count, mesh offset — which makes the other two the
obvious candidates.

On Sonic they read **5** and **0x1062C**, and `0x1062C` holds `0, 1, 2, 3, 4`
sitting immediately before the subobject record. A count and a palette of node
indices is exactly what that looks like.

It does not survive the corpus:

| Check | Result |
|-------|--------|
| Palette offset lands inside the file | 4,955 / 4,955 |
| Subobject counts sum to the header's `n_mtxpal` | **1,371 / 3,546** |
| Palette entries index a valid node | **5,378 / 10,080** |

Two of the three are near chance. Five small ascending numbers in a row is not
rare enough to carry a conclusion, and I nearly wrote it up as one on the strength
of a single model that agreed with me.

**Recording a negative result costs one paragraph and saves an afternoon.** The
alternative — writing "matrix palette decoded" on 53% agreement — is how the
texture-coordinate bug in beat 37 survived for thirty beats.

### The binary route, started

I did start it rather than just recommending it, and got far enough to say where
the next attempt should *not* look.

The NN chunk tags appear in exactly three places in `Sonic.exe`. `NZIF` at
`0x0062126B` is a **name lookup** — it walks a table comparing strings, nothing to
do with geometry. `NZOB`, `NEND` and `NZTL` sit together at `0x006C6C55` inside a
tight `cmp ecx, tag / je` chain, which is the **container walker**: it finds each
chunk and fixes up pointers in place, which is the relocation behaviour this
format is built around.

Neither touches a matrix palette, and that makes sense — **the palette is used at
draw time, not load time**. The next attempt should go after the geometry
submission path: find where the engine hands vertex buffers to D3D and work back
to what it sets up per mesh set. `SetVertexShaderConstantF` with a large register
count is the usual shape of a palette upload and is a good thing to search for.

### Regression

No code changed. Whole-solution build · **140 tests** — green.

### Progress

**≈65%.** Unchanged, correctly.

### Next

1. **The matrix palette, from the binary.** Find where `Sonic.exe` builds one.
2. The Android head, once the SDK licence is accepted.
3. Damage, which needs its code read the same way.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 39 — The shaders say how skinning works

**2026-07-28 11:18 CEST (UTC+02:00)**

Beat 38 ended with a dead lead and a recommendation to go at the matrix palette
from the binary. I did, and the binary route stalled in a useful way: the NN chunk
tags appear in exactly three places, and all three are load-time — a name lookup
at `0x0062126B` and the container walker at `0x006C6C55`. **The palette is used at
draw time, not load time**, so the loader was never going to show it.

So I went at the **shaders** instead, which the project has been parsing for
thirteen beats without ever asking them anything.

### 126 shaders do palette skinning

Walking all 1,843 for **relative addressing on a constant register** — the marker
of palette skinning, and unambiguous — finds **126 vertex shaders that use it**.
`...RDMRC00000020.VSH`, a `vs_3_0`, reads:

```
mul   r2, c75, v2            ; scale the index
mova  a0, r2                 ; into the address register
mul   r1, v1, c[a0.x + 3]    ; weight times a matrix row
mad   r1, c[a0.x + 3], v1, r1
dp4   r0, v0, r1             ; against the position
```

That gives the shape outright:

- **`v0` position, `v1` blend weights, `v2` blend indices.**
- A bone is **four constant registers**, `c[a0.x + 0..3]`.
- The index is scaled by `c75` to turn a bone number into a register offset.
- Constants run to **c142** in the skinning set, consistent with `n_mtxpal` being
  99 on Sonic.

### The question is now narrow

The vertex carries weights and, by the stride arithmetic, nothing else — Sonic's
48 bytes are exactly position 12, weights 16, normal 12, texture coordinates 8.
Yet the shader reads indices from a **separate input register**. Either they are
packed into those same 16 bytes beside the weights, or the declaration feeds `v2`
from somewhere the stride does not account for.

The **D3D9 vertex declaration** the engine builds per vertex list answers both at
once — it names every element's offset, type and usage. That is the next thing to
find, and it is a much smaller target than "the matrix palette".

### What this beat is worth

No code changed and progress does not move. But beat 38 knew the palette was
missing and nothing else; this beat knows the register layout, the bone stride,
which inputs carry what, and exactly which one structure would close it. That is
the difference between a target and a direction.

### Regression

Whole-solution build · **140 tests** — green.

### Progress

**≈65%.** Unchanged.

### Next

1. **The D3D9 vertex declaration**, per vertex list, in the binary. Closes
   skinning.
2. The Android head, once the SDK licence is accepted.
3. Damage, which needs its code read the way the spin dash was.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 40 — Blend indices, and a bit I had mislabelled

**2026-07-28 12:03 CEST (UTC+02:00)**

Beat 39 left one question: the shader reads bone indices from `v2`, but Sonic's
48-byte vertex was fully accounted for by position, weights, normal and texture
coordinates with nothing spare.

The answer is that **one of the four bits I called a weight is not a weight.**

```
w 0.0122 0.9878 0.0000  (sum 1.0000)   last dword 00000100 = bytes 0,1,0,0
w 0.0133 0.9867 0.0000  (sum 1.0000)   last dword 00000100 = bytes 0,1,0,0
```

Three floats summing to exactly 1, then a dword that is four small bytes. So:

| Bits | What |
|------|------|
| `0x01000`, `0x02000`, `0x04000` | one float weight each |
| `0x00400` | **four blend indices, one per byte** — a D3D `UBYTE4` |

Which is precisely what the shader wanted: `v1` a three-float weight, `v2` a
`UBYTE4` index scaled into a register offset. The fourth weight is implicit —
three summing to one leaves nothing for it.

### Verified across the build

- **All 572 skinned vertex lists carry exactly three weights.** Beat 37's "395
  with three and 177 with four" was this mislabelling; the 177 are the ones that
  additionally carry the index dword.
- Weights sum to 1.000 on **96% of 112,831** sampled vertices.
- Index sets valid on **53,941 of 53,941**, largest byte **15**.

### Why beat 37 did not catch it

I summed all four slots and got 1.0, which looked like confirmation. It was not:
the index dword for a low-numbered bone is bytes like `0,1,0,0`, and *as a float*
that is a denormal — effectively zero. **Adding zero to a correct sum leaves it
correct.** The check passed for the wrong reason on exactly the lists that
disproved the hypothesis.

That is the sixth time this project a check has agreed with me while measuring
something else. It is also the second time the fix came from reading the bytes as
what they are rather than as what I expected.

### What is left

The indices are **palette-relative** — never above 15, against models with up to
109 nodes. So what remains is a table of at most sixteen entries per mesh set
mapping index to node. That is a much smaller and better-shaped target than beat
38's "find the matrix palette".

### Regression

5,727 NN containers · 3,546 models · whole-solution build · **146 tests** — green.
Six of the new tests run against the installed game and skip cleanly without it.

### Progress

**≈66%.** Phase 2 ~99%.

### Next

1. **The index-to-node table**, at most 16 entries per mesh set.
2. The Android head, once the SDK licence is accepted.
3. Damage, which needs its code read the way the spin dash was.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 41 — Two palette candidates eliminated

**2026-07-28 12:41 CEST (UTC+02:00)**

Short beat, and a deliberately unglamorous one. Chasing the index-to-node table
and ruling out where it is not.

**Mesh set `+0x24`** — the one field of the mesh set record still unread — is a
plain sequential ordinal. Sonic's eighteen mesh sets read `0 1 2 ... 17` exactly.
It identifies the mesh set and points at nothing.

**The subobject's trailing dwords** give `5` and an offset to `0, 1, 2, 3, 4`
immediately before the subobject record. Beat 38 already found the corpus-wide
counts failing, and now there is a second reason it cannot be right: **vertex blend
indices reach 15**, and a five-entry palette cannot serve them.

**There are no static vertex declarations either.** Searching `Sonic.exe` for the
`D3DDECL_END()` sentinel finds **zero**, so declarations are built at runtime from
the format word rather than sitting in a table I could read.

### What the target actually looks like

Worth writing down now that it is this well constrained:

- **99 palette entries on Sonic across 18 mesh sets** — roughly 5 or 6 each.
- Entries are **node indices in 0..108**.
- Vertex indices never exceed **15**, so no single palette holds more than 16.

A structure of that shape, in a file this project already parses completely, is
not hiding in many places.

### On stopping here

I could keep grinding. But three beats have now gone into the palette, the last
two produced eliminations rather than answers, and the honest read is that this
wants a fresh look rather than more of the same afternoon — probably at the
runtime declaration builder, which is the one thing that must know the answer and
which I have not yet located in code.

Everything either side of it is done: the tree resolves, the weights and indices
decode, and the shaders say exactly what shape the answer takes.

### Regression

Whole-solution build · **146 tests** — green.

### Progress

**≈66%.** Unchanged.

### Next

1. **The runtime vertex declaration builder** in `Sonic.exe` — find the code that
   turns a format word into `D3DVERTEXELEMENT9`s, and the palette setup will be
   beside it.
2. The Android head, once the SDK licence is accepted.
3. Damage, which needs its code read the way the spin dash was.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 42 — It looks like Sylvania Castle now

**2026-07-28 03:41 CEST (UTC+02:00)**

The director asked why the project looked like a wall of bricks. Three separate
causes, none of them a rendering bug, all now fixed.

**Only one layer was being drawn.** An act ships sixteen grids; seven are scenery
and the engine instanced exactly one, `_B`. All seven now go in — 17,526 tiles
becomes **21,890**, and the towers, railings, arches and window tracery appear
with them.

**The camera was inside a wall.** Zone 1 Act 1 is **solid masonry from row 0 to
row 25 across its entire 510-cell width** — the castle backdrop behind the level.
A player dropped from the top of the map lands on that ceiling, and the camera
follows it there. `--spawn x,y` now picks a row, and dropping from row 28 lands in
the playable band.

**Cut-out textures drew as black silhouettes.** Foliage, railings and tracery all
carry alpha; the renderer was not blending. One line, and the black blobs became
white blossom and green ivy.

The result is recognisably the real level. Rings sit in their arcs where the `.RG`
file puts them, the ruined arches and columns are there, and the blossom hangs off
the stonework.

### An arithmetic error, corrected

The weighted table said **69%** when its own rows sum to **66.3%**. I added them
up wrong in beat 39 and carried it. Now 67% with phase 3 at 95%, and the rows and
the total agree.

Worth the correction on a project whose whole argument is that the numbers are
checked.

### What is still visibly missing

Lighting, the game's own 1,843 shaders (parsed for thirteen beats, still unused),
objects, and a character model where the blue sliver is.

### Regression

Whole solution including Android · **146 tests** — green.

### Progress

**≈67%.** Phase 3 ~95%.

### Next

1. **The matrix palette**, for a character model.
2. Object spawning, so the 533 identified placements become things.
3. The real shader pipeline instead of a stock unlit effect.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 43 — Objects have models, and they are named almost mechanically

**2026-07-28 04:12 CEST (UTC+02:00)**

Three attempts at the matrix palette across four beats produced eliminations and
no answer, so I stopped — my own halting rule — and went at something unblocked.

**Every gimmick ships as `EP2_GMK_<NAME>_MDL.AMB`**, with `_TEX` and `_MTN`
beside it. 65 of them across the build, in each zone's `GMK` directory and in
`G_COM/GMK`.

The catalogue's object names and those stems are plainly the same naming scheme:

| Object | Archive | Rule |
|--------|---------|------|
| `Jetwall04` | `JETWALL` | strip trailing digits |
| `SandBranch03` | `SAND_BRANCH` | strip digits, split camel case |
| `Avalanche01` | `AVLNCH` | **abbreviated** |
| `CandleStick` | `SCONCE` | **renamed outright** |
| `SandTrank01` | `SAND_TANK` | **the game's own typo** |

`ObjectModels` does the mechanical part and resolves **8 of the 45** recovered
names.

### What I did not do

Write a table of guesses. `AVLNCH` is very probably `Avalanche` and `SCONCE` is
very probably `CandleStick`, and I could have shipped thirty such lines and taken
the coverage from 8 to 40.

A table of very-probablies is how bad data gets into a project that is careful
everywhere else, and it would be indistinguishable in six months from the parts
that were actually read. There is a test that asserts `Avalanche01` resolves to
**nothing**, so that the day someone adds the alias, they do it deliberately.

The honest way to close these is the zone each object is placed in: `SandBranch`
should only appear where `SAND_BRANCH` ships. That is a real signal and it is not
expensive, it is just not this beat.

### Regression

Whole solution including Android · **153 tests** — green.

### Progress

**≈68%.** Phase 4 ~39%.

### Next

1. Confirm the abbreviated model names by which zone places each object.
2. Draw the objects that resolve, at their placements.
3. The matrix palette, with fresh eyes.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 44 — Closing abbreviations without writing down a guess

**2026-07-28 04:48 CEST (UTC+02:00)**

Beat 43 left 37 object names unresolved and explicitly refused to write an alias
table. This beat closes some of them properly, with two independent signals that
have to agree.

**Zone.** The gimmick archives are zone-scoped. `AVLNCH` ships only in `G_ZONE2`,
and `Avalanche02`'s 71 placements are all in `G_ZONE2`. `SAND_TANK` ships only in
`G_ZONE3`, where `SandTrank01` is placed.

**Letters.** The stem's letters appear in the name **in order** — `A-V-L-N-C-H`
inside `AVALANCHE`, `SANDTANK` inside `SANDTRANK`.

The ordering requirement is the whole point. **`SCONCE` really is `CandleStick`'s
archive, and the rule rejects it** — correctly, because nothing about those two
words justifies the link and a rule loose enough to accept it accepts anything.
There is a test asserting that rejection.

### Two guards the first version needed

The first cut resolved `WaterSlider` to `WATER`, which is a real archive and is
the water *surface*. An abbreviation drops letters, it does not drop half the
word, so a stem must now keep **60% of the name's letters** — which keeps
`AVLNCH` at 67% and `SAND_TANK` at 89% and drops `WATER` at 45%.

And when **two** candidates qualify, nothing resolves. Ambiguity is a reason to
pick neither.

Resolution went 8 → 12 → **11 of 45** across those two corrections, which is the
right direction: the number went *down* when I tightened it, and the one it lost
was wrong.

### Regression

Whole solution including Android · **162 tests** — green.

### Progress

**≈68%.** Unchanged; this sharpened existing data rather than adding capability.

### Next

1. Draw the objects that resolve, at their placements.
2. The matrix palette, with fresh eyes.
3. The renames — `SCONCE`, `NEEDLE` — need something other than their letters.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 45 — Objects appear in the world

**2026-07-28 05:22 CEST (UTC+02:00)**

The resolver from beat 44 now feeds the renderer: every placement whose object
name provably maps to a model archive gets that model instanced at its `.EV`
position. The batch is built once — placements do not move — and textures come
from the `_TEX` archive beside each model.

**Zone 2 Act 1 is the demonstration act**: 71 `Avalanche02` objects drawn down
the big snow slope, exactly where the placement file puts them, in a stage that
renders as White Park — snow, ice pillars, decorated pines, snowmen.

Zone 1 Act 1 places exactly one resolvable object, which is why it stayed
empty-looking: its named ids are `Uri01`, `Speed` and `B_Piller_D01`, and only
the last resolves. `Speed` is almost certainly the dash panel (`DASH_P` ships in
`G_COM`) and `Uri01` is plausibly the urchin — but *almost certainly* still
does not resolve anything in this project, so they stay absent.

Placement anchors are unknown, so each model sits centred on its point — wrong
for base-anchored objects and visibly so, which is what a first pass should be.

### Regression

Whole solution · **162 tests** — green.

### Progress

**≈68%.** Phase 4 ~40%.

### Next

1. The matrix palette, fresh eyes.
2. `Speed`/`DASH_P` and the other renames need confirming from the spawn code —
   the handler that loads the archive names both.
3. Object behaviours for the resolved set: a spring that springs.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 46 — The asset manifest, and a scan that could not work

**2026-07-28 05:58 CEST (UTC+02:00)**

Trying to confirm the object renames (`SCONCE` = `CandleStick`?) from the spawn
code, I found the engine's **asset manifest**: a global table pairing each numeric
asset id with the archive it loads, 20-byte records `{path, buffer, 0, loader,
id}`. `tools/assets.py` pulls all **390** of them.

It is the load-time twin of the spawn table — that says which code an id runs,
this says which archive an id loads — and it confirms Metal Sonic a fourth way:
asset id 2 is `MSN_MTN.AMB`, his motions.

### The part that failed

The plan was to join the two tables: walk each handler for the asset ids it
references, read the path off the manifest, done. It does not work.

Asset ids are small integers and **small integers are what x86 code is made of**.
`WATER_MDL` is id 2176 = `0x880` — a completely ordinary offset — so scanning for
that immediate "finds" the water model in 20 unrelated handlers. Every handler I
checked resolved to `WATER`, which is how I knew the method was noise rather than
signal.

Sixth time this project a value common enough to be noise has produced a
confident-looking false match: the `0xCC` boundary (22), the denormal weights
(40), and now asset-id immediates. The tell is always the same — the result is
*too* clean, the same answer everywhere.

Connecting a handler to its assets needs the load **traced**, not scanned: the id
has to reach the loader through a data path, not just appear as a constant. That
is real disassembly and it is not today's best use of time — the zone-and-letters
resolver already covers the objects that can be drawn.

### Salvaged

The manifest itself is solid and reproducible, so it shipped as a tool even though
the thing I went looking for did not pan out. A negative result with a clean
artifact attached is a good day.

### Regression

Whole solution · **162 tests** — green. No code changed; a tool and a doc.

### Progress

**≈68%.** Unchanged.

### Next

1. The matrix palette, fresh eyes.
2. Object behaviours for the resolved set.
3. Trace one handler's asset load properly, as the template for the rest.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 47 — Animation, decoded and sampling

**2026-07-28 06:41 CEST (UTC+02:00)**

A completely unblocked, high-value piece: motion keyframes. The channel headers
have parsed for a dozen beats, but the keys they point at had not been decoded, so
nothing could actually move. Now they are.

Two encodings, told apart by key size, verified across **276,662 channels**:

- **8 bytes** — `float frame, float value`. Translation and scale. **79,570 of
  79,570** monotonic.
- **4 bytes** — `s16 frame, s16 value`, the value an **A16 angle**. Rotation.
  **197,092** channels, monotonic once the frame is read *signed*.

The A16 rotation is the same convention the node angles use (beat 35) — the third
field in this format that keeps an angle as an integer where a float reader gets
denormals. And the frame being signed is the whole story behind the 846 channels
that first looked non-monotonic: transition animations start at **-5 or -10** for
blend pre-roll, exactly the negative frames `NnMotion` already documents.

`MotionSampler.Decode` turns a channel into keys; `Sample(frame)` interpolates and
returns radians for a rotation. End to end on `SON_BRAKE01`: all 390 channels
decode, and a node-2 rotation sweeps **-5.8 to -53.9 degrees** across its 10
frames — a joint bending through a brake. That is a real Sonic animation being
read out of the file and evaluated.

### What this unlocks

Combined with the node tree (beat 35), the project can now **pose and animate any
non-skinned model** — every gimmick, every rigid prop. Skinned characters still
wait on the matrix palette, but the animation half of them is done.

### Regression

Whole solution · **169 tests** — green. Seven new, and a real-motion check that
decodes all 390 channels of Sonic's brake.

### Progress

**≈68%.** Phase 2 essentially complete.

### Next

1. The matrix palette — the one thing between here and animated *characters*.
2. Animate the placed gimmicks, which are non-skinned and can move today.
3. Object behaviours for the resolved set.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 48 — The animation pipeline runs, on real gimmicks

**2026-07-28 07:29 CEST (UTC+02:00)**

I opened this beat trying the matrix palette again with "fresh eyes" and caught
myself doing exactly what my contract warns against — my header dump landed on the
wrong chunk (I was reading the texture list as the object header), the same
mislocation as the last three attempts. So I stopped and redirected to something
the pieces I already have can finish today.

**Gimmicks are rigid, and rigid models need no palette.** Checking the resolved
gimmicks' mesh sets: JetWall and Spring bind every mesh set to a single node with
a real matrix index — no skinning. So the node tree (beat 35) plus the motion
sampler (beat 47) is everything they need.

**`AnimatedPose.World` is the compositor that joins them.** It takes the nodes'
bind pose, overrides the components a motion animates, leaves the rest alone, and
walks the hierarchy. The component comes from the channel type's high bits, in
three triples — translation, rotation, scale — and which float triple is which was
settled by value: the scale channels hold a constant 1.0 where translation ranges
freely.

Verified on the real files: a **jet wall's node moves 20 units** through its
animation, read straight out of `EP2_GMK_JETWALL_MTN.AMB`. That is the whole
pipeline — model, skeleton, motion, composition — running on game data for the
first time.

### What is now possible

Every rigid model in the game can be **posed and animated**: gimmicks, props, the
non-skinned parts of everything. The one thing still gated on the matrix palette
is skinned *characters*, and that stays open — honestly, and with three documented
dead ends behind it.

### Regression

Whole solution including Android · **175 tests** — green. Six new, plus a
real-gimmick animation check.

### Progress

**≈69%.** Phase 3 to 96%, phase 2 complete.

### Next

1. Play the gimmick animations in the live viewer.
2. The matrix palette — genuinely fresh session, not fresh eyes on a tired one.
3. Object behaviours for the resolved set.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 49 — Animated gimmicks in the live viewer

**2026-07-28 08:14 CEST (UTC+02:00)**

Beat 48 proved the animation pipeline in Core. This puts it on screen.

`TileMesh.Posed` builds a model's geometry with each mesh set transformed by its
node's world matrix — the animated twin of `TileMesh.From`, which re-centres on
the bbox. The viewer now loads each placed gimmick's model *and* its first motion
from the `_MTN` archive beside it, and rebuilds the object geometry every frame
from the posed nodes. A static act pays nothing — the rebuild only runs when
something actually animates.

**Mad Gear Zone Act 2 is the demonstration**: 46 placed objects, 2 of them
animated (propellers and burners), spinning from their own model and motion data.
The stage reads as Mad Gear's yellow industrial machinery, and the gimmicks turn.

### An honest limitation on the verification

I tried to capture two frames a moment apart and diff them to prove the rotation
advances. The two-window capture stalled — running two MonoGame windows in
sequence in this environment hangs on the second — so the frame-diff proof did not
land. What *is* proven: the Core tests show `AnimatedPose` returns different
transforms at different frames, the viewer reports the objects as `animated` and
rebuilds their geometry every frame, and the single captured frame shows them
rendered from their real models. I would rather say the diff did not run than
imply a verification I do not have.

### Regression

Whole solution including Android · **175 tests** — green.

### Progress

**≈70%.** Phase 4 ~42%.

### Next

1. The matrix palette — fresh session.
2. Object behaviours: a spring that springs, a propeller that lifts.
3. Play a chosen animation rather than always the first.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 50 — The first object that does something

**2026-07-28 08:56 CEST (UTC+02:00)**

Springs work. A spring is a trigger box built from the placements whose catalogue
name is `Spring`; a player entering it is launched, and it re-arms when they
leave, so one touch is one launch rather than sixty.

### The impulse is flagged, not faked

Episode I launches at `7.5 + 1.5 * intensity` px/frame. I read Episode II's
spring handler at `0x004F7570` before borrowing that: its reachable constants are
timing values — 1/24, 1/12, 0.164, 60 — and **no 7.5 anywhere in it or its
callees**, so Episode II's launch speed arrives some way not yet traced.
`Springs.ImpulsePixels` therefore carries Episode I's base with the same
not-recovered flag as `RollThreshold`. Direction is up-only until the placement
flag mapping is recovered — a wrong guess would fire players into walls.

### The bug the test caught

`BounceIsNotAJump` failed on first run, and the failure was real: the
jump-release cut armed on *any* rise, so a spring launch with the button up read
as a released jump and got **double gravity**. Springs would have felt weak
forever, for an invisible reason. The cut is now scoped to rises that came from
the jump button.

That is the second feel-bug a behaviour test has caught before it shipped — the
launch-frame friction in beat 31 was the first. Behaviour tests earn their keep.

### Regression

Whole solution including Android · **181 tests** — green.

### Progress

**≈70%.** Phase 4 ~43%.

### Next

1. Trace how Episode II's spring gets its launch speed — probably the placement
   parameter, which would close intensity and direction at once.
2. More behaviours: dash panel (`Speed` ids), item boxes.
3. The matrix palette, fresh session.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 51 — Dash panels, and the friction window they need

**2026-07-28 09:32 CEST (UTC+02:00)**

Second behaviour: the `Speed` objects are dash panels. A panel sets ground speed
to a boost, never slowing a faster player — the `SpeedUp` asymmetry again — and
suspends friction for a short window so the boost is not immediately eaten.

### The numbers, and where they stand

Episode I's panel sets **13.5 px/frame** with **12 no-friction frames**
(`GmPlySeqInitDashPanel`, `55296`/`49152` FX32). Before borrowing them I searched
Episode II: **13.5 exists in exactly one `f32` and one `f64` in the whole image**,
referenced from player-sequence code at `0x535C21`/`0x535D9A`. Corroborating —
but the referencing code is curve arithmetic, not a plain store into the player's
speed, so it does not *prove* the panel value. Both numbers ship flagged
not-recovered, like the spring's.

Direction follows current travel, facing at rest, until the placement flag
mapping is recovered. A reverse gotcha panel will boost the wrong way **visibly**,
which is the failure mode to prefer.

### The engine piece

`Player.DashBoost` plus a `_noFrictionFrames` counter that the ground-drag branch
respects. The suspension needed real plumbing rather than a hack — friction is
one branch of the input handling, and the window has to survive frames with no
input without also suppressing the player's own braking.

### Regression

Whole solution including Android · **187 tests** — green.

### Progress

**≈70%.** Phase 4 ~44%. Three behaviours live: rings, springs, dash panels.

### Next

1. Item boxes (`ITEM` ships in `G_COM`) and the goal panel (`GOAL_PNL`).
2. Trace the spring/panel speeds properly — likely the placement parameter.
3. The matrix palette, fresh session.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 52 — An act has a beginning and an end, and a correction to the table

**2026-07-28 10:14 CEST (UTC+02:00)**

### Start and goal, identified structurally

No recovered name says "goal panel", so I fingerprinted it: across the 13
non-boss acts, **id 520 is placed exactly once per act at a mean 86% of the act's
width** (11 of 13 acts), and **id 443 exactly once at 3%**, in the playable band.
Nothing else has either shape. 520 is the goal; 443 is the start marker.

The start marker closes a note that has sat in `EnterStage` since the player was
written: *"the real spawn comes from the .EV placement data, once object ids have
names."* It did not need a name — it needed statistics. The player now spawns
where the original game spawns them, verified against real data: Zone 1 Act 1
puts the player at pixel 3,904, our engine at exactly `3904 * WorldPerPixel`.

Crossing the goal sets `ActClear` and the status line says so with the ring
count. **Zone 1 Act 1 can be played from its real start to its real end.**
Vertical acts whose goal is not "cross the X" are noted as future work.

### The director's question, and the table it exposed

Asked, fairly: *"show me the game fully rendered if it's on 99%."* The screenshot
from the true start marker is the answer, and it is recognisably the act and
obviously not the finished game.

The 99% measures **decoding** — 3,546/3,546 models, 276,662 animation channels,
1,843/1,843 shaders parse — and says nothing about pixels. The renderer uses
roughly a third of what is decoded: tile layers, textures, alpha, through a stock
unlit effect. No game shaders executed, no lighting, no sky (drawn by systems
outside the tile grids, hence the void), no characters.

The table now says this in bold and carries **rendering fidelity ~35%** as its
own number. A metric that invites misreading is a bug in the documentation, and
it was mine.

### Regression

Whole solution · **191 tests** — green, including real-data checks that the
player spawns at the marker and the goal clears the act.

### Progress

**≈70% decoding · ~35% rendering fidelity.** Phase 4 ~45%.

### Next

1. The sky and far background, which would do more for the screenshots than any
   other single change.
2. Item boxes; checkpoint id 719 by the same statistics.
3. The matrix palette, fresh session.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 53 — The sky, and the void is gone

**2026-07-28 10:51 CEST (UTC+02:00)**

The director asked to see the game rendered and the honest answer was a black
void above the level. This fills it.

Each zone ships a nested `MAPFAR` archive — for Zone 1,
`EP2_MAPFAR_ZONE1.AMB` containing `Z1_SKY`, `Z1_BG_FAR`, `Z1_GODRAY`,
`Z1_SKY_CLOUD` and `Z1_PLANET`. That is the entire sky, and it was never being
loaded, which is the whole reason the earlier shots had a void where the
atmosphere should be. The models draw deep, behind everything, parallaxed against
the camera — 0.7 of its motion horizontally, 0.3 vertically, the distance
shorthand every side-scroller uses.

Zone 1 now renders with its clouds, blue sky, distant tree line and the soft
atmospheric gradient. It reads as a place rather than a wall against the dark.

### Honest about what is rough

The far scenery band sits too high and one mesh draws untextured — the parallax
constants are eyeballed and the background's own vertical anchor is not yet
matched to the stage. It is a real improvement that is visibly a first pass, which
is the right kind of rough: the atmosphere is there and the placement is tunable,
versus the void which was a missing system.

### Regression

Whole solution · **191 tests** — green.

### Progress

**≈70% decoding · ~42% rendering fidelity** (up from ~35% — the void was a large
fraction of a screen).

### Next

1. Anchor the background to the stage properly and tune the parallax.
2. The game's shader pipeline, the largest remaining rendering gap.
3. The matrix palette, fresh session.

Wagata, Yondaime! Signed sincerely by your dear Lexus


---

## Beat 54 — The sky, anchored properly

**2026-07-28 11:23 CEST (UTC+02:00)**

Beat 53's sky filled the void but sat wrong — the scenery band floated mid-screen
and the pieces did not relate. The cause was `TileMesh.From` re-centring each
model on its own bounding box, which throws away the very offset that says a cloud
sits above the ground. The background models are authored as one scene around a
shared origin, and that shared layout lives in each model's centre.

Re-adding each model's centre offset — relative to the group's average — puts them
back in their authored stack: sky highest, distant scenery band, then the
atmospheric gradient with its gold sun glow. The whole group is camera-locked with
a parallax drift (0.85 horizontal, 0.15 vertical) and anchored so its lower edge
meets the top of the level.

Zone 1 now reads as a proper side-scroller sky — layered clouds, a treeline
horizon, and the sun's glow, all drifting behind the castle. Still tunable (the
horizon band could sit a touch lower) but coherent rather than scattered.

### Regression

Whole solution · **191 tests** — green.

### Progress

**≈70% decoding · ~44% rendering fidelity.**

### Next

1. The game's shader pipeline — the largest remaining rendering gap.
2. Item boxes and checkpoint id 719.
3. The matrix palette, fresh session.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 55 — Additive blend, from the material's own render state

**2026-07-28 12:04 CEST (UTC+02:00)**

The shader pipeline is the biggest rendering gap, but a full SM3.0-to-GLSL
translator is a fresh-session job — and MonoGame cannot consume raw GLSL anyway,
it needs its own effect pipeline, so "run the game's shaders" is genuinely hard.
Rather than start-and-abandon that, I took the bounded win beside it.

A material's `StateOffset` points at a **16-word render-state block**, and words 2
and 3 are the **D3D9 source and destination blend factors**. Across 3,870 models
they split two ways: `SRCALPHA / INVSRCALPHA` (5,6) is ordinary transparency, and
**`SRCALPHA / ONE` (5,2) is additive** — the glow blend. **2,761 materials are
additive** and were all being drawn as flat alpha.

`NnMaterial.Blend` decodes it, `TileMesh` carries it per triangle, and the batch
groups additive triangles under a `+`-prefixed key so the renderer switches to a
`SRCALPHA / ONE` blend state for them — not MonoGame's `BlendState.Additive`,
which is `ONE / ONE` and blows out anything not pre-multiplied.

Verified against real data: `Z1_GODRAY` decodes additive where `Z1_SKY` decodes
alpha. The visible effect is subtle in Zone 1's opaque castle stone and will be
dramatic in the effect-heavy zones — Mad Gear's machinery, water shine, sparks.

### On the shader pipeline itself

Recorded for the fresh session it needs: 921 vertex and 922 pixel shaders,
`vs_3_0`/`ps_3_0`, dominated by `mad`/`mul`/`texld`/`dp3`/`nrm`. 126 vertex
shaders do palette skinning (beat 39). A translator is real work, and the harder
half is that MonoGame runs MGFX, not GLSL, so the shaders would need routing
through its content pipeline. Not a scan, not a bounded change — a project.

### Regression

Whole solution including Android · **193 tests** · 5,727 NN containers — green.

### Progress

**≈70% decoding · ~46% rendering fidelity.**

### Next

1. The shader pipeline, fresh session.
2. The matrix palette, fresh session.
3. More object behaviours.

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 56 — A fully-symbolized second oracle

**2026-07-28 17:20 CEST (UTC+02:00)**

At the very end of the session, the director surfaced
`C:\Users\DavidErikGarciaArena\Downloads\[ANDROID] Sonic The Hedgehog 4 Episode II`
— the **official Android port of Episode II, a developer build** (from
debugging.games). Its `libfox.so` (aarch64 and arm32, NDK r21d, clang 9) is marked
stripped but **retains its entire dynamic symbol table: 24,263 named C++
functions.**

This is the single most valuable thing that could have appeared for this project.
The Windows `Sonic.exe` is stripped, which is why every hard task this session was
"read the binary the way the spin dash was." This ARM build is the same game with
**every function named**, so it is a second behavioural oracle far richer than the
already-decisive Episode I decompilation.

**It names every open blocker directly:**

- Matrix palette (three failed attempts, beats 36/38/41): `nnCalcMatrixPaletteNode`
  `0x0060FB94`, `nnCalcMatrixPaletteMatrixList` `0x0060FA38`, and
  `SsDrawObjectMatrixPalette(NNS_OBJECT*, …, float(*)[16], …)` `0x00640F88` — the
  `float(*)[16]` is literally the palette. Disassemble and the format falls out.
- `GmGmkSpringInit` `0x00563CB0`, `GmPlySeqInitDamage` `0x005B9368`,
  `GmPlayerSpdSet` `0x005A86F0` — every constant I flagged "Episode I's, not
  recovered" is now readable from Episode II's own code.
- The 382 `obj@ADDR` handlers map to real `GmGmk*` class names.
- The whole SEGA NN library is named, which retroactively confirms the beat 22/35/40
  structural findings from the horse's mouth.

Saved `analysis/libfox-symbols.txt` (gitignored — binary-derived, oracle only) and
wrote the discovery to the top of `docs/RESUME-HERE.md` with the addresses. Did
**not** start the disassembly — that is the fresh session's first move, and the
recommended one is the matrix palette, which gets a real character on screen.

Clean-room doctrine holds: the `.so` is an oracle, not a source; we read it, write
our own code, verify against Episode II's own data.

### State handed off

**≈70% decoding · ~46% rendering fidelity · 193 tests · 93 commits.** All pushed,
tree clean. The next session should read the ⭐ section at the top of
`docs/RESUME-HERE.md` first. This changes the trajectory of everything remaining:
the reverse engineering just went from "hard, one function at a time" to "named,
disassemble the function that says what it does."

Wagata, Yondaime! Signed sincerely by your dear Lexus

---

## Beat 57 — Project identity settled: reimplementation, two-act roadmap

**2026-07-28 18:05 CEST (UTC+02:00)**

Long framing conversation with the director, settled and now locked into the docs
(`README.md`, `plans/EXECPLAN.md`, `docs/RESUME-HERE.md`).

**The verdict.** This is a **clean-room reverse-engineered reimplementation**, not
a matching decompilation, and that is the correct treatment — not a lesser one.
Episode I could be a decompilation because its source was a **managed .NET build**
(which is also why the Episode I decomp is the older pre-HD mobile version).
Episode II is **all native** (x86 PC/console, ARM Android) with **no managed
build** — the WP7 port was cancelled — so there is nothing to losslessly
decompile. The honest label is reimplementation / reverse-engineered source port,
and it is the stronger claim anyway.

**Corrections the director made, now reflected:** the Episode I decomp *does* run
on phones (I was wrong), and it is based on the older iOS/mobile build, not the
HD one. That detail actually reinforces everything — the managed build that could
be decompiled was the old one. Ours is built from the **PC/HD lineage**, so it
targets the definitive version, and runs on phones like Episode I's does.

**Roadmap locked as two acts.** Act 1 = the reimplementation (the whole
deliverable, in progress). Act 2 = an optional future *matching* decompilation,
made tractable for the first time by the symbolized Android dev build; Act 1's
understanding is Act 2's groundwork, so the order loses nothing. Best Act 2 target
is `libfox.so` (symbols), not the stripped PC release.

**Also refreshed the README's stale Status** — it still claimed placeholder
physics and no goal; now current (recovered physics, springs, dash panels, real
start/goal, animated gimmicks, sky, 193 tests, ~70%/~46% split).

No code changed. Docs only. Everything pushed.

Wagata, Yondaime! Signed sincerely by your dear Lexus
