# BP_Trap_Saw - configurable saw-off trap - build guide

**Status (14 Nov 2026):** `BP_Trap_Saw` is **complete**. The shell, components,
17 variables, all three functions (`ActivateSaw`, `FinishSawMorph`, `ChaseStep`)
and the EventGraph (BeginPlay, Tick, RevealBox overlap) are built and compile.
The event lanes are collapsed into named composite nodes
(`SawBeginPlayer_Setup`, `SawBeginPlay_Visibility`,
`SawBeginPlayer_RevealArm`, `SawTick_Morph`, `SawTick_Chase`);
`_verify_saw.py` follows collapsed graphs, so the verifier sees the wiring
inside them.

**Final verification:**

```text
py -3 Scripts\_verify_saw.py    -> 86 PASS | 0 FAIL | 0 WARN | 0 PENDING
py -3 Scripts\_verify_traps.py  -> 14 PASS | 0 FAIL | 0 WARN | 0 PENDING
```

UE 4.27 Python can create the Blueprint asset and set its parent class, but this
build exposes **no** component/variable/K2-node creation API
(`Scripts/_probe_graph_api.py`, `Scripts/_probe_blueprint_vars.py`). The graphs
were built by pasting T3D generated from existing Blueprint exports; the
verifier is read-only.

**Quick start:** sections 6-7 are the per-instance recipes and the PIE
checklist. Re-run both verifiers after any graph edit.

---

## 1. What this trap is

One Blueprint for every saw instance. The same class covers:

- a normal, always-visible saw obstacle;
- a visible **saw-off** prop that turns on / grows / moves when the player
  walks near (the "sawoff" surprise);
- a hidden trap that is invisible until the player enters its reveal volume;
- a stationary saw whose **size** changes at runtime;
- a movable saw whose **size and location** change at runtime;
- an optional chase mode that follows the player after the initial transform.

The important design split is **visibility is not activation timing**:

```
bStartHidden            = can the player see it before activation?
bActivateOnPlayerNear   = do we wait for the player, or activate at BeginPlay?
bMovable                = may its location change?
bChasePlayer            = after the transform, follow the player?
```

That gives four useful combinations:

| `bStartHidden` | `bActivateOnPlayerNear` | Result |
|---|---|---|
| `false` | `false` | Normal obstacle: visible, spinning, lethal from BeginPlay. |
| `false` | `true` | Visible saw-off prop: the player sees the still blade; when they enter `RevealBox` it turns on, grows/moves over `TransitionDuration`, then spins. |
| `true` | `true` | Hidden trap: invisible and harmless until the player enters `RevealBox`; then it appears, grows/moves, spins, and becomes lethal. |
| `true` | `false` | Hidden at spawn but activates immediately. Rarely useful; kept so the two switches stay independent. |

**Damage rule:** this is an ordinary trap. It inherits `BP_TrapBase`'s
`Trigger` overlap, which is already gated by
`Equal(OtherActor, Get Player Character(0))` and calls `Kill` on the player
only. Do **not** add a second `Kill` or `Apply Damage` to the child. Enemies and
bullets must not trigger or damage it (`Docs/TrapTriggers.md` section 2).

---

## 2. Art facts (verified)

| Fact | Value |
|---|---|
| Spinning art | `FLB_Saw`, 8 frames, 14 fps, frame size 38x38 px |
| Still / off art | `SPR_SawOff`, 38x38 px |
| Pixels per unreal unit | `0.16` |
| Natural blade size | `38 / 0.16 = 237.5` uu |
| One-tile scale | `100 / 237.5 = 0.4211` |
| Off frame | **`FLB_Saw` frame 0 is pixel-identical to `SPR_SawOff`** |

The last row means there is **one** visual component. Do not add a second
PaperSpriteComponent for `SPR_SawOff`. Stop the flipbook at frame 0 for the off
state and call `Play` when the saw turns on.

The `Sprite` pivot is `TOP_LEFT`. To centre a one-tile blade on the actor
origin, put the inherited `Sprite` at relative location `(-50, 0, 50)` and
scale it to `0.4211`. The trigger then sits at the actor origin.

---

## 3. Components (hand work)

### 3.1 One-time pivot in `BP_TrapBase`

**Do not rename, delete or replace `DefaultSceneRoot`.** It is inherited from
`BP_TrapBase` and is the actor root for every trap. UE 4.27 also refuses to
reparent inherited components in a child Blueprint:
`FSCSEditorTreeNodeComponentBase::CanReparent()` returns false for inherited
nodes and for the default root. Dragging inherited `Sprite`/`Trigger` in
`BP_Trap_Saw` therefore cannot work.

If you already added `SawRoot` in `BP_Trap_Saw`, delete that local component
first. It cannot parent inherited components and would collide with the
inherited name.

Create the pivot once in the parent, where `Sprite` and `Trigger` are local:

1. Open `Content/DontTrustTheLevels/Traps/Blueprints/BP_TrapBase`.
2. Components -> **+ Add** -> **Scene Component**, rename it **`SawRoot`**.
   It stays a child of `DefaultSceneRoot`; leave its relative transform at
   `(0,0,0)`, scale `1`.
3. Drag **`Sprite`** onto **`SawRoot`**, then drag **`Trigger`** onto
   **`SawRoot`**. These components are local to `BP_TrapBase`, so the editor
   allows the reparent.
4. Compile and save `BP_TrapBase`.
5. Open `BP_Trap_Saw`; `SawRoot` is now inherited, and `Sprite`/`Trigger` are
   already under it. Do not add a second `SawRoot`, and do not try to reparent
   inherited components in the child.
6. In `BP_Trap_Saw`, add **+ Add** -> **Box Collision**, rename it
   **`RevealBox`**. It must be a child of **`DefaultSceneRoot`**, **not** of
   `SawRoot` (its reveal radius should not scale with the blade).

This is a shared base-class change: `BP_Trap_Spikes` and `BP_Trap_FallPlat`
inherit the new identity pivot too. Their relative transforms are unchanged,
but run `_verify_traps.py` and PIE one instance of each after the edit.

`SawRoot` is the single animation pivot: moving or scaling it moves/scales the
blade and the lethal trigger together. `RevealBox` stays at the original actor
position, so the trap is still detectable where it was placed.

### 3.2 Exact component settings

**Set the `Sprite`/`Trigger` rows in `BP_Trap_Saw`, not in `BP_TrapBase`.**
They are inherited overrides; changing them in the parent would also change
`BP_Trap_Spikes` and `BP_Trap_FallPlat`. In `BP_TrapBase`, the only change is
the identity `SawRoot` pivot and the two attachment links.

| Component | Parent | Relative location | Relative scale | Key settings |
|---|---|---|---|---|
| `DefaultSceneRoot` | - | `(0,0,0)` | `(1,1,1)` | - |
| `SawRoot` | `DefaultSceneRoot` | `(0,0,0)` | `(1,1,1)` | - |
| `Sprite` | `SawRoot` | `(-50, 0, 50)` | `(0.4211, 0.4211, 0.4211)` | Flipbook = **`FLB_Saw`**; **Looping = on**; Collision Presets = **`NoCollision`** |
| `Trigger` | `SawRoot` | `(0, 0, 0)` | `(1,1,1)` | Collision Presets = **`OverlapAllDynamic`**; Generate Overlap Events = **on**; Box Extent = `(50, 50, 50)` |
| `RevealBox` | `DefaultSceneRoot` | `(0, 0, 0)` | `(1,1,1)` | Collision Presets = **`OverlapAllDynamic`**; Collision Enabled = **Query Only**; Generate Overlap Events = **on**; Box Extent = `(300, 100, 300)` (tune per instance) |
| `BillboardComponent_1` | inherited | - | - | editor icon only; leave alone |

**Why `Sprite` is `NoCollision`:** the blade is a visual, not a solid platform.
The inherited `Trigger` is the lethal volume. This also keeps enemies
trap-neutral, exactly like `BP_Trap_Spikes.Sprite`.

If the `Sprite` component exposes **Play Automatically**, turn it **off**.
`BeginPlay` stops/reset the flipbook and `FinishSawMorph` owns the `Play` call;
leaving auto-play on can flash a spin frame before the graph runs.

**Why `Trigger` is `(50,50,50)`:** with `SawRoot` scale 1 the box is a
100x100x100 uu volume around the one-tile blade. Scaling `SawRoot` to
`TargetScale = 2` scales the box to 200 uu, so the hitbox always matches the
new blade size.

**RevealBox extent is per instance.** `(300, 100, 300)` means a 600x200x600 uu
proximity volume around the saw. For a floor trap that should only fire when
the player is almost on it, use `(150, 100, 150)`. For a tall jump-catching
trap, raise Z (for example `(250, 100, 350)`). Keep `RevealBox` off the
`PlayerStart`; an actor spawned already overlapping the player is an
initialization edge case.

**Compile** `BP_TrapBase` after the pivot, then compile `BP_Trap_Saw` after
adding `RevealBox` and the variables. Click the inherited `SawRoot` component
and confirm its relative transform is still `(0,0,0)` with scale `1`.

---

## 4. Variables (hand work)

My Blueprint -> **Variables**. Set the type in Details, tick **Instance
Editable** on the first nine, then **Compile** before wiring anything.

| # | Name | Type | Default | Instance Editable | Meaning |
|---|---|---|---|---|---|
| 1 | `bStartHidden` | Boolean | `false` | yes | If true the blade is invisible before activation. |
| 2 | `bActivateOnPlayerNear` | Boolean | `false` | yes | If true, wait for the player to enter `RevealBox`; if false, activate at BeginPlay. |
| 3 | `bMovable` | Boolean | `false` | yes | If false, the location never changes; only size changes. If true, the saw moves to `TargetOffset`. |
| 4 | `bChasePlayer` | Boolean | `false` | yes | After the transform, follow the player at `ChaseSpeed`. Only used when `bMovable` is true. |
| 5 | `StartScale` | Float | `1.0` | yes | Origin size. `1.0` is a one-tile / 100 uu blade. |
| 6 | `TargetScale` | Float | `1.0` | yes | New size after the transform. `2.0` is two tiles / 200 uu. |
| 7 | `TargetOffset` | Vector | `(0,0,0)` | yes | New relative location. Used only when `bMovable` is true. `(300,0,0)` is 3 tiles right; `(0,0,-200)` is 2 tiles down. |
| 8 | `TransitionDuration` | Float | `0.0` | yes | Seconds to move/resize from the origin form to the new form. `0` = instant. |
| 9 | `ChaseSpeed` | Float | `300.0` | yes | UU per second while chasing. |
| 10 | `bActive` | Boolean | `false` | no | Internal. Set true once, at activation. |
| 11 | `bTransitioning` | Boolean | `false` | no | Internal. True while the timed morph is running. |
| 12 | `MorphElapsed` | Float | `0.0` | no | Internal. Seconds since activation. |
| 13 | `MorphAlpha` | Float | `0.0` | no | Internal. Clamped 0..1 blend. |
| 14 | `StartLocation` | Vector | `(0,0,0)` | no | Internal. Captured from `SawRoot` at activation. |
| 15 | `TargetLocation` | Vector | `(0,0,0)` | no | Internal. `TargetOffset` when movable, otherwise `StartLocation`. |
| 16 | `StartScale3D` | Vector | `(1,1,1)` | no | Internal. Captured from `SawRoot` at activation. |
| 17 | `TargetScale3D` | Vector | `(1,1,1)` | no | Internal. `Make Vector(TargetScale,TargetScale,TargetScale)`. |

---

## 5. Graph (hand work)

Create three functions in **My Blueprint -> Functions**:

- `ActivateSaw` - no inputs, no outputs.
- `FinishSawMorph` - no inputs, no outputs.
- `ChaseStep` - one **Float** input named `DeltaSeconds`, no outputs.

Keep the Event Graph for `BeginPlay`, the `RevealBox` overlap and `Tick`.

> **Polarity warning.** Several branches below have a "do nothing" or
> "wait" side. Read each True/False label twice before compiling. This project
> has already lost time to a correct class check with its exec pins reversed
> (`Docs/DashDamage.md`). The common reversals here are:
>
> - `bActive` **True = return**, False = activate.
> - `bStartHidden` **True = invisible / Trigger NoCollision**.
> - `bActivateOnPlayerNear` **True = arm RevealBox**, False = activate now.
> - `TransitionDuration > 0` **True = start the timed morph**, False = finish now.
> - `bTransitioning` **True = morph this frame**, False = chase gate.
> - `bMovable` is the **`bPickA`** pin of `Select Vector`: True picks
>   `TargetOffset`, False picks `StartLocation`.

### 5.1 `ActivateSaw` function

```
Function Entry (ActivateSaw)
  -> Branch (Condition = GET bActive)
       True  -> (leave the True pin empty - already active)
       False -> SET bActive = true
                Sprite -> Set Visibility
                    New Visibility = true
                    Propagate to Children = true
                Trigger -> Set Collision Enabled
                    New Type = QueryOnly
                RevealBox -> Set Collision Enabled
                    New Type = NoCollision
                Sprite -> Stop
                Sprite -> Set Playback Position
                    New Position = 0.0
                    Fire Events = false
                SET StartLocation = SawRoot -> Get Relative Location
                SET StartScale3D = SawRoot -> Get Relative Scale 3D
                SET TargetLocation = Select Vector
                    bPickA = GET bMovable
                    A      = GET TargetOffset
                    B      = GET StartLocation
                SET TargetScale3D = Make Vector
                    X = GET TargetScale
                    Y = GET TargetScale
                    Z = GET TargetScale
                SET MorphElapsed = 0.0
                Branch (Condition = GET TransitionDuration > 0.0)
                    True  -> SET bTransitioning = true
                    False -> FinishSawMorph (call)
```

**Why `Stop` + `Set Playback Position(0)`:** `FLB_Saw` frame 0 is the still
`SPR_SawOff` image. The saw stays visually off while it grows/moves; the next
section starts the spin only when the morph finishes. For an instant
`TransitionDuration = 0`, the off frame exists for one evaluation and then
`Play` runs immediately.

**Why `Select Vector` and not a Branch:** both outcomes need to continue to the
same `Set TargetScale3D` node. A Branch would give you two exec paths that you
would have to merge; `Select Vector` is pure and keeps one chain.

### 5.2 `FinishSawMorph` function

```
Function Entry (FinishSawMorph)
  -> SawRoot -> Set Relative Location
       New Location = GET TargetLocation
  -> SawRoot -> Set Relative Scale 3D
       New Scale 3D = GET TargetScale3D
  -> SET bTransitioning = false
  -> Sprite -> Play
  -> (Function Result)
```

This is the **only** place that calls `Play` after activation. Do not add a
second `Play` in `ActivateSaw`; otherwise the blade spins during the grow/move
and the off-frame effect is lost.

---

### 5.3 `Event BeginPlay`

Use a **Sequence** with three `then` pins. The Sequence is what runs the three
independent setup lanes without needing to manually merge Branch outputs.

```
Event BeginPlay
  -> Sequence
       Then 0:
         SET bActive = false
         SET bTransitioning = false
         SET StartScale3D = Make Vector(StartScale, StartScale, StartScale)
         SawRoot -> Set Relative Scale 3D (New Scale 3D = StartScale3D)
         SET StartLocation = SawRoot -> Get Relative Location
         Sprite -> Set Collision Enabled (New Type = NoCollision)
         Sprite -> Set Looping (bNewLooping = true)
         Sprite -> Stop
         Sprite -> Set Playback Position (New Position = 0.0, Fire Events = false)

       Then 1:
         Branch (Condition = GET bStartHidden)
           True  -> Sprite -> Set Visibility (false, propagate = true)
                    Trigger -> Set Collision Enabled (NoCollision)
           False -> Sprite -> Set Visibility (true, propagate = true)
                    Trigger -> Set Collision Enabled (QueryOnly)

       Then 2:
         Branch (Condition = GET bActivateOnPlayerNear)
           True  -> RevealBox -> Set Collision Enabled (QueryOnly)
           False -> RevealBox -> Set Collision Enabled (NoCollision)
                    ActivateSaw (call)
```

**Read the combinations:**

- `bStartHidden=false`, `bActivateOnPlayerNear=false`: visible, spinning,
  lethal at BeginPlay. This is the normal obstacle.
- `bStartHidden=false`, `bActivateOnPlayerNear=true`: visible **still** blade,
  lethal trigger already on, `RevealBox` armed. This is the visible saw-off
  surprise.
- `bStartHidden=true`, `bActivateOnPlayerNear=true`: invisible, trigger off,
  `RevealBox` armed. This is the hidden trap.
- `bStartHidden=true`, `bActivateOnPlayerNear=false`: hidden for the first
  evaluation, then `ActivateSaw` shows it immediately.

**The inherited `Trigger` is deliberately still enabled for the visible
saw-off case.** A visible blade is a normal obstacle; touching it kills even
before it turns on. Only hidden saws disable the trigger, because the player
must not die to something they cannot see.

### 5.4 `RevealBox` overlap

Select **`RevealBox`** in the Components panel -> Details -> **Events** -> click
**+** next to **On Component Begin Overlap**. Do not use the actor-level
`Event Receive Actor Begin Overlap`; the inherited disabled event at the top of
the graph is not the saw trigger.

```
On Component Begin Overlap (RevealBox)
  -> Equal (Object)
       A = Other Actor
       B = Get Player Character (Player Index = 0)
  -> Branch (Condition = Equal.Return Value)
       True  -> ActivateSaw
       False -> (nothing)
```

This is the **second** and **only** player gate in the saw. `BP_TrapBase`
already owns the lethal gate on `Trigger`; this one makes sure a bullet or an
enemy walking through `RevealBox` cannot start the trap.

### 5.5 `Event Tick`

```
Event Tick
  -> Branch (Condition = GET bTransitioning)
       True:
         SET MorphElapsed = MorphElapsed + Delta Seconds
         SET MorphAlpha = Clamp
             Value = MorphElapsed / TransitionDuration
             Min = 0.0
             Max = 1.0
         SawRoot -> Set Relative Location
             New Location = Lerp (Vector)
                 A = GET StartLocation
                 B = GET TargetLocation
                 Alpha = GET MorphAlpha
         SawRoot -> Set Relative Scale 3D
             New Scale 3D = Lerp (Vector)
                 A = GET StartScale3D
                 B = GET TargetScale3D
                 Alpha = GET MorphAlpha
         Branch (Condition = MorphAlpha >= 1.0)
           True  -> FinishSawMorph (call)
           False -> (nothing; continue next frame)

       False:
         Branch (Condition = AND)
           A = GET bActive
           B = AND
               A = GET bMovable
               B = GET bChasePlayer
           True  -> ChaseStep (Delta Seconds = Event Tick.Delta Seconds)
           False -> (nothing)
```

`TransitionDuration` is never zero on the True side of `bTransitioning`: the
only place that sets `bTransitioning = true` is the `TransitionDuration > 0`
branch in `ActivateSaw`. Do not move that setter above the branch.

### 5.6 `ChaseStep` function (optional chase)

```
Function Entry (ChaseStep, input DeltaSeconds)
  -> Get Player Character (Player Index = 0)
  -> Is Valid (Object = Return Value)
  -> Branch (Condition = Is Valid.Return Value)
       True:
         SawRoot -> Add World Offset
             Delta Location = Normal (
                 Subtract (Vector)
                     A = Player Character -> Get Actor Location
                     B = SawRoot -> Get World Location
             ) * (ChaseSpeed * DeltaSeconds)
             bSweep = false
             bTeleport = false
       False: (nothing)
```

`Normal` is the vector normalization node; its input is the result of the
`Subtract (Vector)` node. Multiply the unit direction by the
float `ChaseSpeed * DeltaSeconds` with **Multiply (Vector * Float)**. Target
the **`SawRoot` component**, not the actor: `RevealBox` stays at the placement
point, and only the blade + lethal `Trigger` move.

Chase is intentionally behind `bActive AND bMovable AND bChasePlayer`. If you
reverse that gate, a hidden saw will fly toward the player before it is
revealed. Set `TargetOffset = (0,0,0)` and `TransitionDuration = 0` when you
want the saw to start chasing immediately after reveal.

The game is a side-scroller on the **X/Z** plane; keep `TargetOffset.Y = 0` and
the player/saw at the same Y. If your level uses a different gameplay plane,
zero the Y component after `Subtract` before `Normal`.

---

## 6. Per-instance recipes

Set these on the **placed actor** (Details -> Default), not on the class
defaults, unless every saw in the level should share the value.

| Scenario | `bStartHidden` | `bActivateOnPlayerNear` | `bMovable` | `bChasePlayer` | `StartScale` | `TargetScale` | `TargetOffset` | `TransitionDuration` |
|---|---|---|---|---|---|---|---|---|
| Normal visible obstacle | false | false | false | false | 1.0 | 1.0 | (0,0,0) | 0.0 |
| Visible saw-off, grows when player is near | false | true | false | false | 0.5 | 1.5 | (0,0,0) | 0.35 |
| Hidden stationary pop-up | true | true | false | false | 0.05 | 1.0 | (0,0,0) | 0.25 |
| Hidden one-shot lunge to the right | true | true | true | false | 0.5 | 1.25 | (400,0,0) | 0.60 |
| Hidden grow-then-chase | true | true | true | true | 0.5 | 1.0 | (0,0,0) | 0.30 |
| Visible grow-then-chase | false | true | true | true | 0.75 | 1.25 | (0,0,0) | 0.40 |

**Reading the table:** `StartScale = 0.05` is a 5 uu origin form - small enough
to read as "a blade growing out of the floor" but not zero (a zero scale can
make the trigger degenerate). `TargetScale = 2.0` is a 200 uu / two-tile
blade. `TargetOffset` is in UU; one tile is 100 uu.

### 6.1 Normal visible obstacle

1. `bStartHidden = false`.
2. `bActivateOnPlayerNear = false`.
3. `StartScale = TargetScale = 1.0`.
4. `TransitionDuration = 0.0`.
5. Place the actor centre where the blade centre should be.

The saw spins from BeginPlay and the inherited `Trigger` kills the player on
contact. `RevealBox` is disabled at BeginPlay; it has no role in this recipe.

### 6.2 Visible saw-off that grows when the player is near

This is the "normal-looking obstacle that catches a jump" case.

1. `bStartHidden = false` - the player sees the still `SPR_SawOff` frame.
2. `bActivateOnPlayerNear = true` - it waits for `RevealBox`.
3. `bMovable = false` - only size changes.
4. `StartScale = 0.5`, `TargetScale = 1.5` (50 uu -> 150 uu).
5. `TransitionDuration = 0.35`.
6. `RevealBox` extent: `(250, 100, 350)` so it also catches a player who jumps
   over the blade.

The visible still blade is already lethal (the inherited `Trigger` is on), so
the player must avoid it even before it transforms. When they enter
`RevealBox`, the blade grows over 0.35 s and then starts spinning.

### 6.3 Hidden stationary pop-up

1. `bStartHidden = true` - invisible and harmless before reveal.
2. `bActivateOnPlayerNear = true`.
3. `bMovable = false`.
4. `StartScale = 0.05`, `TargetScale = 1.0`.
5. `TransitionDuration = 0.25`.
6. `RevealBox` extent: `(250, 100, 250)`.

`BeginPlay` hides the sprite and sets `Trigger` to `NoCollision`.
`RevealBox` is armed. When the player enters it, `ActivateSaw` shows the blade,
enables the lethal trigger, grows the blade from 5 uu to 100 uu, then calls
`Play`.

### 6.4 Hidden one-shot lunge

1. `bStartHidden = true`.
2. `bActivateOnPlayerNear = true`.
3. `bMovable = true`.
4. `bChasePlayer = false` - it moves once to `TargetOffset` and stops.
5. `StartScale = 0.5`, `TargetScale = 1.25`.
6. `TargetOffset = (400, 0, 0)` - 4 tiles to the right.
7. `TransitionDuration = 0.60`.

This is the "saw comes out of the wall/floor and sweeps across the corridor"
case. If you want it to move down instead, use `TargetOffset = (0, 0, -300)`.

### 6.5 Grow-then-chase

1. `bMovable = true`, `bChasePlayer = true`.
2. `TargetOffset = (0,0,0)` and a small `TransitionDuration` (0.2-0.4) to grow
   in place first.
3. `TargetScale` around `1.0`-`1.5`.
4. `ChaseSpeed = 250`-`400` UU/s.

After the grow finishes, `Event Tick` enters the chase gate and `ChaseStep`
moves `SawRoot` toward the player. The saw does not stop at a distance; the
lethal trigger is expected to catch the player.

---

## 7. PIE checklist

Run in a test map with one instance of each recipe, PIE stopped while editing.

1. **Normal visible saw:** walk into it -> player dies/reloads once.
2. **Normal visible saw:** fire a bullet through it -> no reaction, no damage
   to the bullet, no Kill.
3. **Normal visible saw:** walk an enemy through it -> enemy unharmed, saw does
   not react (`Docs/TrapTriggers.md` enemy contract).
4. **Visible saw-off:** approach without touching -> blade grows over the set
   duration, then starts spinning.
5. **Visible saw-off:** jump over it -> `RevealBox` still catches the player
   when the Z extent is tall enough; the blade grows under the jump.
6. **Hidden pop-up:** stand outside `RevealBox` -> no blade visible. Walk in ->
   blade appears, grows, spins, and kills on contact.
7. **Hidden pop-up:** fire a bullet through `RevealBox` -> nothing happens.
8. **Hidden pop-up:** walk an enemy through `RevealBox` -> nothing happens.
9. **Hidden pop-up:** touch the hidden saw before reveal -> nothing; the
   trigger must be `NoCollision`.
10. **Stationary resize:** set `bMovable=false` and a different
    `TargetScale`; location must not change, size must.
11. **Movable lunge:** set `bMovable=true` and `TargetOffset`; the blade and
    its lethal box move together. The hitbox must not stay at the old place.
12. **Transition timing:** `TransitionDuration=0` -> instant; `0.5` -> visibly
    halfway at 0.25 s; no pop at the end.
13. **Chase:** `bChasePlayer=true` -> after the initial transform, the blade
    follows the player and kills on contact. It must not chase before reveal.
14. **Respawn:** die and reload -> every saw returns to its placed origin form
    and hidden state.

---

## 8. Verify

With the editor open and PIE stopped:

```bat
py -3 Scripts\_verify_saw.py
py -3 Scripts\_verify_traps.py
```

`_verify_saw.py` exports `BP_Trap_Saw` to T3D, spawns it read-only in the
editor, and checks the parent class, component set, collision profiles, exposed
defaults, the player-gated `RevealBox` event, the three function graphs, and the
no-duplicate-Kill rule. It prints `PENDING` while the asset is only the shell.

`_verify_traps.py` must stay at its current **14 PASS / 0 FAIL**. The only
shared change is the identity `SawRoot` pivot in section 3.1: it does not alter
the existing trap graphs, but re-run this verifier and PIE one
`BP_Trap_Spikes` and one `BP_Trap_FallPlat` after the base edit.

Useful read-only helpers:

```bat
py -3 Scripts\_inspect_saw_assets.py   :: component transforms, collision, art facts
py -3 Scripts\_make_saw_shell.py       :: idempotent asset creation, never overwrites
```

---

## 9. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Hidden saw kills before it is revealed | `BeginPlay` True branch for `bStartHidden` did not set `Trigger` to `NoCollision`; check that branch. |
| Hidden saw never appears | `RevealBox` collision is `NoCollision`, has no overlap event, or `bActivateOnPlayerNear` is false; check the component event target and the player-equality Branch. |
| Bullets or enemies reveal the saw | The `Equal` node is missing or wired to the wrong actor. Use `OtherActor` and `Get Player Character(0)`; do not use `BPI_Killable`. |
| Visible saw-off never turns on | `RevealBox` is disabled, the overlap event is missing, or `ActivateSaw` is on the False pin. |
| Nothing resizes | `bTransitioning` never becomes true (`TransitionDuration` is 0), or `Event Tick` is not wired to `FinishSawMorph` at alpha >= 1. |
| It jumps instantly instead of transitioning | `TransitionDuration = 0`, or the `TransitionDuration > 0` Branch is reversed. |
| `bMovable=true` but it does not move | `Select Vector.bPickA` is reversed, `TargetOffset` is zero, or `bMovable` was not saved on the placed instance. |
| It moves but the hitbox stays behind | `Trigger` was not attached to `SawRoot`; reattach it. |
| It resizes but the blade is off-centre | `Sprite` relative location is not `(-50,0,50)` or its scale is not `0.4211`. |
| The saw blocks the player | `Sprite` collision is not `NoCollision`; the lethal volume is `Trigger`, not `Sprite`. |
| Player dies twice / reloads twice | A `Kill` or `Apply Damage` node was added to the child. Delete it; `BP_TrapBase` owns the player death. |
| Chase starts before reveal | The Tick chase gate is missing `bActive`; use `AND(bActive, AND(bMovable, bChasePlayer))`. |
| Chase flies away | `ChaseSpeed` is too high for the frame, or `DeltaSeconds` was not passed into `ChaseStep`; use the Event Tick `Delta Seconds` pin. |
| `Play` or `Stop` is not in the node search | Right-click the graph with **Context Sensitive** off and search `Play` / `Stop`; they are functions on the `Sprite` PaperFlipbookComponent. |
| `Lerp` has the wrong type | Use **Lerp (Vector)** for the two vector pairs; the alpha stays a float. |

---

## 10. Design notes

- **One class, per-instance data.** No per-level saw Blueprint copies. The
  placed actor owns `bStartHidden`, `bActivateOnPlayerNear`, `bMovable`,
  `StartScale`, `TargetScale`, `TargetOffset`, `TransitionDuration` and
  `ChaseSpeed`.
- **The lethal gate stays in `BP_TrapBase`.** The child never calls `Kill` or
  `Apply Damage`; this is what keeps the enemy/bullet trap contract intact.
- **Reveal and damage are different volumes.** `RevealBox` is a proximity
  sensor; `Trigger` is the lethal hitbox. Do not make one volume do both, or a
  hidden saw becomes lethal before it is shown.
- **`SawRoot` is the animation pivot.** `Sprite` and `Trigger` are its
  children. Scaling it grows the visual and the hitbox together; moving it
  moves both.
- **The off frame is real art, not a placeholder.** `FLB_Saw` frame 0 is
  pixel-identical to `SPR_SawOff` (verified against the source PNGs), so the
  same component can show the still blade and the spin.
- **Python cannot build the rest.** `BlueprintFactory` can create the asset and
  set `BP_TrapBase` as parent, but this UE 4.27 build exposes no SCS component,
  variable, or K2-node creation API. The guide is the implementation; the
  verifier is the safety net.





