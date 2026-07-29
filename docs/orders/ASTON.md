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
