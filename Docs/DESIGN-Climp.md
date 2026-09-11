# Climp Mode

Climp is Cubiciousflage's ascent mode. The shipped spelling is **CLIMP**, matching
`/Game/Cube/Map/Map_Climp` and the main-menu label.

## Player promise

Start at the waterline and climb a generated tower of **ten** themed floors to the lighthouse at the
middle of the map. Every run builds its own tower from the run seed: a ten-turn conical spiral of
flying islands, each floor drawn from a matching base-map library and tinted to one of six colour
bands, with the converted ZhangJiajie map as the ground floor. A **lighthouse** ends every floor and
replenishes cubes and ammunition; reaching the roof of the last one - three times the size of the rest,
standing on the tower's own axis - completes the run.

The spaces between the islands carry the run's resources: metal and energy ore to shoot, and
ammunition and skill orbs to take. Neither is decoration - Climp gives no other way to get either.

## Mode boundary

- `AVoxelClimpGameMode` is independent of `AVoxelPropGameMode`; Climp has no Prop/Hunter teams,
  hiding phase, conversion, bots, or round reset.
- The main menu opens `/Game/Cube/Map/Map_Climp` with the Climp game mode stated explicitly in the
  travel URL. This remains correct even though Prop Hunt is the project's global default mode.
- The initial button launches a local run. Session/multiplayer behavior is intentionally not inferred
  until Climp's player count and cooperation/competition rules are defined.
- Tab opens the regular player Cube Editor. Climp materials are displayed in the always-visible HUD;
  there is no inventory window or mode-only Tab panel.
- Q selects the Cube Paint weapon and E selects the carry weapon; left mouse activates the selected
  tool, matching the four regular weapons. Their actions require the regular first-person weapon
  view. While stationary-lock is active, Q/E return to their character-rotation role.

## Authoritative run state

The server owns the replicated run seed, current stage (`1..10`), completion state, checkpoint
activation, recovery points, painted-object placement validation, pickup consumption, and decay order.

**Only the seed goes on the wire.** Every machine builds the same tower locally from it - the geometry,
the ten checkpoints, the ore deposits and the pickup plan are all derived, not replicated. What that
buys and what it costs are covered in `HANDOFF-ClimpTower.md`; the rule that matters here is that
anything needing to know where the tower actually IS reads `AVoxelClimpGameState`, which keeps the
route island tops, the satellite tops, the checkpoints and the lighthouse volumes from the build that
ran. Re-deriving the tower with different inputs produces a different tower, and has twice.

A checkpoint is claimed by going INTO its lighthouse, not by standing on the roof - except the summit,
where the roof is the win condition. Any checkpoint at or ahead of the current stage counts: on a route
made of jumps between flying rocks, landing past a lighthouse is normal and used to leave every later
one permanently inert.

## Building rules

- In first-person view, Q selects Paint, E selects Carry, and left mouse performs the selected tool's
  action. Holding left mouse extends one server-anchored, face-connected line from the previous
  accepted build with no skipped stamps; releasing it drops a carried build. V cycles brush size
  1x, 2x, 4x, 8x, then 1x. Shape cycling is
  removed and the runtime brush is Cube.
- There is one inventory-capacity ceiling shared by all materials. There is no second active-placement
  ceiling; building stops only when the selected material inventory is exhausted.
- B cycles Block, Metal, and Energy material tiers. N cycles 16
  fixed colors and a seventeenth Rainbow option. Rainbow is the initial selection and uses a seeded,
  replicated random palette choice independently for every voxel, including large brushes.
- Material lifetime and rarity increase Block -> Metal -> Energy. Block uses the normal cube surface,
  Metal preserves the selected color with a fully metallic, low-roughness response, and Energy emits
  the selected color at high intensity.
- Carry can grab any aimed point on a painted stroke, selects every owned painted group touching it,
  and moves that connected assembly from the exact grab point. A replicated animated rainbow overlay
  marks the held assembly. Collision stays enabled; the server rejects an obstructed transform before
  teleporting it, preventing builds from entering terrain or passing through unrelated objects.
- Every placed stroke owns an independent lifetime, so continuous painting cannot postpone older
  cubes indefinitely. Within a stroke, decay removes the most recently indexed voxel first and
  proceeds gradually rather than destroying the whole object in one frame.
- A placement is rejected unless the player owns every required material voxel, its volume is clear,
  and its expanded support shell touches the world or an existing painted cube. Authored terrain uses
  its exact impact surface; relative 10-uu snapping is used only when extending a player build. Blue
  preview means accepted, red means genuinely obstructed or unsupported. The server validates all checks.

Paint cubes use the shared `VoxelLevelVoxelSize` edge length (currently 10 uu), so player-built cubes
match the base voxel grid instead of the 100 uu engine primitive. Current tuning is a 600-cube
inventory, selectable brushes up to 8x8x8, and total stroke lifetimes of 30 seconds for Block, 60 seconds for
Metal, and 180 seconds for Energy. Voxels dissolve progressively within that total window; the values
are not multiplied by brush voxel count. They are game-mode defaults rather than rules embedded in the
placed-object actor.

## HUD layout

The build/combat HUD is one general bottom-left HUD used by Lobby, Hunt, and Climp. Block/Metal/Energy counts
share one rainbow-framed panel whose width matches two skill tiles. Q Paint and E Carry are each one
skill tile wide and load their authored `gunui/buildgun` and `gunui/move` art; the resource rows use
`gunui/rock`, `gunui/metal`, and `gunui/energy`. V displays only the current size, while B displays only
the current material icon and N only the current color. Those three 72x72 controls sit beside the three
shared skills. Unselected Q/E tiles stay white outside FPS mode. The active Q/E tool or numbered main
weapon uses the same green selection treatment. The four main weapon buttons span the resulting six-skill width, with a visible gutter
between skill and weapon rows. Text uses unoutlined glyphs. Tool tiles dim outside first-person view,
and the selected tool highlights as equipped. Voice/text chat occupies the bottom-right. In Hunt, only
the four firearm tiles collapse for a Prop. Both teams retain Z/X/C skills and the Q/E build tools;
the server validates those actions for either team.

Height is shown independently as a narrow full-height rail at the middle-right, with exact meters and
summit percentage, ticks at the ten floors' real heights, climber name tags, and a personal best
persisted in `UVoxelUserSettings`. There is no Climp status menu in the top-right or across the top of
the screen. All display values are read from the local controller/player state through constant-size
Slate attributes; this adds no actor scan or collection traversal per frame.

The F prompt for a pickup is drawn as the front end's own button - cream fill, pink outline, dark pixel
text, the key as a keycap - because it is telling the player to press something, and every other thing
in the game that does that is a `Voxel.Button.Primary`.

Arriving at a floor is announced in the match HUD's result-card theme: "Congratulations, you have
reached Floor 3 - Canyon". The summit banner stays up and plays the victory bed and confetti.

## Resource and ammunition loop

**Two systems, because there are two verbs. Ore is MINED; everything else is TAKEN.**

- Weapon ammunition is finite and has no timed or manual free refill in Climp.
- **Ore deposits are level geometry**, not actors: six definitions - metal and energy, three sizes each -
  appended to the generated level as ordinary placements, so they stream, proxy and break like the rock
  around them. Shooting one runs the harvest path that already existed. Metal is twice energy's size and
  carries glowing amber veins; energy emits. Both wear the carry tool's rainbow, because a resource that
  looks like scenery is one nobody finds.
- **Ammunition and skill orbs are actors taken with F**, or shot from a distance. An ammunition pickup is
  shaped like the weapon it feeds, built from that weapon's Legend shop geometry, and its sign names it -
  `LASER AMMO`. Every pickup fully refills the weapon: the prompt shows no number because every capacity
  in the game is far below what a pickup carries.
- **F, not walk-over.** The route is narrow and a player lands where the geometry lets them; a resource
  spent by a landing they did not choose is hostile. A full inventory reads `- no room` rather than the
  prompt vanishing.
- **Spent is spent.** Pickups and deposits stream in and out with distance, so "I took that" has to
  outlive the actor or every crate is renewed by walking away. Both are recorded and persisted against
  the run seed in `UVoxelClimpRunSave`.
- Lighthouses refill all cube materials and ammunition, update the recovery point, advance the stage
  exactly once, and unlock that floor's achievement.

## Generation constraints

Superseded in shape, not in principle. The rules below still hold: a route must be validated before
anything optional is added, and randomly placing attractive rocks without a reachability pass is not an
Only Up-style run. What changed is the geometry they are applied to.

**The tower is now a cone of TEN stages**, generated by `VoxelClimpTower` and described in full in
`Docs/HANDOFF-ClimpTower.md`, which is the authority on the generator. In summary:

- Ten complete turns, one themed floor per turn, 50,000 uu tall, tapering from a 25,000 uu radius at the
  water to 5,000 uu at the summit. The turn count and the two radii are the authored quantities; the
  climb angle falls out of them, which is the reverse of the original design.
- Height is distributed along the PATH rather than by turn. On a cone the upper turns are a fraction of
  the lower ones, so distributing by turn gives the summit a fraction of the islands and leaves gaps.
- Contact points every `IslandSpacing` (700 uu) of measured travel; every contact point gets a cluster.
  A route body must be narrower than that spacing or the spiral fuses into a ramp; satellites and
  landmarks answer to their own, larger limits because nothing is jumped from them.
- The route WANDERS rather than jitters: lateral and vertical offsets are carried and nudged, not drawn
  independently, so it leaves the line by 200 m over many islands while consecutive steps stay a stride
  apart. Independent draws put neighbours at opposite extremes and produced 4,785 uu gaps.
- The route may descend slightly. Forcing every island above the last produced a staircase and, with a
  couple of thousand islands, silently decided the tower's height instead of the setting.
- The last turn is followed by a straight **run-in to the axis**, so the climb ends in the middle of the
  map rather than at an arbitrary point on the top ring. The summit lighthouse stands there at three times
  the size of the nine checkpoint ones, and reaching its roof wins the run.

Reachability is asserted in `Cubiciousflage.Climp.TowerShape` rather than assumed: worst step, worst
rise, descent depth and share, even contact-point density top to bottom, and the summit landing within
5% of the authored height.

## Streaming, and what it costs to exist

Everything on this tower is distance-streamed against the same rule, because everything on it is
several orders of magnitude more than a player can see at once.

- The level's 32,000-odd placements are built inside `AVoxelLevelDirector::StreamingLoadRadius`
  (12,000 uu) and released past its unload radius (16,000 uu), nearest first, time-boxed per pulse.
- **Pickup actors follow the same rule**, decided four times a second by the game mode. Roughly ten of
  a run's thousand planned pickups are alive at any moment. A pickup outside LOD0 is not visible, not
  reachable and not interactable, so there is nothing for it to be.
- Distant objects too far to build are stood in for by instanced two-LOD coarse silhouettes.
- Mesh builds run on worker threads - 84% of the work measured off the game thread.

The consequence that shapes the gameplay code: **an actor's existence is no longer the same statement
as the pickup's existence.** Anything that means "this is gone for good" has to be recorded outside the
actor, or streaming renews it.

## Persistence

`UVoxelClimpRunSave` records what a run has spent - pickups taken, deposits mined - **keyed on the run
seed**. Both are indices into arrays derived from the seed, so they only mean anything within one tower;
a new seed therefore starts clean and the file invalidates itself. Written on a five-second debounce
rather than per take.

## Achievements

Eleven, registered in `UVoxelAchievementSubsystem::GetAchievementDefinitions` and in Steamworks:

- `NC_CLIMP_FLOOR_1` … `NC_CLIMP_FLOOR_9`, one per floor lighthouse.
- `NC_CLIMP_SUMMIT`, for reaching the final lighthouse and completing a run.
- `NC_FALL_RECOVERIES_100`, a second tier on the shared `NC_STAT_FALL_RECOVERIES` stat.

A flat list rather than a stat with tiers, because "reach floor 6" is not "reach a floor six times" -
a milestone cannot tell one climber from six first floors. The ids are derived from the stage number
rather than listed, and `Climp.ModeDefaults` asserts every stage resolves to a name the definition list
contains: `UnlockAchievement` refuses an unknown name silently, and so does Steam.

## Map and conversion workflow

Main travel map:

```text
/Game/Cube/Map/Map_Climp
```

Starter/source area:

```text
/Game/ZhangJiajieMountain/Maps/Demonstration
```

Open a source map in the editor and use **Tools > Voxel Objects > Convert Current Level to Voxel
Copy**, or run:

```text
VoxelConvertCurrentLevel [/Game/Path/OutputMap] [/Game/Cube/Levels/OutputLevelAsset] [WorldScale] [InclusionVolume|none] [MinOccupiedVoxels]
```

The converter includes visible runtime `UStaticMeshComponent` instances, including ISM/HISM foliage,
and preserves their per-instance transforms. Components above 2,048 instances remain native by design:
dense grass would otherwise create one runtime voxel actor per blade. Translucent/additive foliage also
remains native because opaque cubes cannot reproduce its blend mode. Trees, bushes, and rock foliage
below the threshold convert; dense ground cover normally does not.

The 2026-08-17 source-map audit found 17 foliage components in `Demonstration`. Sixteen are within the
converter ceiling (39 to 1,675 instances each), including all six tree components. One
`Grass_01_B` component has 4,025 instances and will intentionally remain native. Thin masked grass may
still produce no occupied cells at the chosen voxel resolution; the conversion result log is the final
authority for those meshes.

The source map has now been converted with the deterministic lowest-start rule. `Player Start2` at
`(20, 560, 112)` was selected as the waterline Lobby marker instead of the summit start near
`Z=121397`. The generated assets are:

```text
/Game/ZhangJiajieMountain/Maps/Demonstration_Voxel_Climp
/Game/Cube/Levels/Level_Map_Climp
```

The current adaptive bake contains 449 object definitions, 13,169 successful placements, one waterline
marker, and 2,531,232 voxels. Fifty-nine large variants were automatically voxelized more coarsely so
their longest grid axis is at most 64 cells. This includes `BigMountain_01` at `40x41x64` (7,553 cells)
and `Big_Mountain_02` at `17x19x64` (3,724 cells); their smooth native components are hidden in the
generated map. The atmospheric `SM_SkySphere` remains native because an opaque voxel sky shell is not
gameplay geometry. `Map_Climp` resolves `Level_Map_Climp` by its map-specific asset name, so no binary
map edit or Base Village fallback is required. Its Climp game mode explicitly chooses the generated
Lobby marker because it is an `AVoxelMatchSpawnPoint`, not an engine `APlayerStart`.

### The ground floor

The converted map is used twice: its object library supplies the flying islands the spiral is built
from, and the level itself is the ground floor every run starts on. `VoxelClimpTower::AppendStartingIsland`
takes it whole - no radius cut, which produced a flat slab with a visible edge - centred horizontally on
the level's own Lobby marker and anchored vertically on the ground measured beneath that marker, so the
player stands at Z=0 at the foot of the spiral.

The big mountains are deliberately not the vertical anchor. They are the floating ZhangJiajie pillars:
`BigMountain_01` sits 63,144 uu above the waterline the player spawns on, and anchoring on its base put
the spawn 631 m underground. Its height is logged instead. `Big_Mountain_02` fits the island size band
and appears in the spiral like any other rock.

The 72 Landscape chunks use their component-specific generated material instances, including painted
weight-map variation. Unattended conversion skips the unreliable scene-capture path and rejects blank
captures; the final component bakes vary from bright tan to dark brown instead of inheriting one gray or
blue fallback. Successfully replaced Landscape and mountain components are hidden, and their native
actor collision is disabled in the generated map.

At runtime this map uses distance materialization rather than constructing all 13,169 voxel actors at
startup. Complete placement metadata and bounds remain resident; 96 spawn-near placements were built
synchronously in the validation run, then actors loaded nearest-first within 12,000 uu and unloaded
beyond 16,000 uu. Work is capped at eight builds per 25 ms pulse, 64 removals per pulse, and 2,500 live
placements. The 13k bounds scan runs four times per second, not every frame, and damaged objects are
pinned so local edits cannot be lost by unloading - except the ore deposits, which are declared
consumable and are recorded as spent rather than pinned, because a run mines several hundred. Checkpoint
positions come from the generator, not from baked bounds, so far-away actors do not need to exist for
Climp generation.

Against the previous eager payload, runtime setup fell from 91.2 seconds and about 13.2 GB working set
to 2.8 seconds in the final offscreen PIE check; a longer streaming run held around 4.7 GB while
preserving every checkpoint. Large-level one-click
prebuild is capped at the 24 highest-value definitions; a repeat conversion reused all 24 by content
hash and left the other 425 definitions for lazy streaming.

Climp now prewarms those remaining shared voxel definitions in bounded batches behind the normal
loading card before releasing player input. Nearby collision actors still materialize first and all
13,169 placements remain distance streamed; prewarming definitions is not the same as spawning every
actor. The gate polls at 10 Hz and has a 60-second real-time safety deadline. If compilation or asset
loading cannot finish by then, the loading card and input lock are released and the same bounded work
continues in the background. Placement decisions remain timer-driven at 4 Hz. The only new per-frame
gameplay work is one constant-time controller check; while E is actively carrying a group it sends at
most 20 unreliable position updates per second.

The held Q stroke now has two phases. Its first stamp is still an authoritative surface-supported
placement; that hit records a camera-space depth. Until mouse release, subsequent targets follow the
camera ray at that fixed depth, so the server's adjacent-axis interpolation can climb vertically or
extend sideways into empty space without gaps. The server remains authoritative over inventory,
collision, reach, adjacency, and the maximum 16 interpolated stamps per 20 Hz update.

Climp skills Z/X/C each carry one replicated use. One seeded skill cache is generated per stage and
restores all three uses; Prop Hunt retains its existing cooldown-only rules. Empty weapon ammo,
consumed Climp skills, and an empty selected Paint material use the HUD's indigo depleted state. The
shared material inventory cap is 9,999 cubes. Inventory is replicated from the common
`AVoxelPropPlayerState`: Lobby starts at 5,000/3,000/1,000, Hunt at 1,000/500/300, and Climp keeps
300/200/100 Block/Metal/Energy. The Q/E viewmodels are shared derivatives of the
shop's Plasma Pod and Riot Grabber geometry: their luminance-ordered palettes are reversed and their
muzzles receive small mode-specific voxel extensions, without changing or granting either shop skin.
The 2026-08-20 validation built both targets, loaded `Map_Climp` under the actual `VoxelClimpGameMode`
in `-game`, generated ten checkpoints, ~430 ore deposits, ~990 planned pickups of which ten or so are
alive at once, and released the initial loading gate. `Automation RunTests Cubiciousflage` runs
**16 pass / 1 fail**, the failure being the pre-existing `Cubiciousflage.Voxel.BaseMapRegistry` defect
recorded in `HANDOFF-ClimpTower.md`.

Carry never suspends decay. Q-built objects continue consuming their original 30/60/180-second
lifetime while E is holding or moving them, so grabbing an almost-expired object cannot refresh it.
Depleted HUD tiles use dark neutral gray, and the compact material counts are left aligned.

## Delivery order

1. **Complete:** mode entry, dedicated rule boundary, map cooking, conversion, and foliage audit.
2. **Complete:** replicated run state, checkpoint lighthouses, refill/recovery, material inventory,
   and always-visible mode HUD.
3. **Complete:** the generated ten-floor tower - themed libraries, colour bands, wandering route,
   satellites, landmarks, distant proxies, the summit run-in, and the ground floor.
4. **Complete:** material harvesting, mined ore deposits, and handed ammunition and skill pickups.
5. **Complete:** continuous Q paint lines, four brush sizes, 17 color choices, three material tiers,
   grab-point E carry with held feedback, validated placement, inventory cap, and independent
   total-lifetime Block/Metal/Energy decay.
6. **Complete:** streaming for both the level and the pickups, per-run persistence of what has been
   spent, and the eleven Steam achievements.

**The mode is feature-complete and unplayed.** Every number in it is measured and nothing in it has
been climbed end to end by a person. What is left is the thing no assertion can do: `HANDOFF-ClimpTower.md`
carries the open questions under *Known defects* - whether a 1,100 uu rise is satisfying to bridge,
whether the summit's roof can be reached on foot, and whether a metal deposit now wide enough to cover
its island makes the route feel blocked.
