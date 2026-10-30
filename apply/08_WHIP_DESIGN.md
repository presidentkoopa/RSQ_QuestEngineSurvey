# 08 APPLY — THE WHIP: wind-up, slack-as-gauge, and three ways to aim

**Mod:** `Grapple/` in `github.com/presidentkoopa/RSQ_UBQSeries`, HEAD `6a5c98c`, 2026-10-30.
**Engine:** `qzd173` (tag `qzd17.3`, emawind84/gzdoom). All `file:line` resolve in those two trees.
**Companion:** `apply/01_GRAPPLE_TETHER.md`. That document diagnosed the sag and proposed deleting it.
**This document supersedes that recommendation** — the diagnosis was right, the prescription was wrong.
**Scope:** ZScript and data only. No engine change anywhere. Nothing is written into the mod's tree.

> **The owner, verbatim:** *"that 'sag' is what made me think it could be a whip in the first place.
> not saying i want it that way, i don't know how it works im not a coder, but i saw what could be a
> whip and i had visions of two handing a whip to build momentum, somehow targeting an enemy and
> letting loose. maybe a lock on somehow or something else, I'm open to anything. indiana jones X
> castlevania"* — and then: all three targeting options, selectable from the menu.

---

# PART ONE — WHAT THE PLAYER DOES

*(No code in this half. Skip to Part Two for the implementation.)*

## The thing you saw was the right thing

The droop in the rope is the only part of the current whip that has any **weight** to it. Everything
else about it is a stopwatch: press the button, and exactly four tics later — 114 milliseconds, the
same every single time — the lash reaches exactly as far as a menu slider says, and does exactly as
much damage as another menu slider says. How hard or fast you moved your arm changes nothing but the
direction.

So the plan is not to delete the droop. The plan is to **delete the stopwatch and keep the droop** —
and then make the droop mean something, because right now it is arbitrary too: it is a fixed 8 % of
the rope's length, held for a fixed four tics, with no relation to anything you did.

## Two hands, and a rope you wind up

You take the whip in both hands and **circle it** — around your body, over your head, however you
like. Each revolution adds to the whip's stored momentum. Keep circling and it keeps building; stop,
and it bleeds away over about a second. You cannot bank a charge and walk around with it, and you
cannot get there in one flick: it is a thing you spend a second or two doing, on purpose, which is
exactly what winding up a whip is.

What counts is **the circling**, not how fast your hands travel in a straight line. Punching forward
builds nothing. Going round builds everything. That is the right rule because it is the real one, and
because it means a nervous player waving their arms about does not accidentally charge a whip.

## The rope tells you how charged you are. There is no meter.

**This is the part the droop buys us.**

At rest the rope hangs off your hand with a real, heavy sag. As you wind up, the slack **pulls out**
of it: the sag shallows, the arc tightens, and the whole rope lifts and starts to travel. At full
charge it is a taut, fast, shallow loop. Let the charge decay and you watch the slack come back.

So the charge gauge is the rope itself. No HUD number, no bar, no glow — you glance at the whip and
you know. That is the whole reason to keep the sag, and it is why a bug report would have been the
wrong read on it.

## Letting loose

Release, and the stored momentum becomes three things at once:

- **Reach.** A cold whip reaches a fraction of its full length. A fully wound one reaches all of it.
- **Damage.** Same curve.
- **The crack itself** — how fast the lash straightens and where in the swing it snaps. A wound-up
  release is a fast, flat, loud crack. A cold one is a floppy slap that barely gets out there.

The two menu sliders that currently *set* reach and damage become **ceilings** instead: the most a
perfect release can do. Nothing about your existing settings breaks; they just become the top of a
range rather than a constant.

## Three ways to aim, and you pick

Winding up a whip and then having to also thread a needle with it is a lot to ask of one gesture, so
the aim gets its own setting with three honest options:

- **Free.** No help at all. The lash goes where the lash goes and hits whatever it crosses. The most
  honest and, bluntly, **the hardest to actually enjoy** — at 35 samples a second a fast lash is
  three or four snapshots, and you will miss things you were sure you hit. Right for people who want
  the raw thing; wrong as a default.
- **Snap within arc** — **the recommended default.** You aim roughly, and on release the tip bends
  toward the best enemy inside a cone around where you threw it. Aim badly and you still miss; aim
  near enough and it connects. This is what makes a whip feel like a whip instead of like a physics
  demo, and it is how every game that has ever shipped a good whip does it.
- **Explicit lock.** Point at something, press a button to lock on, wind up, release, and it goes
  there. Reliable, readable, and **the most likely of the three to stop feeling like a whip and start
  feeling like a gun.** Worth having — some people will want it, and it is the only mode that works
  for someone who cannot aim precisely — but I would not make it the default.

All three are one setting in the Grapple's options page, next to the existing Whip entry.

## What this is not

It is not a new mod, and it is not a rewrite of the grapple. The hook, the rope, the swing, the reel,
the climb and the monster yank are all untouched. And the best part of the current whip is kept
exactly as it is: when the lash catches something **above** you, it already hands the rope over to
the grapple's own swing code, and you are on a pendulum. Indiana Jones is already in there. This
document is about making Castlevania work too.

---

# PART TWO — IMPLEMENTATION

## 0. The reframe, stated against the code

`01_GRAPPLE_TETHER.md` §1 found the sag and was right about where it lives:

```
whip.zs:213   double bow = (age <= OUT_TICS) ? len * 0.25 * (1.0 - double(age)/OUT_TICS) : len * 0.08;
whip.zs:222   pb.z += bow * sin(180.0 * u) * (age <= OUT_TICS ? 1.0 : -1.0);
whip.zs:223   pb += side * (len * 0.06 * u * sin(360.0 * (1.5 * u - phase)));
whip.zs:27    const BACK_TICS = 4;      // with :204-206, the 114 ms those three lines are seen in
```

Its prescription — make the bow decay to zero and freeze the wave — **is withdrawn.** What is wrong
with those lines is not that they draw slack. It is that **every number in them is a constant**:

| Line | What is arbitrary | What it should be |
|---|---|---|
| `:213` second branch | `len * 0.08`, fixed | a function of stored momentum |
| `:222` | sign flips once, at a fixed tic | sign and depth follow the charge, continuously |
| `:223` | amplitude `len*0.06`, phase rate `1.5u - phase` where `phase = age/OUT_TICS` | both scaled by charge |
| `:27`, `:204-206` | 4 tics out, 4 tics back, always | the throw's own measured duration |
| `:78` | `wreach` read from a cvar at `Start` | a cvar **ceiling**, scaled at release |

So: **the slack survives and becomes the gauge; the stopwatch dies.** Three of those four rows are
the same edit — multiply by a charge term — which is why this is an M and not an L.

**Label: the reframe costs nothing. The work below is BUILDABLE / M, with one S and one L-shaped risk
named in §4.3.**

---

## 1. THE TWO-HAND WIND-UP

### 1.1 Where it is measured

Build on the per-frame hand stream, which already exists and is already the only multiplayer-correct
hand read in the mod (`01_GRAPPLE_TETHER.md` §9.3):

```
input.zs:91-106    RenderOverlay -> SendNetworkEvent("rsqgr_pose", hand, RSQGR_Climb.Pack(off))
input.zs:141-146   NetworkProcess -> core.poseOff[i], core.poseTic[i]
climb.zs:48-56     HandOffset() -> (ok, off), with a 3-tic freshness window
climb.zs:68-78     Pack / Unpack: 3 x 10 bits of half-units, +512
```

Three changes, all small, all already scoped in `01_GRAPPLE_TETHER.md` §8.1:

1. **Widen the send gate.** `input.zs:101` sends only while `grabHeld` is set. Add "or a whip is
   armed on this hand". One `||`.
2. **Keep a ring, not a single sample.** `core.poseOff[i]` → an 8-entry ring per hand with its tic.
   8 tics ≈ 229 ms, the budget `01_MELEE` §3.2 settles on.
3. **Keep the height trick verbatim.** `input.zs:103` measures the hand's Z against *that frame's own
   eye* (`e.ViewPos.z`), not `pmo.Pos` and not `player.viewz`; the comment at `input.zs:79-90` records
   the measured 6-14 unit per-stroke wobble from each wrong choice. A wind-up differenced off a
   wobbling Z would charge itself while the player stood still. **Load-bearing.**

**Packing is adequate and needs no change.** `Pack` is half-unit resolution over ±256 units
(`climb.zs:68-78`), and a hand circles within ~30 units of the body, so the quantisation is 0.5 u on
a ~30 u radius — under 2 %. Fine for an angle.

### 1.2 Angular, not linear — and immune to the two-hand trap by construction

Measure the **angle swept about the player's own body axis**, per tic, from the two hand positions:

```zscript
// post-normalisation (Z already multiplied by level.pixelstretch -- see 1.4)
Vector3 mid  = (handRing[MAIN][0] + handRing[OFF][0]) * 0.5;  // offsets from the pawn
Vector2 r    = mid.xy;                                        // radius vector in the body plane
double  rad  = r.Length();
if (rad < MIN_RADIUS) { /* hands too close to the spine: no angle to speak of */ }
double  phi  = atan2(r.y, r.x);                               // absolute, wraps
double  dphi = Actor.DeltaAngle(phiPrev, phi);                // SHORTEST arc, signed, degrees/tic
```

`Actor.DeltaAngle` is the right tool and is already in house — `monster.zs:279` uses it to work out a
tumble's residual facing.

Three properties worth stating, because each one kills a class of false positive:

- **It is signed, so it accumulates only while the circling keeps its direction.** Reverse mid-wind
  and the sign flips and the accumulator drains. A player waving their arms back and forth nets ~0.
  Linear speed cannot do this.
- **It ignores a forward punch entirely** (`dphi ≈ 0` for radial motion), which is exactly the
  owner's "circling builds it".
- **It is derived from hand POSITIONS, never from `AttackAngle` / `AttackPitch`.** This matters. With
  `vr_two_handed_weapons` true (default,
  `qzd173/src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:119`), pressing the engine's offhand
  **grip** with the hands inside 0.50 m latches `weaponStabilised`
  (`qzd173/src/rendering/gl/stereo3d/gl_openvr.cpp:1704-1727`), after which the engine **overwrites
  the main hand's yaw and pitch** with the hand-to-hand vector
  (`qzd173/src/rendering/gl/stereo3d/gl_openvr.cpp:1739-1756`). `weaponoffset` — and therefore
  `AttackPos` — is written just above at `:1733-1738` and is **not** touched. Since a two-handed whip
  grip puts the hands well inside 0.50 m, an angle-based wind-up would be silently hijacked. A
  position-based one is bit-identical stabilised or not, on either VR backend.
  The mod is otherwise **not exposed** to the trap at all, and that was verified, not assumed: it
  defines **no `Weapon` subclass** (24 classes under `zscript/rsqgr/`, none deriving from `Weapon`),
  and grab state comes from its own KEYCONF aliases rather than the engine grip
  (`KEYCONF.txt`, `input.zs:68-71`, `core.zs:536-547`) — which is precisely the defence
  `00_CAPABILITIES` gotcha 10a recommends, reached independently. **Do not change that.**

### 1.3 Accumulate and decay

```zscript
// per tic, while armed and both hands' poses are fresh and good
double gain = abs(dphi) * rad / REF_RADIUS;        // degrees x (radius / a reference radius)
if (sign(dphi) != sign(dphiPrev)) gain = 0;        // direction reversed: this tic banks nothing
charge = min(charge + gain * K_GAIN, CHARGE_MAX);
charge *= DECAY;                                   // every tic, armed or not
double k = clamp(charge / CHARGE_MAX, 0.0, 1.0);   // 0..1, the number everything else reads
```

**The curves, and why these shapes.**

- **Accumulate: linear in swept angle, scaled by radius.** Resist the temptation to make it
  super-linear — a whip's stored energy really is roughly proportional to the work done going round,
  and a linear accumulator with a cap is the only shape a non-coder can feel the edges of. The radius
  term is what makes a big, committed arm circle worth more than a wrist twirl, and it is free.
- **Target: full charge in two to three deliberate revolutions.** Two revolutions is 720°, so with
  `K_GAIN` chosen so `CHARGE_MAX` ≈ 900 "degree-units" at `REF_RADIUS`, a tight fast double circle
  tops out and a single lazy one sits near 40 %. Those are the numbers to put in front of the headset
  first; they are reasoned, not measured.
- **Decay: exponential, with a stated half-life, not a linear drain.** "Bleeds away over roughly a
  second" → a ~0.4 s half-life, so `DECAY = 0.5 ** (1.0 / (0.4 * 35)) ≈ 0.951` per tic. Exponential
  is the correct choice over linear for one concrete reason: it is **scale-free**, so a half-charged
  whip and a full one lose the same *proportion* per tic and the gauge in §2 therefore moves at a
  visually constant rate at every charge level. A linear drain empties a low charge suddenly, and the
  rope would appear to collapse.
- **Decay runs even while winding.** So holding a steady circle finds an equilibrium rather than
  climbing forever, and the equilibrium is set by how hard you are circling. That is a better feel
  than a ratchet, and it is one line.

### 1.4 Units — the one place this can be quietly wrong

`vr_vunits_per_meter` is 34.0 (`hw_vrmodes.cpp:94`) and the tic rate is 35, so **1 map unit/tic ≈
1.03 m/s horizontally**. Vertically the hand transform divides the up axis by `pixelstretch`
(`qzd173/src/rendering/gl/stereo3d/gl_openvr.cpp:1282-1292`), default 1.2
(`qzd173/src/gamedata/g_mapinfo.cpp:314`), so an unnormalised height reads ~17 % short.

**An overhead circle is the one wind-up where this bites**, because its swept angle is measured in a
plane with a large vertical component. The mod already carries the term —
`util.zs:43` has `sin(pitchUp) / PIXELSTRETCH`, `util.zs:104` multiplies Z by it — and `HANDOFF.md`
records the direction verified equal to the engine's own `AttackDir` at five poses. **Already right.**
One change only: `util.zs:21`'s `const PIXELSTRETCH = 1.2` should read `level.pixelstretch`
(`qzd173/wadsrc/static/zscript/doombase.zs:479`, settable from MAPINFO at
`qzd173/src/gamedata/g_mapinfo.cpp:1566`), falling back to 1.2.

### 1.5 Arming, and the spike filter that must land first

Arming reuses the existing press/hold split, which already exists: `KEYCONF.txt` gives
`+rsqgr_main` / `+rsqgr_off` a real press/release pair, and `RSQGR_Core` tracks `held[i]` and
`pressTic[i]` (`core.zs:20-21`, `:59-60`) against an 8-tic tap window (`TAP_TICS`, `core.zs:17`).
Tap = fire the claw, unchanged. **Hold = arm the whip and start integrating.** Release = throw.

**Prerequisite, and it is a real shipped bug.** `RSQGR_Util.PoseGood` (`util.zs:51-60`) correctly
gates on `OverrideAttackPosDir` and rejects a hand beyond `ARM_MAX = 96` (`util.zs:20`), but it does
**not** reject the engine's own hand-collapse: on any frame where the device update does not run,
`VRMode::SetUp` leaves both hands at the body centre
(`qzd173/src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:438-446`), and the device code runs only
inside `if (gamestate == GS_LEVEL && menuactive == MENU_Off && !paused)` (`gl_openvr.cpp:1704`). A
collapsed pose sits ~41 units above the pawn — inside `ARM_MAX` — so `PoseGood` returns **true**.
`HANDOFF.md` already reports the symptom ("A press in the first frame after a menu closes fires from
the body"). For a wind-up it is worse than a wrong fire: the collapse-then-restore pair is a huge
apparent `dphi` and would **charge the whip from a menu close**. Two lines:

```zscript
if ((pmo.AttackPos - pmo.OffhandPos).Length() < 0.01) return false;   // both hands collapsed
if ((now - prev).Length() > SPIKE_REJECT) { invalidate ring; return false; }   // ~40 u/tic
```

**Label: BUILDABLE / M** (the ring and the gate are **S**; the integrator and its curves are the M).

### 1.6 Cvars

| Cvar | Default | Meaning |
|---|---|---|
| `rsqgr_whip_wind` | `1` | two-hand wind-up on; `0` = release throws at a fixed mid charge, so one-handed play still works |
| `rsqgr_whip_wind_full` | `900` | swept degree-units at the reference radius for full charge |
| `rsqgr_whip_wind_decay` | `0.4` | charge half-life, seconds |
| `rsqgr_whip_wind_radius` | `24` | reference radius, map units (≈ 0.7 m) |

Sliders go on the **Tuning and Debug** page (`MENUDEF.txt:84-124`), beside the existing
`Whip Reach` / `Whip Damage` rows at `:117-118`. `rsqgr_whip_wind` is a front-page `Option`, by the
`Whip` row at `MENUDEF.txt:75`. All four must be added to `rsqgr_reset` — see §4.5.

---

## 2. THE SLACK IS THE CHARGE GAUGE

### 2.1 The mapping

`Draw` (`whip.zs:200-243`) keeps its shape. Three terms become functions of `k` (§1.3):

```zscript
// --- replaces whip.zs:213 ---
// Depth of the hanging bow. k = 0 -> deep and heavy; k = 1 -> nearly pulled flat.
double bowRest = len * BOW_REST;              // BOW_REST ~ 0.22  (deeper than today's 0.08)
double bow     = bowRest * (1.0 - 0.85 * k);  // never fully zero: a taut rope still has a little

// --- replaces whip.zs:222 ---
// Sign follows CHARGE, not the tic counter. Below k ~ 0.5 the rope hangs (down, -1);
// above it the loop has lifted and travels (up, +1). Cross-fade so it does not snap over.
double lift = clamp((k - 0.5) * 2.0, -1.0, 1.0);          // -1 hanging .. +1 lifted
pb.z += bow * sin(180.0 * u) * lift;

// --- replaces whip.zs:223 ---
// The travelling wave: amplitude GROWS with charge (a cold rope barely moves, a wound one
// snakes), and its phase rate grows too (a wound rope's wave runs down it faster).
double amp  = len * WAVE_AMP * (0.25 + 0.75 * k);          // WAVE_AMP ~ 0.06, as today
double rate = WAVE_RATE_MIN + (WAVE_RATE_MAX - WAVE_RATE_MIN) * k;   // cycles down the rope
wavePhase += RSQGR_A11y.Deg(rate * 360.0 / 35.0);          // accumulated, NOT age/OUT_TICS
pb += side * (amp * u * sin(360.0 * (1.5 * u) - wavePhase));
```

Four deliberate choices:

1. **`bow` never reaches zero.** A rope drawn as a perfectly straight line stops reading as a rope.
   Keep ~15 % of the rest bow at full charge.
2. **The lift cross-fades through `k = 0.5` instead of flipping.** Today's flip is instantaneous at
   `age == OUT_TICS` (`whip.zs:222`) and is visible as a pop. A `clamp((k-0.5)*2, -1, 1)` fade makes
   the rope *rise through* straight as it charges, which is the single most legible moment in the
   whole gauge: **straight means half.**
3. **`wavePhase` must accumulate, not be derived from `age`.** Today `phase = age / OUT_TICS`
   (`whip.zs:214`) and keeps climbing past 1.0 into the retract, so the wave rolls on a rope that has
   already cracked. An accumulator driven by the charge-dependent rate removes the stopwatch from the
   visual entirely, which is the point of this whole document.
4. **Rate through `RSQGR_A11y.Deg()`.** `a11y.zs:39` caps a per-tic phase step at `MAX_DEG = 25.7`
   (2.5 Hz) when Photosensitive Mode is on (`a11y.zs:20-23`, `:34`). A charge-driven wave is exactly
   the kind of accelerating flicker that rule exists for, and the helper is already written. **Free
   correctness; use it.** Do not reimplement the cap.

### 2.2 Why no HUD

Because the rope is better, and because the alternative costs more. A meter needs `RSQ_Accessibility`'s
text and cue scaling (`a11y.zs:57-61`), a position that works at every height in VR, and a decision
about multiplayer. The rope needs none of that and is already drawn, by code that is already
measured: `hook.zs:143-161` documents the chain look against jpalomo's original (links every 5.0
units, nearest 15.0 units out), and `HANDOFF.md` records the rope draw at mean 0.016 ms / max
0.061 ms per rope.

**One accessibility concession, and route it properly.** A player who cannot read the rope at a glance
gets an optional cue through the service the mod already uses: `RSQGR_A11y.CueVisual()`
(`a11y.zs:64`) for a small charge ring scaled by `RSQGR_A11y.Cue()` (`a11y.zs:57`), and
`RSQGR_A11y.CueRumble()` (`a11y.zs:65`) for a rising per-hand tick. Both read as OFF unless
`RSQ_Accessibility` is loaded — the class-name guard at `a11y.zs:27` — so the mod still plays alone.
**Do not add a charge HUD to the Grapple's own menu.**

### 2.3 The A/B reach test, restated in the new terms

`01_GRAPPLE_TETHER.md` §1 proposed an `rsqgr_whip_reach 192` vs `48` A/B to decide whether the sag
was geometry or timing. **That question is now moot** — both halves are being replaced — so the test
is repurposed into the one that actually matters now:

**Does the gauge read?** Hold the arm at four fixed charges and look at the rope.

```
rsqgr_whip 1; rsqgr_whip_aim 1; rsqgr_debug 1
rsqgr_whip_wind_decay 999          // freeze the charge so a frame can be captured at a known k
```

With decay effectively off, drive `charge` to 0.00 / 0.33 / 0.66 / 1.00 and dump a frame at each.
Four frames side by side answer it in a glance, and §6 gives the exact runs. If the four frames are
not instantly orderable by eye, `BOW_REST` and `WAVE_AMP` are too small — raise `BOW_REST` toward
0.3 before touching anything else, because depth separates better than wave amplitude does at a
glance.

**Label: BUILDABLE / S.** It is three expression swaps inside one existing function.

---

## 3. RELEASE — MOMENTUM INTO REACH, DAMAGE AND TIMING

### 3.1 The conversion

```zscript
// at release, k = the charge at the moment the bind came up
reach  = lerp(REACH_MIN_FRAC * ceilReach, ceilReach, k);   // ceilReach = rsqgr_whip_reach
damage = max(1, int(ceilDamage * (0.25 + 0.75 * k)));      // ceilDamage = rsqgr_whip_damage
outTics = int(round(lerp(OUT_SLOW, OUT_FAST, k)));         // OUT_SLOW ~ 7, OUT_FAST ~ 2
```

- **`rsqgr_whip_reach` (192) and `rsqgr_whip_damage` (12) become ceilings**, keeping their names,
  their CVARINFO rows (`CVARINFO.txt`, the whip block) and their existing sliders
  (`MENUDEF.txt:117-118`). No ini breaks; the values simply become the top of a range. Same
  convention `01_GRAPPLE_TETHER.md` §8.4 proposed for the tether, applied here.
- **`REACH_MIN_FRAC ≈ 0.35`.** A cold whip must visibly fail to get there — that is what teaches the
  wind-up without a tutorial.
- **`outTics` replaces `OUT_TICS` (`whip.zs:26`) and `BACK_TICS` (`whip.zs:27`) as consts.** A wound
  release straightens in 2 tics (57 ms) and reads as a snap; a cold one takes 7 (200 ms) and reads as
  a flop. The retract then runs for `outTics` again, so the whole lash is 4-14 tics instead of
  always 8. **This is the stopwatch dying**, and it is the single change the owner will feel most.
- **Charge is spent.** Set `charge = 0` at release. No double-dipping, and the gauge visibly empties.

### 3.2 Resolving the hit — the swept chord

A single ray fired on one tic (`whip.zs:118`) samples the lash's final line and nothing it swept
through. At 192 units of lash with the wrist turning 60° in a tic, the tip bulges
`192 × (1 − cos 30°) ≈ 25.7` units past the chord — **larger than an imp**, which is radius 20 and,
because Doom's actor test is Chebyshev rather than circular
(`qzd173/src/playsim/p_map.cpp:1417-1419`, trace path
`qzd173/src/playsim/p_trace.cpp:695-696`, neither reading `Angles.Yaw`), a 40×40 axis-aligned square.
A fast lash sweeps clean past and reports a miss. With charge now *shortening* `outTics`, this gets
worse precisely when the player did everything right.

Apply `01_MELEE` §3.5, with the whip's adaptation — interpolate the **angles**, because the tip's
motion is dominated by rotation:

```zscript
const SUBS = 3;                      // 3 below ~24 u/tic of TIP speed; 5 above
Vector3 prevTip = TipAt(0.0), prevRoot = RootAt(0.0);
for (int i = 1; i <= SUBS; i++)
{
    Vector3 tip = TipAt(double(i)/SUBS), root = RootAt(double(i)/SUBS);
    if (Sweep(prevTip, tip)) { resolve; break; }   // what the TIP swept across
    if (Sweep(root, tip))    { resolve; break; }   // something standing IN the lash
    prevTip = tip; prevRoot = root;
}
// angle lerp must take the shortest arc: a0 + Actor.DeltaAngle(a0, a1) * t   (monster.zs:279 idiom)
```

**Cost: 6 traces per tic per lashing hand, 12 with both hands; 10 / 20 at `SUBS = 5`** — and only
during the 4-14 tic lash, not continuously. For scale, `01_MELEE` §3.4 calls 210 traces/second
"nothing", a `LineTracer` call is one `FPathTraverse` (the same order as a hitscan bullet), and the
mod's measured budget is core `WorldTick` mean 0.041 ms / max 0.098 ms under the owner's full load
order with two hooks out (`HANDOFF.md`).

**No new primitive is needed, and this is where the mod is already right.** `RSQGR_Tracer`
(`hook.zs:390-411`) is already a `LineTracer` subclass taking an arbitrary start and direction
(`qzd173/wadsrc/static/zscript/doombase.zs:310`), whose `TraceCallback` already rejects the shooter,
players, the mod's own rope actors, missiles and non-shootables (`hook.zs:396-407`). That is
`TRF_ABSPOSITION` without the flag, shipped and tested. Reuse it; one `Trace` per sub-chord.

One guard, because the collision box is square: after a hit, confirm `Distance3D(HitActor)` against
`victim.radius + fatten` and reject beyond it, or a corner graze registers as a clean body hit from
1.41× the intended reach.

### 3.3 What must not change at release

**The snag path.** `whip.zs:145-153` → `Snag()` (`:168-196`) → `RSQGR_Core.Latch()`
(`core.zs:289-312`). A crack that lands above the hand already builds core's hook exactly as a claw
that hit that spot would have, and everything after it — swing, reel, tap-to-let-go — is core's own
unchanged code. **This is Indiana Jones, already working** (`HANDOFF.md`: "Snag: tip on the alcove
lintel 91 u above the hand -> core's own Latch(); reel and swing are core's"). Every new mode must
enter through `Latch()` the same way; if one needs state `Latch()` does not set, add a defaulted
parameter rather than forking it.

**The damage bracket and its ordering.** `whip.zs:129` calls `RSQGR_Events.Whipped(v, pmo)` **before**
`RSQGR_Perks.Hurt(...)`. The service contract (`service.zs:54-58`) promises every blow is noted
before it lands, so a scorer hearing the death inside `DamageMobj` already reads `"whipped"` as 0
(`service.zs:444`), and `RSQ_TopHPBars` reads `"stunned"` off it (`service.zs:445-454`). **Keep the
call order exactly.**

**Label: BUILDABLE / M.**

---

## 4. THREE TARGETING MODES

One new cvar, three values, one menu row. The owner asked for all three selectable; this is the
shape that matches the mod's own conventions.

### 4.0 The cvar, in the file's own style

```
// CVARINFO.txt, inside the existing "monster side" whip block, after rsqgr_whip_above
server int    rsqgr_whip_aim      = 1;      // where the crack goes: 0 free (what you threw), 1 snap to the best target in an arc, 2 the target you locked
server float  rsqgr_whip_arc      = 28;     // mode 1: half-angle of the arc searched, degrees
server float  rsqgr_whip_bend     = 18;     // mode 1: most the tip is bent toward it, degrees
server int    rsqgr_whip_lock_hold = 10;    // mode 2: tics the grab bind must be held to take a lock
```

Matching `CVARINFO.txt`'s existing conventions exactly: `server`, aligned columns, a trailing `//`
comment that says what it does in plain words, inside the `==== monster side ====` block where every
other `rsqgr_whip_*` cvar already lives.

```
// MENUDEF.txt -- OptionValue beside RSQGR_MonsterModes / RSQGR_Tumbles / RSQGR_LookSets
OptionValue "RSQGR_WhipAims"
{
    0, "Free"
    1, "Snap To Arc"
    2, "Locked Target"
}
// front page, immediately after MENUDEF.txt:75  Option "Whip", "rsqgr_whip", "OnOff"
    Option "Whip Aim", "rsqgr_whip_aim", "RSQGR_WhipAims"
// Tuning and Debug, after MENUDEF.txt:120  Option "Whip Snag", ...
    Slider "Whip Arc", "rsqgr_whip_arc", 0, 60, 4, 0
    Slider "Whip Bend", "rsqgr_whip_bend", 0, 40, 2, 0
    Slider "Whip Lock Hold", "rsqgr_whip_lock_hold", 4, 24, 2, 0
```

The front page earns one row because it is a real choice with a name, which is the rule stated at the
top of `MENUDEF.txt:1-4`; the three numbers go to Tuning, where every other number lives.

### 4.1 Mode 0 — Free

**What it is.** The chord between the last two hand samples is traced (§3.2) and whatever it crosses
is hit. Closest to `whip.zs:118` today, minus the stopwatch.

**What it feels like.** Honest and physical. The lash goes where you threw it and the world does not
help you.

**Where it fails, stated plainly.** The sampling rate. The poses are written per rendered frame
(`gl_openvr.cpp:2549-2552`) but **read once per tic**, so a fast lash is three or four samples of a
motion the player experienced as continuous (`00_CAPABILITIES` §1.4). The swept chord recovers the
*geometry* of what was crossed; it cannot recover the player's *intent* between samples. The result
is misses that feel unjust — the hardest failure mode to forgive, because the player knows what they
meant. **This is the hardest of the three to enjoy and should not be the default.** It is the right
mode for the player who wants the raw thing, and it is the correct baseline for every rig measurement
in §6 because it has no assist to confound a reading.

**Label: EXISTS (modulo §3.2) / S.**

### 4.2 Mode 1 — Snap within arc — **RECOMMENDED DEFAULT**

**What it is.** On release, search for valid targets inside a cone about the release direction and
bend the lash's tip toward the best one. Aim roughly well and connect; aim badly and still miss.

**The search — reuse the mod's furniture, do not write a second picker.**

`monster.zs` already knows how to decide whether a monster is a legitimate whip/yank subject:

```
monster.zs:199-208   Throwable(a, by): rejects bDONTTHRUST, bBOSS, over rsqgr_yank_mass,
                     wider than rsqgr_yank_radius (64), taller than rsqgr_yank_height (110)
hook.zs:396-407      RSQGR_Tracer.TraceCallback: rejects the shooter, players, the mod's own
                     rope actors, missiles, non-shootables, anything at Health <= 0
monster.zs:513       BlockThingsIterator.Create(a, 8) -- the iterator idiom, already in use here
```

So the picker is: one `BlockThingsIterator` over `reach`, filtered by the **existing** `Throwable`
gate and the **existing** `TraceCallback` predicate, then a line-of-sight `Trace` per surviving
candidate through the **existing** `RSQGR_Tracer`. Three reuses, no new policy, and a Champions
champion or a LegenDoom legendary is classified by the same code that already classifies it for the
yank — which is the whole reason to reuse it rather than write a second rule that drifts.

**How "best" is chosen, and why.**

> **Most aligned, tie-broken by nearest. Not lowest health.**

- **Most aligned** (largest `dir dot toTarget.Unit()`) is the right primary because it is the only
  criterion that is a function of **what the player did**. The player aimed; the assist's job is to
  honour the aim, not to second-guess it. An assist keyed on anything else takes authorship away and
  is felt immediately as the game choosing for you.
- **Nearest as the tie-break**, because within a few degrees of each other the near one is the one
  the player was looking at, and because a whip's reach falls off — a far target at full stretch is
  a weaker, less satisfying connection.
- **Explicitly not lowest health.** It is tempting (it finishes things, it reads as "smart") and it
  is wrong: it makes the whip jump to a dying enemy while the player swings at the healthy one in
  front of them. That is the single most-complained-about behaviour in every game that has shipped
  it. Reject it.
- **Hard gate before any of that:** line of sight. A target behind a wall is not a candidate however
  well aligned, which is one `RSQGR_Tracer` call per candidate and is why the iterator's
  wall-blindness is not a problem here.

**Tuning the cone and the bend.**

- **`rsqgr_whip_arc` half-angle 28° default.** Reasoned from the sampling, not guessed: at 35 Hz a
  committed lash turns the wrist roughly 40-60° in a tic, so the player's own uncertainty about where
  the lash pointed is of that order. A cone narrower than ~20° is inside the noise and the assist
  will seem not to work; wider than ~35° and it starts hitting things the player did not aim at. 28°
  sits in the middle with headroom on the slider both ways.
- **`rsqgr_whip_bend` max 18° default, and it is the one that matters.** The bend is what the player
  *sees*. Keep it well under the arc: the cone decides **whether** to help, the bend decides **how
  much it may show**. A target 28° off gets bent toward by at most 18° and is therefore still missed
  — which is correct, and is what keeps the assist honest. Only targets within ~18° are actually
  reachable; the remaining 10° of cone exists so the search does not have a hard visible edge.
- **Scale the bend by charge.** `bend * k`. A wound whip is authoritative and finds its mark; a cold
  flop does not get a free hit. This also means the assist cannot mask a failure to wind up, so the
  gauge in §2 keeps its teaching job.
- **Route the arc through accessibility.** `RSQGR_A11y.Wide(tol)` (`a11y.zs:47`) multiplies a
  tolerance by ×1..×2 under `rsqa_gesture_ease` (`a11y.zs:46`), and `core.zs:566` already applies it
  to `rsqgr_grab_reach`. Pass `rsqgr_whip_arc` through `Wide()` and a player who needs more help gets
  it from the setting that exists for exactly that, without a second knob and without the mod
  hard-linking to anything (`a11y.zs:27`'s guard makes it inert when `RSQ_Accessibility` is absent).

**What it feels like.** Like a whip. You commit to a direction, the tip finds the body, you hear the
crack. This is the mode that makes the feature.

**Where it fails.** Crowds — most-aligned in a pack of five is close to arbitrary from the player's
point of view, and no tie-break fixes that honestly. Mitigation: shrink the effective arc as the
candidate count rises (a soft `arc / sqrt(n)`), so a pack forces a more precise throw. That is a
tuning line, not a redesign, and it should be tried before anyone reaches for a lock-on to solve it.

**Cost: one blockmap walk plus up to a handful of LoS traces, once per release** — not per tic. The
iterator at `monster.zs:513` is the same order and runs inside a budget already measured at
0.036 ms mean for a thrown Baron (`HANDOFF.md`).

**Label: BUILDABLE / M.**

### 4.3 Mode 2 — Explicit lock

**What confirms the lock, and the bind question answered.**

The mod has four binds today (`KEYCONF.txt`): `+rsqgr_main`, `+rsqgr_off`, `+rsqgr_grip_main`,
`+rsqgr_grip_off`. There is room, and three options were considered:

1. **Reuse the grab bind while a whip is armed.** Tempting and free — there is no rope to climb while
   a whip is armed, so `grabHeld` is otherwise idle. **Rejected.** `rsqgr_grab_dg` defaults true and
   makes `RSQ_DistanceGrab`'s own grips call `core.Grab` (`input.zs:149-150`), so a DG grip press
   would silently take a whip lock. A cross-mod surprise is exactly what the series rule exists to
   prevent.
2. **A fifth bind, `+rsqgr_whip_lock`.** One `addmenukey` row and one alias pair, in the file's
   existing style. `KEYCONF.txt:9-12` already notes the owner binds the stick clicks himself and the
   mod ships **no default controller binds**, so a fifth key costs him one binding and nobody else
   anything. **Recommended.**
3. Hold-direction-for-N-tics with no button. Rejected: it fires while the player is just looking
   around, and it is unteachable.

```
// KEYCONF.txt, with the four existing rows
addmenukey "Lock Whip Target" "+rsqgr_whip_lock"
alias "+rsqgr_whip_lock" "event rsqgr_wlock 1"
alias "-rsqgr_whip_lock" "event rsqgr_wlock 0"
```

The event path is already laid: a local `event` reaches `ConsoleProcess` on the machine that owns the
controller, the pose is read there and packed into a network event
(`input.zs:1-20`, `:28-72`, `:108-151`). The lock's candidate is picked by §4.2's picker along the
**current** hand direction, resolved locally, and the chosen actor is confirmed over the net. Follow
that pattern; it is the only multiplayer-correct one in the mod.

**What shows the player they are locked.** Three, in order of cost:

- **A sound at the target**, through the existing one-shot carrier: `Sound2` spawns an
  `RSQGR_WhipTip` at a point and plays a sound on it (`whip.zs:158-164`), and `RSQGR_Looks.Sound`
  resolves the name per look set (`looks.zs:37-63`). One new name, three sets.
- **A per-hand rumble — already in the API, and already named for this.** `RSQGR_A11y.Ready(who,
  hand)` is commented *"a lock / grab-ready cue on a hand (0 main, 1 off): a short rumble when Sound
  Cues has it"* (`a11y.zs:92-98`). It is literally the lock cue, written before anyone asked for a
  lock. Call it and nothing else.
- **A marker on the target** when `RSQGR_A11y.CueVisual()` (`a11y.zs:64`), sized by
  `RSQGR_A11y.Cue()` (`a11y.zs:57`). Optional, off without `RSQ_Accessibility`.

**What breaks the lock.** All of these, and the first four are free because the mod already has the
tests:

| Breaker | Where the test already is |
|---|---|
| target dead, destroyed, or turned unshootable | `core.zs:332`'s live re-check idiom (added after a measured Champions "Dark Red" bug — see `HANDOFF.md`) |
| target out of `reach` | `core.zs:337`'s range guard idiom |
| line of sight lost | `core.zs:640-649` `Sees()`, every 4 tics with 3 strikes |
| target becomes an illegitimate subject mid-lock | `monster.zs:199-208` `Throwable`, re-asked |
| the arm swings more than ~70° away from the locked bearing | new, and **necessary**: without it the lock survives the player deliberately turning away and the whip curves round like a homing missile |
| release, or disarm | — |
| a save/load, and the first tic after it | `core.zs:162-170` `Settle()` — and this is mandatory, see below |

**`Settle()` is not optional.** `core.zs:154-170` resets per-hand latches on the first tic the handler
object has not seen, *not* in `WorldLoaded`, because a saved handler's fields are read back **after**
`WorldLoaded` — the comment records the bug ("a hold reset there came back from the save and kept
climbing"). A lock is precisely that kind of latch, and an `Actor` pointer besides. Clear it in
`Settle()`.

**What it feels like, honestly.** Like a gun. You acquire, you confirm, you fire, it hits. The
intermediate skill — *throwing* at a thing — is removed, and with it most of what makes a whip a
whip. **I recommend against it as the default** and recommend shipping it anyway, for two reasons:
the owner asked for it, and it is the only one of the three that works for a player who cannot aim
precisely, which is a real audience the series already caters to (`a11y.zs:46-48`).

**Where it fails, and the L-shaped risk.** Target churn. A locked actor can be replaced mid-lock by
another mod — Champions and LegenDoom Lite both do this, and `HANDOFF.md`'s `agent-compat` section
records two separate measured instances of a monster mutating under the hook (mass 1200 mid-pull; an
actor turning unshootable and being parked in a map corner). Holding an `Actor` reference across a
wind-up is the same exposure with a longer window. Every breaker above must be re-asked **every
tic**, not at release — that is the lesson `core.zs:327-332`'s comment block already teaches, and
ignoring it is how this mode becomes the one that eats a week.

**Label: BUILDABLE / M** for the mechanism; **the compat hardening is the L-shaped part** and should
be budgeted separately.

### 4.4 Summary

| Mode | Label | Cost | Default? | One-line verdict |
|---|---|---|---|---|
| 0 Free | EXISTS (with §3.2) | S | no | Honest; hardest to enjoy; the correct measurement baseline |
| **1 Snap To Arc** | BUILDABLE | M | **YES** | The mode that makes the feature work |
| 2 Locked Target | BUILDABLE | M (+L hardening) | no | Reliable, readable, most likely to feel like a gun |

### 4.5 **`rsqgr_reset` must be updated — flagged explicitly**

`KEYCONF.txt`'s `rsqgr_reset` alias is one line listing **every** cvar with its default, and today it
lists all six whip cvars (`rsqgr_whip 0; rsqgr_whip_reach 192; rsqgr_whip_damage 12;
rsqgr_whip_stun 1; rsqgr_whip_snag 1; rsqgr_whip_above 16`). It is what the front page's
`SafeCommand "Reset All", "rsqgr_reset"` (`MENUDEF.txt:124`) invokes.

**Any cvar added by this document and not added to that alias will silently survive a "Reset All"**,
leaving the player with a half-reset whip and no indication why. Eight new cvars:

```
rsqgr_whip_aim 1; rsqgr_whip_arc 28; rsqgr_whip_bend 18; rsqgr_whip_lock_hold 10;
rsqgr_whip_wind 1; rsqgr_whip_wind_full 900; rsqgr_whip_wind_decay 0.4; rsqgr_whip_wind_radius 24
```

`HANDOFF.md` ("For core") already warns that the alias is assembled from both halves' lists and that
both must be regenerated when either changes. **Treat the reset line as part of the feature, not as
documentation.**

Two related notes while in those files:

- `MENUDEF.txt` does **not** in fact list every cvar — `rsqgr_whip_above` is in CVARINFO and in
  `rsqgr_reset` but has no menu row. Not a bug, but it means "every cvar keeps its row"
  (`MENUDEF.txt:1-4`) is already aspirational; do not rely on the menu as the inventory.
- The preset rows (`RSQGR_P_Range` / `_Swing` / `_Pull`, `MENUDEF.txt:38-59`) are generated by
  `tools/rsmenu/presets.py`. If the whip gets presets, they belong in that script, not hand-written
  into MENUDEF.

---

## 5. SERIES FIT

**The aim mode stays in the Grapple's own menu. It does not belong in `RSQ_Accessibility`.**

The line is clean and the mod already draws it correctly. `RSQ_Accessibility` carries settings that
are **about the player's body or senses** and apply to every mod the same way: handedness
(`rsqa_onehand` → `RSQGR_A11y.Ctrl`, `a11y.zs:81-84`, consumed at `util.zs:54`, `:65`, `:74`, `:79`),
gesture tolerance (`rsqa_gesture_ease` → `Wide` / `Low`, `a11y.zs:46-48`), photosensitivity
(`a11y.zs:34-43`), comfort (`a11y.zs:53`), cue and text scale (`a11y.zs:57-61`), sound-cue
substitution (`a11y.zs:64-65`), hold-to-toggle (`a11y.zs:50`).

Three things follow:

1. **`rsqgr_whip_aim` is a gameplay choice about one weapon in one mod.** It has no meaning in the
   saber, the wheel or the shield. It is a Grapple cvar on the Grapple's page, exactly as
   `rsqgr_monster_mode` and `rsqgr_geo_mode` are.
2. **But its *tolerances* do belong to accessibility, and must be routed there** — §4.2: pass
   `rsqgr_whip_arc` through `RSQGR_A11y.Wide()` and any speed or hold threshold through
   `RSQGR_A11y.Low()` (`a11y.zs:48`), exactly as `core.zs:566` already does for
   `rsqgr_grab_reach`. One player's "Snap To Arc" is then wider than another's, from the setting
   that exists for it, with no second knob.
3. **Handedness is already handled and must not be re-solved.** `RSQGR_Util.Ctrl` (`util.zs:49`) and
   `RSQGR_A11y.Ctrl` (`a11y.zs:81-84`) already route a hand's *part* to whichever controller plays it
   under the Off-Hand Modifier. A two-handed wind-up reads both hands, so it is unaffected — but the
   **one-handed fallback** (`rsqgr_whip_wind 0`, §1.6) must read its hand through `Ctrl()` like
   everything else in `util.zs` does, or one-handed play breaks for a player using the modifier.

**Nothing hard-links, and the guard is already written.** `a11y.zs:27` resolves `RSQA_Service` only
while a class named `RSQA_Menu` exists, so a value left in the ini does nothing on its own and the
Grapple still loads and plays alone. Same for `RSQ_SlowMo` (`util.zs:90-100`), `RSQ_Score`
(`service.zs:370-381`), `RSQ_DistanceGrab` (`input.zs:149-150`), BD22Q (by class *name* via
`FindStateByString`, `climb.zs:38-44`) and the Gun Bonsai patch (`perks.zs`). **No new hard link, no
new `#include` of another mod, no new class reference.**

**Service contract: additive only.** `service.zs:1-7` promises every unknown request answers `-1`
(or `null`). Add only:

| New request | objectArg | Answer |
|---|---|---|
| `"whip.charge"` | pawn (intArg = hand) | 0..100, the stored wind-up; -1 no whip armed |
| `"whip.aim"` | pawn | the live `rsqgr_whip_aim` mode |
| `"whip.lock"` (GetObject) | pawn (intArg = hand) | the locked actor, or null |

And **do not touch** `"whipped"` (`service.zs:444`) or `"stunned"` (`:445-454`) — `RSQ_TopHPBars`
reads them, and `"stunned"`'s 35-tic `Pain`-state window depends on `whip.zs:131-136`'s forced pain
surviving unchanged. It does, in everything above.

---

## 6. WHAT THE RIG CAN SETTLE BEFORE ANYONE SPENDS DAYS

The rig exists and is better suited to this than to anything it has been used for yet:
`tools/rigtest/gr.sh <name> <secs> "<console script>" [1 = full load order]`,
`tools/rigtest/build_probe.sh` (probe handler + the generated GRTEST map: ledges, a 768-wide pit,
ceiling 512, a 3D slab, a tagged alcove), and probe events `grp_tp x y ang`,
`grp_summon idx dist angofs`, `grp_dir`, `grp_ceil`, `grp_hurt`, with `grp_log 0..4`
(`HANDOFF.md`, "Test kit"). Two of those are exactly the instruments needed:

- **`grp_dir`** already prints, per hand: `OverrideAttackPosDir`, the hand's yaw/pitch/roll, the mod's
  own computed direction, **the engine's own `AttackDir` for comparison**, and
  **`hand-off(x y z)` — the hand's offset from the pawn** (`tools/rigtest/probe/zscript.txt:23-32`,
  the `T_DIR` line). That offset is the wind-up's entire input.
- **`grp_summon idx dist angofs`** places a monster at a given distance and angular offset — which is
  the targeting cone's test, by construction.

### Test A — does slack-as-gauge read at a glance? (half a day)

The question is purely visual, so it needs frames, not logs.

```
gr.sh gauge 20 "rsqgr_whip 1; rsqgr_whip_aim 0; rsqgr_debug 1;
               rsqgr_whip_wind_decay 999;            // freeze the charge
               grp_tp 512 512 0; grp_log 2"
```

With decay frozen, hold the posed hands at four driven charges and **dump one left-eye frame at each**:
`k = 0.00, 0.33, 0.66, 1.00`. Capture as `test/evidence/frames_gauge.png` (a 4-up, matching the
existing `frames_spin.png` / `frames_flip.png` / `frames_voxel_barrel.png` convention).

**Pass:** a person shown the four frames in random order sorts them correctly, first try, without
being told what to look for. **That is the whole bar** — if it needs explaining it has failed, because
in play there is no explanation available.
**If it fails:** raise `BOW_REST` toward 0.3 before touching `WAVE_AMP`; depth separates better than
wave amplitude at a glance (§2.3). Capture a second 4-up and compare.

Also capture two frames at `rsqgr_whip_reach 192` and `48` at fixed `k = 0.5`, to confirm the gauge
reads at both ends of the reach slider — the bow is proportional to `len`, so a short whip has a
shallower absolute sag and may need `BOW_REST` to be reach-compensated. Cheap to check now, annoying
to discover later.

### Test B — does a wound two-hand release land where a player expects? (one day)

This is the one that decides whether §1 is worth building, and it wants numbers.

```
gr.sh wind 40 "rsqgr_whip 1; rsqgr_whip_aim 0;        // mode 0: NO assist, so the reading is clean
              rsqgr_debug 1; grp_log 3;
              grp_tp 512 512 0;
              grp_summon 0 160 0; grp_summon 0 160 15; grp_summon 0 160 30; grp_summon 0 160 45"
```

Four imps at 160 units and 0° / 15° / 30° / 45° off the throw bearing. Pose a two-hand circle, drive
the wind-up to a known `k`, release along 0°, and from the `T_GRW` lines record: the charge at
release, the resolved `reach`, the resolved `outTics`, which imp (if any) was hit, and the angular
error between the throw bearing and the hit.

Run the sweep at `k = 0.25 / 0.50 / 0.75 / 1.00`, ten releases each.

**Three things it settles, in one afternoon of runs:**

1. **Whether the angular wind-up accumulates at all from a posed hand**, which is the cheapest
   possible falsification of §1.2. If `dphi` does not integrate as predicted from the
   `hand-off(x y z)` values the probe already prints, stop and fix the sampler before designing
   curves. *(Honest caveat: a cvar-posed hand is not a tracked hand. The rig can prove the integrator
   is arithmetically right and can prove it does not charge from a still hand or from a menu close —
   which is `PoseGood`'s shipped bug, §1.5, and is worth a dedicated run. It cannot tell anyone
   whether two revolutions is the right ask. That is a headset question and nothing on a desk will
   answer it.)*
2. **The scatter of the angular error.** This is the number that chooses `rsqgr_whip_arc`: the cone
   should be roughly the 90th percentile of the measured error, not a guess. My 28° default is
   reasoned from the 35 Hz sampling (§4.2); measure it and replace it.
3. **Whether `outTics` 2 reads as a snap.** Dump `frames_wind.png` at `k = 1.00` across the lash's
   whole life — 2 tics out is two frames of straightening, and if the crack is invisible at that
   speed then `OUT_FAST` wants to be 3 and the §3.1 range narrows.

### Test C — the regressions, and they are not optional

Everything in §3.3. One run, the owner's full load order (`gr.sh ... 1`):

```
gr.sh whipreg 60 "rsqgr_whip 1; rsqgr_whip_snag 1; rsqgr_debug 1; grp_log 2;
                 grp_tp 1600 512 180; grp_ceil 0"      // the tagged alcove, for the lintel snag
```

Four assertions, all against behaviour `HANDOFF.md` already measured, so the comparison is exact:

| Assertion | Baseline to match |
|---|---|
| a lintel snag still reaches `core.Latch()` and swings | `HANDOFF.md`: "tip on the alcove lintel 91 u above the hand -> core's own Latch()" |
| `"whipped"` still reads 0 at the kill | `HANDOFF.md` (agent-svc): "whip kill `whipped` 0" |
| save → load ×2 mid-wind and mid-lash loses no state and clears the lock | the existing save/load discipline, `core.zs:162-170` |
| no new script errors beyond `BD_VoxelPack`'s 4 `voxeldef` lines | every round's baseline in `HANDOFF.md` |

**Do Test A first.** It is half a day, it is the cheapest, and it tests the premise the owner actually
supplied — that the droop is worth keeping. If the gauge does not read, §§1 and 3 still stand on their
own merits but §2 becomes an ordinary visual polish item and the ordering of the whole plan changes.

---

## 7. WHERE THE MOD IS ALREADY RIGHT (whip-specific)

`01_GRAPPLE_TETHER.md` §9 lists fourteen for the mod as a whole. These five are the ones this
document could plausibly have broken:

1. **The sag itself** (`whip.zs:213`, `:222`, `:223`). The owner is right and the first draft of
   `01` was wrong. It is the only thing in the file with weight, and it is now the gauge.
2. **`whip.zs:166-196` `Snag()` → `core.Latch()`.** A crack that resolves to geometry builds core's
   hook exactly as a claw that hit that spot would have, so swing, reel and release are core's
   unchanged code. The whip-to-rope handover is the tether thesis already implemented, and it is
   measured working. **Do not touch it.**
3. **`whip.zs:97-98` re-reads the hand every tic so the lash follows the wrist** ("a whip goes where
   your wrist sends it"). The wind-up adds a *magnitude*; this line already supplies the direction,
   correctly, through `PoseGood`'s gate.
4. **`whip.zs:118` passes `TRACE_HitSky` in the `traceFlags` slot** — correct, it is a real
   `ETraceFlags` value `0x0040` (`qzd173/wadsrc/static/zscript/doombase.zs:256`,
   `qzd173/src/playsim/p_trace.h:102`), not a confusion with `ETraceResult`.
5. **`whip.zs:129`'s call order** — `RSQGR_Events.Whipped` before `RSQGR_Perks.Hurt`, which is what
   makes the service answers safe for a scorer reading `WorldThingDied` (`service.zs:54-58`).

And one that is right in a way worth naming separately: **`whip.zs:1-22`'s header comment**. It
states the design, the tic timeline and the no-edit-to-core constraint in plain words before any
code. Every number this document changes is named there. Rewrite that comment with the feature;
leaving it describing a stopwatch that no longer exists would cost the next session a day.

---

## 8. OPEN

1. **Two revolutions for full charge.** Reasoned from "a second or two of deliberate winding", not
   measured. The one number that most wants a slider before it wants a default. Test B cannot settle
   it (§6).
2. **`rsqgr_whip_arc` 28°.** Derived from the 35 Hz sampling rate; Test B measures the real scatter
   and should replace it.
3. **Whether crowds need the `arc / sqrt(n)` narrowing** (§4.2) or whether most-aligned is good
   enough. Unknown until someone whips into a pack.
4. **Whether a cold release should be *possible* at all**, or whether below some `k` the bind should
   simply fire the claw instead. The latter is tidier (no useless action) and riskier (a bind that
   does two different things depending on an invisible threshold). I lean to allowing the cold flop:
   it is how the player learns that winding matters.
5. **Whether `rsqgr_whip_aim` should have a fourth value, "Snap, but only while wound"** — assist
   that itself has to be earned. Attractive, and one `&&`. Out of scope until modes 0-2 are measured.
6. **Could not find** any existing arc-search or target-picker helper in the mod to build mode 1 on
   beyond `Throwable` (`monster.zs:199-208`) and the `TraceCallback` predicate (`hook.zs:396-407`).
   The cone search itself is new code, not a wrapper — a reuse of policy, not of implementation.
