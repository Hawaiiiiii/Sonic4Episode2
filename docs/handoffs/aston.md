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
