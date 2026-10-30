# 00 — Capability map

**Engine:** tag `qzd17.3` of `github.com/emawind84/gzdoom` — emawind84's QuestZDoom PC edition.
All citations are repo-relative paths at that tag.

**What this file is:** the shared foundation every topic report builds on. If you only read one
file before designing a mechanic, read this one. If a topic report contradicts it, the topic
report has to say so out loud and show its citation.

**What this engine is, in one line:** GZDoom with a real two-handed VR layer bolted into the
playsim — a second weapon, a second psprite layer, a second set of attack buttons, and exactly
one extra hand pose exposed to script.

---

## 0. Read this before you build anything — which VR backend fills the hand poses

Everything in this survey that reads a hand depends on four C++ globals: `weaponangles`,
`offhandangles`, `weaponoffset`, `offhandoffset`. In this source tree they are written in
**exactly one file**:

```
src/rendering/gl/stereo3d/gl_openvr.cpp:1331-1333   weaponangles
src/rendering/gl/stereo3d/gl_openvr.cpp:1341-1343   offhandangles
src/rendering/gl/stereo3d/gl_openvr.cpp:1733-1740   weaponoffset   (from dominantControllerPose)
src/rendering/gl/stereo3d/gl_openvr.cpp:1763-1770   offhandoffset  (from offhandControllerPose)
```

That is the **OpenVR / SteamVR** path. Three facts follow, all verified by grep over the whole tree:

1. **There is no OpenXR input code anywhere in this tree.** Zero hits for `xrSyncActions`,
   `xrLocateSpace`, `xrGetActionStatePose`, `xrGetActionStateFloat`.
   `src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp` only *reads* those globals
   (`extern` at `:116`, consumed at `:452-458` and `:586-611`) — it never fills them.
   They are defined, unwritten, in `src/QzDoom/qzdoom_common.cpp:10`.
2. **`ENABLE_OPENVR` defaults to `NO`** — `src/CMakeLists.txt:418`.
3. **`VR_HapticEvent` is an empty function body** — `src/QzDoom/qzdoom_common.cpp:65-67`.
   Haptics are not merely unexposed to ZScript; on this path they do nothing at all.

**What this means for a modder:** on PC, hand tracking reaches script only in a build configured
with `ENABLE_OPENVR=YES` and running through SteamVR/OpenVR. If the shipped binary your team
targets runs on an OpenXR runtime instead (Virtual Desktop's VDXR, Oculus's own OpenXR runtime,
WMR), then on the evidence in *this* tree the hand-pose fields are never populated and every
hand-driven design in this survey reads zeros.

**This is unresolved and it is a go/no-go.** It is not a reason to discard the survey — the
engine capabilities documented here are real, and on the Quest build the Android/QzDoom native
layer supplies these same globals. But before a team commits a week to a hand-driven mechanic,
somebody must confirm which backend the target build actually uses. The test is cheap: print
`AttackPos` and `OffhandPos` from ZScript in the real build and see whether they move and
whether they differ from each other.

---

## 1. VR data exposed to script

### 1.1 The complete list

There is nothing else. This is the whole surface.

| ZScript | Declared | Native | Thunk |
|---|---|---|---|
| `Actor.HmdPosition` — vector3, readonly | `wadsrc/static/zscript/actors/actor.zs:277` | `src/playsim/actor.h:1668` | `src/scripting/vmthunks_actors.cpp:2195` |
| `Actor.OffhandPos` — vector3, readonly | `actor.zs:272` | `actor.h:1661` | `vmthunks_actors.cpp:2191` |
| `Actor.OffhandPitch` — double, readonly | `actor.zs:273` | `actor.h:1662` | `vmthunks_actors.cpp:2192` |
| `Actor.OffhandRoll` — double, readonly | `actor.zs:274` | `actor.h:1664` | `vmthunks_actors.cpp:2193` |
| `Actor.OffhandAngle` — double, readonly | `actor.zs:275` | `actor.h:1663` | `vmthunks_actors.cpp:2194` |
| `Actor.OffhandDir(actor, angle, pitch)` — vector3 | `actor.zs:859` | `actor.h:1666` (function pointer) | — |
| `Actor.AttackPos` — vector3, readonly | `actor.zs:268` | `actor.h:1654` | `vmthunks_actors.cpp:2187` |
| `Actor.AttackPitch` — double, readonly | `actor.zs:269` | `actor.h:1655` | `vmthunks_actors.cpp:2188` |
| `Actor.AttackRoll` — double, readonly | `actor.zs:270` | `actor.h:1657` | `vmthunks_actors.cpp:2189` |
| `Actor.AttackAngle` — double, readonly | `actor.zs:271` | `actor.h:1656` | `vmthunks_actors.cpp:2190` |
| `Actor.OverrideAttackPosDir` — bool, readonly | `actor.zs:276` | `actor.h:1652` | `vmthunks_actors.cpp:2186` |
| `PlayerInfo.PlayInVR` — bool, readonly | `wadsrc/static/zscript/actors/player/player.zs:3136` | `src/playsim/d_player.h:502` | — |

### 1.2 Both hands are fully tracked, plus the head

> **Correction, 2026-10-30.** The first draft of this file said there was no main-hand pose.
> That was wrong. Three topic surveys independently caught it and it was then verified by hand.
> The error came from reading `hw_vrmodes.cpp:442-444` — the **non-VR fallback path**, where
> `AttackPos` is simply assigned the same value as `OffhandPos` and the player position — and
> mistaking it for the live VR path. If you read an early copy of this survey, re-read this
> section; it is the most load-bearing fact in the document.

The live VR paths write a real, independent weapon-hand transform from the controller,
exactly symmetric with the offhand:

```
hw_openxrdevice.cpp:586-597   AttackPitch/Angle/Roll/Pos  from weaponangles[] / weaponoffset[]
hw_openxrdevice.cpp:600-611   OffhandPitch/Angle/Roll/Pos from offhandangles[] / offhandoffset[]
gl_openvr.cpp:2525-2537       the OpenVR path, same shape
```
(full paths: `src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp`,
`src/rendering/gl/stereo3d/gl_openvr.cpp`)

So you have:

- **Main hand** — `AttackPos`, `AttackAngle`, `AttackPitch`, `AttackRoll`, plus an `AttackDir`
  basis helper (`actor.h:1659`). Full 6DOF.
- **Offhand** — `OffhandPos`, `OffhandAngle`, `OffhandPitch`, `OffhandRoll`, plus `OffhandDir`
  (`actor.zs:859`). Full 6DOF.
- **Head** — `HmdPosition`. Your reference frame for everything body-relative: guard height,
  duck, lean, holster slots, body shots vs head shots.

**Two independently tracked hands and a tracked head.** Boxing with both fists, two-hand
handlebars, a bow drawn between two hands, hand-over-hand climbing, sword-and-board — none of
these are blocked by the input layer.

**Gate every hand-reading path on `OverrideAttackPosDir`** (`actor.zs:276`). It is set at
`hw_vrmodes.cpp:438` to `!puristmode && (IsVR() || vr_override_weap_pos)`, and when it is false
you are not in VR and both hand poses collapse to the viewpoint. The engine branches on it
itself throughout `src/playsim/p_map.cpp` (`:4597`, `:4647`, `:4784`, `:4888`, `:5185`, `:5219`,
`:5649`) and `src/playsim/p_actionfunctions.cpp:1302`.

Scale: `vr_vunits_per_meter` is 34 (`hw_vrmodes.cpp:94`) — use it to convert hand separations
and swing speeds into real-world units.

### 1.3 There are no velocity fields

Nothing above is a velocity. No hand speed, no head speed, no acceleration. Every punch, swing,
flick, draw and throw in this survey is built the same way:

> store `OffhandPos` (and/or `HmdPosition`) each tic in a small ring buffer; difference it.

Four samples is a generous history at 35 Hz. Budget for that everywhere.

### 1.4 Update rate: written per frame, read per tic

The poses are written from the **renderer**, not the playsim:

- `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:442-445`
- `src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:600-603` (OpenXR path)
- `src/rendering/gl/stereo3d/gl_openvr.cpp:2549-2552` (OpenVR path)

Two consequences that pull against each other:

1. **The value you read is fresh.** It is the most recent rendered frame's pose, not a stale
   35 Hz sample. Thresholds behave better than you would expect.
2. **You still only sample at 35 Hz.** Motion between tics is gone for good. A fast punch is two
   to four samples; a fast sword swing may put the blade on the far side of a target without ever
   sampling inside it.

**The standard remedy, used all over this survey:** don't test the hand's *position*, test the
*chord* between the last two samples — trace along it. Any mechanic that tests only "is the hand
inside the target right now" will miss fast motion, which is exactly the motion players use.

### 1.5 Buttons

`player.cmd.buttons` is readable from script — `player.zs:3048` (`native readonly uint buttons`),
with `oldbuttons` at `player.zs:3064` for edge detection.

The engine adds real VR buttons to the standard set (`wadsrc/static/zscript/constants.zs`):

| Constant | Bit | Line |
|---|---|---|
| `BT_OFFHANDATTACK` | 1<<26 | `constants.zs:866` |
| `BT_OFFHANDALTATTACK` | 1<<27 | `constants.zs:867` |
| `BT_OFFHANDRELOAD` | 1<<28 | `constants.zs:868` |
| `BT_MAINHANDRELOAD` | 1<<20 | `constants.zs:869` |
| `BT_USER1` … `BT_USER4` | 1<<21 … 1<<24 | `constants.zs:859-862` |

Note `BT_SHOWSCORES` was set to 0 and its bit reused for `BT_MAINHANDRELOAD` (`constants.zs:857`).

**So you have per-hand attack, alt-attack and reload, plus four free user buttons.** That is a
lot of input, and several mechanics in this survey that look like they need gesture recognition
are better served by a button plus a pose check.

Per-hand button dispatch already exists in the player code: `CheckWeaponButtons(hand)` at
`player.zs:588` and `:2031`, selecting `BT_OFFHANDATTACK`/`BT_OFFHANDALTATTACK` at
`player.zs:473-474`.

### 1.6 Haptics — present in C++, not reachable from ZScript

- `VRMode::Vibrate(duration, channel, intensity)` — `src/common/rendering/hwrenderer/data/hw_vrmodes.h:94`
- `VR_HapticEvent(event, position, intensity, angle, yHeight)` — `hw_vrmodes.h:97`
- Intensity cvars `ext_haptic_level_*` — `src/playsim/p_actionfunctions.cpp:92-101`
  (global, damage_projectile, pickup_weapon, pickup, fire_weapon, poison, healstation,
  heartbeat, rumble)

**Zero hits for haptic / rumble / vibrate anywhere in `wadsrc/static/zscript`.**

Treat "fire a custom rumble from script" as **unproven**. The cvar list shows the engine fires
haptics on its own hardcoded events (taking projectile damage, picking something up, firing a
weapon). A mod that wants rumble on a punch landing has to provoke one of those events, which
is a HACK, not a clean capability. Topic reports that depend on haptic feedback say so.

### 1.7 VR cvars a mod can read

From `src/QzDoom/VrCommon.h:16-27`: `vr_control_scheme`, `vr_move_use_offhand`,
`vr_weaponRotate`, `vr_snapTurn`, `vr_ipd`, `vr_weaponScale`, `vr_teleport`, `vr_switch_sticks`,
`vr_secondary_button_mappings`, `vr_two_handed_weapons`, `vr_crouch_use_button`.

`vr_two_handed_weapons` and `vr_move_use_offhand` matter most — they are engine-level toggles
that change what the player's hands are already doing, so a mod should read them rather than
assume.

---

## 2. The sprite layer and hand-attached models

### 2.1 Two weapons, one per hand — this is shipped, not a hack

- `PlayerInfo.OffhandWeapon` — `src/playsim/d_player.h:384`
- Weapon flag `bOffhandWeapon` routes a weapon to that hand — `player.zs:1977-1998`
- Weapon flag `bTwoHanded` forcibly clears the offhand weapon — `player.zs:1989-1992`
- Ready / fire / down state selection is keyed on `hand` throughout:
  `player.zs:398`, `:431`, `:466`, `:2092`, `:2429`
- Morphing preserves it — `PremorphWeaponOffhand`, `d_player.h:410`

**Two independent weapon state machines, one per hand, each with its own psprite layer.**
This is the single most valuable feature in the engine for this project, and most topic reports
lean on it somewhere.

### 2.2 Psprite layers

From `wadsrc/static/zscript/constants.zs:798-801` (mirrored at `src/playsim/p_pspr.h:56-59`):

| Layer | Value |
|---|---|
| `PSP_WEAPON` | 1 |
| `PSP_FLASH` | 1000 |
| `PSP_OFFHANDWEAPON` | 1000000 |
| `PSP_TARGETCENTER` | `int.max - 2` |

Layers are integers, so you may create your own at any free index — the named ones are
conventions, not a fixed set. `GetPSprite(id)` / `SetPsprite(id, state)` take arbitrary ids.
The gap between 1 and 1000, and between 1000 and 1000000, is yours.

That gives each hand its own stack of independently positioned, independently animated overlay
layers. Per-layer position is script-writable each tic via the `PSprite` object.

### 2.3 Models

MODELDEF attaches models to weapon states as normal. The per-frame hand pose means a model
drawn on the offhand follows the real controller without any script work. Driving a model's
position or rotation *per tic from script* is where it gets interesting, and the topic reports
that need it (vehicles, mechs, shields) each work out their own route.

---

## 3. Tracing and collision

### 3.1 `LineTrace` is the workhorse of this entire survey

```
wadsrc/static/zscript/actors/actor.zs:805
native bool LineTrace(double angle, double distance, double pitch, int flags = 0,
                      double offsetz = 0., double offsetforward = 0., double offsetside = 0.,
                      out FLineTraceData data = null);
```

`FLineTraceData` — `actor.zs:36-61`:

| Field | Use |
|---|---|
| `HitActor` | what you hit, if an actor |
| `HitLine` | the wall line — **this is how you get a wall normal** (line delta rotated 90°, sided by `LineSide`) |
| `HitSector` | floor/ceiling sector |
| `Hit3DFloor` | 3D floor hit |
| `HitTexture` | the texture — material-specific reactions |
| `HitLocation` | the exact point |
| `HitDir` | the trace direction |
| `Distance` | how far |
| `LineSide` | which side of the line |
| `LinePart` | top / middle / bottom |
| `SectorPlane` | which plane |
| `HitType` | one of `TRACE_HitNone / HitFloor / HitCeiling / HitWall / HitActor / CrossingPortal / HasHitSky` (`actor.zs:38-47`) |

**`HitLine` plus `LineSide` is the single most important detail here.** It means a mod can
recover a wall's facing at runtime, which is what makes wall-running, rope anchoring, shield
alignment and surface-aware impacts possible *without touching a map*. Several topic reports
turn on this.

### 3.2 Collision is cylinders

Actors are a `radius` and a `height`; there is no mesh collision. A flat shield, a car body,
a sword blade — none of them collide as their shape. Every mechanic in this survey that wants
a non-cylindrical collision volume either approximates it with several actors, fakes it with
traces, or accepts the cylinder. The shields report works this through in the most detail.

---

## 4. Time

- `Actor.isFrozen()` — `wadsrc/static/zscript/actors/actor.zs:513`
- `PlayerInfo.timefreezer` — `player.zs:3079`, an int bitmask, not a boolean
- `PowerTimeFreezer` — `wadsrc/static/zscript/actors/inventory/powerups.zs:1502`, setting and
  clearing `timefreezer` bits at `:1529`, `:1537`, `:1604`
- Level clocks: `Level.time`, `Level.maptime`, `Level.totaltime`, `Level.starttime` —
  `wadsrc/static/zscript/doombase.zs:425-428`, all readonly

`timefreezer` is a **mask**, not a flag — the freeze is already expressed per-player in bits.

**`10_TIME.md` settled this topic, and the answer is better than expected.** Headlines, so other
reports can rely on them:

- **`Actor.freezetics`** (`actor.zs:265`, native `actor.h:1284`) is writable and checked at the
  *top* of `AActor::Tick` (`p_mobj.cpp:3948-3952`), before states, velocity, inventory and AI.
  Driven as a duty cycle (freeze N tics, run 1) it slows an actor by exactly 1/(N+1) with
  animation, velocity and AI **inherently in sync** — you never touch `tics` or `Vel`, so the
  classic desync trap never arises. A frozen actor **stays in the blockmap**, so it remains
  shootable and collidable.
- **Global freeze with exceptions is O(1) and already shipped:** `Level.SetFrozen()`
  (`doombase.zs:511`) plus the writable `player.timefreezer` bitmask (exemption at
  `actorinlines.h:186-190`) plus per-actor `+NOTIMEFREEZE` (`actor.h:315`). A mod does not need
  `PowerTimeFreezer` at all.
- **A global time scale exists but is a WALL for mods:** `i_timescale` (`d_main.cpp:217-234`,
  applied at `i_time.cpp:67-72`) cannot be set from script — `CVar.SetFloat` throws outside menu
  code (`vmnatives.cpp:981-990`) and there is no script `ConsoleCommand` in `src/`. Per-actor
  scaling gets you there anyway, quantised to 1/2, 1/3, 1/4…
- **Freezing the playsim does not stutter the VR view.** Head orientation is re-derived from the
  live pose every rendered frame and written into the viewpoint *after* interpolation
  (`hw_openxrdevice.cpp:736-751`). Player translation and slowed actors stutter; your head does not.

### 4.1 Where to inject per-tic logic

`WorldTick` runs after all player input processing and before `P_XYMovement` / `P_ZMovement`
(`src/p_tick.cpp:166-172` → `p_mobj.cpp:4344`, `:4372`), so `Vel` / `bFly` / `bNoGravity` /
`Gravity` writes land the same tic. `TickPSprites` (`player.zs:1796`) is the other clean hook.
Thinkers at `STAT_BOT` (104) tick after all actors (`src/playsim/dthinker.cpp:145-149`) if you
need a controller that runs last.

**Trap:** thinker iterators only visit statnums ≥ 32 (`dthinker.cpp:938-940`), so actors parked
at a low statnum vanish from your own sweep loops.

---

## 5. Camera and view

- `SetCamera(Actor cam, bool revert)` — `actor.zs:941` (action function)
- `PlayerInfo.camera` — `player.zs:3110`
- `Actor.CameraHeight` — `actor.zs:249`
- `SetCameraToTexture(viewpoint, texture, fov)` — `doombase.zs:30`, plus
  `SetCameraTextureAspectRatio` at `:31` — render-to-texture, which is how you build a working
  mirror, a rear-view mirror in a vehicle, or a cockpit display

**The VR caveat:** the HMD supplies view orientation. How much of a script-driven view change
actually survives that is the question the mech and flight reports have to settle, and it is
flagged in both as decisive rather than incidental.

---

## 6. Rendering a mod can use

- **Post-process shaders** are a supported mod feature —
  `src/common/rendering/hwrenderer/postprocessing/hw_postprocessshader.cpp`, with console
  control at `hw_postprocessshader_ccmds.cpp`. Declared from GLDEFS, uniforms settable from
  script. This is the route for bullet-time tinting, damage vignettes, heat haze.
- Dynamic lights, translucency and render styles are stock GZDoom and behave normally.
- `SetCameraToTexture` (above) for in-world screens and mirrors.

---

## 7. Gotchas that bite every topic

These are the ones that silently eat a prototype. Several were found the hard way by topic
surveys; each names the report that proved it.

1. **`MovePlayer` wipes `vel.XY` every grounded tic.** It multiplies by 0.0001 unless
   `player.keepmomentum` is set — `player.zs:1373-1383`; `keepmomentum` is writable
   (`d_player.h:414`) and the engine sets it itself at `actions.zs:78`; cvar `vr_momentum`
   defaults false (`hw_vrmodes.cpp:120`). **Every momentum mechanic in this survey dies without
   this** — slides, dashes, rope swings, vehicles. Set it every tic. (09_MOVEMENT, 05_GRAPPLE)
2. **`A_ChangeVelocity` on the player is a no-op by default.** Gated on `vr_recoil`, default
   false — `p_actionfunctions.cpp:90`, gate `actions.zs:63-79`. Write `Actor.Vel` directly
   (`actor.zs:112`). (05_GRAPPLE, 04_FLIGHT)
3. **Script cannot change view pitch or roll in VR.** `vp.HWAngles.Pitch/Roll` are overwritten
   from the raw HMD after `R_SetupFrame` — `hw_entrypoint.cpp:117` then `:143` →
   `gl_openvr.cpp:1467,1471`; OpenXR `hw_openxrdevice.cpp:740-741`; view matrix built at
   `hw_drawinfo.cpp:443`. Screen shake, camera bob, dash roll and wall-run tilt are **WALLs in
   VR**, not design choices. (04_FLIGHT, 03_MECHS, 09_MOVEMENT)
4. **Room-scale Z is discarded.** Positional tracking is XY-only and explicitly restores Z
   (`gl_openvr.cpp:2613-2632`); head height is a render-only eye offset
   (`hw_openxrdevice.cpp:221`) plus crouchfactor. You can *detect* a duck via `HmdPosition`,
   but the collision volume does not follow. (04_FLIGHT)
5. **Hard speed cap: `MAXMOVE` = 30 units/tic** — `src/playsim/p_local.h:61`. Airborne friction
   is skipped (`p_mobj.cpp:2447-2463`), so air movement is lossless. (05_GRAPPLE, 02_VEHICLES)
6. **No velocities — you difference positions.** (§1.3)
7. **35 Hz sampling of a per-frame value.** Fresh, but sparse. Test chords, not points. (§1.4)
8. **Haptics are not scriptable.** `VR_HapticEvent` is an empty stub at
   `src/QzDoom/qzdoom_common.cpp:64`. Any design whose feel depends on rumble is on thin ice,
   and this is named across several reports as the highest-value engine addition. (§1.6)
9. **Collision is cylinders.** No shape you draw is the shape that blocks. (§3.2)
10. **`bTwoHanded` clears the offhand weapon** (`player.zs:1989-1992`) — a two-handed weapon
    and an offhand gadget cannot coexist without working around this.
11. **`+NOINTERACTION` is a trap** — it forces `MF_NOBLOCKMAP` (`p_mobj.cpp:3988-3992`), taking
    the actor out of the world. To hold something still *and* keep it collidable, use
    `freezetics` (`actor.zs:265`), which early-returns out of `Tick` (`p_mobj.cpp:3948-3952`)
    but stays in the blockmap. (06_GRAB, 10_TIME)
12. **`SetOrigin(pos, false)` and bare `Warp` snap** — they call `ClearInterpolation`
    (`p_maputl.cpp:570`, `p_things.cpp:805`). `SetOrigin(pos, true)` interpolates. `Prev` is
    re-snapshotted for every actor at the top of every tic before any thinking
    (`src/p_tick.cpp:151-162`), so any position write during the tic renders smoothly.
    This decides whether hand-followed objects feel solid or jittery. (06_GRAB, 07_SHIELDS)
13. **This is not REMA.** `E:\DOOMWork\UZDXREMA` has a far larger VR surface. Anything you
    remember from there and cannot find here is a WALL here.

---

## 8. How to read the rest of this survey

`README.md` is the index: a flat table of every mechanic across all ten reports, with its label,
its cost, and the file and section that holds the design. Go there first, then to the section.

Labels: **EXISTS** (already in the engine, cited) · **BUILDABLE** (script, with named APIs) ·
**HACK** (works, but abuses something — the report says what breaks) · **WALL** (needs engine
changes — the report names the missing piece).

Costs: **S** an afternoon · **M** a few days · **L** a week or more.
