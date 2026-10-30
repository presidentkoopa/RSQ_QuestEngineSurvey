# 02 — VEHICLES AND DRIVING

**Engine surveyed:** tag `qzd17.3` of `github.com/emawind84/gzdoom`.
All citations are repo-relative at that tag. Read `_BASELINE_FACTS.md` first.

**Scope rule for this project:** we live on the sprite layer. No map rebuilds, no new
world geometry. The player *is* the vehicle, or rides one.

---

## 0. BOTH HANDS ARE READABLE — the premise this report is built on

The baseline's original §1a ("there is no main-hand pose field") has since been corrected
upstream. I verified the correction independently from source before relying on it, and it
holds. Recorded here because it is the load-bearing fact for the whole topic.

The main hand's pose is exposed to ZScript as four native readonly fields, declared
alongside the offhand ones and registered in the VM in the same block:

| ZScript | Declared | Native field | VM registration |
|---|---|---|---|
| `Actor.AttackPos` (vector3) | `wadsrc/static/zscript/actors/actor.zs:268` | `src/playsim/actor.h:1654` | `src/scripting/vmthunks_actors.cpp:2187` |
| `Actor.AttackPitch` (double) | `actor.zs:269` | `actor.h:1655` | `vmthunks_actors.cpp:2188` |
| `Actor.AttackRoll` (double) | `actor.zs:270` | `actor.h:1657` | `vmthunks_actors.cpp:2189` |
| `Actor.AttackAngle` (double) | `actor.zs:271` | `actor.h:1656` | `vmthunks_actors.cpp:2190` |
| `Actor.OverrideAttackPosDir` (bool) | `actor.zs:276` | `actor.h:1652` | `vmthunks_actors.cpp:2186` |

They are written per rendered frame from the real main-hand controller transform, next to
the offhand writes:

- OpenVR/SteamVR backend: `src/rendering/gl/stereo3d/gl_openvr.cpp:2525-2538` — `AttackPos`
  is taken straight out of `GetWeaponTransform(&mat, VR_MAINHAND)`, i.e. the hand's world
  matrix translation, and `AttackRoll` from `weaponangles[ROLL]`.
- Offhand, identical shape: `gl_openvr.cpp:2541-2554`.

So **both hands are freely readable in world space, with full yaw/pitch/roll each.** The
"one-readable-hand problem" does not exist at this tag. Every two-handed mechanic below —
handlebars, steering wheel, gear lever — is designed on two real hands, not on one hand
plus the aim vector.

`OverrideAttackPosDir` (`actor.zs:276`) is the gate: it is set true only when VR (or
`vr_override_weap_pos`) is active and purist mode is off
(`src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:436`). When it is false, the base
`VRMode::SetUp` collapses both hands onto the player's own position
(`hw_vrmodes.cpp:441-443`: `AttackPos = OffhandPos = player->mo->PosAtZ(shootz)`), so any
hand-difference maths yields exactly zero. **Gate every hand-derived control on
`OverrideAttackPosDir`** and provide a stick fallback, or the mod is dead on flat-screen.

### 0a. A second, worse caveat: which VR backend actually feeds those fields

In this tree, the globals `hmdPosition`, `weaponoffset` and `offhandoffset` are declared in
`src/QzDoom/qzdoom_common.cpp:11-15` and are **written by exactly one place**:
`src/rendering/gl/stereo3d/gl_openvr.cpp:1733-1740` (main hand), `:1763-1770` (offhand),
and `VR_SetHMDPosition` called from `gl_openvr.cpp:2400`. A full-tree grep finds no other
writer.

The OpenXR desktop backend `src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp`
*reads* those globals (`:593-595`, `:606-608`) but nothing in this tree writes them on that
path — the writer lives in the Android/Quest native layer, which is not in this checkout.
I could not find an OpenXR-path writer. Treat it as **unverified** that `HmdPosition` and
`AttackPos`/`OffhandPos` are live when running through OpenXR on PC (VDXR, etc.). Test it
before building on it; see §7.

`Actor.HmdPosition` itself (`actor.zs:277`, `actor.h:1668`, written at
`hw_vrmodes.cpp:445`) is the raw tracking-space HMD position **in metres**, not map units,
and it is *not* rotated into world yaw. Conversion factor is `vr_vunits_per_meter`, used
all through `hw_openxrdevice.cpp:593-608`. `HmdPosition[1]` is the up axis.

---

## 1. Verdict (3-5 sentences)

A vehicle on this engine must be **the player pawn itself**, not an actor the player rides:
`PlayerPawn.MovePlayer` and `PlayerPawn.PlayerThink` are plain ZScript virtuals
(`wadsrc/static/zscript/actors/player/player.zs:1340`, `:1734`), so a mod can replace the
entire locomotion model with car physics in about two hundred lines and no engine change —
whereas riding a separate actor hits two hard walls, an engine that never carries a rider's
XY velocity and a VR layer that stops updating hand and head poses the moment
`player.camera` is not a player (`gl_openvr.cpp:2406`, `hw_openxrdevice.cpp:569`).
Speed is **not** the make-or-break problem it is usually assumed to be: the classic
`MAXMOVE` 30-units/tic clamp is applied only in water, while crouched, or under the
wallrun compat flag (`src/playsim/p_mobj.cpp:2096-2105`), the real ceiling is 5000 units
per tic (`p_mobj.cpp:2107`), and tunnelling is already solved by radius-based substepping
inside `P_XYMovement` (`p_mobj.cpp:2166-2192`). The genuine make-or-break gotcha is
QuestZDoom-specific and nastier: `MovePlayer` multiplies the player's horizontal velocity
by `0.0001` every single tic unless `vr_momentum` or `player.keepmomentum` is set
(`player.zs:1374-1379`), and `A_ChangeVelocity` silently does nothing to a player unless
`vr_recoil` is on, which defaults off (`wadsrc/static/zscript/actors/actions.zs:76`,
`src/playsim/p_actionfunctions.cpp:90`). Two-handed handlebar steering, lean, wheelies,
gear levers, hardpoint guns and per-part damage are all BUILDABLE on top of that; the only
true WALL in the list is a view-locked cockpit model, because a psprite's VR transform is
chosen by which hand owns the weapon and there is no head slot (`src/r_data/models.cpp:222`).

---

## 2. What already exists

**Player locomotion is ZScript, top to bottom.**
- `PlayerPawn.PlayerThink()` — `player.zs:1734`, invoked by name from C++ at
  `src/playsim/p_user.cpp:1356-1360` (`IFVIRTUALPTRNAME(... NAME_PlayerPawn, PlayerThink)`).
  A subclass can replace the whole think.
- `PlayerPawn.HandleMovement()` — `player.zs:1611`, calls `MovePlayer()`, `CheckJump()`,
  `CheckMoveUpDown()`.
- `PlayerPawn.MovePlayer()` — `player.zs:1340`. Reads `player.cmd` (`UserCmd`), applies yaw,
  friction, `TweakSpeeds`, `ForwardThrust`, `Thrust`, `Bob`.
- `PlayerPawn.TweakSpeeds(forward, side)` — `player.zs:1286`, virtual.
- `PlayerPawn.ApplyAirControl(movefactor, bobfactor)` — `player.zs:1325`, virtual.
- `Actor.FallAndSink(grav, oldfloorz)` — `actor.zs:522`, `virtual native`, called through
  `AActor::CallFallAndSink` (`src/playsim/p_mobj.cpp:3042`). Custom gravity curves per
  vehicle without touching `sv_gravity`.

**Per-hand input already reaches script.** The ticcmd button word carries separate bits for
each hand: `BT_ATTACK`, `BT_ALTATTACK`, `BT_RELOAD`, `BT_MAINHANDRELOAD`,
`BT_OFFHANDATTACK`, `BT_OFFHANDALTATTACK`, `BT_OFFHANDRELOAD`
(`src/d_event.h:42-77`), filled in `src/g_game.cpp:775-777`, and read in ZScript as
`player.cmd.buttons` (e.g. `player.zs:473`: `int bt_attack = hand ? BT_OFFHANDATTACK : BT_ATTACK`).
So "left trigger = throttle, right trigger = brake" is already wired; no new input plumbing.

**Analog stick values** arrive as `cmd.forwardmove` / `cmd.sidemove` (`usercmd_t`,
`src/d_protocol.h:64-73`; ZScript fields registered at `src/playsim/p_user.cpp:1952-1953`).
These are the only analog axes script can see.

**Two independent weapons, one per hand**, with their own psprite layers and state machines
— `PSP_OFFHANDWEAPON` / `PSprite.OFFHANDWEAPON` (`player.zs:2939`), `PlayerInfo.OffhandWeapon`
(`src/playsim/d_player.h:384`). This is the Lucky & Wild substrate and it is shipped, not a hack.

**Moving an actor (including a player) along a path is shipped.** `PathFollower`
(`wadsrc/static/zscript/actors/shared/movingcamera.zs:146`) and `ActorMover`
(`:434`) move their `tracer` with `tracer.SetZ()` + `tracer.TryMove()` each tic
(`:465-478`), saving and restoring the tracer's `bNoGravity` / `bSolid` / `bNoBlockmap`
(`:489-507`) and calling `tracer.ClearInterpolation()` so the renderer does not smear the
jump (`:506`). That is a working, in-tree reference implementation of on-rails motion.

**Bridges / standable actors exist** — `+ACTLIKEBRIDGE` (`src/scripting/thingdef_data.cpp:211`)
and `+CANPASS` / `MF2_PASSMOBJ` (`thingdef_data.cpp:158`). Player defaults already include
`+CANPASS` (`player.zs:113`), so a player can stand on a solid actor out of the box.

**Standing on an actor tracks its Z, and only its Z.** `AActor::Tick` calls `P_CheckOnmobj`;
if something is under you it does `SetZ(onmo->Top())` every tic (`p_mobj.cpp:4386-4396`),
sets `MF2_ONMOBJ` and zeroes `Vel.Z` (`p_mobj.cpp:4426-4428`). **Nothing anywhere copies the
carrier's XY velocity to the rider.** Grep for a carry in `p_mobj.cpp` / `p_map.cpp` finds
only the flag writes at `p_mobj.cpp:4373` and `:4427`.

**Landing on an actor is reported to script, once.** `PlayerLandedOnThing(mo, onmo)`
(`p_mobj.cpp:4382`) forwards to the ZScript virtual
`Actor.PlayerLandedMakeGruntSound(Actor onmobj)` (`actor.zs:1440`, dispatched at
`p_mobj.cpp:2916-2920`). That hook hands you the actor you just landed on — the Matrix
"land on the truck" event, free.

**Continuous "what am I standing on" is exposed.** `Actor.TestMobjZ(bool quick)` returns
`bool, Actor` (`actor.zs:818`, thunk `src/scripting/vmthunks_actors.cpp:1168-1176`). Call it
each tic to get the carrier.

**Haptics, honestly.** `VRMode::Vibrate(200, 0/1, level)` fires on *both* controllers
whenever the console player takes damage, and that call is **outside** the `__MOBILE__`
guard, so it runs on PC (`src/playsim/p_interaction.cpp:1441-1444`), with
`level = 0.4 + 0.6 * min(damage,100)/100`. Everything else routes through `VR_HapticEvent`
with hardcoded event names (`p_map.cpp:4769`, `p_mobj.cpp:7556`, `a_weapons.cpp:107`, …) and
nothing in `wadsrc/static/zscript` can call it. Rumble from script is a HACK only: deal
yourself real damage. See §5.

**QuestZDoom already tracks player speed** for its own purposes:
`primaryLevel->max_velocity` and `avg_velocity` updated in `src/p_tick.cpp:186-188`. I found
no ZScript exposure of those two fields.

---

## 3. Building blocks — exact APIs

### Movement and physics
| API | Where | Note |
|---|---|---|
| `Actor.Vel` (vector3, read/write) | `actor.zs` | map units per tic. The one true control. |
| `Actor.Thrust(speed, angle)` | `actor.zs:860` | adds to `Vel` along an angle |
| `Actor.VelFromAngle(speed, angle)` / `Vel3DFromAngle(speed, angle, pitch)` | `actor.zs:856-857` | sets, does not add |
| `Actor.TryMove(vector2 newpos, int dropoff, bool missilecheck, FCheckPosition tm)` | `actor.zs:831` | full collision move, returns false if blocked |
| `Actor.CheckMove(vector2 newpos, int flags, FCheckPosition tm)` | `actor.zs:832` | test without moving |
| `Actor.CheckPosition(Vector2 pos, bool actorsonly, FCheckPosition tm)` | `actor.zs:763` | ditto, fills `tm.stepthing`, `tm.floorz`, `tm.floorterrain` |
| `Actor.SetOrigin(vector3 newpos, bool moving)` | `actor.zs:753` | teleport; `moving=true` keeps interpolation |
| `Actor.SetXYZ(vector3)` / `SetZ(double)` | `actor.zs:754`, `:846` | no collision, no relink |
| `Actor.Warp(dest, xofs, yofs, zofs, angle, flags, …)` | `actor.zs:942` | `WARPF_COPYVELOCITY` = `0x4000`, `WARPF_INTERPOLATE` = `0x10` (`constants.zs:522-538`) |
| `Actor.TestMobjZ(bool quick)` → `bool, Actor` | `actor.zs:818` | who am I standing on |
| `Actor.GetFriction()` → `double, double` | `actor.zs:817` | friction, movefactor |
| `Actor.A_SetSize(radius, height, testpos)` | `actor.zs:1314` | grow the hitbox into a car |
| `Actor.Gravity`, `Mass`, `Speed`, `radius`, `height`, `Friction` | `actor.zs:176`, `:165`, `:113`, property at `:433` | per-actor scalars |
| `Actor.Pitch`, `Actor.Roll` | `actor.zs:110-111` | lean and pitch, see MODELDEF below |
| `A_SetAngle/A_SetPitch/A_SetRoll(x, flags, ptr)` | `actor.zs:1272-1274` | `SPF_INTERPOLATE` keeps the renderer smooth |
| `Actor.ClearInterpolation()` | `actor.zs:734` | after any teleport-grade move |
| `Actor.LinkToWorld` / `UnlinkFromWorld(out LinkContext)` | `actor.zs:841-842` | required around `bSolid`/`bNoBlockmap` changes, see `movingcamera.zs:491-495` |
| flags `bNOGRAVITY`, `bFLOAT`, `bSOLID`, `bNOCLIP`, `bCANPASS`, `bACTLIKEBRIDGE`, `bTHRUACTORS`, `bNOFRICTION`, `bBLOCKEDBYSOLIDACTORS` | `thingdef_data.cpp:127`, `:130`, `:119`, `:129`, `:158`, `:211`, `:166`, `:324`, `:280` | |

**Do not use `A_ChangeVelocity` on the player.** `wadsrc/static/zscript/actors/actions.zs:63`
is QuestZDoom's rewritten version; at `:74-76` it does
`if (player != NULL && player.mo == ref) { if (!vr_recoil) return; … }` and `vr_recoil`
defaults **false** (`src/playsim/p_actionfunctions.cpp:90`). Write `Vel` directly.

### View and camera
| API | Where | Note |
|---|---|---|
| `Actor.SetViewPos(Vector3 offset, int flags = -1)` | `actor.zs:1278` | seat the eyes off the pawn origin |
| `Actor.ViewPos` (readonly `ViewPosition`) | `actor.zs:97`, class at `:70` | null until `SetViewPos` is called once |
| `VPSF_ABSOLUTEOFFSET` `1<<1`, `VPSF_ABSOLUTEPOS` `1<<2`, `VPSF_ALLOWOUTOFBOUNDS` `1<<3` | `constants.zs:431-433` | `ALLOWOUTOFBOUNDS` is hardware-renderer only |
| `PlayerInfo.camera` | `player.zs:3110` | **see the wall in §5** |
| `player.crouchfactor` | `player.zs:3126` | in VR with `vr_crouch_use_button false` this is driven by live HMD height (`hw_openxrdevice.cpp:576-579`, `gl_openvr.cpp:2513-2516`) — a free head-height signal, and a speed trap |

### VR pose (the full list for this topic)
`AttackPos/Pitch/Roll/Angle` (`actor.zs:268-271`), `OffhandPos/Pitch/Roll/Angle`
(`actor.zs:272-275`), `OverrideAttackPosDir` (`:276`), `HmdPosition` (`:277`),
`Actor.OffhandDir(actor, angle, pitch)` (`:859`), `PlayerInfo.PlayInVR` (`player.zs:3136`).
No velocities — difference consecutive tics (baseline §1b). Values are written per rendered
frame, read per tic: fresh, but 35 samples/second.

### Model and psprite presentation
| API | Where |
|---|---|
| `A_Overlay(layer, state, nooverride)` | `actor.zs:1323` |
| `A_OverlayOffset / Scale / Rotate / Pivot / PivotAlign / VertexOffset / Alpha / Flags / RenderStyle / Translation` | `actor.zs:1325-1334` |
| `PSprite` fields `x, y, scale, pivot, rotation, alpha, HAlign, VAlign, Coord0..3` | `player.zs:2954-2972` |
| `A_ChangeModel(modeldef, modelindex, modelpath, model, skinindex, …, flags)` | `actor.zs:1232` |
| MODELDEF flags `USEACTORPITCH` `1<<5`, `USEACTORROLL` `1<<6`, `PITCHFROMMOMENTUM` `1<<1`, `ROTATING` `1<<2`, `USEROTATIONCENTER` `1<<9`, `MODELSAREATTACHMENTS` `1<<12`, `CORRECTPIXELSTRETCH` `1<<13` | `src/r_data/models.h:48-62` |
| HUD model VR transform (hand chosen by weapon ownership) | `src/r_data/models.cpp:208-232` |

---

## 4. Mechanic designs

Each entry: what the player physically does → how script detects it → how it resolves →
feel knobs → label + cost.

### 4.1 Vehicle as player-movement override  — **BUILDABLE, cost M** ← *this is the one*

**Physical:** the player stands still, holds the bars, and the world moves. No separate
vehicle actor.

**Detection:** a custom `PlayerPawn` subclass, or `MovePlayer()` left alone and a
`VehicleThinker` running from `WorldTick`. Preferred shape:

```
class RSVehiclePawn : DoomPlayer
{
    override void MovePlayer()
    {
        if (!driving) { Super.MovePlayer(); return; }
        player.keepmomentum = true;       // MANDATORY, see below
        Angle += steerRate * dt;          // we do the yaw, not cmd.yaw
        Vel.XY = AngleToVector(Angle, speed);
    }
}
```

**Resolution:** `Vel` is consumed by `P_XYMovement` in the pawn's own `Tick`, which gives
you wall sliding (`+SLIDESONWALLS`, already on the player — `player.zs:111`), step-ups,
slopes (`P_CheckSlopeWalk`, `p_mobj.cpp:2160`) and radius-substepped collision for free.

**The three traps, all QuestZDoom-specific:**

1. **The momentum damper.** `player.zs:1374-1379`:
   ```
   if (!vr_momentum && !player.keepmomentum && player.onground && friction == ORIG_FRICTION)
   { vel.XY *= 0.0001; Speed = Default.Speed * 8; }
   ```
   `vr_momentum` defaults **false** (`hw_vrmodes.cpp:120`). So on normal ground, the engine
   deletes 99.99% of your horizontal velocity **every tic**. A vehicle must set
   `player.keepmomentum = true` every tic, or override `MovePlayer` entirely.
2. **keepmomentum self-clears.** `player.zs:1385-1388` clears it once both
   `|vel.x|` and `|vel.y|` fall below `vr_momentum_threshold` (default `1.0`,
   `hw_vrmodes.cpp:121`). Re-set it each tic; do not set it once.
3. **`A_ChangeVelocity` is a no-op on players** by default (`actions.zs:74-76`).

**Feel knobs:** `speed` ramp per tic (accel), a per-tic `speed *= drag`, `Angle` rate as a
function of `speed` (slow at speed = arcade-correct), `Actor.Friction` /
`GetFriction()` for surface grip, `level.aircontrol` through `ApplyAirControl`
(`player.zs:1325`) for mid-jump steering, `A_SetSize` to widen the hitbox at speed.

**Why this wins over riding an actor:** it keeps `player.camera == player.mo`, which is the
only configuration in which the VR backends write the hand and head poses at all
(`gl_openvr.cpp:2406`, `hw_openxrdevice.cpp:569` — both do
`r_viewpoint.camera ? r_viewpoint.camera->player : nullptr`). It keeps the player's own
collision, Z handling, damage, and `PlayerLandedMakeGruntSound`. And it needs zero engine
changes.

### 4.2 Vehicle as a separate actor the player rides — **HACK, cost L**

**Physical:** identical to the player; the difference is internal.

**Detection / resolution, two sub-variants:**

*(a) Player stands on the vehicle actor.* Give the vehicle `+SOLID +ACTLIKEBRIDGE`, a wide
`radius`, and let the player jump on. Z tracking is automatic (`p_mobj.cpp:4386-4396`).
**XY is not** — nothing in the engine carries a rider (§2). You must, every tic, do
`[ok, carrier] = TestMobjZ(); if (carrier == vehicle) { Vel.XY += vehicle.Vel.XY; player.keepmomentum = true; }`
in `PlayerThink`, which runs for all players *before* `RunThinkers` moves anything
(`src/p_tick.cpp:166-171`). Doing it in the vehicle's `Tick` instead races the thinker order
and jitters.

*(b) Player is hard-slaved to the vehicle.* Copy `ActorMover`: each tic
`player.SetZ(vehicle.pos.Z + seatZ); player.TryMove(vehicle.Pos.XY + seatOffset, true);`
exactly as `movingcamera.zs:465-478` does, with `bNoGravity` forced on and restored on exit
(`:489-507`). Robust, and the shipped precedent for it is in the tree.

**Do not set `player.camera = vehicle`.** See §5, Wall 1.

**Feel knobs:** seat offset via `SetViewPos` (`actor.zs:1278`), carrier lag (lerp the copy
to get suspension float), `bTHRUACTORS` on the vehicle so traffic does not shove it.

**Cost L** because every player system (damage, landing, weapon reach) now has two bodies to
reason about, for no capability the override approach lacks.

### 4.3 Handlebars / steering wheel gripped with both hands — **BUILDABLE, cost S**

**Physical:** both hands out in front, roughly shoulder width, turned like bars or a wheel.
Squeeze a trigger to "grab".

**Detection:** with §0 established, this is a two-point measurement.
```
if (!OverrideAttackPosDir) { fall back to cmd.sidemove; }
vector3 L = OffhandPos, R = AttackPos;          // actor.zs:272, :268
vector3 d = R - L;
double barAngle = atan2(d.y, d.x);              // wheel/bar yaw
double barTilt  = atan2(d.z, d.xy.Length());    // bar roll, the motorbike one
double grip     = d.Length();                   // hands apart = wide bars
```
`barAngle - restAngle` is the steer input; `barTilt` is the lean/roll input. Both hands come
from the same frame's render, so the difference is coherent. Grab state: `BT_ATTACK` for the
main hand and `BT_OFFHANDATTACK` for the offhand (`src/d_event.h:42`, `:74`), read from
`player.cmd.buttons` — both hands at once, independently.

**Resolution:** `Angle += steerGain * deadzone(barAngle - restAngle) * speedFactor(speed)`
inside the overridden `MovePlayer`.

**One-hand fallback (if §0a turns out to bite and the main hand is dead on your backend):**
use `OffhandPos` against the *player's aim yaw* (`Angle`, which is the weapon hand folded
into the viewpoint, baseline §1a) as the second point. Worse — you lose roll, and head turns
pollute it — but it steers.

**Feel knobs:** `restAngle` calibrated on grab (store the hand vector at the tic both grips
close), dead zone in degrees, gain curve, return-to-centre spring, a per-tic slew limit so a
35 Hz sample jump cannot snap the bars.

### 4.4 Throttle and brake — **BUILDABLE, cost S**

**Physical:** right trigger = throttle, left trigger = brake. Or twist-grip: roll the main
hand forward.

**Detection:** `player.cmd.buttons & BT_ATTACK` / `& BT_OFFHANDATTACK`. Twist-grip:
`AttackRoll` (`actor.zs:270`) delta from the grab-time rest roll.

**Resolution:** `targetSpeed = maxSpeed * throttle01; speed = approach(speed, targetSpeed, accel)`.

**WALL inside this one:** there is **no analog trigger value in ZScript**. `usercmd_t`
(`src/d_protocol.h:64-73`) has only `buttons` (a bitfield) plus `forwardmove`/`sidemove`/
`upmove`/`pitch`/`yaw`/`roll`. Analog throttle must come either from the stick
(`cmd.forwardmove`) or from `AttackRoll` as a twist-grip. **The twist-grip is the better
answer and it is free.** Label: trigger-as-analog = WALL (missing piece: an analog axis in
`usercmd_t` plus a ZScript field); twist-grip analog = BUILDABLE, cost S.

### 4.5 Motorbike lean from head and body — **BUILDABLE, cost M**

**Physical:** lean your torso into the corner.

**Detection:** two independent signals, use both.
- `HmdPosition` (`actor.zs:277`) — metres, tracking space, unrotated. Difference it against a
  calibrated centre captured when the player mounts: `leanX = HmdPosition.x - centre.x`.
  Because it is unrotated tracking space, you must counter-rotate by the player's own yaw
  change since calibration, or recalibrate on every snap turn.
- `barTilt` from §4.3 — the hands roll with the bike naturally, and this one is already in
  world space.
- Free bonus: with `vr_crouch_use_button false`, `player.crouchfactor` (`player.zs:3126`) is
  continuously recomputed from HMD height (`gl_openvr.cpp:2513-2516`), i.e. "how far the
  rider has tucked down".

**Resolution:** `Roll = lerp(Roll, maxLean * leanInput, leanSpeed)` (`actor.zs:111`), and feed
the same number into the turn rate so leaning actually steers. The bike model is drawn rolled
by giving its MODELDEF `USEACTORROLL` (`src/r_data/models.h:54`).

**Feel knobs:** max lean degrees, lean-to-turn coupling, counter-steer lag, lean recentre
rate, a low-speed cutoff so the bike does not lie down at a standstill.

**Gotcha — the crouch speed trap:** if the player tucks far enough that
`player.crouchfactor < 0.75`, `P_XYMovement` switches on the `MAXMOVE` clamp branch
(`p_mobj.cpp:2096-2105`) and hard-limits you to 30 units/tic, *and* `MovePlayer` scales
fm/sm by `clamp(crouchfactor, 0.5, 1)` (`player.zs:1418-1423`). Tucking in would silently
halve your top speed. Either keep `vr_crouch_use_button true` while driving, or force
`player.crouchfactor = 1` each tic in the vehicle pawn.

### 4.6 Wheelies, jumps, landing angle (ExciteBike) — **BUILDABLE, cost M**

**Physical:** pull the bars back (hands toward you, rising) to wheelie; hold the angle in the
air; level out before you land.

**Detection:** wheelie = `barTilt` (§4.3) plus hand-toward-body distance; both from
`AttackPos`/`OffhandPos`. Airborne = `!player.onground` (set each tic at `player.zs:1370`:
`pos.z <= floorz + 2 || bOnMobj || …`). In-air pitch authority = continued `barTilt`.

**Resolution:**
- Wheelie: `Pitch` (`actor.zs:110`) drives the model via MODELDEF `USEACTORPITCH`
  (`models.h:53`). Note `Pitch` is *also* the VR aim pitch, so for a player pawn you want the
  bike to be a separate attached model actor carrying the pitch, not the pawn itself.
- Jump: write `Vel.Z` directly, or override `FallAndSink` (`actor.zs:522`) for a custom arc.
  Baseline gravity is `sv_gravity` 800 (`p_mobj.cpp:156`); per-actor `Gravity` scales it
  (`actor.zs:176`).
- Landing quality: the engine hands you the landing event with the actor you hit —
  `PlayerLandedMakeGruntSound(Actor onmobj)` (`actor.zs:1440`, call at `p_mobj.cpp:4382` for
  actor landings and `p_mobj.cpp:2767` for floor landings). Override it, compare the bike's
  pitch to the landing surface, and apply a speed penalty or a crash. `LandingSpeed`
  (`actor.zs:267`, default `-8` at `:475`) is the engine's own "that was a landing" threshold.

**Feel knobs:** air pitch rate, landing tolerance window in degrees, speed loss curve,
`Vel.Z` on launch, per-vehicle `Gravity`, a brief `bNOFRICTION` on a bad landing to make it
skid (`thingdef_data.cpp:324`).

### 4.7 Speed limits and tunnelling — **EXISTS (mostly solved), cost S to verify**

This is the section the brief called make-or-break. Quantified:

| Clamp | Value | Where | When it applies |
|---|---|---|---|
| `MAXMOVE` | **30.0** units/tic = 1050 units/sec | `src/playsim/p_local.h:61` | **only** if `COMPATF_WALLRUN` is set for a player, **or** `waterlevel >= 1`, **or** `player->crouchfactor < 0.75`, and `!bNOFRICTION` — `p_mobj.cpp:2096-2105` |
| `MAXMOVE/4` | 7.5 units/tic | `p_mobj.cpp:2065-2066` | underwater and not crouched past `crouchoffset < -10` |
| `VELOCITY_THRESHOLD` | **5000.0** units/tic | `p_mobj.cpp:2107-2111` | always; `Vel.MakeResize(5000)` |
| `cmd.upmove` | `±0x300` | `player.zs:1579` | fly/swim vertical only |
| line special thrust | `±MAXMOVE` | `p_lnspec.cpp:1229-1230` | `ThrustThing` without the nolimit arg |

**So a dry, upright, non-wallrun player is clamped at 5000 units/tic, not 30.** For reference,
Doom running speed is roughly 12.5 units/tic and the player radius is 16 (`player.zs:101`).
A 200 units/tic vehicle — about 16× running, a credible highway speed — is nowhere near any
clamp. Note the clamp preserves direction rather than clamping axes independently
(`p_mobj.cpp:2100-2105`), so it decelerates rather than skewing your heading.

**Tunnelling is already handled.** `P_XYMovement` splits the move into
`steps = 1 + speed / (radius - 1)` substeps and runs a full `P_TryMove` per substep
(`p_mobj.cpp:2166-2192`, loop at `:2216-2429`). At radius 16 that is a `TryMove` roughly
every 15 units, so a 200 units/tic vehicle does ~14 collision tests per tic. **You do not
tunnel; you pay CPU.** Two real consequences:

- **Cost scales linearly with speed and inversely with radius.** Growing the hitbox with
  `A_SetSize` (`actor.zs:1314`) to car dimensions *reduces* the substep count. A radius-48
  car at 200 u/tic does ~5 `TryMove`s instead of 14.
- **Blocked substeps abort the rest of the move** — `P_SlideMove` is invoked with
  `(onestep, totalsteps)` (`p_mobj.cpp:2283`) and if velocity goes to zero, `steps = 0`
  (`:2285`). A glancing hit at speed will *stop* a vehicle rather than deflect it unless you
  re-apply thrust next tic. Build the "scrape along the barrier" feel yourself by detecting
  `Vel` collapse and restoring the tangential component.

**Hitscan and projectile reach is separate:** `PLAYERMISSILERANGE` is 8192
(`src/playsim/p_local.h:69`) and `MISSILERANGE` 2048 (`:68`) — fine at highway speeds, but a
projectile fired *forward* from a 200 u/tic vehicle will be overtaken by the vehicle unless
you add the vehicle's velocity to the missile's `Vel` on spawn.

**Label:** EXISTS. Cost S, and the S is one measured test run, not code.

### 4.8 Cockpit / handlebar model attached to view or hand — **split verdict**

**On a hand: BUILDABLE, cost S.** A psprite layer whose `Caller` is the ready weapon is given
the main-hand transform; one whose `Caller` is `player.OffhandWeapon` is given the offhand
transform. The decision is exactly one line: `src/r_data/models.cpp:222`
```
int hand = psp->GetCaller() == playermo->player->OffhandWeapon ? 1 : 0;
```
then `vrmode->GetWeaponTransform(&objectToWorldMatrix, hand)` (`models.cpp:223`,
implementation `hw_vrmodes.cpp:455-472`). So: make the left handlebar grip an offhand
"weapon" and the right grip the ready weapon, each with a MODELDEF model, and they track the
real hands with no extra work. Per-tic positioning via `A_OverlayOffset` / `A_OverlayRotate`
/ `A_OverlayScale` (`actor.zs:1325-1330`) and the `PSprite` fields (`player.zs:2954-2972`)
layers fine adjustments on top; `huds->mx/my` and `huds->rotation` are folded into the model
matrix at `src/rendering/hwrenderer/scene/hw_weapon.cpp:108`.

**Pinned to the view / head: WALL.** There is no head slot. Every HUD model goes through
`RenderHUDModel` and gets either a hand transform or, when `GetWeaponTransform` fails but VR
is on, a *player-position* fallback with the pawn's yaw (`models.cpp:228-233`) — never the
HMD pose. Missing piece: a third `GetWeaponTransform` hand index (or a `VR_HEAD`) plus a
psprite flag selecting it.

**Workaround (HACK, cost M):** spawn a real world actor with the dashboard model,
`+NOINTERACTION`-style flags, and `SetOrigin(viewPos + forward*d, true)` every tic with
`ClearInterpolation()` discipline, or parent it to the pawn and let `USEACTORROLL`/
`USEACTORPITCH` carry the lean. It will lag the head by one tic (28 ms) because script runs
at 35 Hz and the head is sampled per frame. For a dashboard that *should* be body-locked
rather than head-locked — which is what a real cockpit is — that is correct behaviour, not a
defect. **Recommendation: build the cockpit as a body-locked world model and the controls as
hand psprites. Do not chase a head-locked cockpit.**

### 4.9 Interstate '76 — vehicle-mounted weapons and hardpoints — **BUILDABLE, cost M**

**Physical:** the guns are bolted to the car and point where the car points; the player
drives, and separately aims a turret with one hand.

**Detection / resolution:**
- *Fixed hardpoints:* ordinary `Weapon` actors on the player. Fire from
  `AttackPos`-independent origins by spawning the projectile with `A_SpawnItemEx` at a
  muzzle offset from the pawn and `Vel3DFromAngle(speed, Angle, 0)` using the **vehicle's**
  `Angle`, not the aim angle. Add the vehicle's `Vel` to the projectile's `Vel` so forward
  shots do not get rear-ended by the car.
- *Aimed turret:* the offhand weapon system. `PlayerInfo.OffhandWeapon`
  (`d_player.h:384`), `bOffhandWeapon` routing (`player.zs:1977-1998`), and firing from
  `OffhandPos` along `OffhandDir(self, angle, pitch)` (`actor.zs:859`) give a hand-aimed gun
  that is completely independent of where the car is heading.
- *Hardpoint inventory:* per-slot `Inventory` items on the pawn, each holding a model name
  for `A_ChangeModel(…, modelindex: slot, …)` (`actor.zs:1232`) so the car's silhouette
  changes with the loadout.

**Feel knobs:** convergence distance for paired guns, muzzle offsets, recoil as a `Vel`
nudge (**not** `A_ChangeVelocity` — §3), per-hardpoint heat/ammo, `bNOFRICTION` yaw kick on
a big shot.

**Aiming while driving:** this is where the two-hand read earns its keep. Steering comes from
the *pair* (`AttackPos - OffhandPos`), so a one-handed grip plus a free hand is detectable as
"the hands are too far apart / the grab button released" — switch the free hand to turret
mode and auto-centre the bars.

### 4.10 Lucky & Wild — one drives, hands shoot — **BUILDABLE, cost M**

**Physical:** the vehicle drives itself (on rails or auto-steer); both of the player's hands
are free to aim and fire independently.

**Detection / resolution:**
- *On-rails:* `PathFollower` / `ActorMover` with the player as `tracer`
  (`movingcamera.zs:434-481`). `ActorMover` already does the right things: it saves
  `bNoGravity`/`bSolid`/`bNoBlockmap` (`:489`), forces `bNoGravity` (`:490`), moves with
  `TryMove` so the player still collides with monsters, and calls `ClearInterpolation()`
  (`:506`). Setting `args[2] & 2` also slaves the actor's `angle` to the path (`:473`) —
  for VR, **leave that off**: the player's yaw must stay the player's.
- *Auto-steer:* in the overridden `MovePlayer`, run a forward `CheckMove`
  (`actor.zs:832`) a few tics ahead at several candidate yaws and pick the clearest; blend
  that into `Angle` at a bounded rate.
- *Shooting:* the shipped two-weapon system. Main hand = `player.ReadyWeapon` fired by
  `BT_ATTACK`; offhand = `player.OffhandWeapon` fired by `BT_OFFHANDATTACK`
  (`d_event.h:74`), each with its own psprite layer and state machine (baseline §2). No work
  needed beyond authoring two weapons.

**Feel knobs:** rail speed curve, how much yaw authority the player keeps, lateral "duck
left/right" from `HmdPosition.x` as a dodge, auto-brake on a near-miss.

### 4.11 Modular damage: per-part health — **BUILDABLE, cost M**

Entirely script-side; the engine imposes nothing and offers nothing.

**Detection:** a `VehicleState` Thinker or an `Inventory` item on the pawn holding
`int partHealth[N]` for tyres FL/FR/RL/RR, engine, radiator, armour faces, each weapon
hardpoint. Hits land through the normal damage path: override
`PlayerPawn.DamageMobj` / the `WorldThingDamaged` event, and resolve *which* part was hit
from the hit direction relative to `Angle` plus the hit Z against `height`.

**Resolution (handling changes):**
- Flat tyre → raise the `Friction` asymmetry: bias `Angle` by a constant pull per tic and
  cut `maxSpeed`.
- Dead engine → `maxSpeed = 0`, let drag bleed `speed`.
- Blown armour → raise incoming damage multiplier.
- Destroyed hardpoint → remove the weapon, `A_ChangeModel` that model index to a wrecked
  mesh or to nothing (`actor.zs:1232`).
- Visuals: one `A_ChangeModel` call per part index; MODELDEF
  `MODELSAREATTACHMENTS` (`models.h:60`) lets indices above 0 ride the base model's bones.

**Feel knobs:** per-part HP, pull-per-tic on each tyre, smoke thresholds, whether a part can
be repaired, whether the steering bias is felt in the bars (you cannot push the player's real
hands, so express it as yaw drift the player must counter — which is the correct and more
legible choice in VR anyway).

**Label: BUILDABLE, cost M** — the engine contributes nothing, but blocks nothing.

### 4.12 Car-as-mech: gauges, gear shifting by hand, systems — **BUILDABLE, cost M**

**Gear lever — physical:** reach to a fixed spot beside the seat, squeeze, move, release.

**Detection:** `OffhandPos` (`actor.zs:272`) is a world-space point. Define the lever's gate
as a small world-space box offset from the pawn by the pawn's yaw; detect entry, latch on
`BT_OFFHANDATTACK`, read the displacement while held, release to commit. This is an H-pattern
in thin air and it works because the hand position is absolute and the player's body does not
move relative to the seat. **Gate on `OverrideAttackPosDir`.**

**Resolution:** gear index changes the torque curve and `maxSpeed`; a missed shift costs
speed. 35 Hz sampling is plenty for a shift (it takes ~0.3 s = ~10 samples).

**Gauges:** a world-model dashboard (§4.8) with per-needle actors you rotate via
`A_SetAngle`/`A_SetRoll` + MODELDEF `USEACTORROLL`, or a ZScript `StatusBar`/`HUDMessage`
overlay. Both are plain ZScript.

**Systems management:** ignition, headlights (dynamic lights are ZScript-spawnable), boost,
damage repair — each is a hand-zone button in the same world-space-box style as the lever.

**Feel knobs:** gate box size (start generous: 6-8 map units), latch hysteresis, haptic/audio
detent (audio only — §5), shift timing window, stall behaviour.

### 4.13 Matrix highway: jumping between vehicles, traffic — **HACK, cost L**

**Traffic:** ordinary actors with `+SOLID` and a velocity, or on `PathFollower` rails. Trivial
— **EXISTS, cost S.**

**Standing on a moving vehicle: this is the researched answer the brief asked for.** The
engine tracks the carrier's **Z and nothing else**:
- `AActor::Tick` → `P_CheckOnmobj`; if something is beneath you and within `MaxStepHeight`,
  `SetZ(onmo->Top())` every tic, `MF2_ONMOBJ` set, `Vel.Z = 0` (`p_mobj.cpp:4370-4428`).
- No code path copies `onmo->Vel` to the rider. Confirmed by grep across
  `src/playsim/p_mobj.cpp` and `src/playsim/p_map.cpp`; the only `MF2_ONMOBJ` writes are the
  flag itself (`p_mobj.cpp:4373`, `:4427`).
- The rider *can* find the carrier: `TestMobjZ()` (`actor.zs:818`) returns it, and
  `PlayerLandedMakeGruntSound(Actor onmobj)` (`actor.zs:1440`) fires once on landing with the
  carrier passed in.

So: **you stand on a speeding truck and it drives out from under you.** The fix is script,
every tic, in `PlayerThink` (before thinkers run — `src/p_tick.cpp:166-171`):
```
[ok, carrier] = TestMobjZ(true);
if (carrier && carrier is "RSVehicle") {
    Vel.XY += carrier.Vel.XY;         // inherit for this tic
    player.keepmomentum = true;       // or the damper eats it (player.zs:1374)
}
```
That is a HACK — a correct one, but the engine should be doing it. Expect two artefacts:
one-tic lag at high speed (a 200 u/tic truck moves 200 units between samples, so your feet
slide), and a fight with `P_SlideMove` when the truck's own hull blocks you. Mitigate with
`bTHRUACTORS` on the rider while mounted, or go to the `ActorMover` hard-slave (§4.2b).

**Jumping between vehicles:** `CheckJump` (`player.zs:1510`) and `Player.JumpZ 8`
(`player.zs:120`) are fine as-is, but the jump must inherit the source vehicle's velocity
(same `Vel.XY +=` as above, applied at the moment of launch) or every jump will land behind
the target. On landing, `PlayerLandedMakeGruntSound(onmobj)` identifies the new carrier.

**Label:** standing/jumping between moving vehicles = **HACK, cost L**. The missing engine
piece is named in §5, Wall 2.

---

## 5. Walls

**Wall 1 — `player.camera` on a non-player actor kills all VR input.**
Both VR backends select the player as `r_viewpoint.camera ? r_viewpoint.camera->player : nullptr`
(`src/rendering/gl/stereo3d/gl_openvr.cpp:2406`,
`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:569`), and `r_viewpoint.camera`
is `player->camera` (`src/rendering/r_utility.cpp:968`). Point `player.camera` at a vehicle
actor and `AttackPos`, `OffhandPos`, `HmdPosition`, `crouchfactor` and the teleport logic all
stop updating, because a vehicle actor's `->player` is null.
*Missing piece:* resolve the player as `r_viewpoint.camera->player ?: players[consoleplayer]`,
or honour `RF2_CAMFOLLOWSPLAYER` (which already exists at `r_utility.cpp:991`) in the VR
`SetUp` path. One-line class of change, but it is an engine change and this engine is frozen.
**Consequence for the mod: §4.1 is not a preference, it is the only supported shape.**

**Wall 2 — no rider carry.** Standing on a moving actor inherits Z only
(`p_mobj.cpp:4386-4396`); nothing copies XY velocity.
*Missing piece:* in `AActor::Tick`, when `MF2_ONMOBJ` is set and the carrier moved, add
`onmo->Vel.XY` to the rider's position or velocity (Hexen-style platform carry).
Scriptable workaround in §4.13, with named artefacts.

**Wall 3 — no head-locked psprite.** `RenderHUDModel` picks hand 0 or 1 from weapon
ownership (`src/r_data/models.cpp:222-223`); the only non-hand fallback is player-position +
pawn yaw (`models.cpp:228-233`). No HMD transform is available to a psprite.
*Missing piece:* a `VR_HEAD` index in `VRMode::GetWeaponTransform` and a `PSPF_`-style flag
to request it. Workaround in §4.8.

**Wall 4 — no analog trigger in ZScript.** `usercmd_t` (`src/d_protocol.h:64-73`) carries
`buttons` as a bitfield plus six shorts; no controller axis beyond the sticks reaches script.
*Missing piece:* an analog field in `usercmd_t` fed from the VR input layer, plus a
`UserCmd` field definition next to `src/playsim/p_user.cpp:1952-1953`.
Workaround: twist-grip from `AttackRoll` (§4.4), which is arguably better VR anyway.

**Wall 5 — no script-callable haptics.** Baseline §4 holds. The one PC-reachable rumble is
`VRMode::Vibrate(200, 0/1, 0.4 + 0.6*min(damage,100)/100)` on taking damage
(`src/playsim/p_interaction.cpp:1441-1444`), outside the `__MOBILE__` guard; everything else
goes through `VR_HapticEvent` with hardcoded event names and no ZScript binding.
*HACK:* damage the driver 1 point to get a rumble — but it also adds to `player.damagecount`
(`p_interaction.cpp:1431`), i.e. a red pain flash, and it must be real damage, so it is
unusable as continuous road feel. **Build engine feel out of sound and camera shake
(`DEarthquake`, `src/rendering/r_utility.cpp:1064`), not rumble.**
*Missing piece:* a ZScript `Vibrate(hand, duration, intensity)` binding onto
`VRMode::Vibrate` (`hw_vrmodes.h:94`).

**Wall 6 — 35 Hz is the whole budget for hand motion** (baseline §3). A hand crossing 1 m in
0.2 s is 7 samples. For steering and levers this is fine (they are positional, not
gestural). For anything that needs the *path* of a hand — a flicked gear change, a sharp
bar-slap — you get the endpoints and must infer the middle.

**Wall 7 (not an engine wall, a QuestZDoom booby trap, listed here because it will cost
someone a day):** `A_ChangeVelocity` returns without doing anything on a player unless
`vr_recoil` is set, and it defaults false (`wadsrc/static/zscript/actors/actions.zs:74-76`,
`src/playsim/p_actionfunctions.cpp:90`). Likewise `MovePlayer` annihilates player velocity
every tic unless `keepmomentum` is re-set (`player.zs:1374-1379`, cvar default false at
`hw_vrmodes.cpp:120`). Both are silent.

---

## 6. Smallest prototype

Target: a rideable motorbike that steers from two hands, leans, and goes fast enough to
prove the collision budget — in one pk3, no engine change, no map work.

1. **`class RSBikePawn : DoomPlayer`** with `bool mounted` and `double speed`.
2. **Override `MovePlayer()`** (`player.zs:1340`). When `!mounted`, `Super.MovePlayer()`.
   When mounted:
   - `player.keepmomentum = true;` **every tic** (`player.zs:1374` damper).
   - `player.crouchfactor = 1;` every tic (avoids the `MAXMOVE` crouch branch,
     `p_mobj.cpp:2096-2105` — only needed if `vr_crouch_use_button` is false).
   - throttle: `speed = approach(speed, maxSpeed * ((player.cmd.buttons & BT_ATTACK) ? 1 : 0), accel)`,
     then `speed *= drag`.
   - steering: if `OverrideAttackPosDir`, `vector3 d = AttackPos - OffhandPos;`
     `Angle += steerGain * deadzone(atan2(d.y, d.x) - restYaw) * clamp(speed/maxSpeed, .2, 1);`
     else `Angle += cmd.sidemove * fallbackGain / 256.`
   - lean: `Roll = lerp(Roll, maxLean * (atan2(d.z, d.xy.Length()) - restTilt) * k, 0.25);`
   - `Vel.XY = AngleToVector(Angle, speed);`
3. **Calibrate on mount:** on the tic both grab buttons go down, store `restYaw`, `restTilt`,
   `HmdPosition` as `leanCentre`. Re-calibrate on snap turn.
4. **Bike model:** one world actor spawned at the pawn, `SetOrigin(pawn.pos, true)` each tic,
   MODELDEF with `USEACTORROLL` + `USEACTORPITCH` (`src/r_data/models.h:53-54`), `Roll` and
   `Pitch` copied from the pawn. This is the body-locked cockpit of §4.8.
5. **Handlebar grips:** a `+OFFHANDWEAPON` weapon holding the left grip model and an ordinary
   ready weapon holding the right. They track the real hands with zero code
   (`src/r_data/models.cpp:222`).
6. **The one measurement that decides everything:** log
   `Vel.Length()`, `player.onground`, and a tic-time sample while driving a long straight at
   `maxSpeed = 50`, then `100`, then `200` units/tic, against a wall-dense map. You are
   measuring `TryMove` substep cost (`p_mobj.cpp:2166-2192`), not correctness — the clamp at
   5000 (`p_mobj.cpp:2107`) will not be approached.
7. **Verify §0a first** (see Open Questions). If `AttackPos == OffhandPos == pawn.pos` on your
   backend, everything hand-driven in this document is inert and you must fall back to
   sticks before writing anything else.

Steps 1-3 are the whole physics model and are perhaps 120 lines. Steps 4-5 are content.

---

## 7. Open questions

1. **Does the OpenXR desktop path populate `HmdPosition`, `AttackPos` and `OffhandPos` at
   all?** In this tree only `gl_openvr.cpp` writes the backing globals (§0a). Resolve with a
   five-minute test, not by reading more code: print
   `AttackPos`, `OffhandPos`, `HmdPosition`, `OverrideAttackPosDir`, `PlayInVR` each tic from
   a trivial EventHandler, run once under SteamVR and once under the OpenXR runtime you
   actually ship on, and compare. **Every hand-driven mechanic in §4 depends on the answer.**
2. **Does `_BASELINE_FACTS.md` §1a need rewriting for every other topic report, or only this
   one?** The main-hand fields are real (§0). Other reports built on "one readable hand"
   should be re-checked.
3. **What is the real cost of a 200 units/tic player** in substepped `TryMove`s on a dense
   map at Quest-class CPU budget? Unknown; §6 step 6 measures it.
4. **Does `P_SlideMove` at vehicle speed produce a usable "scrape the barrier" feel, or does
   it just stop the vehicle dead?** `p_mobj.cpp:2283-2290` suggests the latter is likely and
   needs a script-side tangential restore. Not tested.
5. **Is `primaryLevel->max_velocity` / `avg_velocity` (`src/p_tick.cpp:186-188`) reachable
   from ZScript?** I did not find a field definition for either. If it is, it is a free
   speedometer.
6. **Does `SetViewPos` with `VPSF_ABSOLUTEOFFSET` compose correctly with the VR head pose, or
   does the VR path overwrite the view position?** `SetViewPos` is `actor.zs:1278`; the VR
   viewpoint is built in `hw_openxrdevice.cpp:695` (`updateHmdPose`). Not traced. This decides
   whether a seat offset (driver sits low and left) is available.
7. **Can a vehicle's `radius` be grown with `A_SetSize` (`actor.zs:1314`) on a *player* pawn
   without breaking map traversal** (doorways, lifts, the player start)? Cheap to test, and it
   is the main lever on the substep cost in §4.7.
