# Baseline facts — verified by hand before the topic surveys

**Engine surveyed:** tag `qzd17.3` of `github.com/emawind84/gzdoom` (emawind84 QuestZDoom PC edition).
All citations in every report are **repo-relative** paths at that tag (`src/...`, `wadsrc/...`).

This file is ground truth. Every topic report may rely on it without re-deriving it.
If a report contradicts something here, the report must say so explicitly and show its citation.

---

## 1. The entire VR surface exposed to ZScript

This is the whole list. There is nothing else.

| ZScript | Declared | Native field | Thunk |
|---|---|---|---|
| `Actor.HmdPosition` (vector3, readonly) | `wadsrc/static/zscript/actors/actor.zs:277` | `src/playsim/actor.h:1668` | `src/scripting/vmthunks_actors.cpp:2195` |
| `Actor.OffhandPos` (vector3, readonly) | `actor.zs:272` | `actor.h:1661` | `vmthunks_actors.cpp:2191` |
| `Actor.OffhandPitch` (double, readonly) | `actor.zs:273` | `actor.h:1662` | `vmthunks_actors.cpp:2192` |
| `Actor.OffhandRoll` (double, readonly) | `actor.zs:274` | `actor.h:1664` | `vmthunks_actors.cpp:2193` |
| `Actor.OffhandAngle` (double, readonly) | `actor.zs:275` | `actor.h:1663` | `vmthunks_actors.cpp:2194` |
| `Actor.OffhandDir(actor, angle, pitch)` | `actor.zs:859` | `actor.h:1666` (fn ptr) | — |
| `PlayerInfo.PlayInVR` (bool, readonly) | `wadsrc/static/zscript/actors/player/player.zs:3136` | `src/playsim/d_player.h:502` | — |

### 1a. There is no main-hand pose field. This is the single most important fact in the survey.

`Offhand*` exists; there is no `MainhandPos`, no `WeaponhandPos`, no `WeaponAngles` in ZScript.
The **weapon hand is the aim**: its pose reaches script as the player's own
`angle` / `pitch` / `roll` and view position, because the renderer folds the weapon
controller into the viewpoint (`src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:442-444`,
where `AttackPos`/`AttackAngle`/`AttackPitch` are set alongside the offhand values).

Consequence for every topic: **you get one freely-readable hand (the offhand) and one hand
that you can only read as "where the player is aiming."** Any two-handed mechanic has to be
designed around that asymmetry, not against it.

### 1b. There are no hand or head velocity fields.

Nothing in the table above is a velocity. Any speed, swing, flick, draw or throw
must be computed by storing the previous tic's `OffhandPos` / `HmdPosition` and
differencing. Budget for that in every design, and see §3 for the rate you get.

---

## 2. The offhand weapon system (this is real and substantial)

- `PlayerInfo.OffhandWeapon` — `src/playsim/d_player.h:384`
- `PSP_OFFHANDWEAPON` psprite layer, used throughout `player.zs`
  (`player.zs:411`, `:563`, `:1947`, `:1953`, `:1998`, `:2095`)
- Weapon flag `bOffhandWeapon` routes a weapon to that hand — `player.zs:1977-1998`
- Weapon flag `bTwoHanded` forcibly clears the offhand weapon — `player.zs:1989-1992`
- Per-hand button checking: `CheckWeaponButtons(hand)` — `player.zs:588`, `:2031`
- Per-hand ready/fire/down state selection keyed on `hand` — `player.zs:398`, `:431`,
  `:466`, `:2092`, `:2429`
- Morph preserves it: `PremorphWeaponOffhand` — `d_player.h:410`

So: **two independently-driven weapons, one per hand, each with its own psprite layer and
its own state machine, is a shipped engine feature, not a hack.**

---

## 3. Update rate: hand pose is written per *frame*, read per *tic*

The hand and head poses are written from the **renderer**, not the playsim:

- `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:442-445`
- `src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:600-603`
- `src/rendering/gl/stereo3d/gl_openvr.cpp:2549-2552`

Two consequences, and they pull in opposite directions:

1. **The value script reads is fresh** — it is the latest rendered frame's pose, not a
   35 Hz-stale one. Good for thresholds.
2. **Script still only samples at 35 Hz**, so any motion detail between tics is gone
   forever. A fast swing is 2-4 samples. Every gesture design in this survey must survive
   on that budget, and "trace the path between the last two samples" is the standard
   remedy — do not assume you can see the arc.

---

## 4. Haptics: present in C++, **not reachable from ZScript**

- `VRMode::Vibrate(duration, channel, intensity)` — `src/common/rendering/hwrenderer/data/hw_vrmodes.h:94`
- `VR_HapticEvent(event, position, intensity, angle, yHeight)` — `hw_vrmodes.h:97`
- Intensity cvars `ext_haptic_level_*` — `src/playsim/p_actionfunctions.cpp:92-101`
- **Zero hits for haptic/rumble/vibrate anywhere in `wadsrc/static/zscript`.**

Treat "fire a custom rumble from script" as **unproven** until someone cites a hook.
If your topic needs haptics, find the caller of `VR_HapticEvent` and report whether a mod
can reach it (a hardcoded damage/pickup/fire event you could provoke is a HACK, not a BUILDABLE).

---

## 5. VR cvars a mod can read or set

From `src/QzDoom/VrCommon.h:16-27`:
`vr_control_scheme`, `vr_move_use_offhand`, `vr_weaponRotate`, `vr_snapTurn`, `vr_ipd`,
`vr_weaponScale`, `vr_teleport`, `vr_switch_sticks`, `vr_secondary_button_mappings`,
`vr_two_handed_weapons`, `vr_crouch_use_button`.

`vr_two_handed_weapons` in particular is an engine-level toggle, not something a mod invents.

---

## 6. Standing rules for every report

- **Cite or say you couldn't.** `path:line` at the `qzd17.3` tag. "I could not find this"
  is an acceptable and useful answer; a confident guess is not.
- **Label every mechanic** EXISTS / BUILDABLE / HACK / WALL, and cost S / M / L.
- **A WALL must name the missing piece**, not just say it's impossible.
- **Do not cite REMA.** `E:\DOOMWork\UZDXREMA` is a different, heavily modified engine.
  Anything found only there is a WALL here.
