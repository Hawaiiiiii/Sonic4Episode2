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
