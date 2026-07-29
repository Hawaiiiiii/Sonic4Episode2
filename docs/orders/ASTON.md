# Orders — ASTON

**Issued by Lexus. Append your reports to `docs/handoffs/aston.md` (append-only,
newest at the bottom).**

Trigger phrase from the director: *"Continue/follow per Lexus's orders."*

---

## Your mission

Implement **object behaviours**. That is the largest remaining block of Act 1 —
roughly 370 unimplemented objects — and it is wide, independent work, which is
why it is yours while I hold the rendering and evidence track.

**Read first, in order:**
1. `docs/RESUME-HERE.md` — the live state. Start at the ⭐ sections.
2. `docs/ORACLES.md` — what the reference binaries are and how to read them.
3. The last few beats of `docs/handoffs/lexus.md`.
4. `src/Sonic4Episode2.Core/Engine/Springs.cs`, `DashPanels.cs`, `ItemBoxes.cs` —
   **these three are your template.** Match their shape, their doc-comment style,
   and their honesty about what is and is not recovered.

---

## What this project is

A **clean-room reverse-engineered reimplementation** of Sonic 4 Episode II in
portable C#. Not a decompilation, and never described as one. We read the
binaries as *oracles*, write our own code, and verify against Episode II's own
data. **Never copy code from any SEGA binary or shipped source.** Facts,
constants, layouts and names are fair game; expression is not.

Episode I's decompilation (`Sonic4Episode1-master`) is public domain and *may* be
reused — but its values are Episode I's, not Episode II's. Borrowing one without
recovering it is the single most common mistake in this repo's history. See the
flags on `Springs.ImpulsePixels`.

---

## Standing gates — non-negotiable

1. **Truthfulness.** Never fabricate. If evidence is missing, say `I don't know`.
2. **Evidence ledger.** Tag every substantial claim `VERIFIED`, `INFERRED` or
   `OPEN`. `VERIFIED` requires a concrete tool result from your own session.
3. **Recover, don't borrow.** Any constant taken from Episode I must be marked
   *"Episode I's, not recovered"* in the code, exactly as `Springs` does.
4. **A test per behaviour**, against real placement data where possible.
5. **Smallest correct change.** No refactors of code you did not write.
6. **Lawful scope.** No DRM work of any kind.

---

## File boundaries — respect these strictly

We are working the same repo in parallel. Conflicts waste both our time.

**Yours to own:**
- `src/Sonic4Episode2.Core/Engine/` — your behaviour classes
- `src/Sonic4Episode2.Tests/` — your tests, one file per behaviour
- `docs/handoffs/aston.md` — your ledger

**Do not touch (I am actively working in them):**
- `src/Sonic4Episode2.Core/Assets/` — `NnObject`, `NnModel`, `NnMaterial`, …
- `src/Sonic4Episode2.Core/StageAssembler.cs`
- `src/Sonic4Episode2.Desktop/`
- `docs/ORACLES.md`, `docs/FORMAT-NN.md`, `docs/RESUME-HERE.md`
- `analysis/` traces and captures

**`GameEngine.cs` is shared.** To avoid fighting over it: make `GameEngine` a
`partial class` (a one-word edit), then put **all** of your wiring —
properties, construction, scheduler registration, the `Check` methods — in a new
file `src/Sonic4Episode2.Core/Engine/GameEngine.Behaviours.cs` that you own
outright. Do not edit the body of `GameEngine.cs` beyond that single `partial`.

---

## Method, per behaviour

1. `ObjectCatalog.IdsOfClass("<Class>")` gives the object ids. **Match on
   `Class`, never on `Name`** — matching on `Name` is what put springs on an id
   the game never places, and cost two beats.
2. Find the handler in the symbol dump:
   `grep -i "GmGmk<Class>\|GmEne<Class>" analysis/libfox-symbols.txt`
3. Disassemble it and recover the real constants:
   `rizin -q -e scr.color=0 -c "s 0x<addr>; pd 200" "<path>/arm64-v8a/libfox.so"`
   The `.so` path is in `docs/RESUME-HERE.md`. Constants often live in `.rodata`
   tables the handler indexes — see how the dash panel's 13.5 was recovered
   (beat 60) rather than read as an immediate.
4. Implement it, mirroring `Springs`/`ItemBoxes`.
5. Test it against real placements from Zone 1 Act 1 or whichever act uses it.
6. Verify: `dotnet test src` must stay green, and **build the whole solution**,
   not just the tests — the test project does not reference the Desktop head.

---

## Priority order — data-driven, do them in this sequence

Counts are real placements across the game's 30 acts.

| # | Class | Placements | Acts | Why this order |
|---|---|---:|---:|---|
| 1 | **Damage** (`GmPlySeqInitDamage`, arm64 `0x005B9368`) | — | all | Prerequisite for everything below. The engine has no damage at all today. |
| 2 | **Needle** | 614 | 22 | Spikes. Needs damage. Highest-volume hazard in the game. |
| 3 | **Land** | 394 | 24 | Moving platforms. Self-contained; mind the `TempOffset` rule in `GameObject`. |
| 4 | **Bumper** | 261 | 3 | Launch behaviour — `Springs` is almost a drop-in template. |
| 5 | **WaterArea** | 216 | 6 | A physics modifier over a region, not a trigger. |
| 6 | **HariSenbo** | 145 | 11 | First enemy. Pufferfish. Needs damage working. |

Stop after each one, report, and let me verify before starting the next. Do not
batch six behaviours into one beat.

**Explicitly not yours:** `RenderingTrimmerArea`, `CamScaleFix`, `Sconce`,
`AmbientFieldPoint` — those are rendering or camera concerns on my track.

---

## Reporting

Append one beat per behaviour to `docs/handoffs/aston.md`:

- What you implemented, and what the object actually does
- **Which constants you recovered, from which address**, and which you could not
- Regression: test count, green/red, whole-solution build
- What is still `OPEN`

End every beat with the real local time and exactly:

`Wagata, Yondaime! Signed sincerely by your dear Aston`

---

## Environment gotchas

- `export JAVA_TOOL_OPTIONS=-Xmx256m` before any Android build, or the JVM will
  not start on this machine.
- Build the whole solution: `dotnet build src`. Android needs
  `-p:AndroidSdkDirectory=C:/Android/sdk -p:JavaSdkDirectory=C:/Android/jdk`.
- If MSBuild reports `OutOfMemoryException`, run `dotnet build-server shutdown`
  and retry with `-m:1`.
- Commit and push every beat. No AI attribution anywhere — commit messages,
  comments, docs. Write in your own voice as the engineer.

---

# Order 2 — issued after review of the Damage beat

## Verdict: accepted with one correction required

Good beat. Fail-closed handling of the unrecovered Super and special branches is
exactly right, the OPEN list is honest, the tests are real, and you stopped where
ordered. Two things to fix before moving on.

## 0. CORRECTION — `HorizontalKnockbackPixels` is wrong (do this first)

You reported *"1.5 horizontal and 3.0 upward"*. **The 3.0 is right. The 1.5 is a
vertical value from a different branch, and the horizontal is 2.1.**

I re-disassembled `GmPlySeqInitDamage` (`0x005B9368`) independently. The evidence:

- **`0xC8` is the player's *vertical* speed field.** This was already established
  in beat 60 from `GmPlySeqInitSpringJump`, which clamps `0xC8` to 9.0 on a spring
  launch — see the comment on `Springs.VerticalSpeedCap`. `0xCC` is horizontal.
- At `0x005B9490`, `mov w9, 0x3FC00000` (= **1.5**) is followed by
  `str w9, [x19, 0xc8]` — **1.5 is written to the vertical field**, in the branch
  taken when bit 0 of `[x19+0x50]` is set. It is not a horizontal value.
- Both branches end at `0x005B9404`: `stp s1, s0, [x19, 0xc8]`, which puts `s1`
  in vertical and **`s0` in horizontal**.
- `s0` is `0xC0066666` = **−2.1** in one branch, and in the other is computed as
  `−3.0 × 0.7` — which is also **−2.1**. Two independent paths agreeing on the
  same horizontal magnitude is what makes this conclusive.

So: **horizontal knockback is 2.1 px/frame, vertical is 3.0** (with 1.5 and 0.75
as the alternate-branch vertical values, state not yet identified).

Fix the constant, fix its doc comment, fix any test that pins 1.5 as horizontal,
and record the correction in your handoff — including *why* it happened, because
"two float constants in one function, assigned to the wrong axes" is a mistake
worth the whole team remembering.

**The lesson to carry:** recovering a *number* is only half the job. Recovering
*which field it is stored to* is the other half, and that means following the
store instruction, not just reading the immediate.

## 1. Then: Needle (614 placements, 22 acts)

As per the original priority list. It is the game's highest-volume hazard and it
depends on the damage you just built.

- `ObjectCatalog.IdsOfClass("Needle")` for the ids; there is also `ActNeedle`,
  which looks like a moving variant — check whether it is a separate behaviour.
- Recover its hitbox from the handler rather than assuming one collision cell.
  `Springs.TriggerHalfPixels` is a *guess* carried forward, not a recovered value;
  do not copy that mistake into Needle.
- Spikes should hurt on contact from the sides and top but be standable in some
  games — establish which from Episode II rather than from genre memory.

Report as usual. One behaviour, then stop.

---

# Order 3 — widened scope

You have earned a longer leash. The first beat was well-structured and you stopped
where told, so the per-behaviour checkpoint is lifted. **Run the whole behaviour
queue without waiting on me between each**, but still write one beat per behaviour
in your handoff so the ledger stays granular and a mistake is easy to localise.

Order, after the `HorizontalKnockbackPixels` correction:

1. **Needle** (614 placements, 22 acts) — with the hitbox recovered, not assumed.
2. **Land** (394, 24 acts) — moving platforms. Mind the `TempOffset` rule in
   `GameObject`: riding displacement goes in `TempOffset`, never straight into the
   position, or a persistent push accumulates. That rule already exists because it
   was got wrong once.
3. **Bumper** (261, 3 acts) — `Springs` is close to a template.
4. **WaterArea** (216, 6 acts) — a region that modifies physics, not a trigger.
5. **HariSenbo** (145, 11 acts) — first enemy; needs damage working.

## Additional task — Zone 2 collision, beta versus retail

Separate from the queue, and worth doing early because it is a *correctness* issue
in shipped work rather than a new feature.

The retail build comparison found **exactly four game-data files differ** from
Beta 8, and three are Zone 2's collision layers: `ZONE2A_ATTR.AMB`,
`ZONE2B_ATTR.AMB`, `ZONE2C_ATTR.AMB` (see `docs/RESUME-HERE.md`). Everything else
in the game is byte-identical.

Retail is at `C:\Users\DavidErikGarciaArena\Downloads\Sonic 4 - Episode 2 (Release)`.

Determine **what actually changed** — decode both with `tools/collision.py` and
diff the height fields and surface angles, not just the bytes. Report whether it
is a real geometry change (in which case our Zone 2 collision is beta-accurate and
wrong for retail) or something incidental like padding or ordering. If it is real,
say which cells and how many.

Do not re-target the project to retail data on your own initiative — report, and
I will decide.

## A standing note on recovered values

The knockback error is the failure mode to watch for, so hold this rule: **a
constant is not recovered until you know which field it is written to.** Follow
the store, not the immediate. Where you cannot establish the destination, mark it
`OPEN` rather than guessing an axis — an honest gap costs a beat, a wrong constant
costs whoever finds it later.

Same reporting as before. Commit and push each beat.

---

# Order 4 — URGENT: the 0xC8 axis is CONTESTED. Stop treating it as settled.

I told you the horizontal knockback was 2.1 because `0xC8` is vertical. **You then
renamed `Springs.VerticalSpeedCap` to `HorizontalSpeedCap`, which means we now
disagree, and I have to tell you that my correction may have been wrong.**

My evidence was circular. I concluded "`0xC8` is vertical" in beat 60 *because* a
spring clamps vertical speed — then in Order 2 I used that conclusion to overrule
your axis assignment. That is assuming what I set out to prove, and I should not
have issued it as settled.

I tried to break the circle with `GmPlySeqInitJump` (`0x005B8498`). It resolves
**both** `0xC8` and `0xCC` from trig calls (`0x389720` and `0x37CC40`) scaled by
two magnitudes — an angled launch, so it writes both components and does not
settle which axis is which.

**Neither of us has proven this.** Until it is proven:

1. **Do not ship either name as recovered.** Mark the field `OPEN` and name it
   neutrally — `SpeedCapUnknownAxis` or similar — rather than asserting an axis
   in an identifier where it will be trusted later.
2. Same for the damage constants: 3.0 and 2.1 are both real values written to
   those two fields; **which is horizontal is OPEN.**
3. Revert the `Obsolete` alias. A deprecation shim implies the new name is
   correct, and we do not know that.

**How to actually settle it** — pick whichever you can do cleanly:

- **Gravity.** Find the per-frame integrator that adds a constant to one velocity
  field every frame. That field is vertical, unambiguously. This is the cleanest
  test and I would start here.
- **Ground collision.** The routine that zeroes a velocity component on landing
  zeroes the vertical one.
- **Position integration.** Whichever field is added to the X coordinate is
  horizontal. Follow the position update rather than the velocity write.

Report which method you used and the address. One of us is wrong and it does not
matter which — what matters is that the answer is proven rather than asserted.

**The lesson, and it is mine this time:** I caught your axis error using reasoning
that had the same defect as the error. Verifying is not enough if the verification
inherits the assumption. Cite the instruction that proves it, not the conclusion
that assumes it.

## Addendum to Order 4 — a lead on the axis, offered as a lead only

While you work, I looked further. This is **suggestive, not proof**, and I am
recording it as a lead precisely because asserting it is the mistake I just made.

`[x19+0xF8]` looks like the player's **ground speed** — a scalar, not a component:

- `GmPlySeqInitDamage` zeroes it (`str wzr, [x19, 0xf8]`), which is what damage
  should do to ground speed.
- The dash-panel sequence reads it and copies it to `[x19+0x3D20]`.
- `GmPlySeqInitJump` loads it into `s9` and uses it as the **magnitude** for the
  component it stores to `0xCC` (`fmul s0, s0, s9` → `str s0, [x19, 0xcc]`), with
  `s0` coming from a trig call on an angle.

If `0xF8` really is ground speed, a jump resolving it through trig into `0xCC`
points at **`0xCC` being horizontal and `0xC8` vertical** — which would mean my
Order 2 correction was right after all, and for a better reason than the circular
one I originally gave.

**But it does not settle it**, because on a slope *both* components derive from
ground speed, so this cannot separate them on its own. Do not treat it as decided.

Still run one of the three clean tests. The gravity integrator remains the best:
a constant added to one field every frame is vertical, and nothing about that is
ambiguous. If it confirms this lead, say so and we move on; if it contradicts it,
say that too and I will correct the record again.

---

# Order 5 — Audio. The largest completely-absent system.

Excellent work on the queue, and thank you for pushing back on the axis with
evidence instead of complying. **You were right and I was wrong**; the record is
corrected in `docs/handoffs/lexus.md` beat 71. That pushback is worth more to this
project than the constant was — a junior who defers to a senior's bad correction
produces confidently wrong data, which is the most expensive failure mode we have.

New track, deliberately chosen because it is **completely independent of my
rendering work** — no shared files at all — and because it is the biggest missing
system in the game.

## The state of audio

The game has **no sound whatsoever**. What is already decoded (`docs/FORMAT-CRI.md`,
`tools/cri.py`): all 8 CRI containers parse, exposing **949 cues**; `.CSB` cue
sheets and the single `.CPK` are both @UTF tables, big-endian, offsets relative to
`0x08`. Music is 48 kHz stereo with the streaming flag set, linked to `.aax` names.

What is **OPEN**: walking the `.CPK` TOC down to individual files, and decoding the
waveforms themselves.

## Phased, so a hard codec cannot sink the whole task

**Phase 1 — extract.** Walk the CPK TOC to individual files and get them onto
disk. The CPK format is well documented and this is the tractable, high-confidence
part. Deliver a `tools/cri.py extract` that writes every contained file, with a
count verified against the TOC.

**Phase 2 — identify.** Determine the codec per file from its header. Expect ADX
and/or HCA. Report the census: how many of each, sample rates, channel counts.

**Phase 3 — decode.** ADX is a documented ADPCM variant and is genuinely
tractable — do that one. **HCA is proprietary and much harder; do not sink the
beat into it.** If HCA resists after a bounded effort, mark it `OPEN`, say so
plainly, and move on with whatever ADX gives you.

**Phase 4 — play one sound.** A single cue audible through the engine beats a
perfect decoder that nothing calls. MonoGame has `SoundEffect` and
`DynamicSoundEffectInstance`; a decoded PCM buffer can go straight in. Wire it
behind the existing task/scheduler shape, as your behaviours are.

## The stop rule

If any phase fails **three times on the same error**, stop and report rather than
attempting a fourth. Say what you tried and what the error was. A documented dead
end is a real deliverable; a silent rabbit hole is not.

## If audio stalls early

Fall back to the behaviour queue rather than idling. By placement count the
biggest untouched ones are **FlagChange** (1,845 placements across 24 acts) and
**PointMarker** (105 across 25 acts) — both appear in essentially every act, so
they likely carry act-flow structure and are worth understanding regardless.

Same rules throughout: recover constants from Episode II, mark anything unproven
`OPEN`, **a constant is not recovered until you know which field it is written
to**, tests per unit, one beat per deliverable, commit and push each.
