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
| `PlayerInfo.PlayInVR` — bool, readonly | `wadsrc/static/zscript/actors/player/player.zs:3136` | `src/playsim/d_player.h:502` | — |

### 1.2 There is no main-hand pose. Design around this, not against it.

Search the tree: there is no `MainhandPos`, no `WeaponhandPos`, no `WeaponAngles` reaching
ZScript. Only `Offhand*`.

The weapon hand is not missing — it *is the aim*. The renderer folds the weapon controller
into the viewpoint, setting the attack origin and orientation alongside the offhand values:

```
src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:442   AttackPos   = OffhandPos   = PosAtZ(shootz)
src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:443   AttackAngle = OffhandAngle = viewpoint yaw - 90
src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:444   AttackPitch = OffhandPitch = -viewpoint pitch
src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:445   HmdPosition = hmdPosition
```

So in practice you have:

- **Offhand** — a freely readable 6DOF pose (position, yaw, pitch, roll). Do what you like with it.
- **Main hand** — readable only as "where the player is pointing": the player's `angle`, `pitch`
  and the attack origin. You get direction and origin, not an independent hand position.
- **Head** — `HmdPosition`, a real tracked position. This is your reference frame for
  everything body-relative: guard height, duck, lean, holster slots, body shots vs head shots.

**The practical rule for every mechanic in this survey:** put the thing that needs a *free-moving
hand* on the offhand, and the thing that needs *aiming* on the main hand. A design that needs two
independently tracked hands doing different free-space motions at once is fighting the engine.
Several topic reports hit this wall; they say so where they do.

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

Note `timefreezer` is a **mask**, not a flag — the freeze is already expressed per-player in bits.
Whether that can be bent into a partial or selective freeze is the time report's job; it is the
obvious first place to look.

There is no global time *scale* in this list. The time report searches for one and reports
what it finds.

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

1. **One free hand.** The offhand is tracked; the main hand is aim. (§1.2)
2. **No velocities — you difference positions.** (§1.3)
3. **35 Hz sampling of a per-frame value.** Fresh, but sparse. Test chords, not points. (§1.4)
4. **Haptics are not scriptable.** Any design whose feel depends on rumble is on thin ice. (§1.6)
5. **Collision is cylinders.** No shape you draw is the shape that blocks. (§3.2)
6. **`bTwoHanded` clears the offhand weapon** (`player.zs:1989-1992`) — a two-handed weapon and
   an offhand gadget cannot coexist without working around this.
7. **The HMD owns the view.** Script view changes in VR are suspect until proven. (§5)
8. **This is not REMA.** `E:\DOOMWork\UZDXREMA` has a far larger VR surface. Anything you
   remember from there and cannot find here is a WALL here.

---

## 8. How to read the rest of this survey

`README.md` is the index: a flat table of every mechanic across all ten reports, with its label,
its cost, and the file and section that holds the design. Go there first, then to the section.

Labels: **EXISTS** (already in the engine, cited) · **BUILDABLE** (script, with named APIs) ·
**HACK** (works, but abuses something — the report says what breaks) · **WALL** (needs engine
changes — the report names the missing piece).

Costs: **S** an afternoon · **M** a few days · **L** a week or more.
