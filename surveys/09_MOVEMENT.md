# 09 — Advanced movement mechanics as a MOD

Engine surveyed: tag `qzd17.3` of `github.com/emawind84/gzdoom` (QuestZDoom PC edition).
All citations are repo-relative at that tag. Sprite-layer only: nothing below authors a map,
a linedef, a sector or a thing. Walls are found at runtime by trace or by block iterator.

Builds on `_BASELINE_FACTS.md` (offhand-only hand pose, no hand velocity, 35 Hz script sampling).

---

## 1. Verdict

The whole player movement loop — `PlayerThink`, `MovePlayer`, `CheckJump`, `CheckCrouch`,
`CheckMoveUpDown`, `CalcHeight`, `ApplyAirControl` — is **ZScript, virtual, and overridable**
(`wadsrc/static/zscript/actors/player/player.zs:1734`, `:1340`, `:1514`, `:1194`, `:1565`, `:604`,
`:1328`), so every mechanic in this brief is script work, not engine work. `LineTrace` returns
`HitLine` and `LineSide` (`wadsrc/static/zscript/actors/actor.zs:50`, `:59`), and `Line.delta`
is readable (`wadsrc/static/zscript/mapdata.zs:210`), so **a true 2D wall normal is derivable
at runtime with no map changes** — that one fact is what makes wall running good rather than
a fudge. Gravity is per-actor and writable (`Actor.Gravity`, `actor.zs:176`, consumed at
`src/playsim/actorinlines.h:85-88`), friction is killable per-actor (`bNOFRICTION`,
`src/scripting/thingdef_data.cpp:324`), and `P_SlideMove` already rewrites `Vel` to the
wall-parallel component (`src/playsim/p_map.cpp:3328-3329`), so holding a player against a
wall while keeping tangential speed is close to free. The one engine-specific landmine is
QuestZDoom's VR momentum kill: `MovePlayer` zeroes XY velocity every tic on the ground unless
`vr_momentum` or `player.keepmomentum` is set (`player.zs:1374-1383`) — every mechanic here must
set `player.keepmomentum = true` each tic or it is silently deleted. Comfort is the real
constraint, not capability: artificial linear acceleration is the nausea source, and the only
per-eye overlay script can drive is the flat full-screen blend `player.BlendR/G/B/A`
(`player.zs:3120-3123`, drawn per eye at `src/rendering/hwrenderer/hw_entrypoint.cpp:200-201`),
so every mechanic gets a blink/fade variant rather than a tunnelling vignette.

---

## 2. What already exists

**The player think path is ZScript and virtual.**

| Thing | Where |
|---|---|
| `PlayerPawn.PlayerThink()` | `wadsrc/static/zscript/actors/player/player.zs:1734` |
| `HandleMovement()` | `player.zs:1612` (calls MovePlayer, CheckJump, CheckMoveUpDown) |
| `MovePlayer()` | `player.zs:1340` |
| `CheckJump()` | `player.zs:1514` |
| `CheckMoveUpDown()` | `player.zs:1565` |
| `CheckCrouch(bool)` | `player.zs:1194` |
| `CheckPitch()` | `player.zs:1460` |
| `CalcHeight()` | `player.zs:604` |
| `ApplyAirControl(out movefactor, out bobfactor)` | `player.zs:1328` |
| `Actor.FallAndSink(grav, oldfloorz)` — virtual, applies gravity to `Vel.Z` | `wadsrc/static/zscript/actors/actor.zs:522`; native body `src/playsim/p_mobj.cpp:2954-3029`; called from `P_ZMovement` at `src/playsim/p_mobj.cpp:2604` |

Native `P_PlayerThink` does almost nothing itself — it dispatches to the ZScript virtual
(`src/playsim/p_user.cpp:1356-1360`).

**QuestZDoom's VR momentum kill is already in `MovePlayer`.**

```
player.zs:1374   if (!vr_momentum && !player.keepmomentum
player.zs:1375   && player.onground && friction == ORIG_FRICTION)
player.zs:1377       vel.XY *= 0.0001;
player.zs:1378       Speed = Default.Speed * 8;
```
`vr_momentum` defaults false (`src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:120`).
So the stock VR feel is "no momentum, 8× per-tic speed, instant start/stop". `player.keepmomentum`
(`player.zs:3108`, native `src/playsim/d_player.h:414`) opts out of the kill — and the engine
clears it automatically once speed drops below `vr_momentum_threshold` (default 1.0),
`player.zs:1385-1387`. That auto-clear is the natural end of a dash or slide.

**Physical crouch is already engine-driven.** With `vr_crouch_use_button` false the OpenXR
layer writes `player->crouching = 10` and `player->crouchfactor = hmdHeight / defaultViewHeight`
every frame (`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:574-579`), and
`CheckCrouch` honours the `crouching == 10` sentinel (`player.zs:1209-1213`). So a mod that reads
`player.crouchfactor` gets real physical crouch depth for free in that mode.

**Stick and head translation are separate.** In VR the stick goes through `VR_GetMove`'s
joystick pair into `cmd.forwardmove` / `cmd.sidemove` (`src/g_game.cpp:834-842`), while
*physical* head translation is applied by the renderer as a save/override/`P_XYMovement`/restore
on `player->mo->Vel` (`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:665-688`;
OpenVR equivalent `src/rendering/gl/stereo3d/gl_openvr.cpp:2613-2637`). So **`cmd.forwardmove`
in VR is pure stick** — leaning does not pollute it. That is what makes stick-flick detection viable.

**Already-shipped movement modifiers a mod can reuse:** `PowerSpeed`
(`wadsrc/static/zscript/actors/inventory/powerups.zs:1164`, applied via `GetSpeedFactor` at
`player.zs:1320`), `PowerHighJump` (`powerups.zs:1827`, applied at `player.zs:1537-1548`),
`Player.JumpZ` property (`player.zs:68`, default 8 at `:120`), `bFLY` / `bNOGRAVITY`
flight through `CheckMoveUpDown` (`player.zs:1577-1590`).

**Wall sliding already preserves tangential speed.** `P_XYMovement` routes blocked player moves
into `P_SlideMove` (`src/playsim/p_mobj.cpp:2277`, `:2281`), and `FSlide::SlideMove` writes the
clipped wall-parallel move straight back into `mo->Vel` (`src/playsim/p_map.cpp:3328-3329`).
A player with velocity pressed into a wall keeps the along-wall component with no script help.

---

## 3. Building blocks

### 3.1 Where to inject each tic

Per-tic order inside one game tic (`src/p_tick.cpp:165-172`):

1. `P_PlayerThink` for every player — `src/p_tick.cpp:168` → ZScript `PlayerThink`
   (`src/playsim/p_user.cpp:1356`). This is where `MovePlayer` zeroes velocity and
   `CheckJump` adds jump velocity.
2. `EventHandler.WorldTick` — `src/p_tick.cpp:170` (`wadsrc/static/zscript/events.zs:177`).
3. `Level->Thinkers.RunThinkers` — `src/p_tick.cpp:172` → `AActor::Tick`, which runs
   **`Inventory.DoEffect` on every carried item** (`src/playsim/p_mobj.cpp:4005-4017`,
   declared `wadsrc/static/zscript/actors/inventory/inventory.zs:982`) and *then*
   `P_XYMovement` (`src/playsim/p_mobj.cpp:4344`) and `P_ZMovement`
   (`src/playsim/p_mobj.cpp:4372`, `:4435`).

**Hook points, in order of preference:**

- **`Inventory.DoEffect()` on a carried controller item** — runs after `MovePlayer` has already
  applied its velocity kill, and before `P_XYMovement` consumes `Vel`. Velocity written here lands
  this same tic and cannot be clobbered by `MovePlayer`. Per-player, net-safe, ordered by inventory
  order. **This is the right hook for a mod that must coexist with arbitrary other mods**, because
  it needs no player-class replacement.
- **Overriding `PlayerPawn.MovePlayer` / `CheckJump` / `ApplyAirControl` / `FallAndSink`** — cleanest
  and most precise, but requires replacing the player class, which collides with any other mod that
  does the same.
- **`EventHandler.WorldTick`** — fine for detection and bookkeeping; same timing window as
  `DoEffect` for velocity writes, but not per-player, so key everything on `players[i]` explicitly.

### 3.2 Moving the player

| API | Where | Notes |
|---|---|---|
| `Actor.Vel` (vector3, writable) | `actor.zs:112` | The one reliable mover. Write it, then set `player.keepmomentum = true`. |
| `Actor.Thrust(speed, angle)` | `actor.zs:860` | **Adds `ThrustAngleOffset`** (`src/playsim/actor.h:1607-1613`), which the OpenVR layer sets from the offhand controller when `vr_move_use_offhand` (`src/rendering/gl/stereo3d/gl_openvr.cpp:2603-2606`) and which is **not exposed to ZScript** (no `DEFINE_FIELD`). So `Thrust` has an unreadable direction bias in VR. Do not use it for a directional dash. |
| `A_ChangeVelocity` | `wadsrc/static/zscript/actors/actions.zs:63` | **Trap: `if (!vr_recoil) return;` on the player** (`actions.zs:77`), and `vr_recoil` defaults false (`src/playsim/p_actionfunctions.cpp:90`). Unusable as a mover. It does show the correct idiom: it sets `player.keepmomentum = true` (`actions.zs:78`). |
| `Actor.TryMove(newpos, dropoff, missilecheck, tm)` | `actor.zs:831` | Collision-checked relocation; use for blink/teleport comfort variants. |
| `Actor.CheckMove(newpos, flags, tm)` | `actor.zs:832`, flags `PCM_DROPOFF/NOACTORS/NOLINES` at `constants.zs:545-547` | Probe a position without moving — ledge and clearance tests. |
| `Actor.SetOrigin(newpos, moving)` | `actor.zs:753` | Unchecked relocation. Only after a `CheckMove`. |
| `Actor.Vel3DFromAngle` | `src/playsim/actor.h:1620` (native) | |

### 3.3 Gravity and friction — what a mod may change, and what is baked in

**Changeable:**

| Knob | Where | Effect |
|---|---|---|
| `Actor.Gravity` (double, writable) | `actor.zs:176`, default `1` at `actor.zs:432` | Per-actor multiplier. `GetGravity() = Level->gravity * Sector->gravity * Gravity * 0.00125` — `src/playsim/actorinlines.h:85-88`. Per-player, net-safe, instant. **The right lever for wall-run and hang-time.** |
| `bNOGRAVITY` | `MF_NOGRAVITY`, `src/scripting/thingdef_data.cpp:127` | Zeroes gravity entirely, but on a player above the floor it *also* enables FlyBob vertical bobbing and applies `FRICTION_FLY` (0.918) to `Vel.Z` — `src/playsim/p_mobj.cpp:2630-2645`. The bobbing is a VR nausea source. Prefer `Gravity = small` over `bNOGRAVITY`. |
| `bNOFRICTION` (`MF8_NOFRICTION`) | `src/scripting/thingdef_data.cpp:324` | Early-returns out of all ground friction, air friction and the stop-speed clamp — `src/playsim/p_mobj.cpp:2442-2444`. A momentum-preserving slide in one flag. Also suppresses water sink damping (`p_mobj.cpp:2646`) and icy bounce (`p_map.cpp:3037`). |
| `level.gravity`, `level.aircontrol`, `level.airfriction` | `wadsrc/static/zscript/doombase.zs:454-456` — all declared `play`, i.e. **writable** | Global, shared by every player. **Writing `level.aircontrol` does not recompute `airfriction`** — the native `AirControlChanged()` (`src/g_level.cpp:1927-1938`) is not exposed, so a mod must write `level.airfriction` itself with the same formula: `airfriction = (aircontrol <= 1/256) ? 1.0 : aircontrol * -0.0941 + 1.0004`. In multiplayer, don't. |
| `ApplyAirControl(out movefactor, out bobfactor)` override | `player.zs:1328-1332` | **Per-player air control with no global side effects.** The correct lever if you own the player class. |
| `Sector.friction`, `Sector.movefactor` | `wadsrc/static/zscript/mapdata.zs:370` | Writable, but this is a world-geometry edit in all but name. Out of scope under the hard rule. |
| `Player.JumpZ` property / `PowerHighJump.Strength` | `player.zs:68`, `powerups.zs:1827` | Jump impulse. |
| `player.keepmomentum` | `player.zs:3108` | Opts out of the VR velocity kill. **Side effect: in the non-kill branch `Speed = Default.Speed` instead of `Default.Speed * 8`** (`player.zs:1378` vs `:1382`), so stick walking is 8× weaker for as long as you hold it. Hold it only for the duration of a dash/slide. |

**Baked in — a mod cannot change these:**

- `ORIG_FRICTION = 0.90625`, `ORIG_FRICTION_FACTOR = 0.03125` — compile-time, `src/doomdef.h:274-275`,
  mirrored as ZScript consts at `actor.zs:88-89`. The *application* of friction in
  `P_XYMovement` (`src/playsim/p_mobj.cpp:2531-2533`) is native and has no virtual.
- The `STOPSPEED` stop clamp and `MinVel` zeroing — `src/playsim/p_mobj.cpp:2497-2512`, `:2546-2555`. Native.
  Only `bNOFRICTION` bypasses them.
- `FRICTION_FLY = 0.91797` — `src/doomdef.h:277`.
- The air-friction formula itself (`aircontrol * -0.0941 + 1.0004`) — `src/g_level.cpp:1936`. You can
  write the result, not change the curve.
- `MAXPLMOVE` clamping of `cmd.forwardmove`/`sidemove` — `src/g_game.cpp:863-870`. Script reads the
  clamped value.
- The per-frame head-translation `P_XYMovement` call in the renderer
  (`hw_openxrdevice.cpp:665-688`). It saves and restores `Vel`, so it will not eat your dash velocity,
  but it *will* keep shoving the player when they move their head — including while wall-running.
- `ThrustAngleOffset` — written by the renderer, no ZScript field.

### 3.4 Reading the wall — the trace API, and yes you get a normal

`Actor.LineTrace(angle, distance, pitch, flags, offsetz, offsetforward, offsetside, out FLineTraceData data)`
— `wadsrc/static/zscript/actors/actor.zs:805`, native `src/playsim/p_map.cpp:5165`,
thunk `src/scripting/vmthunks_actors.cpp:1272`.

`FLineTraceData` (`actor.zs:36-62`) gives you:

- `HitType` — `TRACE_HitWall` / `HitFloor` / `HitCeiling` / `HitActor` / `HasHitSky` (`actor.zs:38-47`)
- `HitLine` — **the actual `Line`** (`actor.zs:51`), populated at `src/playsim/p_map.cpp:5274`
- `LineSide` — 0 front / 1 back, set only on `TRACE_HitWall` (`actor.zs:58`, `p_map.cpp:5278-5280`)
- `LinePart` — upper(0) / middle(1) / lower(2) (`actor.zs:59`, `p_map.cpp:5281-5300`)
- `HitLocation`, `HitDir`, `Distance` (`actor.zs:55-57`)
- `HitSector`, `Hit3DFloor`, `HitTexture` (`actor.zs:52-54`)
- `SectorPlane` — 0 floor / 1 ceiling (`actor.zs:60`)

**It does not hand you a normal. You derive one, exactly, and this is cheap:**

`Line` exposes `v1`, `v2`, `delta` (`wadsrc/static/zscript/mapdata.zs:210-211`). The engine's own
side test is `P_PointOnLineSidePrecise`: `(p.y - v1.y)*delta.x + (v1.x - p.x)*delta.y > eps` → back
(`src/playsim/p_maputl.h:42-45`). Front side is therefore the half-plane in direction
`(delta.y, -delta.x)`. So:

```
Vector2 d = data.HitLine.delta;
Vector2 n = (d.y, -d.x).Unit();      // outward normal of the FRONT side
if (data.LineSide == Line.back) n = -n;   // n now points back toward the tracer
```

That is a surveyed-geometry-exact 2D wall normal, derived entirely at runtime. It is 2D only —
Doom walls are vertical, so there is no vertical component to recover and none is needed.

**Richer alternative:** `LineTracer` (`wadsrc/static/zscript/doombase.zs:306-321`) with an overridable
`TraceCallback()` and a `TraceResults` struct carrying `Side`, `Tier`, `HitLine`, `HitPos`,
`Fraction` (`doombase.zs:278-304`). Use it when you want to skip non-solid hits (e.g. ignore
two-sided lines whose back sector is walkable) rather than stopping on the first thing.

**Cheaper alternative for wall-run, and the one to actually ship:** `BlockLinesIterator`
(`doombase.zs:229-238`) — `Create(origin, checkradius)` walks every line in the blockmap cells
around the player. One iteration per tic replaces an 8-trace fan, and gives `CurLine` directly,
so you get `delta`, `flags` (`ML_BLOCKING`, `ML_TWOSIDED`, `ML_BLOCKEVERYTHING` — `mapdata.zs:175-203`),
`frontsector`/`backsector` and `sidedef[]` for free. Compute perpendicular distance from the
player to each candidate line yourself; keep the nearest one whose normal opposes your velocity.

**Deciding a line is a runnable wall (no map data, all runtime):**

1. `ML_BLOCKING` set, or not `ML_TWOSIDED` → one-sided wall, always runnable.
2. If `ML_TWOSIDED`: compare `frontsector`/`backsector` floor and ceiling at the hit point via
   `Sector.floorplane.ZatPoint(xy)` / `ceilingplane.ZatPoint(xy)` (`SecPlane.ZatPoint`,
   `mapdata.zs:271`). If the opening is shorter than the player's height, it is a wall.
3. Verticality: Doom lines are always vertical, so any `TRACE_HitWall` is a vertical surface.
   There is no sloped-wall case to handle.
4. Sky check: pass `TRF_NOSKY` (`constants.zs:957`) so you don't try to run on a sky edge.

### 3.5 Reading VR input: body, button, stick

| Input | How | Reliability |
|---|---|---|
| **Button** | `player.cmd.buttons & BT_*` (`constants.zs:834-869`) with `player.oldbuttons` (`player.zs:3064`) for edges. `BT_USER1..4` (`:859-862`) are free; `BT_JUMP`, `BT_CROUCH`, `BT_RUN`, `BT_OFFHANDATTACK` (`:866`) are in use. | **Highest.** No false positives, no 35 Hz loss. Default every mechanic to a button. |
| **Stick flick** | `cmd.forwardmove` / `cmd.sidemove` — pure stick in VR (`src/g_game.cpp:834-842`), clamped to `MAXPLMOVE` (`:863-870`). Detect a double-tap: value crosses zero then exceeds ~80% of max within ~8 tics. | **Good.** Clean signal, but a double-tap costs you ~0.2 s of latency, and players who ride the stick at full deflection will mis-fire it. Make the threshold and window cvars. |
| **Physical crouch** | `player.crouchfactor` (`player.zs:3127`) when `vr_crouch_use_button` is false — the engine writes it from head height each frame (`hw_openxrdevice.cpp:574-579`). Or read `HmdPosition.y` directly (metres; `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:445`, raw tracking-space pose from `VR_SetHMDPosition`, `src/QzDoom/qzdoom_common.cpp:93-95`, fed from the raw HMD matrix at `src/rendering/gl/stereo3d/gl_openvr.cpp:2400`). | **Good for a held state, poor as a trigger.** Depth is reliable; *rate* of crouch is 2-3 samples at 35 Hz (§3 of BASELINE). Use "below X% of standing height" as a gate, never "crouched fast" as a trigger. |
| **Lean** | Differencing `HmdPosition.xz` across tics, in metres. | **Weakest, and it fights the engine.** Head translation is already converted to movement every frame by the renderer's own `P_XYMovement` call (`hw_openxrdevice.cpp:665-688`), so a lean *already* moves the player; a lean gesture is therefore read against a moving baseline. Also: standing-height and play-space origin vary per player, and nothing recentres `HmdPosition` for you. Use lean only as a **modifier on an already-committed mechanic** (which way you leave a wall run), never as the trigger. |
| **Offhand motion** | `OffhandPos` differenced per tic (BASELINE §1b — no velocity field). | Usable, 2-4 samples per gesture. Competes with weapon handling. Not recommended for locomotion. |

Calibration note: `HmdPosition` is raw tracking-space metres. A mod that wants "is the player
crouching" must learn the standing height itself (running max over the first few seconds, or a
cvar) — the engine's own code does exactly this hack (`hw_openxrdevice.cpp:157-171`).

### 3.6 Feedback channels available to script

- **Full-screen blend** — `player.BlendR/G/B/A` (`player.zs:3120-3123`, native
  `src/playsim/d_player.h:434`, thunk `src/playsim/p_user.cpp:1926`). Composited into
  `V_CalcBlend` (`src/rendering/2d/v_blend.cpp:370`) and drawn by `V_DrawBlend` **inside the
  per-eye loop** (`src/rendering/hwrenderer/hw_entrypoint.cpp:200-201`). So it works in VR.
  It is a flat full-screen `Dim` (`src/rendering/2d/v_blend.cpp:392-393`) — a fade, not a vignette.
- **View height** — `player.viewheight` / `player.deltaviewheight` (`player.zs:681-699`) feed
  `player.viewz`, which becomes the camera position (`src/rendering/r_utility.cpp:1023`) and
  then `CenterEyePos` (`:1236`). **Writing viewheight moves the VR camera vertically.** Real lever,
  real hazard.
- **View offset** — `Actor.SetViewPos(offset, flags)` (`actor.zs:1278`, flags `VPSF_*` at
  `constants.zs:431-434`), applied at `src/rendering/r_utility.cpp:648-673` before `CenterEyePos`
  is taken. Lets you shove the VR camera laterally. Comfort-hostile; off by default.
- **View roll** — `A_SetViewRoll` (`actor.zs:1277`) → `viewPoint.Angles.Roll` →
  `HWAngles.Roll` (`r_utility.cpp:1230`) → used by the VR eye pose
  (`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:205-206`). **Artificial roll does
  reach the headset.** Default it off for every mechanic.
- **Haptics** — per BASELINE §4, not reachable from ZScript. Treat as unavailable.
- **No eye-space vignette.** Script 2D drawing in VR is composited as a world-locked quad at
  `vr_hud_distance`/`vr_hud_scale` (`hw_openxrdevice.cpp:299-301`), not as an eye overlay, so a
  tunnelling ring drawn in `RenderOverlay` would appear as a floating billboard. See §5.

---

## 4. Mechanic designs

Throughout: "controller item" means a `CustomInventory`/`Inventory` subclass given to the player on
spawn, whose `DoEffect()` is the per-tic hook (§3.1). Every design assumes
`player.keepmomentum = true` is re-asserted each tic while the mechanic is active, and relies on the
engine clearing it automatically below `vr_momentum_threshold` (`player.zs:1385-1387`).

---

### 4.1 Ground dash

**What the player does.** Presses a bound button (recommended), or double-flicks the movement stick
in the direction they want.

**How script detects it.** In `DoEffect`: rising edge of `cmd.buttons & BT_USER1` against
`player.oldbuttons`. Stick variant: track `cmd.sidemove`/`cmd.forwardmove`
(`src/g_game.cpp:834-842`); a dash fires when the magnitude goes below 20% of `MAXPLMOVE` and back
above 80% within 8 tics with the same sign.

**How it resolves.** Direction = the player's facing (`angle`) combined with the current
`cmd.forwardmove`/`sidemove` ratio, normalised — **not `Thrust`**, which carries the unreadable
`ThrustAngleOffset` (§3.2). Write `Vel.XY = dir * dashSpeed` directly, set
`player.keepmomentum = true`, set `bNOFRICTION` for `dashTics` so friction does not eat it
(`src/playsim/p_mobj.cpp:2442-2444`), then clear the flag and let the normal friction path
(`p_mobj.cpp:2531-2533`) decelerate the player back to walking. Gate on `player.onground`
(`player.zs:1369`) and a cooldown counter.

**Feel tuning knobs.** `dashSpeed`, `dashTics`, cooldown, whether `bNOFRICTION` holds for the whole
dash or releases at 60%, whether direction locks at the start or steers during.

**Comfort variant.** *Blink dash.* Instead of velocity: `CheckMove` the destination
(`actor.zs:832`, with `PCM_DROPOFF` cleared so you don't blink off a ledge), raise
`player.BlendA` to 1.0 over 2 tics, `SetOrigin` (`actor.zs:753`) or `TryMove` (`actor.zs:831`) to
the destination, fade `BlendA` back over 3 tics. No optical flow at all, so no vestibular conflict.
The engine already ships a comfort teleport (`vr_teleport`,
`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:634-651`), so this is a familiar idiom
for the user. Make blink-vs-smooth a cvar and **default it to blink**; smooth dash must additionally
ramp velocity over 3-4 tics rather than stepping it, because the step is the nausea, not the speed.

**Label: BUILDABLE. Cost: S.**

---

### 4.2 Air dash

**What the player does.** Same input as the ground dash, while airborne.

**How script detects it.** Same edge detection, but gated on `!player.onground` (`player.zs:1369`)
and on an `airDashesUsed` counter that resets on landing.

**How it resolves.** Write `Vel.XY` directly as in 4.1. Additionally set `Vel.Z = 0` at the moment of
the dash to give it a clean horizontal read, then restore gravity. Air friction is the thing to watch:
if `level.aircontrol > 1/256` the engine multiplies XY by `airfriction` every tic
(`src/playsim/p_mobj.cpp:2451-2465`), which will bleed the dash. `bNOFRICTION` skips that path
entirely (`p_mobj.cpp:2442`), which is the clean fix and needs no global cvar write.

**Feel tuning knobs.** Air dash count per jump, whether `Vel.Z` is zeroed, hang-time (set
`Gravity = 0.2` for the dash's duration via `actor.zs:176`), horizontal speed relative to the
ground dash.

**Comfort variant.** The blink variant is *worse* in the air than on the ground — a blink with no
ground plane for reference is badly disorienting. Comfort air dash instead = short, slow, and with
a hard duration cap (≤6 tics), plus a partial `BlendA` dim (0.35) for the duration so peripheral
flow is damped without a full blackout. Offer "air dash off" as a setting; it is the one mechanic
here some players will simply not tolerate.

**Label: BUILDABLE. Cost: S.**

---

### 4.3 Slide

**What the player does.** Crouches — physically, if `vr_crouch_use_button` is false, or on the
crouch button — while moving at speed.

**How script detects it.** In `DoEffect`: `player.crouchfactor < 0.7` (`player.zs:3127`; in physical
crouch mode the engine writes it from head height every frame,
`hw_openxrdevice.cpp:574-579`) **and** `Vel.XY.Length() > slideEntrySpeed` **and**
`player.onground`. Physical crouch is the better input here precisely because it is a *held state*,
not a gesture — §3.5 says crouch depth is reliable where crouch *rate* is not, and a slide only
needs depth.

**How it resolves.** Set `bNOFRICTION` (`thingdef_data.cpp:324`) so `P_XYMovement`'s friction block
is skipped entirely (`p_mobj.cpp:2442-2444`), re-assert `player.keepmomentum = true` each tic, and
apply your own per-tic decay (`Vel.XY *= 0.97`) so you control the curve instead of inheriting
`ORIG_FRICTION`. Suppress steering by zeroing or scaling your own contribution — you cannot stop
`MovePlayer` from thrusting on `cmd.forwardmove` without overriding it (`player.zs:1401-1437`), so
either override `MovePlayer` (if you own the player class) or counter-thrust in `DoEffect`, which
runs after it. Exit when speed drops below threshold or the player stands up; clear `bNOFRICTION`.
Slide under low ceilings is free: `CheckCrouch` already refuses to stand up when
`pos.Z + height >= ceilingz` (`player.zs:1221`).

**Feel tuning knobs.** Entry speed threshold, decay rate, whether the slide gets a speed *boost* on
entry, steer authority during the slide, minimum slide duration.

**Comfort variant.** **Do not dip the camera.** The obvious flourish — drop `player.viewheight`
(`player.zs:681`) for the slide — moves the VR camera (`r_utility.cpp:1023`, `:1236`) with no matching
head movement, and that is textbook vestibular conflict. In VR the player's head is *already* low,
because they physically crouched. Comfort slide = constant velocity decay (constant velocity is far
easier on the inner ear than acceleration), no viewheight change, no roll, and a cap on entry speed.
If you want the slide to read visually, do it on the weapon psprite, not the camera.

**Label: BUILDABLE. Cost: M.**

---

### 4.4 Backstep

**What the player does.** Button press, or back-flick on the stick.

**How script detects it.** Rising edge of a bound button, or `cmd.forwardmove` crossing strongly
negative twice within the double-tap window (§3.5).

**How it resolves.** Identical to 4.1 with direction = `angle + 180`, a shorter duration, and a
lower speed. Worth adding: a `CheckMove` probe at the destination with `PCM_DROPOFF` *not* set, so
a backstep cannot walk you off a ledge you cannot see.

**Feel tuning knobs.** Distance, duration, i-frames (if the mod has them), whether it cancels a
reload.

**Comfort variant.** Backward optical flow is the worst case in VR — the player sees the world
rushing toward them from a direction their head is not facing. **Default the backstep to blink**
(fade / `SetOrigin` / fade, as in 4.1), with smooth available but not the default. If smooth is
chosen, cap distance hard and keep duration ≥6 tics so the acceleration is gentle.

**Label: BUILDABLE. Cost: S.**

---

### 4.5 Wall run

**What the player does.** Jumps toward a wall and holds the movement stick along it (and, optionally,
leans into the wall to extend it).

**How script detects it.** Per tic in `DoEffect`, while `!player.onground`:

1. Build a `BlockLinesIterator` around the player (`doombase.zs:235`, radius ≈ `radius + 24`).
2. For each `CurLine`: reject if not blocking (§3.4 step 1-2 — `ML_BLOCKING` /
   `!ML_TWOSIDED` / opening shorter than the player). Compute the perpendicular distance from
   `pos.xy` to the line, and the normal `n = (delta.y, -delta.x).Unit()`, flipped so it points at
   the player.
3. Keep the nearest candidate within `wallGrabDist` whose normal satisfies
   `(Vel.XY.Unit() dot n) < -0.2` (you are moving into it) and
   `abs(Vel.XY.Unit() dot tangent) > 0.5` (you have along-wall speed, not just a face-plant).
4. Confirm with a single `LineTrace` along `-n` from the player's chest height
   (`actor.zs:805`, flags `TRF_NOSKY | TRF_THRUACTORS`), check `data.HitType == TRACE_HitWall`
   and `data.Distance` — this rejects lines the blockmap offered you through a corner.

**How it resolves.**

- **Hold against the wall:** add a small inward velocity each tic, `Vel.XY += -n * stickForce`.
  You do not need to zero the outward component yourself: `P_SlideMove` already rewrites `Vel` to
  the wall-parallel component when the move is blocked (`src/playsim/p_map.cpp:3328-3329`,
  reached from `src/playsim/p_mobj.cpp:2277`). This is why wall running is cheap on this engine.
- **Reduce gravity:** `Gravity = wallRunGravity` (e.g. 0.12) — `actor.zs:176`,
  consumed at `src/playsim/actorinlines.h:85-88`. Restore to `1` on exit. **Do not use
  `bNOGRAVITY`**: on a player above the floor it switches on FlyBob vertical bobbing and
  `FRICTION_FLY` damping of `Vel.Z` (`src/playsim/p_mobj.cpp:2630-2645`), and the bobbing is a
  nausea source you did not ask for.
- **Drive along the wall:** project the player's stick input onto the wall tangent and add it as
  velocity. `player.keepmomentum = true` each tic.
- **Exit jump:** on `BT_JUMP` rising edge, `Vel.XY = (tangent * exitForward + n * exitOut).Unit() * exitSpeed`
  and `Vel.Z = JumpZ * exitUp` — note `CheckJump` will not fire for you, because it requires
  `player.onground` (`player.zs:1532`), so you must do the whole jump yourself.
- **Exit conditions:** the trace stops finding the wall, a timer expires, the player lets go of
  the along-wall input, or `Vel.XY` tangential magnitude falls below a floor. Restore `Gravity = 1`
  and clear any flags on every exit path, including death and level change.

**Feel tuning knobs.** `wallRunGravity`, max duration, `stickForce`, whether tangential speed is
held constant or decays, minimum entry speed, re-grab cooldown on the same `Line` (store
`HitLine.Index()`, `mapdata.zs:239`, to stop re-attaching to the wall you just left).

**Caveat you cannot design away:** the renderer's per-frame head-translation
`P_XYMovement` (`hw_openxrdevice.cpp:665-688`) keeps pushing the player while they move their head,
including during the run. It saves and restores `Vel`, so it will not eat your velocity, but a
player who leans away from the wall will physically translate off it. Build the re-grab window to
tolerate a tic or two of lost contact.

**Comfort variant.** This is the mechanic most likely to make people ill, because it is sustained
and lateral. Comfort wall run =

- **No camera roll.** The cinematic instinct is `A_SetViewRoll` (`actor.zs:1277`) to bank into the
  wall; it reaches the headset (`r_utility.cpp:1230` → `hw_openxrdevice.cpp:205-206`) and it is a
  reliable way to make someone take the headset off. Default 0, expose as a slider that starts at 0.
- **No `SetViewPos` lateral shove** (`actor.zs:1278`).
- **Constant tangential speed, not accelerating.** Steady optical flow is tolerable; changing flow
  is not.
- **Hard duration cap** (2 s default) so the exposure is bounded.
- **Partial blend dim** at `BlendA ≈ 0.3` for the duration (`player.zs:3123`) to damp peripheral
  flow — the nearest thing to a vignette this engine gives script (§5).
- **Comfort-max setting: wall *hang* instead of wall *run*** — same detection, same gravity
  reduction, but tangential velocity is clamped to zero, so the player is stationary against the
  wall and only the exit jump moves them. Preserves the traversal (reach a ledge you otherwise
  can't) with almost no optical flow.

**Label: BUILDABLE. Cost: L.**

---

### 4.6 Double jump

**What the player does.** Presses jump again while airborne.

**How script detects it.** `cmd.buttons & BT_JUMP` rising edge against `player.oldbuttons`
(`player.zs:3064`) while `!player.onground` (`player.zs:1369`), with a `jumpsUsed` counter that
resets on landing. The stock `CheckJump` will ignore the press for you — it requires
`player.onground && player.jumpTics == 0` (`player.zs:1532`) — so there is no conflict to resolve.
One trap: `CheckCrouch` clears `BT_CROUCH` whenever `BT_JUMP` is held (`player.zs:1199-1202`),
so do not try to read a crouch+jump chord.

**How it resolves.** `Vel.Z = JumpZ * 35 / TICRATE * doubleJumpFactor` — mirroring the stock
computation at `player.zs:1535` — plus `player.jumpTics = -1` and `bOnMobj = false`
(`player.zs:1551-1553`) so the engine's own bookkeeping stays consistent. If you want air steering
on the second jump, that is the `ApplyAirControl` override (`player.zs:1328`) or a transient
`level.aircontrol`/`level.airfriction` pair (§3.3) — prefer the override.

**Feel tuning knobs.** `doubleJumpFactor`, number of extra jumps, whether the second jump zeroes
downward velocity first (it should — a double jump that cannot save a bad fall feels broken),
whether it adds horizontal velocity in the stick direction.

**Comfort variant.** The gentlest mechanic here: vertical motion with the head upright is the best
tolerated axis, and the arc is self-limiting. Comfort variant = cap the second jump's `Vel.Z` so
the total apex is not much higher than a single jump, and **do not add a landing viewheight kick**
(`player.deltaviewheight`, `player.zs:681`) — the stock engine already applies one on hard landings
(`player.zs:1476`), and in VR it is a camera move with no head move. If the mod ships a landing kick
at all, make it a weapon-psprite dip, not a camera dip.

**Label: BUILDABLE. Cost: S.**

---

### 4.7 Summary table

| Mechanic | Label | Cost | Best input | Comfort default |
|---|---|---|---|---|
| Ground dash | BUILDABLE | S | Button | Blink |
| Air dash | BUILDABLE | S | Button | Short + dim, off-switch |
| Slide | BUILDABLE | M | Physical crouch (held state) | No camera dip |
| Backstep | BUILDABLE | S | Button | Blink |
| Wall run | BUILDABLE | L | Auto-detect + stick | Wall *hang*, no roll |
| Double jump | BUILDABLE | S | Button | Capped apex, no landing kick |
| Eye-space comfort vignette | **WALL** | — | — | — (see §5) |
| Script-driven haptic feedback | **WALL** | — | — | — (BASELINE §4) |

---

## 5. Walls

**W1 — No eye-space comfort vignette.** The standard VR locomotion comfort tool is a tunnelling
vignette that darkens the periphery of each eye during motion. Script cannot draw one.
The only per-eye overlay a mod can drive is `player.BlendR/G/B/A`
(`player.zs:3120-3123` → `src/rendering/2d/v_blend.cpp:370` → `V_DrawBlend`,
`src/rendering/hwrenderer/hw_entrypoint.cpp:200-201`), and `V_DrawBlend` is a flat full-screen
`Dim` over the whole viewport (`src/rendering/2d/v_blend.cpp:392-393`) — uniform, no radial falloff.
Script 2D drawing (`RenderOverlay`) is not an alternative: in VR it is composited as a world-locked
quad at `vr_hud_distance` / `vr_hud_scale` (`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:299-301`),
so a drawn ring would float in the world rather than mask the eye.
**Exact missing piece:** a per-eye radial mask in the post-process chain with a ZScript-settable
strength — e.g. a `vr_vignette`-style uniform applied in `screen->PostProcessScene`
(`src/rendering/hwrenderer/hw_entrypoint.cpp:198`) and a `PlayerInfo` field to drive it.
Until then, the comfort fallback is uniform dimming via `BlendA`, which costs the whole image.

**W2 — No haptics from script.** Per BASELINE §4, confirmed here: `VR_HapticEvent` is called only
from hardcoded native sites (e.g. `src/playsim/p_user.cpp:1301-1304` for heal/heartbeat) and has
no ZScript thunk. A dash, a wall-grab and a landing therefore have no tactile confirmation, which
in VR is a real loss — haptic onset is one of the cheapest ways to make a locomotion mechanic
*feel* grounded. **Exact missing piece:** a ZScript-callable `PlayerInfo.Vibrate(hand, duration, intensity)`
thunking to `VRMode::Vibrate` (`src/common/rendering/hwrenderer/data/hw_vrmodes.h:94`).

**W3 — `A_ChangeVelocity` is dead on the player.** `actions.zs:77` short-circuits on `!vr_recoil`,
and `vr_recoil` defaults false (`src/playsim/p_actionfunctions.cpp:90`). Any mod code (or borrowed
mod code) that moves the player with `A_ChangeVelocity` silently does nothing in a default install.
Not a blocker — write `Vel` directly — but it will bite anyone porting a flat-screen movement mod,
and it is worth stating loudly.

**W4 — `Thrust` has an unreadable direction bias.** `AActor::Thrust` adds `ThrustAngleOffset`
(`src/playsim/actor.h:1607-1613`), which the OpenVR path sets from the offhand controller when
`vr_move_use_offhand` (`src/rendering/gl/stereo3d/gl_openvr.cpp:2603-2606`), and the field has no
`DEFINE_FIELD`. So script can neither read nor cancel it. **Exact missing piece:** a readonly
ZScript field for `ThrustAngleOffset`, or a `ThrustAbsolute()` that skips it. Workaround is total
(write `Vel`), so this is a WALL only for code that insists on `Thrust`.

**W5 — `level.aircontrol` has no recompute hook.** The field is writable from ZScript
(`wadsrc/static/zscript/doombase.zs:455`) but `FLevelLocals::AirControlChanged()`
(`src/g_level.cpp:1927-1938`) is not exposed, so writing `aircontrol` alone leaves `airfriction`
stale. A mod must reimplement the formula. Minor, and documented in §3.3, but it is exactly the kind
of thing that produces a mechanic that works on one map and not another (map `aircontrol` is applied
at `src/g_level.cpp:1849-1858`).

**W6 — No per-player gravity *scaling* hook that survives other mods.** `Actor.Gravity` is writable
and per-player, which covers it — but two mods that both write `Gravity` on the same player will
fight, and the last writer wins with no arbitration. Not an engine defect; a coordination hazard
worth stating because this engine family is explicitly shared ground.

---

## 6. Smallest prototype

**Goal: prove the wall normal is real, in the engine, in one build.** Everything else in this report
rests on §3.4; if the normal is wrong, wall run is a fudge.

One `EventHandler` plus one `Inventory` controller item, no player class replacement, no map data:

1. `Inventory` subclass `WallProbe`, given on `WorldThingSpawned` for the player. In `DoEffect()`:
2. `BlockLinesIterator.Create(owner, owner.radius + 32)` (`doombase.zs:235`). For each `CurLine`,
   skip unless `ML_BLOCKING` or `!ML_TWOSIDED` (`mapdata.zs:175-177`). Compute
   `n = (CurLine.delta.y, -CurLine.delta.x).Unit()` and perpendicular distance.
3. Confirm the nearest candidate with `owner.LineTrace(atan2(-n.y,-n.x), 64, 0, TRF_NOSKY|TRF_THRUACTORS, offsetz: 20, data: d)`
   (`actor.zs:805`); require `d.HitType == FLineTraceData.TRACE_HitWall`.
4. Flip the normal by `d.LineSide` (`actor.zs:58`) per §3.4.
5. **Verification that is not circular:** spawn a short-lived marker actor at
   `d.HitLocation + n * 8` each tic, and `Console.Printf` the dot product
   `n dot (owner.pos.xy - d.HitLocation.xy).Unit()`. **If the derived normal is correct that dot
   product is ≈ +1 on every wall, at every approach angle, on both one-sided and two-sided lines, and
   the markers stand off the wall rather than inside it.** A value near -1 means the `LineSide` flip
   is inverted; a value wandering near 0 means the blockmap handed you a line you are not actually
   facing and step 3's trace confirmation is not filtering.
6. Only once that reads +1 everywhere, add: `Gravity = 0.12` while a wall is detected and
   `!player.onground`, `Vel.XY += -n * 0.6`, `player.keepmomentum = true`. That is a wall hang —
   the comfort-max variant of 4.5 — in about 60 lines, and it is also the safest thing to put in
   front of a headset first.

Deliberately excluded from the prototype: camera roll, viewheight changes, `SetViewPos`, blink
fades. Those are polish on a mechanic that is not yet proven, and each is an independent nausea risk.

---

## 7. Open questions

1. **Is `HmdPosition` recentred by the engine's own recentre, or only by the runtime's?**
   `VR_SetHMDPosition` (`src/QzDoom/qzdoom_common.cpp:93-101`) stores the raw pose and never applies
   an origin offset, and `vr_height_adjust` is added downstream at use sites
   (`hw_openxrdevice.cpp:154`) rather than at the source. So script reading `HmdPosition` sees raw
   tracking space *without* `vr_height_adjust`. I did not find where (or whether) a VDXR recentre
   resets that origin. Any lean or crouch design needs to know, and it should be measured in the
   headset, not reasoned about.
2. **How much does the renderer's per-frame head-translation `P_XYMovement`
   (`hw_openxrdevice.cpp:665-688`) actually displace a wall-running player?** It runs at frame rate,
   not tic rate, so its magnitude depends on framerate and on how much the player's head moves.
   It save/restores `Vel` so it cannot eat the run, but it can break contact. Needs a measured
   number before `wallGrabDist` and the re-grab window can be chosen.
3. **Does the `Speed = Default.Speed` / `Default.Speed * 8` split (`player.zs:1378` vs `:1382`)
   make held `keepmomentum` feel broken?** Holding `keepmomentum` through a long slide drops stick
   walking to 1/8 authority. I expect this reads as "heavy", which may be *right* for a slide and
   wrong for a wall run. Only the headset can answer.
4. **Which one-sided-wall edge cases does the `BlockLinesIterator` + confirm-trace pair miss?**
   Specifically: 3D floors (`Hit3DFloor`, `actor.zs:53`), line portals (`Line.isLinePortal()`,
   `mapdata.zs:229`), and polyobjects. None of these are map *edits*, but a wall run that silently
   grabs a portal line will teleport the player mid-run.
5. **Can a mod own the player class at all in the target deployment?** Every "cleanest" option in
   §3.1 (overriding `MovePlayer`, `ApplyAirControl`, `FallAndSink`) needs the player class. If this
   has to coexist with mods that replace the player, everything must go through `Inventory.DoEffect`,
   which is strictly weaker for `MovePlayer`-stage behaviour (you can counter-thrust after it, not
   prevent it). This is a scoping decision, not an engine fact.
6. **Netplay.** `level.gravity` / `aircontrol` / `airfriction` are level-global
   (`doombase.zs:454-456`) and shared by every player. Per-player `Actor.Gravity` is the net-safe
   lever. I did not audit prediction (`CF_PREDICTING`, e.g. `player.zs:1442`) against velocity
   written in `DoEffect`; a dash may mispredict. Worth a pass before any of this ships multiplayer.
