# 99 — Overlooked

Things found while reading the source that suggest a mechanic nobody asked for.
One entry each: the citation, then a sentence on what it could become.

This file grows as the topic reports come in. Entries marked **(mine)** were found during the
capability pass; the rest come from topic surveys and name their source report.

---

## 1. There is a persistent, separately-rendered psprite band above `PSP_TARGETCENTER` **(mine)**

`PSP_TARGETCENTER = INT_MAX - 2` (`src/playsim/p_pspr.h:59`) is not just the crosshair's layer —
it is the boundary of a **second render pass with different clearing rules**:

- The weapon pass iterates only layers *below* it — `src/rendering/hwrenderer/scene/hw_weapon.cpp:830`, `:915`
- Layers at or above it are drawn in their own pass — `hw_weapon.cpp:1068`
- `A_ClearOverlays(start, stop, safety)` with `safety` set **breaks out before touching them** —
  `src/playsim/p_pspr.cpp:1273-1274` — and also skips `PSP_STRIFEHANDS`, `PSP_WEAPON`,
  `PSP_OFFHANDWEAPON` and `PSP_FLASH` (`p_pspr.cpp:1275-1276`)
- They are given `PSPF_CVARFAST|PSPF_POWDOUBLE` automatically — `p_pspr.cpp:360-362`

**What it could become:** a cockpit frame, a helmet interior, a permanent VR reticle, a damage
vignette or a persistent hand model that survives weapon switches and overlay clears without a
custom statusbar. For mechs and vehicles this is close to purpose-built.

**The trap:** a layer created at or above `PSP_TARGETCENTER` defaults its `Caller` to the
player's `PowerTargeter` inventory item (`p_pspr.cpp:313-318`). Without that powerup the layer
has a null caller and gets culled. Pass `newcaller` explicitly when creating it.

---

## 2. Psprite layer ids are arbitrary integers, and the gaps are enormous **(mine)**

`PSP_WEAPON = 1`, `PSP_FLASH = 1000`, `PSP_OFFHANDWEAPON = 1000000`
(`wadsrc/static/zscript/constants.zs:798-801`).

The named layers are conventions, not a fixed set. `GetPSprite(id)` / `SetPsprite(id, state)`
take any int. There are 998 free indices between the weapon and its flash, and roughly a million
between the flash and the offhand weapon.

**What it could become:** per-hand stacks of independently animated, independently positioned
overlay layers — a held item, a forearm panel, a glove, a muzzle device, a reload prop — all
without fighting the weapon's own two layers for space.

---

## 3. Four completely free input bits **(mine)**

`BT_USER1` … `BT_USER4` — `wadsrc/static/zscript/constants.zs:859-862`, readable via
`player.cmd.buttons` (`player.zs:3048`) with `oldbuttons` (`player.zs:3064`) for edge detection.

**What it could become:** the cheap answer to half the gesture-recognition problems in this
survey. A pose check gated by a held button is far more reliable at 35 Hz than a pure motion
gesture, and it costs nothing. Before designing a gesture, ask whether a button plus a pose
would do.

---

## 4. `BT_SHOWSCORES` was sacrificed, and the engine says so **(mine)**

`BT_SHOWSCORES = 0,  // replaced with BT_MAINHANDRELOAD` — `constants.zs:857`,
with `BT_MAINHANDRELOAD = 1<<20` at `:869`.

**What it means:** the scores button is gone — any mod binding it is binding nothing. Also a
useful signal about this fork's priorities: it spent a scarce input bit on a per-hand reload,
which tells you the two-handed weapon layer is meant to be used.

---

## 5. Offhand projectile firing already resolves direction correctly **(mine)**

`OffhandDir` is not a loose utility — it is wired into the standard attack functions:
`wadsrc/static/zscript/actors/actions.zs:87-89`, `attacks.zs:476-478`, `attacks.zs:596-597`,
and even a Heretic weapon at `heretic/weaponphoenix.zs:185`. All three basis vectors
(forward, side, up) are derived from it.

**What it could become:** it means `A_FireProjectile` and friends already fire *from the offhand,
in the offhand's direction, with correct offsets* when the weapon is an offhand weapon. Anything
thrown, fired or launched from the free hand is far closer to working out of the box than the
rest of this survey would lead you to expect.

---

## 6. `timefreezer` is a bitmask, not a flag **(mine)**

`native int timefreezer` — `wadsrc/static/zscript/actors/player/player.zs:3079`, with
`PowerTimeFreezer` setting and clearing *bits* per player at
`wadsrc/static/zscript/actors/inventory/powerups.zs:1529`, `:1537`, `:1604`.

**What it could become:** the engine already models "frozen for some participants and not
others" at the bit level. That is the natural first place to look for selective or partial
time effects rather than a global stop. See `10_TIME.md`.

---

## 7. `LineTrace` reports the texture it hit **(mine)**

`FLineTraceData.HitTexture` — `wadsrc/static/zscript/actors/actor.zs:53`, alongside `HitLine`,
`LineSide` and `LinePart` (`:50`, `:58`, `:59`).

**What it could become:** material-aware reactions with no map authoring at all — a whip that
cracks differently on metal, footsteps that match the floor, sparks only on stone, a hookshot
that refuses to bite on glass. The texture name is the material database you already have.

And `HitLine` + `LineSide` is how you recover a **wall normal** at runtime, which is the
foundation under wall-running, rope anchoring and shield alignment. It is the most
load-bearing field in the whole trace struct.

---

## 8. Render-to-texture is available to mods **(mine)**

`SetCameraToTexture(viewpoint, texture, fov)` — `wadsrc/static/zscript/doombase.zs:30`, with
`SetCameraTextureAspectRatio(texture, aspectScale, useTextureRatio)` at `:31`.

**What it could become:** a working mirror (useful well beyond vanity — it is how a VR player
sees their own body), a vehicle rear-view mirror, a mech cockpit display, a security monitor,
a scope picture-in-picture.
