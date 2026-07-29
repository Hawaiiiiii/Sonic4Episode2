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
