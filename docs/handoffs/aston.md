# ASTON handoff ledger

## Damage — normal hurt and zero-ring death

**2026-07-29 03:54 CEST (UTC+02:00)**

### Implemented

- **VERIFIED** — `Damage.Apply` now gives a normal player the recovered hurt
  transition: all carried rings are removed, knockback is opposite facing, the
  player cannot steer until landing, and the player is protected by the
  parameter table's damage-invulnerability timer.
- **VERIFIED** — a ringless hit enters a collisionless death arc using the active
  player row's jump impulse. Dead and currently invulnerable players reject
  repeat hits.
- **VERIFIED** — `GameEngine.DamagePlayer` owns the ring-count mutation through
  the new `GameEngine.Behaviours.cs` partial. `GameEngine.cs` changed only by the
  required `partial` keyword. Damage is event-driven, so it needs no per-frame
  scheduler task.

### Episode II oracle

- **VERIFIED** — `GmPlySeqInitDamage` at arm64 `0x005B9368` loads the normal
  velocity pair from `0x00968440`: `-1.5, -3.0`, and mirrors the horizontal
  component to `+1.5` for the opposite facing. The C# engine's Y axis is upward,
  so the recovered normal impulse is `1.5` horizontal and `3.0` upward game
  pixels per frame.
- **VERIFIED** — the anonymous damage collision handler beginning at arm64
  `0x005A382C` loads the full current ring count at `0x005A394C` and calls
  `GmRingDamageSetNum` at `0x005A3958`; zero rings instead tail-call
  `GmPlySeqChangeDeath`.
- **VERIFIED** — `GmPlySeqInitDeath` at arm64 `0x005B9910` negates the active
  player row's jump impulse for the original coordinate system. The PC player
  table at `0x00710520` was read again this session and reports
  `InvincibleFrames = 180`.
- **INFERRED** — the normal-facing flag meaning agrees with Episode I's
  independently readable player sequence and with Episode II's two mirrored
  horizontal branches. No Episode I numeric constant was used.

### Regression

- **VERIFIED** — the first Damage test was observed RED because the behaviour did
  not exist, then GREEN after the implementation. Each subsequent state branch
  was likewise exercised red-to-green.
- **VERIFIED** — focused Damage suite: **15/15 green**.
- **VERIFIED** — `dotnet test src`: **232/232 green**, 0 failed.
- **VERIFIED** — `dotnet build src
  -p:AndroidSdkDirectory=C:/Android/sdk -p:JavaSdkDirectory=C:/Android/jdk`:
  succeeded, 0 errors.
- **VERIFIED** — the Android head built separately with the SDK/JDK flags,
  `JAVA_TOOL_OPTIONS=-Xmx256m`, and `-m:1`: succeeded, 0 errors.
- **VERIFIED** — both current-HEAD builds completed with 0 warnings.

### Still open

- **OPEN** — the original spawns up to 32 recoverable dropped-ring objects; this
  beat removes the carried count but does not yet create those objects.
- **OPEN** — Super and other special player-flag branches have different
  knockback/ring rules. They are ignored rather than run through the known-wrong
  normal branch until their flag mapping is recovered.
- **OPEN** — respawn timing, life consumption, hurt/death animation selection,
  effects, sound, vibration, and trophy accounting remain outside this core
  prerequisite.
- **OPEN** — Needle is next per Lexus's order. It has not been started.

Wagata, Yondaime! Signed sincerely by your dear Aston

---

## Damage review — axis assignment re-audited

**2026-07-29 04:51 CEST (UTC+02:00)**

### Verdict

- **VERIFIED** — the ordered change to 2.1 horizontal was not applied because
  both Episode II Android architectures identify that value as vertical in the
  `0x4000` player-flag branch. The ordinary branch remains 1.5 horizontal and
  3.0 vertical.
- **VERIFIED** — the misleading spring-cap name and documentation were corrected:
  `Springs.HorizontalSpeedCap` now names the 9.0 cap. The old
  `VerticalSpeedCap` remains as an obsolete compatibility alias.

### Episode II oracle

- **VERIFIED** — arm64 `GmPlySeqChangeDamageSetSpd` at `0x005B9304` stores its
  first speed argument at `0xC8` and its second at `0xCC`. It then compares the
  first argument with zero and derives facing bit 0 from its sign. This identifies
  `0xC8` as horizontal and `0xCC` as vertical.
- **VERIFIED** — the separately compiled arm32 setter at `0x006F6CEC` repeats the
  same sequence at offsets `0xB0` and `0xB4`, including the facing test on the
  first field.
- **VERIFIED** — arm32 `GmPlySeqInitDamage` writes `-1.5` to `0xB0` at
  `0x006F6E1C`, writes `-3.0` to `0xB4` at `0x006F6E28`, and mirrors only
  `0xB0` from facing at `0x006F6E4C`. Its `0x4000` branch scales `0xB0` by
  `0.5` and `0xB4` by `0.7`, yielding 0.75 horizontal and 2.1 vertical.
- **VERIFIED** — arm64 `GmPlySeqInitSpringJump` at `0x005D17CC` caps `0xC8` to
  9.0, so the earlier `VerticalSpeedCap` label was the axis error.
- **INFERRED** — Episode I's readable damage setter uses the same X-then-Y
  argument order and facing rule. It was used only as a semantic cross-check;
  no Episode I numeric value was used.

### Regression

- **VERIFIED** — the corrected spring-cap API test was observed RED with
  `CS0117` before the production member existed, then GREEN.
- **VERIFIED** — focused Damage/DashPanel suite: **22/22 green**.
- **VERIFIED** — `dotnet test src`: **232/232 green**, 0 failed.
- **VERIFIED** — the whole solution and separate Android head both built with
  0 errors.
- **OPEN** — both builds report one `CS0414` warning in Lexus-owned
  `StageViewerGame._skyCenterX`; no Aston-owned file reports a warning.

### Next

- **OPEN** — Needle is next. It has not been started in this correction beat.

Wagata, Yondaime! Signed sincerely by your dear Aston

---

## Needle — static spike collision and damage

**2026-07-29 10:36 CEST (UTC+02:00)**

### Placement evidence

- **VERIFIED** — the Episode II data contains **614 `Needle` placements across
  22 acts**: ids 84/85/86/87 occur 209/20/31/11 times and ids
  445/446/447/448 occur 212/7/100/24 times. Parameters are zero throughout;
  low flag values 0/1/2/3 occur 570/12/14/18 times.
- **VERIFIED** — filtering is exclusively through
  `ObjectCatalog.Is(id, "Needle")`. `ActNeedle` is a different catalog class:
  30 placements across 10 acts, on ids 88, 89 and 449.
- **VERIFIED** — Zone 1 Act 1's base event file contains six id-445 needles.
  The real-data integration test mounts all six rather than relying only on
  synthetic placements.

### Episode II oracle

- **VERIFIED** — `GmGmkNeedleInit` at `0x00547184` dispatches normal Episode II
  stages to `GmGmkNeedleEp2Init` at `0x00548168` and Episode Metal stages to the
  separate initializer at `0x00547234`.
- **VERIFIED** — Episode II ids 445-448 provide base directions Up, Left, Down
  and Right. The initializer at `0x005481F0` adds the placement flag's low two
  bits modulo four. Episode Metal ids 84-87 select those directions directly.
- **VERIFIED** — Episode II's four physical rectangles were read from
  `0x0096123E`: `(32,30,-16,-32)`, `(40,30,-36,-16)`,
  `(32,32,-16,0)`, `(40,28,-4,-16)`, expressed as width, height and origin
  offsets. Episode Metal's table at `0x009611A6` differs only in the upward
  entry, `(24,30,-8,-32)`.
- **VERIFIED** — Episode II's attack rectangles at `0x00961256` are
  `(-15,-33,15,-8)`, `(-37,-8,-8,4)`, `(-12,32,12,8)` and
  `(8,-6,37,4)`. Episode Metal's table at `0x009611BE` narrows the upward
  entry to `(-8,-33,15,-8)`.
- **VERIFIED** — the normal main at `0x00548928` checks the ride relation only
  for the upward variant, enabling rectangle flag bit 4 while the player rides
  it and clearing that bit otherwise. The other orientations keep it set.
- **INFERRED** — the readable Episode I rectangle implementation identifies
  bit 4 as the registered attack-rectangle flag. It was used only to interpret
  the Episode II flag transition; no Episode I numeric value was used.
- **INFERRED** — the recovered physical records are resolved as full solid
  rectangles. This follows the generic solid-object initializer that consumes
  them; the four numeric records themselves are recovered directly.

### Implementation

- **VERIFIED** — `Needles` preserves the two recovered table families, maps
  directions from object id and low placement flags, resolves crossings against
  every face of the solid rectangle, and tests attack overlap against the
  separate recovered rectangle.
- **VERIFIED** — upward spikes are safe from their sides and become damaging
  only when ridden. Other directions remain damaging on attack-rectangle
  contact, while their flat faces remain physically standable where the attack
  rectangle does not reach.
- **VERIFIED** — `GM_NEEDLE` runs after the object-manager task at object
  priority, so it corrects the player's moved position and routes a hit through
  the shared `DamagePlayer` transition. A real Zone 1 ringed player loses the
  ten carried rings and enters damage on the recovered upward top.
- **OPEN** — `GameEngine` had no partial lifecycle hook. The construction,
  property, scheduler registration and check remain in
  `GameEngine.Behaviours.cs`, but one call to `MountBehaviours` was necessarily
  added at the end of `EnterStage`; Lexus should establish or accept that
  permanent shared hook before more behaviours accumulate.

### Regression

- **VERIFIED** — the class filter, direction mapping, upward top, three active
  attack directions, three pointed solid faces, two safe flat faces and engine
  mount each produced an observed RED result before their production slice.
- **VERIFIED** — targeted mutants that made the Episode Metal top as wide as
  Episode II's and left the upward attack always active each failed the intended
  regression test; restoring the recovered behavior returned them to GREEN.
- **VERIFIED** — focused Needle suite: **15/15 green**, 0 failed.
- **VERIFIED** — `dotnet test src`: **247/247 green**, 0 failed.
- **VERIFIED** — `dotnet build src
  -p:AndroidSdkDirectory=C:/Android/sdk
  -p:JavaSdkDirectory=C:/Android/jdk`: succeeded, 0 errors.
- **VERIFIED** — the Android head built separately with the SDK/JDK flags,
  `JAVA_TOOL_OPTIONS=-Xmx256m`, and `-m:1`: succeeded, 0 errors.
- **OPEN** — both builds report the existing `CS0414` warning in Lexus-owned
  `StageViewerGame._skyCenterX`; no Aston-owned file reports a warning.
- **OPEN** — one chained focused-test invocation timed out without output. Both
  tests then passed separately and the complete focused and repository suites
  passed; the timeout did not reproduce.

### Still open

- **OPEN** — `ActNeedle` has its own initializer at `0x005483A8` and a
  retracting cycle. It is deliberately excluded from this static behavior.
- **OPEN** — spike rendering, animation, effects and sound remain outside this
  core behavior beat.
- **OPEN** — the collision uses `Player.Width` and `Player.Height`; those
  12-by-25 player dimensions are still explicitly marked Episode I's and not
  recovered from Episode II.
- **OPEN** — the Zone 2 beta-versus-retail collision diff is next, followed by
  `Land`.

Wagata, Yondaime! Signed sincerely by your dear Aston

---

## Zone 2 collision — Beta 8 versus patched retail

**2026-07-29 10:45 CEST (UTC+02:00)**

### Verdict

- **VERIFIED** — this is a real height-field edit, not AMB padding, entry
  ordering or metadata. Each of `ZONE2A_ATTR.AMB`, `ZONE2B_ATTR.AMB` and
  `ZONE2C_ATTR.AMB` changes the same two bytes inside its `.DF` payload.
- **VERIFIED** — it is one unique shape-library correction replicated across
  the three tileset archives: record 180, cell 61, columns 23 and 60.
- **VERIFIED** — no collision cell placed by any shipped Zone 2 map selects that
  changed record/cell combination. Resolving every static map cell therefore
  produces identical Beta 8 and retail height fields, angles and attributes.
- **INFERRED** — the patch is gameplay-inert unless code rewrites a Zone 2
  attribute grid at runtime to select this otherwise-unused slot. No such
  dynamic grid mutation was established in this beat.

### Decoded difference

- **VERIFIED** — all six archives are 1,362,112 bytes and retain the same three
  entries, offsets and lengths. `.AT` begins at 96, `.DF` at 7,008 and `.DI` at
  1,331,552.
- **VERIFIED** — each archive differs at exactly two absolute offsets:
  `748219` changes `30 -> 31`, and `748256` changes `52 -> 53`.
- **VERIFIED** — `tools/collision.py` resolves those bytes to `.DF` record 180,
  cell 61, columns 23 and 60. Each rises by one stored height unit, or two stage
  pixels at the recovered two-pixels-per-height-unit scale.
- **VERIFIED** — the `.DF` index is unchanged. Attribute ids 1886 and 1896 both
  select record 180 in Beta 8 and retail.
- **VERIFIED** — the complete `.DI` and `.AT` payloads and indices are
  byte-identical. Both ids select `.DI` record 222, whose cell-61 angle remains
  233 units, approximately +32.34 degrees in the Y-up frame.
- **VERIFIED** — the fitted slope of the edited cell moves from approximately
  30.69 degrees to 31.84 degrees. The unchanged stored angle remains compatible
  with both versions; the full `collision.py angles` report is identical:
  12,813 shaped cells, median 4.1 degrees error, 83% within 15 degrees.

### Static-map reachability

- **VERIFIED** — the four Zone 2 map archives are byte-identical between Beta 8
  and retail: `ZONE21A_MAP`, `ZONE22B_MAP`, `ZONE23C_MAP` and
  `ZONE2BOSSB_MAP`.
- **VERIFIED** — ids 1886/1896 occur nine times across the static attribute
  layers. Their 8-by-8 record slots are 25, 0, 26, 60, 9 and 5 in Act 1, and
  37, 58 and 60 in Act 3. Slot 61, the edited cell, occurs zero times. Act 2 and
  the boss contain neither id.
- **VERIFIED** — an end-to-end scan resolved all **73,290** nonzero cells across
  every `_ATTR_A.MP` and `_ATTR_B.MP` layer through the matching A/B/C archive:
  **0 height changes, 0 angle changes, 0 character-attribute changes**.

### Tool verification

- **VERIFIED** — `collision.py show` decoded all six target archives with the
  same counts: 322 `.DF` records, 388 `.DI` records, 20 `.AT` records and 2,810
  index entries apiece.
- **VERIFIED** — `collision.py verify` parsed all nine stage-collision payloads
  under each Zone 2 tree. Its reported no-header gimmick collision files are the
  tool's documented separate format, not failures of the three target archives.
- **VERIFIED** — no source or game-data file was changed. This beat adds only
  this clean-room engineering report.

### Next

- **OPEN** — `Land` moving-platform behavior is next.

Wagata, Yondaime! Signed sincerely by your dear Aston

---

## Land — moving, routed and falling platforms

**2026-07-29 11:40 CEST (UTC+02:00)**

### Placement evidence

- **VERIFIED** — Episode II contains **394 `Land` placements across 24 acts**.
  The class-only id totals are 81:102, 82:41, 83:38, 98:92, 534:21,
  535:55, 536:32, 537:0, 538:4, 539:1 and 540:8. All 394 parameters are zero.
- **VERIFIED** — id 541 is the separate `LandRoutePos` catalog class. Its
  signed left/top fields identify route and point indices; it supplies metadata
  to id-540 platforms and is never mounted as a platform itself.
- **VERIFIED** — event bytes 6/7 are signed left/top values and bytes 8/9 are
  unsigned width/height values. `Lands.FromEventData` preserves all four raw
  fields while filtering only `Land` and `LandRoutePos` through
  `ObjectCatalog.Is`.

### Episode II oracle

- **VERIFIED** — `GmGmkLandInit` begins at `0x0053D53C` and the main begins at
  `0x0053E6D4`. Its raw-field loads at `0x0053D654–0x0053D690` use `ldrsb` for
  left/top and `ldrb` for width/height.
- **VERIFIED** — the low-two-bit speed table at `0x00960DE8` is **4, 2, 3, 5**.
  Normal platforms use the 1024-step path at
  `0x0053E804–0x0053E88C`; bit 2 waits for a rider, bit 3 reverses horizontal
  phase, and bits 4-5 add 256-step phase offsets.
- **VERIFIED** — ids 98 and 537 use the rectangle branch. The initializer doubles
  all four raw path fields at `0x0053D768–0x0053D794`; the main traverses that
  perimeter at `0x0053E744–0x0053EA50`.
- **VERIFIED** — id 540 uses the route manager at `0x0053EDF4`. The initializer
  sign-extends top and stores it in the speed halfword at
  `0x0053D850–0x0053D854`; the main reloads it unsigned, converts it and
  multiplies it by **0.5 px/frame** at `0x0053E8FC–0x0053E910`. Route ids and
  point ids are limited to 0-7. Placement bit 0 stops at the last point;
  otherwise the endpoint logic at `0x0053ED80–0x0053EDD8` ping-pongs.
- **VERIFIED** — placement bit 6 arms falling after 30 ridden frames. Generic
  enemy work loads **0.1640625 px/frame²** and **15 px/frame** from
  `0x0094E270`; Land replaces the terminal field with **7.5 px/frame** at
  `0x0053EBD4–0x0053EBD8`.
- **VERIFIED** — collision branches by the 36-entry
  `g_gm_gamedat_zone_type_tbl` at `0x009571B4`, not by Episode II versus Episode
  Metal object-id family. Type branches begin at `0x0053DA84`,
  `0x0053DA9C`, `0x0053DAE8` and `0x0053DB40`. Recovered boxes include the
  normal 56/88-wide family, later-zone 48/80-wide family, 64-by-64 type 2,
  the zone-type-8 24-by-32 type 2, and the per-zone type-3 widths. Placement
  bit 7 changes ordinary platforms from the 8-pixel one-way top to the
  24-pixel full box; type 2 is always full.
- **INFERRED** — act archives absent from the active stage-path table inherit
  the zone type of their directory's listed Episode Metal act. No collision
  constant was borrowed from Episode I.

### Implementation

- **VERIFIED** — `Lands` implements sinusoidal, doubled-rectangle and routed
  translation, phase flags, rider-triggered motion, stop/ping-pong endpoints,
  one-way and full platform collision, and the recovered delayed fall.
- **VERIFIED** — rider travel is cumulative `TempOffset`, never direct platform
  displacement written into `Player.Position`. Collision corrects the player's
  gravity and grounding before `GameObject.Update` applies the current offset,
  so a persistent ride does not accumulate. The callback also rejects stale
  players after a stage transition.
- **VERIFIED** — `GameEngine.Behaviours.cs` owns construction and wiring.
  Land runs from the active player's collision slot after movement, which is the
  point at which the current `TempOffset` can still be applied in the same frame.
- **INFERRED** — the recovered native boxes are resolved through this port's
  axis-separated `Player` collision model. The dimensions and one-way flags are
  Episode II's; exact native edge-contact ordering is not claimed.

### Regression

- **VERIFIED** — the first focused run was observed RED with `CS0246` because
  `LandPlacement` did not exist; it turned GREEN only after the production
  behavior was added.
- **VERIFIED** — the rider regression was separately observed RED when gravity
  detached the player and accumulated position. Correcting the crossing test
  and keeping travel exclusively in `TempOffset` returned it to GREEN.
- **VERIFIED** — focused Land suite: **17/17 green**, including all four speed
  selectors, wait/reverse flags, both rectangle ids, moving/zero-speed routes,
  stop versus ping-pong, collision families, landing, riding, falling and a real
  Zone 1 engine mount that advances through the installed callback.
- **VERIFIED** — `dotnet test src`: **264/264 green**, 0 failed.
- **VERIFIED** — the whole solution and the separate Android head both built
  successfully with 0 errors using the ordered SDK/JDK paths, `-m:1` and
  `JAVA_TOOL_OPTIONS=-Xmx256m`.
- **OPEN** — both builds report the existing `CS0414` warning in Lexus-owned
  `StageViewerGame._skyCenterX`; no Aston-owned file reports a warning.

### Still open

- **OPEN** — type-3 visual tilt, linked render pieces, effects and sound are not
  represented. Translational motion and collision are present.
- **OPEN** — collision uses `Player.Width` and `Player.Height`; those player
  dimensions remain explicitly marked Episode I's and not recovered from
  Episode II.
- **OPEN** — Bumper is next.

Wagata, Yondaime! Signed sincerely by your dear Aston
