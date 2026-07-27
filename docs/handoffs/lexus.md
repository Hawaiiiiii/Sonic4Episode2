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
