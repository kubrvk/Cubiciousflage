# Handoff — Climp generated tower

State as of 2026-08-20. Written for a fresh session picking up the generated Climp climb.

## How to read this document

It is **long, and it is append-only on purpose.** Most of it is not a description of the tower - it is
the reasoning behind each decision, including the ones that were wrong first. That is the part worth
having: nearly every section exists because something measured perfectly and was still broken, and the
sentence explaining why is what stops the next person re-deriving the same mistake.

If you want the *contract* rather than the construction, read `Docs/DESIGN-Climp.md`. If you are about
to change something, find its section here first.

**The mode is feature-complete and has never been played.** Everything below is measured; nothing below
has been climbed. The open questions are collected under *Known defects*, and they are all questions
about how it feels rather than whether it works.

### The four traps that have each cost real time

Repeated here because they are spread across four sections and all four were expensive:

1. **Do not re-derive the tower.** Deterministic from the seed only for identical INPUTS, and
   `BuildAndApply` supplies resolvers a bare `Build()` does not. Read `AVoxelClimpGameState`.
2. **Measure the ACTORS, not the plan.** A correct plan and a heap of actors at the origin look
   identical from inside the planning code.
3. **A component that is the ROOT cannot be offset from its actor**, so `SetRelativeLocation` moves the
   actor instead of the mesh.
4. **A parameter that appears to do nothing may be drowned, not dead** - and a font, a facing or a
   collision volume can be perfectly configured and still render nothing. Look at the thing.

## What Climp is now

`Map_Climp` is **procedural**. It no longer stands up its baked ZhangJiajie level; instead
`AVoxelLevelDirector` sees the map listed in `DefaultGame.ini` under `ProceduralMapPackages` and waits
for a generated level, which `VoxelClimpTower::BuildAndApply` supplies from the run seed.

- `Source/CubeCubeCube/VoxelRuntimeEditor/VoxelClimpTower.h/.cpp` — the generator. Pure functions plus
  one `BuildAndApply` that knows where content lives.
- Spiral is anchored on the world origin. **Ten complete turns and the two radii are the authored
  quantities** and the climb angle falls out of them - the reverse of the original design, which
  authored the angle and let the shape follow. Summit at ~50,300 uu, tapering from a 25,000 uu radius
  at the water to 5,000 uu at the top. See *The spiral*.
- 10 floors, bottom to top: **Forest, Woodlands, Highlands, Canyon, Sea Cliffs, Deep Water, Stone
  Peaks, Fire Magma, Nightfall, Galaxy**, in six colour bands - paired floors share a colour on purpose,
  so a band a player spends two floors inside is a place and a band they pass through is a transition.
  Each draws its geometry from two shipped base-map libraries chosen to match its colour.
- Islands come from the climb map's baked rock library, filtered to a **size band** (400–2600 uu wide)
  and capped to 24 variants. Themed props are drawn from the object catalog by theme tag, capped to 16
  per tag. The caps are the load-time lever: instances of one definition share one runtime voxel asset,
  so uncapped variety builds nearly one mesh per placement.
- Every machine builds the tower locally from the replicated run seed
  (`AVoxelClimpGameState::BuildGeneratedWorldOnce`, called on the server when the seed is picked and on
  a client from `OnRep_RunState`). No geometry goes on the wire. Verified identical across two
  processes.
- The startup loading gate holds while a procedural map has no level yet — see
  `AVoxelLevelDirector::IsInitialStreamingReady`.

## The spiral

Authored as TURNS AND SIZE, not as a climb angle. `FSettings` carries four numbers that say what the
tower is: `StageCount = 10` (ten complete turns, one themed level per turn), `TotalHeight = 100000`
(1 km), `Radius = 25000` (500 m across) and `ForwardShiftPerTurn = 0.5` - each turn's centre steps
forward by half a coil diameter, so the climb is a spring laid over at an angle rather than one standing
on its end. A shift of half a diameter is exactly one radius, which is what keeps consecutive turns
overlapping enough to jump between. The turn count used to fall out of integrating a climb
angle, which meant nobody could say how many turns the tower had without running it.

**The vertical budget.** The path provides `TotalHeight / contact points` of rise between two contact
points - 44 uu at the current settings - and everything else vertical has to stay under it. The route
may never descend, and that rule at a fixed 60 uu minimum step over 2,264 contact points forces 136,000
uu of rise on its own: a climb asked to be 1 km arrived 1.36 km tall, with every other measurement still
passing. The minimum step and the height jitter are both derived from the natural rise now, and
`TowerShape` asserts the summit lands within 5% of the height that was asked for.

The path does not stop on the coil. After the last turn it runs straight in to the AXIS - see "The climb
ends in the middle of the map" below - so the climb ends at the middle of the map under a lighthouse three
times the size of the other nine, and reaching that lighthouse's roof completes the run.

`BuildPath` **walks and measures** the curve rather than solving it. A coil that also travels forward
has no closed form for its arc length, and arc length is the one thing that has to be right: contact
points are placed at a fixed distance along the path, so getting it wrong bunches objects where the
curve is tight. Sub-steps are 1/256 of the spacing, which holds the worst gap under half a percent.

The cluster loop places at EVERY sample and no longer re-measures the spacing itself. Two pieces of code
measuring the same thing to different precisions quietly dropped a third of the route: a sample landing a
hundredth of a unit short of the spacing failed the second test, and the next one paid for both.

`BuildPath` samples by **arc length**, not by height. Ten turns of a 25,000 uu spiral is 1.58 million uu
of travel against 150,000 uu of rise, so the path is very nearly horizontal; stepping by height would
stack every sample into a short column and leave the sweep between turns empty. One sample every
`IslandSpacing` uu of actual path, and every sample is a contact point that gets a cluster of objects.

The step count uses **floor**, not ceil. Rounding it up makes each step marginally shorter than the
spacing, and the cluster loop - which fires once its accumulated distance reaches the spacing - then
skips every second sample and silently drops half the objects.

## The ground floor

`/Game/Cube/Levels/Level_Map_Climp` is the converted ZhangJiajie `Demonstration` map, produced by
`Saved/ConvertClimpVoxel.py` (whole map, scale 1.0, no inclusion volume - the same process as the base
eight maps). It is loaded **once and used twice**: its object library is what the flying islands are cut
from, and the level itself is the ground floor the run starts on.

Re-converted 2026-08-18: 449 objects, 13,169 placements, 1 marker, 2,531,232 voxels.

`AppendStartingIsland` takes it **whole** - no radius cut, which is what produced a flat slab with a
visible edge - and re-centres it so that:

- horizontally, the level's own `Lobby` marker sits on the spiral's axis, so the player starts at the
  foot of the tower;
- vertically, the **ground under that marker** is Z=0, measured from the geometry rather than read off
  the marker. A PlayerStart's pivot is its capsule centre, about 90 uu above the floor.

The big mountains are **not** the anchor. In this source map they are the floating ZhangJiajie pillars:
`BigMountain_01`'s base sits 63,144 uu above the waterline the player spawns on. Anchoring on it put the
spawn 631 m underground. The mountain's base is reported in the log instead, so a floating mountain is
visible there rather than discovered in the world.

`Big_Mountain_02` is 2,234 x 2,497 uu and passes the island size band, so it appears in the spiral like
any other rock. `BigMountain_01` is far too wide and is kept out by the band - by size, not by name.

## Streaming, and why the tower never streamed

`StartPlacementStreaming()` was only ever called from `BeginPlay`. A procedural map builds its level
*after* BeginPlay, when the run seed arrives, so for Climp the streaming timer was never started at all:
nothing loaded as the player approached, nothing unloaded behind them, and only the placements spawned
during `BuildFrom` ever existed. `BuildFrom` now starts it itself when the actor has already begun play.

The startup gate no longer waits for every shared definition to be built. Readiness is about what is
under and around the player: the nearby placements and their visuals. `SpawnPlacement` materialises a
definition on demand, so the remaining library keeps building behind the player while they are already
standing on the ground floor. Loading progress is measured against the nearby placements alone, for the
same reason.

## Each stage is built from its own base map

`FStageTheme` names two baked libraries per stage, and `BuildAndApply` resolves one body pool per stage
from them. A city stage is built out of `newLevel_Map_1_City`, a factory stage out of the factory, an
alien stage out of the alien world. The eight base maps and the lobby are 87 MB of authored converted
voxel content, and the tower was previously assembled entirely from one mountain's rocks while all of it
sat unused.

Libraries are cached per package, so nine loads serve ten stages. A stage whose library is missing or too
thin falls back to the shared mountain pool rather than leaving a hole in the climb, and the tests assert
every stage names two DIFFERENT existing packages.

Islands, satellites AND props all come from the stage's own libraries. A city island wearing generic
scenery is still generic, so the small objects are filtered out of the same load - `FPools::Dressing`,
without the minimum footprint the bodies are held to, since the smallest things in a library are exactly
what dresses an island. The catalog's tagged sets extend that pool rather than replacing it.

Each pool answers to its OWN budget: `MaxIslandVariants` for bodies and satellites,
`MaxDressingVariants` for props. Letting props spend the island budget quietly tripled the definition
count, and every definition is a mesh build at load.

The dressing palette is assembled ONCE PER STAGE. It used to be rebuilt for every island - two array
appends, a couple of thousand times, arriving at the same answer each time within a stage.

**Cost, measured:** definitions 273 -> 1,297, voxels 983k -> 5.79M, prepare 3.2 s -> 6.5 s. The startup
gate stayed between 1 and 5 s depending on how much geometry sits near the spawn, because the gate waits
only for what is around the player - the rest of that library builds behind them.

## The massif

A spiral of small rock with nothing behind it reads as debris in the sky however well the route works.
`FLandmarkResolver` supplies the big landforms - ZhangJiajie's own mountains out of the climb library,
plus anything the bulk converter has filed under the `Landform` category - and they are placed as
scenery, never as route steps or satellites, answering to no size band:

- `MountainsPerStage` (4) float beside every stage, two to five lateral-jitters out so one never lands
  on the path a player is following.
- `BaseMountainCount` (44) stand in the water around the foot, biased to the side the spiral leans into
  because that is where a player looks on the way up, and lower the further out they stand so the range
  recedes rather than walling the horizon.
- Both are placed at 3-9x their authored size (`MinMountainScale`/`MaxMountainScale`) and dropped by
  their own scaled height, so they hang below the altitude given instead of sprouting through the route.

**One shade per floor.** `DarkenDefinition` re-slots a definition against a darkened palette - a
definition carries palette INDICES, not colours, so darkening means finding the nearest entry to each
cell's darkened colour. Built once per stage, not per placement, since each is a new definition and a
new definition per placement is a mesh build per placement. `SummitDarkening` (0.4) is how dark the top
is against the bottom, which makes the mountains the tower's own altimeter.

**Brushify.** `VoxelBulkConvert` now has `Landform` category rules for `volcano` and `cliff_large`;
without a rule an asset is rejected, which is why the first sweep converted nothing. The two large
cliffs convert (118x80x67 and 192x175x160 cells). The four volcanoes are DISTANCE meshes, kilometres
across, and are still rejected against the 280-cell axis cap - they would need adaptive resolution like
the map converter's, or a coarser voxel size the bulk command does not currently expose.

## The route wanders instead of following the line

HOW FAR the route wanders and HOW FAST are separate numbers. Sizing the step as a fraction of the
amplitude means asking for a wider climb asks for bigger gaps in the same breath - at 7,000 uu of spread
that produced 6,319 uu steps. The step is measured against what it threatens instead: the gap to the
next contact point, and the rise the path provides.

The wander is NOT reset at stage boundaries, and the last sample takes its forward direction from the
sample behind it rather than from world-forward. Both were single-step discontinuities of the whole
amplitude, and the second was worth 2,200 uu of the worst step on its own - at the summit, and at the
checkpoints, which are the two places a climb cannot afford a surprise.

The scatter is a WANDER, not a jitter: the lateral and vertical
offsets are carried from island to island and nudged, rather than drawn independently. Independent draws
put neighbours at opposite extremes - at a 2,400 uu spread the worst step between contact points 700 uu
apart came out at 4,785 uu, a route made of gaps. Carrying the offset means the route snakes: it leaves
the line by as much as ever, over several islands, and consecutive steps stay a stride apart (worst step
2,263 uu).

The "route never descends" rule is **gone**, replaced by two weaker and truer ones: no step down is more
than four times the natural rise, and fewer than a quarter of steps descend. Forcing every island above
the last is what produced a staircase - the scripted look - and with a couple of thousand islands it also
decided the tower's height instead of the setting.

## The ground floor is cut by height

The source map is a whole ZhangJiajie showcase: a beach at the waterline with floating pillars hanging
hundreds of metres over it. Only the beach is the ground floor. `GroundFloorCutHeight` (8,000 uu above
the standing surface) drops everything whose BASE floats above it - testing the base rather than the top
is what keeps a tree standing on the beach instead of beheading it.

The pillars were the flying islands of a climb that had none of its own. The spiral generates its own
now, so keeping them put a second, unclimbable set of rocks in the same sky as the route and cost
thousands of placements to do it. 13,169 placements become **342**; the island spans Z -1,587..2,729
over 13,550 x 12,050 uu, and sits at the foot of the spiral at (25,000, 0).

The cut lands in empty air: nothing kept reaches above 2,729 uu, so there is 5,000 uu of clearance
between the island and the threshold. That gap is why the number is not delicate.

## Creator mode builds the generated world too

`IsProceduralMap()` used to return false while creator mode was active, so the level editor on
`Map_Climp` built its BAKED asset - the whole uncut source, pillars and all - and showed a different
world from the one players get. Two modes disagreeing about what a map is cost more than it saved.

The scope is the config list and nothing else: only `/Game/Cube/Map/Map_Climp` is in
`ProceduralMapPackages`, so every authored map's editor is untouched. `Cubiciousflage.Climp.TowerShape`
asserts that the eight prop maps and Base Village are NOT in that list, because a map appearing there
would have its editor silently replaced by a generated world.

**Saving is refused on a procedural map.** `SaveCreatorLevel` overwrites the baked asset in its own
package, and on `Map_Climp` that asset is the source the world is generated FROM - the rock library the
spiral is cut out of and the island it stands on. Saving would write the generated tower over the
library that generated it, leaving neither. The generated world is authored in the generator.

## Distant proxies

Streaming makes a 20,000-object world playable and leaves the sky empty past 12,000 uu. On a 1.5 km
climb that is the difference between a route you can read from the beach and one that does not exist
until you are inside it.

**The palette slot is not a colour.** `MI_VoxelCube` reads the slot INDEX out of vertex colour red as
`slot/255` and looks it up in the palette texture; green and blue carry health and stress. The first
version wrote a literal RGB colour there, so every proxy sampled whatever palette entry its red channel
happened to name - which is why they rendered magenta. `VoxelDistantProxy::EncodeSlot` is the one place
that encoding lives, and the tests round-trip it. The material's cube shade mask is likewise scaled by a
`VoxelSize` scalar, so each tier gets a dynamic material instance with that set to ITS cell size -
leaving it at the real voxel size drew the pattern at a fraction of the size of the cubes it was
shading, which is what made the first attempt look like giant tiles.

**Detail is the renderer's decision, not this code's.** A proxy mesh carries BOTH detail levels -
`NearCoarseAxis` (24) as LOD 0, `CoarseAxis` (8) as LOD 1 - built in one `UStaticMesh` through
`BuildFromMeshDescriptions`, which takes one description per LOD. A hierarchical instanced component
then picks a LOD per instance from its screen size, every frame, on the GPU, for nothing.

The first version chose tiers here instead: a pass over every placement four times a second, two
components per definition, and instances moved between them. It was worse in both ways the choice can be
worse. Every instance in a distance band changed at the same moment, so a whole skyline flicked together
as the player walked; and any single change marked EVERY proxy component dirty, which rebuilds each
one's cluster tree - a stutter per step. Screen size is per instance, so a large rock holds its detail
further out than a small one and no two ever swap together, and `DirtyProxyComponents` now marks only
the components an instance actually moved in.

`VoxelDistantProxy` builds a coarse silhouette per large DEFINITION: the occupancy grid downsampled to
eight cells on its longest axis, one box per occupied coarse cell, interior faces dropped, vertex
colours averaged from the palette slots. `AVoxelLevelDirector` builds one of those meshes per proxied
definition, puts it in a `UHierarchicalInstancedStaticMeshComponent`, and adds one instance per
placement. Measured on the current tower: **75 definitions with two LODs each, standing in for 2,408
placements**, built in about 500 ms with the startup gate at 0.6 s.

Why per definition rather than per placement: the tower draws its bodies from a couple of dozen
variants, so thousands of visible objects cost a couple of dozen components. That is the only reason
"show the whole tower at once" is affordable.

Why a silhouette rather than a bounding box: a player picking a line up a tower has to tell a spire from
a slab. `Cubiciousflage.Climp.TowerShape` asserts a proxy keeps a narrow spire above a broad base, and
that it costs less than an eighth of the body it stands for.

- A proxy is hidden by scaling its instance to nothing when the real object spawns, and restored when it
  unloads. Removing the instance instead would renumber every instance after it.
- Instance moves are pushed to the renderer ONCE per streaming pulse. A hierarchical instanced component
  rebuilds its cluster tree when dirtied, so per-instance marking costs more than the streaming does.
- Proxies never have collision and never cast shadows: the world must not answer differently depending
  on how far away the player is standing.
- **Per-frame cost: none.** Meshes are built once per level; instances are toggled inside the 4 Hz pulse
  that already runs.
- `bDistantProxiesEnabled` on the director turns the whole thing off.

## Stage colour, and the checkpoint lighthouses

Each `FStageTheme` carries a `Tint`, and every stage's bodies and satellites are re-slotted towards it -
sand, forest, sea, canyon, bazaar, city, factory, magma, snow, galaxy. `TintDefinition` blends each cell
around ITS OWN LUMINANCE rather than pulling everything flat at one colour: the variation between
neighbouring voxels is what tells a player which face they can stand on, and a tint that swallows it
makes a stage harder to climb in exchange for looking themed. `ThemeTintStrength` is 0.45 for the same
reason. Tests assert adjacent stages are tellable apart in Oklab and that a tinted body keeps a range of
slots.

The stage tints are SIX BANDS OVER TEN FLOORS - green, earth, sea, stone, fire, space - with paired
floors sharing a band on purpose: a band spent two floors inside is a place, one passed through in a
single floor is a transition. The libraries were re-paired to match, because a sea-coloured floor built
from a temple's masonry reads as blue stonework rather than as the sea.

`TintDefinition` takes the hue whole and interpolates only BRIGHTNESS. Blending in RGB drags every
colour through grey on the way, which is literally what a half-tinted grey rock is - that, not the
strength, is why the first attempt left so much of the tower looking untinted. `ThemeTintStrength` is
0.85.

`GatherRegion` cuts a box out of any level and re-centres it on the middle of its footprint at the
height of its lowest piece - so a stamp's origin is where it STANDS. It tests a placement's ORIGIN, not
its bounds: clipping by overlap would take half of every object the volume's face passes through.

The checkpoint lighthouse is cut from `newLevel_Map_0_BaseVillage` by the box that Map_0_BaseVillage's
own `takeVoxelLobbyhere` blocking volume describes - centre (-20269, -569, 0), half-extents
(1000, 1000, 5000). 85 placements, stamped at all ten checkpoints and tinted to each stage, for 850
placements total. The coordinates live in code rather than the geometry being duplicated into an asset,
so the lobby stays the single source of what that piece is.

### Rounding the lighthouse corners

The building is a square tower, and ten square towers on a spiral of rounded rock read as the one thing
on the route that was imported rather than made. `RoundStampFootprint` cuts the four vertical corner
edges to a quarter-circle at `LighthouseCornerRounding` (0.35) of the stamp half-width, leaving the flat
walls - and therefore the windows and the doorway - alone.

**It has to be a per-voxel cut.** 85 placements over a 2420 x 2251 uu footprint means a placement is
several metres of wall; dropping the ones that fall outside a circle bites lumps out of the corners
rather than rounding them. Placements that clear the rounded plan whole keep the definition they share
with everything else (four plan-corner tests settle it, because a box is the convex hull of its four
corners and the rounded rectangle is convex); only the ones that straddle get a re-cut definition.

**Run once, on the shared stamp, before the per-stage tint.** Measured seed 1852300545: 85 placements,
18 definitions, **4 added by the cut**. Running it after the tint would multiply those four by ten.
Orphaned originals are pruned for the same reason - one unreferenced definition here is ten wasted mesh
builds later.

## Streaming: three separate causes of one symptom

"Lag spikes when loading" was three things, and fixing the first two without the third looked like no
fix at all.

1. **The decision pass ran on a timer.** Standing still and shuffling re-queued thousands of placements
   four times a second. It now runs only when a viewer has moved `StreamingRefreshDistance` (3,000 uu, a
   quarter of the load radius), with `StreamingRefreshSeconds` (3 s) as a ceiling so a stationary player
   still sees a changing world.

   **The first version of this gate did nothing**, because it measured every streaming source - including
   the look-ahead point, which sits up to 8,000 uu along the pawn's velocity and swings by 16,000 when
   somebody turns round. `GatherStreamingAnchors` exists to give the gate the pawns alone.

2. **Unloading was instant.** Crossing a boundary and stepping back destroyed and rebuilt the same
   objects - the most expensive possible way to stand still. A placement out of range is now marked and
   destroyed only if still out of range `StreamingUnloadDelaySeconds` (10 s) later.

3. **The budget was a COUNT.** A spawn costs anything from nothing - another instance of a definition
   already built - to a mesh build for a mountain, so "eight per pulse" bounds nothing about a frame.
   Two changes: the pulse is time-boxed at `StreamingPulseBudgetMilliseconds` (1.5 ms), and - the one
   that matters - `StreamingColdBuildsPerPulse` (1) limits how many NEVER-BEFORE-BUILT definitions a
   pulse may build. A time box stops the second expensive thing; it cannot stop the first.

Requests behind the viewer are WEIGHTED by `StreamingBehindPenalty` (x3 effective distance), not
filtered: what is behind you is in front of you the moment you turn round. Direction comes from control
rotation rather than velocity, because someone walking backwards is still looking where they will land.

## Streaming: it is now verifiable, and it was measured (2026-08-19)

The three fixes above could not be checked, for one reason: every one of them is driven by A PLAYER
MOVING. The movement gate only opens when a pawn travels, the deferred unload only fires when one leaves
and stays gone, and the cold-build throttle only matters when somebody walks into geometry nobody has
compiled. A `-benchmark` run has a pawn that stands still, so it exercised none of the three and
reported that everything was fine.

`ClimpStreamWalk [Seconds=90] [Speed=1200] [StartAlpha=0]` drags a streaming anchor up the spiral at a
chosen speed. `AVoxelLevelDirector::SetDebugStreamingAnchor` substitutes that point for the pawns in
BOTH `GatherStreamingSources` and `GatherStreamingAnchors`, so the same decision pass, the same queues,
the same spawns and the same mesh builds run without a person at the keyboard. It WAITS for the level
rather than refusing, so `-ExecCmds="ClimpStreamWalk 90 1200"` works on a procedural map whose tower
arrives after BeginPlay. `VoxelStreamStats` prints the profile on any map with a director;
`VoxelStreamStatsReset` zeroes it; `ClimpStreamWalkStop` ends a walk early and reports.

**It flies the PAWN along the route**, not just a disembodied anchor. Moving the anchor alone left the
pawn at the spawn marker; the ground under it unloaded once the anchor was far enough away, and it fell
and was recovered forty times in one run. In the log that is indistinguishable from the real spawn fall
loop and is nothing of the kind - the worst thing a harness can do. The pawn is put into `MOVE_Flying`
with damage off for the duration and handed back to `MOVE_Walking` at the end.

It drives streaming, not rendering, physics or input, so it is not a substitute for playing the map. It
answers one question repeatably: whether the streaming pulse is what costs the frame.

### The first three findings

**1. `WarmDefinitions` was a lie, and it defeated the throttle it fed.** The prewarm added a definition
to the warm set after calling `GetOrCreateSharedAsset`, which fills in a definition's voxel array and
builds no geometry at all. The mesh, its clusters and its collision were still built by whichever
placement arrived first, in the frame it arrived. Eighty definitions a second were being declared free
while every one of them was still a cold build, so `StreamingColdBuildsPerPulse = 1` was enforced
against a set already emptied of what it existed to catch. Nothing enters `WarmDefinitions` now until a
shared mesh actually exists for it.

**2. The mesh build moved off the arrival path.** `PrewarmDefinitionMesh` calls
`FVoxelRuntimeTemplateCache::GetOrCreate` with the same arguments `UVoxelComponent` passes in a game
world, so it fills the cache entry the spawn will look for rather than building a parallel copy. That is
the whole of the difference: the spike is not caused by there being a lot of objects, it is caused by
the work landing at the moment of arrival.

**3. The prewarm is range-GATED, not just range-ordered.** Its order used to be the definition array
reversed, which is unrelated to where anybody is standing - the tower compiled the summit's galaxy rock
while the player was on the beach. Sorting alone still compiles the whole tower, merely in a better
order. `StreamingPrewarmRadius` (30,000 uu, against a 12,000 uu load radius) stops it: past that a
definition is represented by its distant proxy, which is what the proxies are for, and it is built when
somebody heads for it. A LEFTOVER PREWARM QUEUE IS THE INTENDED END STATE - a run that empties it has
compiled the whole tower, which is the cost the gate refuses.

| 90 s walk at 1200 uu/s | before | ordered | ordered + gated |
| --- | --- | --- | --- |
| mean streaming pulse | 38.4 ms | - | **14.2 ms** |
| pulses over 33 ms | 700 | - | **292** (of 3,601) |
| pulses over 16 ms | 939 | - | 407 |
| total streaming work | 138 s | 63 s | **51 s** |
| definitions compiled | 1,362 (all) | 639 | 600 |
| placements resident at end | 545 | 953 | - |

The seed differs per run, so these are not perfectly controlled - placement counts ranged 12,451-12,767
and definitions 1,386-1,414 - but the deltas are far larger than that variance.

### The mesh build moved to worker threads (2026-08-19, second pass)

The scheduling work above was still scheduling around a 96 ms unit. `VoxelDefinitionMeshBuilder` makes
the unit smaller by moving the part of it that can move.

**The split the plugin already drew for us.** `FVoxelRuntimeStaticMeshBuilder::BuildMeshDescription` is
documented in the plugin's own header as "pure data, safe on a worker thread - the CPU-dominant half of
BuildStaticMesh", with `BuildStaticMeshFromDescription` as the game-thread remainder. So the director
asks the template cache for the CHEAP half only (`bRequireSharedVisual` and `bRequireSharedCollision`
both false - the sparse grid and the greedy mesher, about 5 ms), and hands the rest to a worker:

- **worker**: assemble each visual cluster's buffers from the template's chunk meshes, fill an
  `FMeshDescription` per cluster, cook the Chaos triangle mesh;
- **game thread**: `NewObject<UStaticMesh>`, `BuildFromMeshDescriptions`, hang the cooked triangle mesh
  on a body setup, publish the clusters into the template.

**No plugin patch.** `FVoxelRuntimeTemplate` is a public struct with public members, and
`FVoxelRuntimeTemplateCache::GetOrCreate` early-outs on a cache hit once `bSharedVisualReady` and
`bSharedCollisionReady` are set. Filling them in ourselves is indistinguishable, to the plugin, from
having built them synchronously. This is the "configure from Cubiciousflage C++" tier of `CLAUDE.md`,
not a behavioural patch to re-apply on every plugin update.

**The one race, and who closes it.** While a build is in flight the template sits in the cache with
`bSharedVisualReady` false, so a placement spawning in that window would make `UVoxelComponent` build
the same clusters synchronously and write the same fields. The director refuses to spawn placements of
a definition `VoxelDefinitionMeshBuilder::IsBuilding` reports, deferring them exactly as it defers any
other cold build - the object stands as its silhouette proxy for the fraction of a second involved.
There is no lock on purpose: a lock would make the game thread wait on the worker.

**Publishing is queued, not done on arrival.** The game-thread residue is ~21 ms per definition and 257 ms
for the worst, and several workers finishing in one frame would stack theirs into it. Finished builds
wait in a queue that the pulse drains under `StreamingPublishBudgetMilliseconds`. One publish is still
atomic - a 257 ms definition costs 257 ms whenever it lands. That unit is a triangle-count question now,
not a scheduling one.

**A cold spawn is the last resort, not the mechanism.** Arriving at an uncompiled definition used to
build its mesh in the frame of arrival (67 ms mean, 343 ms worst - the largest cost left after the
prewarm moved). The placement now jumps the prewarm queue: its definition is dispatched to a worker and
the spawn is deferred. Cold spawns measured **zero** on the final walk.

**Startup.** `BuildFrom` used to spawn the nearest ~100 placements synchronously, before the streaming
timer existed to throttle anything - every definition among them a cold build at full price, in one
unbroken stretch. That was 1,535 ms of frozen game thread, and it is the freeze on startup. Those
placements are queued now and built by the pulse on workers behind the loading card. `Prepared streamed
level` fell to **14 ms**.

**Two bugs the gate rework exposed, both worth keeping in mind:**

- The startup gate waited for the whole stream-in queue to drain. Once the near placements stopped being
  spawned synchronously that meant everything inside the 12,000 uu load radius, and the loading card went
  from 0.9 s to 10.9 s. It waits on a NAMED set now, `InitialGatePlacements`, which is what "readiness is
  about what is under and around the player" was always supposed to mean.
- **The pawn does not spawn where the level's marker is.** On a procedural map the level arrives after
  the first `PostLogin`, so `ChoosePlayerStart` runs before any marker actor exists and the pawn is
  placed at the world origin - 34,775 uu from the tower's Lobby marker. With the pawn as the only
  streaming source, every ground-floor placement sat outside the load radius: nothing under the spawn was
  ever requested, and a player teleported to the marker afterwards arrived over a hole and fell. This is
  why the character kept falling and respawning. `GatherStreamingSources` now keeps the level's markers
  as sources until the gate opens, so the ground somebody is about to stand on is the first thing built.
  97 of 97 gate placements now stand before the card goes away.

### The spawn, and the fall loop

**A procedural map has no spawn marker when players log in.** The level is generated from the run seed,
which `AVoxelClimpGameMode::BeginPlay` sets; `ChoosePlayerStart` has already run by then, found no Lobby
point, fallen through to `AGameModeBase` and put the pawn at the **world origin** - 34,775 uu from where
the tower's ground floor is. The comment claiming "the level director builds in PostInitializeComponents
specifically so its markers exist before the game mode starts players" is true of a BAKED map and false
of this one.

That is not merely a bad spawn, it is an inescapable one. `HandleStartingNewPlayer` records the pawn's
transform as its recovery point, so the origin became the place falling players were returned to - and
there is no ground at the origin either. Fall, recover to the origin, fall again. That loop is what
"the character keeps dropping and respawning" was.

Three things fix it, and all three are needed:

1. **`AVoxelLevelDirector::OnLevelReady`** is broadcast at the end of `BuildFrom`, once markers are real
   actors. `AVoxelClimpGameMode` binds to it **before** setting the run seed - the seed builds the tower
   synchronously, so a listener registered afterwards has already missed the only broadcast it gets.
2. **`PlacePlayersAtLobbyMarker`** teleports every player to the Lobby marker AND rewrites their recovery
   transform. Moving the player while leaving the bad recovery point behind fixes only the first fall.
3. **Gravity is held while the startup gate is closed.** Locking input stops the player walking off the
   world; it does nothing about the world not being there yet. The pawn was being placed on the marker
   before the ground under it had streamed in, falling through, being recovered, and falling again -
   two full cycles in the five seconds before the gate opened. From the player's seat that IS the
   loading screen. `ApplyInitialLoadInputLock` now also puts the pawn in `MOVE_Flying` with no velocity;
   `FinishInitialLoadGate` restores `MOVE_Walking`, including on the safety-timeout path, because a
   player frozen in the air forever is worse than one who falls once and is recovered.

Verified: **0 falls**, player standing at the Lobby marker, gate released after 6.0 s. `RecoverDroppedPlayer`
logs every recovery now, because a recovery that REPEATS is the signature of a recovery point with no
ground under it - the failure presents as a loop, not as one fall.

**Distant objects no longer cast shadows.** A voxel object is not one primitive - the plugin gives it one
`UStaticMeshComponent` per visual cluster, so a thousand resident placements is several thousand shadow
casters in every cascade. `StreamingShadowRadius` (20,000 uu) switches casting off past the band where a
rock's shadow is a few pixels; the silhouette proxies beyond the load radius never cast one anyway. It is
decided on the decision pass, which already has the distances, not per frame. `AVoxelLevelObject::
SetVoxelShadowsEnabled` applies it to the CHILDREN - setting it on the parent `UVoxelComponent` changes
nothing visible, because that component draws nothing itself in a game world.

| 90 s walk at 1200 uu/s, real RHI | session start | after scheduling | after workers |
| --- | --- | --- | --- |
| mean streaming pulse | 38.4 ms | 14.2 ms | **4.1 ms** |
| pulses over 33 ms | 700 | 292 | **134** (of 3,600) |
| pulses over 16 ms | 939 | 407 | **209** |
| total game-thread streaming | 138 s | 51 s | **14.8 s** |
| synchronous cold spawns | - | 33 | **0** |
| `Prepared streamed level` | 1,535 ms | 1,535 ms | **14 ms** |
| startup gate | 0.9 s, 0 placements standing | - | 6.7 s, **97 standing** |

83% of all mesh-build work now happens on worker threads: 42.9 s moved, 8.6 s left behind.

### What is left, and it is not a scheduling problem

**One voxel definition's runtime mesh costs about 96 ms of GAME THREAD, and 93% of that is one thing.**
Split by asking the template cache for the two halves separately - the second call is a cache hit, so
the split is free:

| half | mean | share |
| --- | --- | --- |
| sparse grid + greedy chunk meshing | 4.6 ms | 5% |
| `UStaticMesh` construction + collision cook | **86-99 ms** | **95%** |

The cost is `FVoxelRuntimeStaticMeshBuilder::BuildStaticMesh`: an `FMeshDescription` built one
`CreateVertex`/`CreateVertexInstance`/`CreatePolygon` at a time, then `BuildFromMeshDescriptions`. A cold
spawn measures 72-107x a warm one - the premise the throttle rests on is confirmed, and then some.

No scheduling scheme makes a 96 ms unit of work invisible. One per pulse is a 96 ms hitch, forty times a
second. The worst single definition measured 2.4 s. Everything above is scheduling around an
unaffordable unit; the unit itself is the remaining fix. **The first candidate below is DONE** - see the
worker-thread section above, which took 83% of it off the game thread. The remaining ~8.6 s per walk is
`BuildFromMeshDescriptions`, which cannot leave the game thread, and the two levers on it are both about
building FEWER and SMALLER meshes:

- ~~**Get it off the game thread.**~~ Done. `VoxelDefinitionMeshBuilder`.
- **Build 42% fewer meshes.** `VoxelStreamStats` reports it: *1,411 definitions carry 814 distinct
  shapes*. `TintDefinition` and `DarkenDefinition` each return a NEW definition with the same grid and
  the same occupied cells, differing only in which palette slot each cell names - and the plugin bakes
  that slot into vertex colour, so every recolour is a full mesh build of geometry already built.
  `SM_Lighthouse` (50,049 voxels, ~1.1 s each) appeared ELEVEN times in one run's slowest-24 list: ten
  stage tints of one building, about 11 s of game thread. Tinting through a per-stage palette texture
  rather than by re-slotting cells would collapse those to one mesh and many materials.
- **Runtime Nanite.** `FVoxelEditorNaniteRuntimeMeshBuilder` writes vertex buffers directly and never
  touches `FMeshDescription`. `UVoxelLevelSubsystem::PopulateAssetFromDefinition` records this as an
  untaken tuning opportunity - but CHECKED AND REJECTED as a quick experiment: `bDefaultUseRuntimeNanite`
  is a project-wide setting requiring a restart and a rebuild of every baked asset, and
  `UVoxelComponent::ApplyRuntimeStreamState` makes Nanite BYPASS distance streaming entirely
  (`bNaniteBypassesDistanceStreaming`), which would disable the system this all rests on. The per-asset
  flag only disables Nanite; it cannot enable it against the project default.

- **Build smaller meshes.** `SM_tree_01` is 102,043 voxels and **235,388 triangles**; `SM_Ship_Cabin_01`
  is 59,557 voxels and 215,264. That is roughly two triangles per exposed voxel face with no merging
  across faces of the same palette slot, and triangle count is what both remaining costs scale with -
  the worker's `FMeshDescription` and the game thread's `BuildFromMeshDescriptions`. A greedy merge in
  the plugin's chunk mesher, or a coarser voxel size for large definitions, is the lever. This is the
  single biggest one left.

Debug commands added with this work: `VoxelFly [0|1] [Speed]` puts the local player into free flight
with damage off, because testing a 1.5 km tower on foot is a sequence of respawns rather than a look at
the world; `ClimpGoto [Stage]` teleports to a stage checkpoint, 1,500 uu above the route so the teleport
does not end inside an island; `ClimpSummit` puts you on the summit lighthouse's roof, which is the one
place the run can be won and is otherwise a kilometre of climbing away; `ClimpAfter <Seconds> <Command>`
runs any of them once the loading gate is out of the way, and `ClimpShot [Seconds]` photographs the result
with the HUD in frame.

To bisect a surviving spike that is NOT the builds: set `StreamingColdBuildsPerPulse` to 0 and
`StreamingDefinitionPrewarmBudgetPerPulse` to 1 with a small `StreamingPrewarmRadius`, and walk the same
route. If it persists, it is not mesh building at all.

## Checkpoints and the lighthouse

`AVoxelClimpCheckpoint` actors are spawned by `AVoxelClimpGameMode` at `Build.StageCheckpoints`, and
already carried a trigger and a text label. Stamping the lighthouse at the same point put the marker,
its label and its trigger INSIDE the building - which is why walking up to one appeared to do nothing.

The marker spawns at the lighthouse's own height + `AVoxelClimpCheckpoint::MarkerLiftAboveRoof` (600)
above the island, and its trigger reaches back down 900 to the roof a player stands on. Both are sized
against the building now rather than fixed, because the summit's is three times the others' - see below.
The label reads "3 - CANYON": the number says how far along the climb is, the name says where you are,
and ten identical labels say neither.

## The summit, and the three things that were wrong with the lighthouses (2026-08-19)

### The climb ends in the middle of the map

A spiral ends on its own radius, which is a point on a RING - so "the top of the tower" was wherever the
last turn happened to be cut, 5,000 uu out from the axis, and the middle of the map was empty. `BuildPath`
now appends a **run-in**: contact points at the usual spacing travelling straight in from the last coil to
the axis, marked `FPathSample::bSummitApproach`, gaining the same height per step the spiral does.
Measured: **summit at (0, 0, 50,297), 0 uu from the axis**, on every seed.

**Radial, not a tightening spiral.** A curve that wound inwards would add a fraction of a turn to a tower
whose turn count is the AUTHORED quantity - ten turns is something a player counts from the beach. Holding
the heading and changing only the radius leaves the turn assertion measuring the spiral alone.

**The wander converges across it.** The route wanders by thousands of units either side of the spline,
which is what stops it reading as a spline - but the summit is a NAMED PLACE, and a last island left four
thousand units off the axis puts the lighthouse somewhere else. The offsets are faded to exactly zero over
the run-in rather than switched off at its start, which would be the whole-amplitude discontinuity the
wander exists to avoid, at the one place a climb can least afford it.

**Two frame bugs, and they are the same bug twice.** The lateral offset is expressed in a frame derived
from the direction of travel, and the run-in turns ninety degrees off the coil. Recomputing that frame
rotates several thousand units of accumulated offset in ONE step:

- the run-in's own samples keep the frame the spiral left them (`CarriedLateral`);
- and the LAST COIL SAMPLE now looks BACKWARDS for its heading, exactly as the final sample of the path
  always did, because its successor is no longer more of the same curve. Missing that one sample was worth
  a **4,731 uu** gap between the last two coil islands - caught by the reachability assertion, at a place
  no measurement of the run-in itself would have looked. Route worst step is 1,632 uu now.

### The summit lighthouse

The last stage's lighthouse is the same building at `Settings.SummitStampScale` (3.0), standing where the
run-in ends. **7,260 x 6,753 uu against 2,420 x 2,251 uu on the other nine.** Reaching its roof completes
the run, because the summit checkpoint is the tenth and `ReachCheckpoint` completes on the last stage.

Scaling rather than a second bake, and it is free: a stamp's origin is its own base at the middle of its
footprint (`GatherRegion`), so scaling a copy is one multiply on each placement's offset and one on its
scale - and instances of a definition share one runtime voxel asset whatever scale they are drawn at. A
separately authored large lighthouse would be a whole second library of definitions, every one of them a
mesh build.

`FBuild` records `CheckpointStampHeight`/`CheckpointStampRadius` and the summit's own pair, and
`AVoxelClimpGameState` keeps them. The checkpoint actor places its marker, its trigger and its NAME from
those two numbers; a constant there is a caption inside the masonry on one building and floating beside it
on another.

### Checkpoints six through ten did nothing at all

`InitializeCheckpoint` clamped its stage number to `1..5`, from when Climp had five stages. With ten
floors that turned the top five lighthouses into five more copies of checkpoint five: `ReachCheckpoint`
required the stage number to MATCH the stage being expected, so the sixth asked to advance to a stage
already passed, was refused, and every one above it with it. **The run could not be completed**, and
nothing in the world or the log said why - each lighthouse looked like it worked, and the one that
mattered was ten floors up.

Two changes, and both are needed: the clamp is gone, and `AVoxelClimpGameState::ReachCheckpoint` accepts a
checkpoint AT OR AHEAD OF the current stage rather than exactly it. On a route made of jumps between
flying rocks a player will land past a lighthouse, and under the old rule that left every later checkpoint
permanently inert. Reaching one is evidence the player physically got there.

### Entering one is now visible

Walking into a checkpoint changed nothing a player could see AT THE MOMENT THEY DID IT: the label
recoloured, which is behind them by then, and the inventory refilled, which is on the other side of the
screen. `EntryField` is the project's own barrier - `M_Barrier_Inst`, the same shield the Prop barrier
skill raises, on the engine sphere - swelling from inside the trigger to a little past it and fading out
over 1.25 s. Amber while the checkpoint is unclaimed, green once it is claimed.

- Fired on EVERY machine that sees the overlap and BEFORE the authority check, because the effect is what
  tells a player their feet are inside the volume and it has to happen when they are, not a round trip
  later. It replicates nothing.
- Driven by a **repeating timer that cancels itself**, not a tick: forty frames of one component's scale
  and one scalar parameter, once. Ten checkpoints each registering a permanent tick function so that each
  can animate for a second is the cost this avoids.
- Reusing the barrier rather than authoring a second effect means a player who has seen one shield already
  knows what being inside something looks like.

### The name is written on the lighthouse

**A world-space `UWidgetComponent` ON TOP of the roof, not a `UTextRenderComponent` on the wall.** The
first attempt wrote the name two thirds of the way up the outward face, and in play it could not be found:
from below it is inside the building's own silhouette, from above it is behind the parapet, and a bare
glyph run at a kilometre is a smudge whichever way you are looking. The sign is a widget now - a dark
pill, the floor's own tint as a bar down each edge, white pixel text - sitting directly over the roof
where a climber approaching the building is already looking.

- **World space, never Screen.** A screen-space widget component is HUD rather than scene: not occluded,
  no falloff, and it ignores its own cull distance. `AVoxelClimpPickup::IconWidget` learned this the
  expensive way - see "The icon widget was drawing every weapon on the map onto the screen at once".
- Yawed **outward from the tower's axis at spawn**. The spiral is climbed from the outside, so facing
  outward is right almost always and costs nothing ever, where turning it to the camera would be a
  rotation per sign per frame. Two-sided, so the far side still reads. The summit stands ON the axis,
  where "outward" means nothing, so it faces back down the run-in the player arrives along.
- Scaled against the building - the sign is about two thirds as wide as the roof - so the summit's name
  comes out three times a floor's, which is the whole reason that lighthouse is three times the size.
  1,634 uu wide on a checkpoint, 4,900 uu on the summit.
- The text is **white in every state**, in the project's Gome Pixel face. The stage tint is already the
  colour of the rock the building is made of, so a coloured caption competes with the one signal the
  tower gives for free; whether a checkpoint has been claimed is said by the dome that fires when it is.
- The last one reads "SUMMIT - GALAXY": "10" is a position in a sequence, and the top of the tower is a
  destination.

## Every floor gets a floor's worth of height (2026-08-20)

The tower's top three floors were arriving within a few hundred units of one another - close enough that
the height rail drew two of them on the same row, and close enough that reaching floor 9 did not feel like
reaching anywhere.

**It was the height mapping, not the rail.** Contact points are sampled by ARC LENGTH, which is what makes
the island density even from the water to the peak, and the height was hung off the same quantity. But the
coil TAPERS: turn ten is a fifth of the length of turn one, so it collected a fifth of the height.

`FSettings::EqualHeightPerStage` (0.8) blends the two mappings. Alpha is how far round the whole coil a
sample is - which, one turn being one floor, is exactly "floors completed" - so height that follows Alpha
gives every floor an equal share, and height that follows travelled distance gives the old weighting. Both
are monotonic in Alpha, so any blend of them still only goes up. **The sampling is untouched**, so the
density the arc-length walk exists to give is unchanged.

Held short of 1 on purpose: the top of a mountain IS closer together than the bottom, just not five times
so. `Cubiciousflage.Climp.TowerShape` asserts it from both ends - no floor thinner than 60% of an even
share, none thicker than 160%, and the last floor still tighter than the first, because the first two
assertions alone are satisfied by a cylinder. Reachability is unmoved: worst step 1,632 uu, worst rise
582 uu.

## Arriving is announced, and winning ends the run properly (2026-08-19)

### It was not possible to tell you had entered one

Three things happened when a player walked into a lighthouse and none of them were where the player was
looking: the inventory refilled (other side of the screen), a sign recoloured (behind them by then), and
the stage counter moved (a rail on the right edge). Now:

- **Sound.** Two new cues, `EVoxelCue::ClimpCheckpoint` and `EVoxelCue::ClimpSummit`, played at the
  building. On the **Effects** bus rather than Interface, because arriving somewhere is a thing the world
  does and it should duck with the rest of the world.
- **A dome around the WHOLE BUILDING.** The barrier was previously sized to the trigger, which is a box
  over the roof - a shimmer the size of a doorway, which is precisely something a player walks through
  without noticing. It now swells to `max(radius x 2.4, height x 1.15)` and is centred on the building
  rather than on the marker floating above it, so at full swell the lighthouse is inside it from any
  angle. Amber unclaimed, green claimed, holding full while it grows and fading over the last third.
- **A burst under the player's FEET.** The dome says where the boundary is; this says it was YOU who
  crossed it. Placed at the pawn's base, not its origin - a capsule's pivot is its centre, and an effect
  at chest height reads as an aura rather than as ground being stepped on. Tinted to the floor's colour.

### "Congratulations, you have reached Floor 3 - Canyon"

`AVoxelClimpPlayerController::Client_AnnounceFloor` is a **client RPC**, not something the HUD notices for
itself. Arriving is a server decision - the run state accepts or refuses the checkpoint - and no
replicated value says "and it was YOU who did it": the stage number changes for everybody in the session.

The banner is the **match HUD's result card**, structure unchanged: rainbow frame, cream panel, small
caption, headline, body line. Reaching a floor and winning a round are the same kind of event, and a mode
that invents its own place for "you finished something" teaches the player a second layout for nothing.

**The summit banner does not clear itself.** A floor banner is an interruption and goes after six seconds;
the summit one is the end of the run, and a congratulation that vanishes while the player is still looking
at the view they climbed a kilometre for is not one. It also plays `PlayVictoryMusic()` and the same
`EVoxelVfx::Victory` confetti the match HUD fires on a won round - one ending, presented one way.

Verified: `-ExecCmds="ClimpSummit"` on a real-RHI run logs `reached Climp checkpoint 10/10 and completed
the run`, with the banner up and no Slate ensure in the log.

## Looking at the HUD, and what looking found (2026-08-19, third pass)

Everything in this section was found by taking a screenshot, and none of it was visible in a log. The
rail measured perfectly while drawing its top two floors on the same pixel row; the sign was built,
placed and replicated while being clipped to `2 - WOODL`; the shell was created, sized and coloured while
rendering nothing at all.

**`ClimpAfter <Seconds> <Command...>` and `ClimpShot [Seconds]` exist so that can keep happening.**
`-ExecCmds` fires while the loading card is still up, and on this map everything worth driving happens
after it: the tower is generated from the run seed, the pawn is moved to the Lobby marker, and the startup
gate puts the pawn back into `MOVE_Walking` - which undoes a `VoxelFly` issued at load. A headless session
could start the game and could photograph it, and could do nothing in between. `ClimpShot` uses
`FScreenshotRequest` with **bShowUI true**, not the `HighResShot` console command: that command renders the
scene, produced nothing under `-unattended`, and the HUD is the point.

```
UnrealEditor-Cmd.exe <uproject> /Game/Cube/Map/Map_Climp -game -windowed -resx=1280 -resy=720 -nosplash -unattended "-ExecCmds=ClimpAfter 20 VoxelFly 1 3000,ClimpAfter 23 ClimpGoto 2,ClimpAfter 85 ClimpShot 0"
```

Note the SECONDS ARE GAME TIME. Under heavy streaming a 70 s delay took over two minutes of wall clock,
which looks exactly like the command never firing.

### `ClimpGoto` was teleporting into open sky

It re-derived the path and dropped the pawn on the first SPLINE SAMPLE of the requested stage. The route
wanders up to `IslandLateralJitter` from the line it is threaded on - two hundred metres at the base - so
every arrival was somewhere with nothing under it or near it. In a screenshot that is indistinguishable
from streaming having failed, and it is not. It reads `GetStageCheckpoints()` now, stands off the building
at a distance sized from the lighthouse, and turns to look at it. Same lesson as the ore pickups: go to
where the geometry IS.

### The trigger was above the building, not inside it

`Trigger` was the actor's ROOT, and a root component cannot be offset from its actor - the actor sits
clear above the roof so its sign is not buried in the masonry. So the only volume in the world was a box
floating six metres over the lighthouse, and the only way to claim a floor was to climb on top of it.

`CheckpointRoot` is a bare scene component now and everything hangs off it, exactly as
`AVoxelClimpPickup` had to learn. A floor's volume is **the inside of the building** - base to roof with a
little over, so walking in at ground level or dropping onto the roof both count. The summit's stays on the
roof, because reaching the top is the win condition and the roof is the top.

### The sign

Three separate faults: it was clipped, it did not turn, and it was too high.

- **Clipped.** A widget component renders to a fixed draw size, so a name longer than the pill simply ran
  off the end - `2 - WOODL`. The text sits in an `SScaleBox` on `ScaleToFit`/`DownOnly` now: long names
  shrink, short ones keep the authored size rather than being blown up to twice their neighbours' height.
- **Fixed facing.** It was yawed outward from the tower axis at spawn, which is right almost always and
  wrong exactly when a player approaches from the inside of the coil. It turns to the viewer on a **10 Hz
  timer** - ten actors, ten yaw updates a second, against ten registered tick functions for the same
  result. Yaw only, like the pickup signs: a sign that pitches at a player far below reads as a billboard
  rather than as something standing on a building.
- **Too high.** It floated a further nine metres over the roof, which photographs as an unattached label.
  It sits just clear of the parapet now, half its own height above the roof.
- **Only there once you were nearly on top of it**, which had two causes and needed both fixed. Its cull
  distance was 20,000 uu on a tower whose coils are hundreds of metres apart - 80,000 now. And a
  `UWidgetComponent` only redraws its render target when it TICKS, and by default does not tick while
  nothing is looking at it: a sign first seen from a distance came up blank and filled in only once the
  player had turned towards it and closed the gap. `SetTickWhenOffscreen(true)`. Ten of these redrawing
  static text is nothing; the symptom it removes is the whole feature appearing not to work.

### The shell, and the one thing that cannot be done from code

The barrier is up permanently rather than fired-and-gone: a shell you can see from the coil below is what
tells a climber the building ahead is a thing you go INTO. A floor's is a sphere around the whole
building; the summit's sits ON TOP of it, because that is the storey that matters there.

**It drew nothing at first, and the reason is size.** `M_Barrier_Inst` is authored for a shield around a
CHARACTER - two metres across, seen from a few metres - and this shell is sixty metres across seen from
eighty. At its authored `Radius`, `Fade Distance`, `SamplingScale` and `Noise Tiling` it is completely
invisible at that scale, with no warning anywhere. Those four are set from the lighthouse's own size now.

**Its `Color` parameter looked dead for two rounds, and was not.** Set to amber, set to green, set as a
vector and as a double vector, the shell kept rendering the cyan of its hex texture - so this doc briefly
recorded the parameter as unreachable. It was **drowned, not dead**: at a dense grid and a high line boost
the texture's own emissive swamps the tint entirely, and opening the grid up let it straight through.
Worth remembering the next time a parameter appears to do nothing - "no effect" and "an effect you cannot
see past" look identical from where the call is made.

The shell is **energy blue** unclaimed and **green** claimed, on both the barrier and the sign, so the two
halves of a checkpoint say the same thing: the shell is what is legible from the coil below, the sign is
what is legible standing on the roof.

### It was a black honeycomb, and the building was behind it

At a dense grid the shell's hex cells are opaque black, so the lighthouse inside it simply was not there -
and a shell you cannot see the building through is a wall, when the whole job of the shell is to point AT
the building.

**`Sphere power` is the parameter that fixes it**, not the opacity. It is the fresnel exponent: pushed up,
the shell goes clear where it faces the camera and keeps its glow at the rim, which is the difference
between an energy bubble and a mesh. With the grid opened up, the line brightness raised to carry what is
left, `HardnessPercent` down and the resting alpha at 0.18, the result is a translucent blue dome with the
building plainly inside it.

The alpha is set **for the pair of shells, not for one**: two translucent spheres stacked are not one
translucent sphere, and that doubling was half of why the first version read as opaque.

A second sphere at **negative scale** sits just inside the first. A sphere seen from within shows the
renderer its back faces, and whether those draw depends on one checkbox in a material this code does not
own - a bad thing to depend on when the entire point of the shell is the moment the player is inside it.

### Sound, and a burst under the player

`EVoxelCue::ClimpCheckpoint` and `EVoxelCue::ClimpSummit`, on the **Effects** bus rather than Interface:
arriving somewhere is a thing the world does and should duck with the rest of the world. The VFX burst is
placed at the pawn's BASE, not its origin - a capsule's pivot is its centre, and an effect at chest height
reads as an aura rather than as ground being stepped on.

## The height rail was still the five-stage one

The rail on the right read `STAGE :` over `0 m 0%` with ticks 1 to 5 on a ten-floor tower. Two separate
faults, both from when Climp had five stages:

- the tick loop was a literal `for (Stage = 5; Stage >= 1; --Stage)`;
- and `"STAGE 1/10"` in the heading face does not fit a 126 px panel, so it was **clipped to the word
  STAGE** - the number, which is the entire content of that line, was the part cut off.

Rebuilt against the reference layout, mirrored to the right edge where this HUD lives:

- **FIN** caps the top, a blue-bordered **BASE** caps the bottom, and the track fills **blue** from the
  base to the player. One blue for the fill, the base cap and the player's own name tag, so the three
  read as one instrument.
- The **height readout is green** and leads, because metres is the number a climber quotes; the floor
  counter sits under it, quieter, as `F 3/10` - short enough to fit the panel it is drawn in.
- **Ten floor ticks, placed at the floors' OWN heights**, read from `GetStageCheckpoints()` rather than
  spaced evenly. The checkpoints are where the generator put them, and a rail that spaces them evenly is
  telling the player something about the tower that is not true. Ticks past the run's total collapse, so
  a shorter run needs no rebuild.
- **A name tag per climber**, riding at that climber's own height with its metres beside it and a pip
  pinning it to the rail: you in the rail's blue, everyone else on a stable hue taken from their player
  id - taking hues from the array index instead would recolour the whole rail whenever somebody joins.
  Eight slots; beyond that the rail is a wall of names.

**How a marker is positioned, because Slate has no "put this at 40% of my height" slot.** Each marker is a
lane: an empty box, the marker, another empty box, with the two empties' fill weights bound to `1-f` and
`f`. A constraint canvas cannot bind its offsets to attributes and a custom panel would be a hand-written
layout pass for a HUD element; two bound fill weights get there with the layout code that already exists
and update on the same attribute read as everything else. **The weights are floored at 0.0001** - a
zero-weight fill slot collapses, which snaps a marker at either extreme onto the opposite end, and the
two extremes are exactly where the summit tag and the base tag live.

**100% is the generator's summit, not the tallest thing in the level.** `Construct` scans every placement's
bounds as a fallback, which picks up whichever landmark floats highest - so the rail's top was somewhere
nobody climbs to and a player standing on the summit read well short of it. `ResolveFraction` prefers
`GetSummitLocation().Z + GetSummitStampHeight()`, which is the roof that ends the run.

### What the screenshot then said about it

- **It is the full height of the screen**, less a margin at each end. At 460 px in the middle of the view
  ten floors were compressed into a space where the top two landed on the same row.
- **The border is not the panel colour.** It was, which is not a border - it is two more pixels of
  background, and the rail read as a floating smudge rather than as an instrument with an edge. Two nested
  borders: the rule, then the panel. The panel is 0.52 alpha, because a rail this tall at 0.80 is a black
  column down a fifth of the screen.
- **White is the ordinary colour**, with two readings coloured on purpose: the live height GREEN and the
  personal best BLUE, one above the other, so which of the two moved is answerable without reading either
  label. A floor number turns green when it has been climbed.
- **A name tag was running across the floor numbers and out through the side of the panel.** Slate does not
  clip an overflowing child, and the tag is right-aligned against a rail it grows away from - so a long
  name is cut in code at nine characters AND the column clips to its own bounds. The rail also went from
  208 to 300 wide, because at 208 the tag column was about ninety pixels once DPI scaling had its share
  and `DESKTOP-4F2H` was simply wider than the space it had, metres and pip included.

**Floors 9 and 10 sat on top of each other, and it was NOT the rail.** The rail was reporting the tower
honestly: height followed the length of the coil, the coil tapers, and turn ten is a fifth of the length of
turn one - so it collected a fifth of the height. See "Every floor gets a floor's worth of height" below;
the fix is in the generator, and the ticks separated on their own once it landed.

### And then the panel went away entirely

- **No background and no border.** The rail is a gauge drawn over the world, not a window cut into it. The
  floor numbers and the readouts carry a **1 px dark outline** instead - a white glyph on a bright sky is
  not there, and an outline is what lets the background go away without taking legibility with it. Every
  dark thing left is a PILL around something that has to be read: the two caps, the best mark, a name.
- **The name tags sit against the line.** The floor-number column was 84 wide against two-digit labels,
  which parked every tag most of a centimetre clear of the rail it points at. 46 now, with 3 px gutters.

### The personal best

`UVoxelUserSettings::ClimpBestHeightMetres`, beside the volume sliders, because a best that resets with
the run is not one. Updated in memory from the controller tick that already runs and flushed to disk on
`EndPlay` and on completing the run - saving on every improvement would write a file on most frames of a
climb, and a player who closes the game standing on the summit is the likeliest person to have just set
one.

**Measured from the waterline**, which the generator puts at Z=0. The first version measured from wherever
the pawn happened to be on its first tick, which is not a baseline: a player teleported up the tower
before their first tick recorded a best of zero metres from the summit, and two runs that started
differently produced two bests that could not be compared.

## Looking at it

**Edit level mode does not generate the tower.** `IsProceduralMap()` returns false while creator mode is
active, so the director builds the map's *baked* asset - the converted ZhangJiajie map - and the seeded
world is never run. A level editor showing the island, the floating pillars and no spiral is that rule
working, not the generator failing.

`ClimpPreviewTower [Seed]` builds it anyway, in whatever world is running, keeping distance streaming on
so creator mode does not construct thirty thousand placements at once. `ClimpDrawRoute [Seconds]` draws
the spline alone.

## The flat squares, and why the size band let them in

The tower drew its bodies from converted map libraries filtered by a SIZE BAND - narrow enough to jump
between, wide enough to stand on. That band has nothing to say about SHAPE, and a modular map kit is
largely made of modular ground tiles. Measured on the shipped libraries with `ClimpShapeReport`:

> 1,395 definitions, 235 classified as slabs (17%). Of the 646 inside the body band, **98 are slabs** -
> `Tiled Floor`, `Cobbled Street`, `Pavement Slab`, `Concrete Floor Plate`, `SM_building_floor_01`,
> `Open Water`, `Sand Ground`, `Forest Floor` - all of them 320x320x10.

Hung in the sky as flying islands they read as exactly what they are: flat grey squares.

**`MeasureShape` measures three things and `IsSlab` needs all three.** Each on its own is wrong about a
real shape: footprint fill alone condemns a solid boulder, top-flatness alone condemns a rock with a
shelf, thinness alone condemns a wide low one. A ground tile is the only thing that is all three at
once - fill 1.00, top-flat 1.00, flatness 0.03. The report's "least tile-like" tail is walls, which have
fill 1.00 and top-flat 1.00 and are saved only by the flatness test; that tail is why the report prints
both ends rather than only what it caught.

**It refuses to judge on too little data.** A definition with fewer than `SlabMinimumColumns` occupied
columns is not measurable - a single column is trivially "all at the same height" and any top-surface
test calls it a slab. Unmeasurable means NOT a slab: a wrongly rejected body is a hole in the route, a
wrongly kept one is a flat rock. This is also what keeps the generator's own tests working, since
`MakeTestBody` builds definitions with one occupied cell.

**There were FOUR ways in, and filtering three of them left the job looking undone.** Bodies and
satellites go through `MakePools`; dressing goes through `MakePools` too but on a separate filter; and
the object catalog is APPENDED to the stage palette without going through `MakePools` at all. The
catalog carries a ground entry per theme, which is where `Sand Ground` and `Open Water` were still
coming from after the first two fixes - 15 to 22 placements each, spread the full height of the climb.
The report's per-definition placement counts and Z range are what found it; without them a surviving
tile inside the checkpoint lighthouse and one hanging on the route look identical in the output.

Result: **0 of 506 body-band definitions are slabs.** The only slabs left in a generated tower are the
ZhangJiajie landscape components at Z about -100, which are the beach the run starts on.

## Rocks are scaled up to do the job the tiles were doing

Rejecting the tiles takes the widest, flattest, most landable things out of the pool. Scaling is the one
lever that costs NOTHING to pull: instances of a definition share a single runtime voxel asset, so a rock
placed at 4x is the same mesh build as the same rock at 1x, while a differently-shaped rock is a whole
new one at about 96 ms of it.

**Bodies are grown only if they are too small, and only until they clear the minimum.** Scaling every
body towards a single target size was the first attempt and it worked - every island came out at 520 uu,
which is a tower of one rock at one size. That is the same failure the even-spread variant cap exists to
prevent, reached from the other direction: the cap was carefully keeping a range of SHAPES while the
scale quietly threw away the range of SIZES. Measured after the fix: bodies x1.0 on average, x1.5 at most.

**Satellites are where the enlargement gets to be dramatic**, and the reason is the reason the two pools
exist at all. A route step must stay under `MaximumIslandFootprint` (650 uu) or the spiral fuses into a
ramp - `Cubiciousflage.Climp.TowerShape` asserts it - so no amount of scaling can make a route body big.
Nothing is jumped from a satellite, so it answers to `MaximumSatelliteFootprint` (2,600 uu), four times
the room. `SatelliteFillShare` grows them toward it: measured **x2.5 on average and x5.0 at most**, which
is what turns a string of stepping stones into a field of flying rock.

Scale is per placement and rides in `FVoxelLevelPlacement::Scale`, which `GetTransform` and
`CalculatePlacementBounds` both already honour - so streaming bounds, distant proxies and collision all
follow a scaled body without further work.

`ClimpShapeReport [Count]` prints the whole classification for the standing level: the three metrics, the
size, whether it is a slab, whether it is in the body band, how many placements use it and over what Z
range. Both ends of the sort, because a threshold is only right if it also leaves alone what it should.

## Scaling broke the distant proxies, silently

``ShouldProxy`` measured a definition's AUTHORED footprint against `MinimumFootprint` (900 uu). Once
satellites started being placed at up to 5x, a definition authored at 300 uu became a 1,500 uu boulder in
the world and was still judged at 300 - so it was refused a proxy. The effect was the exact inverse of
what the system is for: **the biggest rocks on the tower were the only ones with no distant stand-in**,
appearing at the load radius with nothing before them.

`ShouldProxy` takes a scale now, and `BuildDistantProxies` passes the LARGEST scale any placement of
that definition uses - one proxy mesh serves every instance, so eligibility has to be decided for the
biggest of them. The material's `VoxelSize` scalar is multiplied by the same factor, for the reason that
parameter exists at all: the cube shade mask has to match the size of the cubes it is shading, and an
instance drawn at 5x has cubes five times the definition's own.

Nothing else needed changing: instances were already added with `Placement.GetTransform()`, which carries
scale, and `SetProxyVisible` restores the same transform.

**Measured: 75 definitions standing in for 2,408 placements, before, against 259 for 3,562 now.** The
cost is about 5 s of the startup gate - 259 definitions x 2 LODs is 518 `BuildFromMeshDescriptions`
calls, on the game thread, in `BuildDistantProxies`. It sits behind the loading card so it is not a
spike, but it is now the dominant startup cost, and the same worker-thread split `VoxelDefinitionMeshBuilder`
uses would apply to it directly if that becomes worth doing.

## Pickups: two systems, because there are two verbs

**Ores are MINED. Everything else is TAKEN.** They started as one system and splitting them fixed three
problems at once.

**Ore deposits are level geometry.** Six designs - metal and energy, three sizes each - authored
`destructible true` and appended to the generated level by `BuildAndApply` as ordinary placements. Being
placements rather than actors means they stream with the rest of the level, get the two-LOD silhouette
proxies, and break like the rock around them; none of that had to be written. The previous version
spawned them as several hundred always-resident replicated actors and then hand-rolled a draw-distance
cull to hide the cost, which was a worse copy of the streaming that already existed.

**The harvest already existed too, and it keys on the DISPLAY NAME.** `AVoxelClimpGameMode::AwardHarvest`
credits cubes from shot scenery, choosing the material by searching the definition's name for
`energy`/`crystal` or `metal`/`iron`/`steel`. `Metal Ore Large` and `Energy Ore Small` are already the
names it is looking for, so shooting a deposit credits the right material with no new code. **Those names
are load-bearing** - a rename that reads as harmless turns an energy deposit into a Block deposit with no
error anywhere - so `Cubiciousflage.Climp.ModeDefaults` now asserts each ore's name still contains its
keyword, that all six are destructible, that the three handed pickups are NOT, and that the energy ores
carry emissive cells (which also covers the `glow` keyword surviving `Build`).

### Four bugs the first playable pass found

**F did nothing on an ammo crate.** `GetOreAmount` returned 0 for every non-ore kind, so `TryTake` asked
`AddAmmo` for zero rounds, which added nothing, reported that it added nothing, and the pickup correctly
refused to consume itself. Every part of the chain behaved properly around a number that meant "nothing".
Ammo carries 30, a weapon cache 90, a skill orb 3 - and the last one matters even though skills are
restored wholesale, because zero is what the pipeline reads as empty.

**An energy deposit credited Block cubes.** `AwardHarvest` identifies shot scenery by searching its
DISPLAY NAME, which is all there is for converted map geometry - but Climp's own deposits have
deterministic ids. `VoxelClimpPickupDesigns::TryGetOreMaterial` answers by id and is checked first; the
name heuristic remains for everything else.

**The ores were still small.** Tripling the CELL COUNT made them denser but only about half again as
wide, because size in the world is cells times cell size and only the first half had been touched. Cell
size went 12/14/16 to 34/40/46 uu.

**The icon widget was drawing every weapon on the map onto the screen at once.** It was
`EWidgetSpace::Screen`, which is HUD rather than scene: not occluded, no falloff with distance, and it
IGNORES the cull distance set on it. `EWidgetSpace::World` makes it a quad in the level like the sign
beside it - occluded by rock, smaller further away, gone past the pickup's cull range - two-sided and
yawed outward from the tower axis so it still needs no per-frame rotation.

**No weapon caches on the route.** A run starts with all four weapons, so a cache granted nothing a
player did not have and cost a detour to reach. Its share went to skills: the split is two thirds
ammunition, one third skill orbs. The Weapon kind is kept - a hand-placed encounter could still use one,
and an ammo crate borrows its icon and weapon type - but the generator never chooses it.

**Signs and icons face the viewer, from ONE tick.** Billboarding is per-frame work and there are several
hundred pickups on a route, so a pickup that turned itself would register several hundred tick functions
for a rotation - most of them for things nobody can see. `AVoxelClimpPlayerController::FaceNearbyPickups`
does it instead: one overlap inside `PickupFacingRadius` at 10 Hz, from the controller that was already
ticking, calling `FaceViewer` on what it finds. YAW ONLY - a sign that pitches to point at a player
standing above it reads as a billboard rather than as something standing on a rock, and on a tower the
viewer is usually well above or below. Purely local: nothing about it replicates.

**Label height comes from the design, not a constant.** It assumed 25 cells - the tallest ore - for every
kind, which floated the sign 420 uu over an ammo crate seven cells tall. It reads `GridSize.Z` now, and
the icon sits 90 uu above the sign rather than 260.

**The icons are the HUD's own brushes.** A skill orb shows all three ability icons because it restores
all three; `PropSkill.0` is the dashboard glyph, a generic icon rather than an ability, which is why
these looked like they had never been hooked up - the real ones are 1, 2 and 3.

**Ammo and skill orbs stay actors taken with F**, because they are handed over rather than broken
out of the world. Roughly 180 of them against 400 ore deposits; the F prompt, the icon widget and the
rainbow apply only to these.

Five kinds - ammo, weapon, skill, metal ore, energy ore - as real voxel objects.

**Emissive is authorable now.** The gap this doc recorded is closed: `FVoxelObjectBuilder` carries per-cell
emissive and the object script has a `glow` keyword. It is a MODE, not a per-shape argument - `glow 200`,
shapes, `glow 0` - because emissive describes a surface rather than a shape. Two things that would have
been silent bugs: `Mirror` reflects the emissive map with the cells, and `Build` emits `CellEmissive` only
when something actually glows, since the validator wants it empty-or-exactly-as-long and every catalog
object is unlit.

**F, not walk-over.** The old pickups were consumed by overlap, which on a climb is hostile: the route is
narrow, a player lands where the geometry lets them, and a resource they were saving is spent by a
landing they did not choose. The prompt is resolved once per controller tick from an overlap query,
weighted by facing rather than filtered by it, and a full inventory reads `- no room` rather than the
prompt vanishing - a prompt that disappears is indistinguishable from a pickup that is not there. Range
is re-checked on the server; the client chooses WHICH, not whether it is close enough.

**The rainbow is the carry tool's.** `M_PlayerHitRainbowOverlay` is the material E marks a held assembly
with, applied here as an overlay on the pickup's cluster components. A player who has used E once already
knows what an animated rainbow edge means. Applied to the CHILDREN - `UVoxelComponent` draws nothing
itself in a game world - on a self-clearing TIMER rather than a tick, which is both what CLAUDE.md asks
for deadline-shaped work and what keeps `Climp.ModeDefaults` passing: a pickup is event-driven.

**Weapons and skills carry their own icon.** A `UWidgetComponent` in SCREEN space, which the renderer
orients at the player for nothing - several hundred of these cost no rotation, no tick and no manager,
where a world-space quad turned to face the camera would be a per-frame system forever. Black icon on a
white disc at 55% opacity, because the icons are authored dark and most of this tower is sky behind them.
Ores get none: a glowing blue crystal and a grey rock already say what they are, and a disc over every
deposit is a screen of discs.

**Ore voxel counts were tripled without scaling anything.** The grids went 7/11/15 to 10/16/22 cells,
which is about three times the cubes at the SAME cube size. A transform scale would have given the same
handful of cubes at triple size - bigger blocks, not more of them - which is the opposite of what a
denser ore should look like.

**Labels use the project face.** `FVoxelUIStyle::WorldFont()` already exists for exactly this - an offline
UFont of the same Gome Pixel face the menus use, kept because TextRender cannot consume Slate composite
fonts. A hand-rolled 5x7 voxel glyph table was written first and thrown away when that turned up; the
project having one answer for world text is worth more than a second one. One `UTextRenderComponent`,
yawed OUTWARD FROM THE TOWER AXIS at build time rather than turned to face the camera each frame: a
billboard is a rotation per label per frame and several hundred pickups make that a permanent per-frame
system, while the spiral is climbed from the outside so facing outward is right almost always and costs
nothing ever.

**Density, and why it was invisible.** At one ore per 22 route islands the tower carried 61 ores over
944,000 uu of route - about one every 15,000 uu, which is why they could not be found. One per THREE
islands is one every 2,100 uu of path, and a spiral packs several coils into any 100 m sphere: 449 ores.

**The bug that made them invisible: RE-DERIVING THE TOWER.** The game mode used to regenerate the tower
to find out where the route islands were, on the argument that generation is deterministic from the seed.
It is - for identical INPUTS. `BuildAndApply` supplies a dressing resolver, a body resolver, a landmark
resolver and a checkpoint stamp; the re-derivation supplied none of them, and pools of a different size
draw a different number of values from the same random stream. The second tower came out the right
height, the right width, with the right island count - and its islands were somewhere else. Checkpoints
landed close enough to look right; four hundred ore pickups placed on those island tops hung in the air
beside the route, which is how it was finally caught.

`AVoxelClimpGameState` keeps `RouteIslandTops`, `StageCheckpoints` and `CheckpointStampHeight` from the
build that ACTUALLY RAN, and everything that puts an object on the route reads those. Not replicated -
every machine builds the same tower from the same seed with the same resolvers, so putting a few thousand
vectors on the wire would undo the design.

Measured after the fix: **mean 2 uu from real geometry, worst 30 uu**, over a sample of the planned ores.
Note that SPREAD IS NOT THE TEST - ores scattered evenly through the tower's volume and ores sitting on
its islands have the same bounding box, which is exactly what the broken version looked like in a log.

**The bug that ACTUALLY piled them up: `AVoxelActor`'s root IS the voxel component.** The pickup centres
its visual on the actor, because a voxel object's origin is its LOW CORNER - and
`VoxelComponent->SetRelativeLocation(-Extent)` moved THE ACTOR rather than the mesh, because that
component is the root. Every pickup teleported to roughly (-90, -90, 0), which is the spiral's own axis
at the waterline: 449 ores in one heap at the foot of the climb. It broke the interaction too, since
`TryTake` range-checks against `GetActorLocation`, so a pickup you were standing next to was at the
bottom of the tower as far as the server was concerned. `AVoxelClimpPickup` owns a `USceneComponent` root
now and attaches the voxel component under it.

**The lesson, and it cost two rounds of "it is fixed now": MEASURE THE ACTORS, NOT THE PLAN.** The plan
being right and the actors being in one heap are entirely compatible, and the log said "spread over
sixty kilometres" throughout both of them. The check reports the bounding box of the SPAWNED pickups now
- the number that collapses to a point when this breaks. Verified after the fix: planned spread and
actor spread are identical at 68,437 x 69,330 x 52,882 uu.

**They are built behind the loading gate, and then behind it again.** 449 actors each with a voxel
component put ELEVEN SECONDS on the startup gate for objects nobody needs in order to stand up. The
layout is planned in one deterministic pass and the actors are built sixteen per 0.1 s on a timer - and
that timer refuses to run at all until `IsInitialStreamingReady`, because taking them off the gate is
only half the win if they then compete with it for the same game thread.

Per-frame cost of a pickup: none. Nothing ticks; the prompt rides the controller tick that already ran.

## Two pools, and why

A route step and a piece of scenery are asked different questions, and one size band cannot answer both:

- a **route step** must be narrower than the gap to the next contact point, or the spiral fuses into a
  ramp, and close enough to be jumped to. Together those cap it at `MaximumIslandFootprint = 650` uu
  against `IslandSpacing = 700`.
- a **satellite** is jumped from by nobody and sits off the route at `SatelliteRadius`, so it may be as
  large as the library allows: `MaximumSatelliteFootprint = 2600` uu. This is where `Big_Mountain_02`
  (2,234 x 2,497 uu) lives - in the spiral, beside the route rather than on it.

Raising the spacing instead was tried and measured: at 2,800 uu the widest bodies stop overlapping, but
the route's worst step becomes 3,486 uu, which the reachability check refuses. The two requirements
genuinely conflict on one pool.

The variant cap now takes an **even spread** across the sorted pool rather than its head. Truncating to
the front built the whole tower from the two dozen largest bodies in the band - one repeated slab.

## Measured, 2026-08-18

Kept as a dated record of what the tower's shape work bought. Note that its height figure predates
`EqualHeightPerStage` and the summit run-in; the current numbers are in the table after it.

| | before | now |
| --- | --- | --- |
| tower | 60,000 uu tall, turns unknown | 150,000 uu tall, radius 25,000, exactly 10 turns |
| contact points | - | 2,255, one cluster each |
| spawn to nearest tower object | ~22,000 uu, against a 12,000 uu load radius | **78 uu** |
| ground floor | 8,433 sliced out | 13,169, the whole converted map |
| total placements | 9,454 | 32,288 |
| distance streaming | never started | running, 64 resident at spawn |
| startup gate | 7.7 s | **0.9 s** |

## Measured, 2026-08-20 — the state it is being left in

One seed's figures, from a headless `-game` run. They move a little run to run because the tower does.

| | |
| --- | --- |
| tower | ~50,300 uu to the summit, radius 25,000 → 5,000, exactly 10 turns |
| route | 1,356 route island tops, worst step 1,756 uu, worst rise 1,103 uu, never descending |
| scatter | 5,429 satellite tops, a mean 2,093 uu clear of the route |
| total placements | ~13,000 generated + the whole 13,169-placement ground floor |
| ore deposits | ~430, roughly 2:1 metal to energy, ~40% of them off the route |
| pickups | ~990 planned; **7–11 alive at once** inside the 12,000 uu load radius |
| lighthouse clearance | 354 placements culled out of 10 footprints |
| streaming walk | 7.7 ms mean pulse, 84% of build work on worker threads |
| startup gate | 0.9 s |
| tests | 16 pass, 1 fail (`Voxel.BaseMapRegistry`, pre-existing) |

## The rewards: found, scattered, streamed and spent (2026-08-20)

Six things were wrong with the pickups at once, and four of them shared a cause: everything about them
was checked as a PLAN and nothing was checked as a thing a player meets.

### The metal ore was there all along, and could not be seen

Every count was right. Every run placed two to three hundred metal deposits, on real geometry, mean 2 uu
from a surface — and the report of "I can't see any metal ore anywhere" was completely accurate. Two
causes, and both of them are about the difference between existing and being findable.

**It was a grey rock on a grey rock.** The energy deposit was found immediately because it EMITS; metal
was authored `iron`, `slate`, `silver`, `steel`, which is exactly the palette the tower's rock is tinted
from. At streaming distance a two-metre grey lump standing on a rock island IS the island. Metal now
carries glowing amber veins — `glow 170..200` on two outcrops per size — in a body that stays grey, so
the two materials still read as different things. `Climp.ModeDefaults` asserts the veins exist AND that
they are under a quarter of the cells: a deposit that glowed all over would be a second energy crystal.

**They were still at scenery scale, and the recorded fix had never reached them.** This doc records ore
cell size going from 12/14/16 uu to 34/40/46 uu because "the ores were still small". That number lives in
`VoxelClimpPickupDesigns::GetVoxelSize`, which is the ACTOR path — and the ores stopped being actors when
they became level placements. A placement carries the definition's own `VoxelSize`, and the object script
gives every definition the 10 uu scenery cell, so the deposits quietly went back to 220 uu for a large
one. A fix landing in the half of a split system that no longer runs is invisible in every test.

`GetDepositScale` puts it back as `FVoxelLevelPlacement::Scale`, at 1.5/1.7/1.9 rather than the actor
path's 3.4–4.6. **The two paths have different constraints and reapplying the actor's number would have
been the opposite mistake**: an actor pickup has no collision, a deposit is level geometry a player lands
on, and at 4.6x a large deposit is 1,012 uu — wider than the 650 uu route island under it, and a wall
across a route whose reachability was checked without it. `Climp.ModeDefaults` asserts a deposit stays
inside `MaximumIslandFootprint`. Scale rather than voxel size, so all deposits of a design still share one
mesh build.

Measured after: 305 metal and 179 energy deposits, a medium metal one 375 x 375 x 272 uu, and
`ClimpGotoOre metal` puts the camera in front of one — which is the only check that would have caught
this.

### The signs had never drawn a single character

Not "were hard to read". Had never rendered, at all, since they were written. `UTextRenderComponent` can
only draw an OFFLINE font — one baked to texture pages at import — and **there is no offline font in this
project**: `GomePixel-ARJd7_Font` reports `FontCacheType 1`, which is Runtime, and so do `dogica_Font`
and `Pixel_Rand_Font`. `FVoxelUIStyle::WorldFont()` is documented as "the offline UFont companion asset".
It is not one. Handed a runtime font, TextRender draws nothing and says nothing.

So the labels were built, sized, coloured, positioned, turned to face the player, culled at the right
distance — and invisible. Every measurement anyone could take of them was correct. Its comment now
records what the asset actually is, because the next person to reach for world text will otherwise reach
for the same trap.

The word moved into the `UWidgetComponent` that was already on the actor and already visibly working.
**One component now, not two**: the disc, its glyph and the word are one quad, one render target and one
rotation, which makes a skill orb cheaper than it was. `SetTickWhenOffscreen(true)`, for the reason the
lighthouse signs need it.

While fixing it: a text render component and a widget component face OPPOSITE ways — the widget is read
from the side its forward axis points at, the text render from the side it points away from. That is now
moot here, but it is why "face the player" and the code agreeing did not mean the sign faced the player.

### The widgets were sized for landmarks, on a field

`GetLabelWorldSize` was 260 for a skill orb — against an orb 154 uu tall, so every LETTER was two thirds
again as tall as the whole pickup. Now a third of that (75/60/65/55/45), about half the height of the
thing it names, and the icon disc is 96 uu rather than 256. The old argument for the large sizes was
legibility from the coil above, and it never held: at two hundred metres a 260 uu glyph is nine pixels.
It holds even less now that a pickup does not exist at all past the load radius.

**The skill orb's disc shows ONE icon, not three.** Drawing all three abilities was the literal statement
of "this restores all three", but three glyphs share the width one had, so each arrived a third of the
size on a disc that is small at range and the row read as texture. The prompt says what it gives; the
disc only has to say which pickup it is.

### They are streamed now, on the level's own rule

Nine hundred pickup actors were spawned during startup and then kept alive for the whole run — each with
a voxel component, a sign, a reach sphere and a replicated body — on a tower of which a player can see
one coil. The level's own 32,288 placements have never worked that way, and `AVoxelClimpGameMode::
ProcessPickupStreaming` now applies the same rule: built inside 12,000 uu, released past 16,000 uu, four
times a second, sixteen per pulse, nearest first. Outside LOD0 a pickup is not visible, not reachable and
not interactable, so there is nothing for it to be.

Measured at a spawn: **7 to 11 of ~600 planned pickups live**. The pass is a few hundred distance
comparisons at 4 Hz; the streaming walk is unchanged at a 7.7 ms mean pulse with 84% of build work on
workers.

The plan is now the permanent thing and the actor the temporary one, which is a change of ownership
rather than an optimisation. **`FPlannedPickup::bTaken` is not optional bookkeeping**: the moment a
pickup can leave memory and come back, "taken" and "unloaded" are the same observation from outside the
plan, and without it every crate on the route renews itself by being walked away from.

### They spawned along a line, because the route is a line

Everything on this tower was placed on `RouteIslandTops`, and a spiral's islands are a thread — so ore,
ammunition and skill orbs arrived strung along one path like markers on a track. The generator already
scatters rock left, right, above and below every contact point; the satellites are now recorded as
`FBuild::ScatterIslandTops` and rewards are placed on both pools.

**The two densities are not the same number and cannot be**, which is arithmetic rather than taste: there
are two to six satellites per contact point, so one shared chance puts four times as much off the route
as on it and quadruples the total. Route 0.18/0.26, satellite 0.035/0.045 — about the same totals as
before, with a bit under half of them off the line. Measured: 5,414 satellite tops against 1,356 route
tops, a satellite a mean 2,093 uu clear of the route, asserted in `Climp.TowerShape` against
`IslandSpacing` because a scatter whose members sit on the route tops is the same dotted line renamed.

Nothing on the guaranteed route depends on reaching a satellite: it is a detour, which is what a reward
off the path should be.

### Taken stays taken, mined stays mined

Two mechanisms, because a pickup and a deposit are different kinds of object.

**Pickups** are plan entries; `NotifyPickupTaken` marks one and the streamer never builds it again.

**Deposits are level placements**, so "it was damaged" lives in the director. Its default is to PIN a
damaged streamed placement resident forever — right for a wall a player shot a hole in, wrong for a
resource: a run mines several hundred, and pinning them all is a slow leak, while not pinning them makes
every deposit infinite. `AVoxelLevelDirector::SetConsumableDefinitions` names the six ore definitions as
resources; a placement of one is recorded consumed on first damage, unpinned, its silhouette taken down
with it, and never built again. **It is not destroyed under the player** — a deposit that vanishes out of
your hands mid-mine is a bug, not a rule — it simply does not come back once you have gone. This runs on
every peer, from the multicast damage path, because streaming is local.

Both are persisted by `UVoxelClimpRunSave`, **keyed on the run seed**. That is the design, not a detail:
a plan index and a placement index only mean anything within one seed, and the same numbers against a
different tower point at different objects entirely. A new seed therefore starts clean and the file
invalidates itself; nothing has to be cleared when a run ends. Indices, not positions — a few kilobytes
of integers, and a mismatch that cannot be mistaken for a near miss. Written on a five-second debounce
rather than per take, because a save is a file write and a player crossing a field of crates would
otherwise cause one per crate.

### Verified end to end, because F and a trigger pull are what a headless run cannot do

Everything past "it spawned" was unverifiable without a person at the keyboard, which is exactly where
the interesting bugs are. `ClimpTakeNearby` and `ClimpMineNearby` drive the REAL server paths —
`TryTake` and `FVoxelWorldDamage::ApplySphere` — not copies of them. `ClimpTakeNearby` walks the pawn to
each pickup rather than reaching across the map, because `TryTake`'s server-side range check is the point
of it and bypassing it would test a path no player runs.

`RunSeedOverride` is `Config` now, so `-ini:Game:[/Script/CubeCubeCube.VoxelClimpGameMode]:RunSeedOverride=424242`
pins a tower from a command line. Any check that spans two sessions needs the two sessions to be the same
tower, and without it they never are.

What that measured, in order:

- took 14 nearby, took 0 on an immediate repeat;
- left the area and came back: the 14 did not return, 60 others streamed in and were takeable;
- restarted the same seed: `Climp run 424242 resumed: 26 pickups already taken`, and taking again in the
  same place found nothing while 74 other pickups still stood;
- mined 13 of 13 deposits, left, returned: `broke 5 of 5 deposits in range (13 were already mined)`;
- restarted: `18 ore deposits already mined stay gone`, and `broke 0 of 0 deposits in range`.

## The rewards, second pass: shootable, marked, and legible (2026-08-20)

Six more, from looking at the first pass in the world.

### The deposits wear the rainbow now, and metal is twice the size

`M_PlayerHitRainbowOverlay` is the carry tool's material and the handed pickups have worn it since they
existed - a player who has used E once already knows what an animated rainbow edge means. The deposits
were the only resource on the tower without it, which is exactly the property that made them read as
scenery.

**It could not be done the way the pickups do it.** A pickup is an actor and applies the overlay to its
own chunks; a deposit is one of thirty thousand PLACEMENTS of six shared definitions. Writing the overlay
into the shared `UVoxelMeshAsset` would mark every instance of that definition everywhere and invalidate
the template, making every instance rebuild - the sharing is the whole reason a level is a library rather
than one grid. An overlay material is a per-primitive property, so `AVoxelLevelObject::SetChunkOverlayMaterial`
marks one instance and costs no mesh work at all.

**On the DEFINITION, resolved as each placement spawns.** `AVoxelLevelDirector::SetHighlightedDefinitions`
is a standing rule rather than a pass over what is resident: the placements stream, so a one-shot pass
marks the few dozen deposits near the player and leaves every one they walk to afterwards plain. It rides
the same short tick window `NotifyDamaged` already opens, because chunk primitives are built
asynchronously and there is usually nothing to put a material on at the moment of the call.

**Metal is now twice energy's scale** - 3.0/3.4/3.8 against 1.5/1.7/1.9. Energy announces itself by
emitting; metal is a rock among rocks and reads by bulk as well as by its veins. Worth stating rather
than burying: a large metal deposit is now **836 uu against a 650 uu route island**, so one can cover the
island it stands on. That is a deliberate trade of route width for legibility - the reachability check
knows nothing about deposits - and `GetDepositScale` is the number to bring down if the route starts
feeling blocked. The test's bound moved from the route island to `MaximumSatelliteFootprint` accordingly,
which still refuses a deposit larger than the biggest rock the generator will place at all.

### Both definition rules moved to the game state

`SetConsumableDefinitions` and `SetHighlightedDefinitions` were being called from the game mode, which
**only exists on the server** - so on a listen server a client would build deposits that never went away
when mined and never wore the rainbow. Both are LOCAL rules about what this machine builds and draws, so
they belong with the local build in `AVoxelClimpGameState::BuildGeneratedWorldOnce`, which every machine
runs. What stayed on the server is the one thing that genuinely is the server's: the `OnPlacementConsumed`
binding that writes the run's progress.

### The sign was clipped because its quad was a guess

`SKILL` came out as `SKI`. The sign is a world-space widget, whose draw size is a fixed quad in world
units that Slate clips to, and the first version computed that quad from a character count times a
fraction of the glyph height. That is wrong for any proportional face and wrong even for this one, since
dogica's advance is not its cap height - and every longer word would have been worse. It is
`FSlateFontMeasure::Measure` now, the same service Slate lays the text out with, so the quad cannot
disagree with what gets drawn.

**The icon disc is gone.** A white disc with a black glyph answered "a weapon cache does not say WHICH
weapon", which was true while the generator still placed weapon caches - it has not since a run started
with all four. What was left was a disc over every skill orb repeating what the word under it already
said, in a second visual language, at roughly the size of the pickup. Font sizes came down again with it:
48/40/42/36/30, about a third the height of the object.

### The F prompt is a button

It was a dark translucent slab with cream text - the HUD's own overlay language - and it read as a status
line rather than as a control. Every other thing in this game that says "press this" is a
`Voxel.Button.Primary`: cream box, two-pixel pink outline, square corners, dark pixel text. It is drawn
as two nested `SBorder`s rather than an `SButton`, because an SButton is focusable and hit-testable and
this sits in a HUD that must never take a click.

### Ammunition and skill orbs can be shot off a rock

The route is a spiral of flying islands and a pickup sits on top of one, so a crate you can see is often
a crate you would have to plan a detour to touch. F stays the close verb; a round is the one for the
island you are looking at rather than standing on.

Three things had to change, and the middle one is the interesting one.

**A pickup had to become a target.** Its voxel body carries no collision on purpose - a pickup a player
can stand on makes the climb solvable in ways the generator never checked - so a shot aimed at a crate
went through it and detonated on whatever was behind. `ShotVolume` is a query-only sphere that blocks the
channels a weapon uses and **ignores `ECC_Pawn`**, so it stops a round without ever becoming a step.
Sized to the design's half-DIAGONAL, not half its longest side: a sphere sized to a face leaves the
corners of a box outside it, and a crate is a box.

**`Reach` had to go, and this was the real bug.** A 260 uu sphere existed so the controller could find
nearby pickups with one scene query. An object-type trace does not consult response channels, only object
types - so the weapon's own trace stopped dead on that sphere, two and a half metres short of the crate,
and the damage that followed reported touching nothing. A shot visibly landing on a pickup collected it
or not depending on how far behind it the next rock happened to be. The controller's search is its own
400 uu sphere against `ECC_WorldDynamic` and finds `ShotVolume` just as well; `InteractionRadius` is
enforced in `CanBeTakenBy`/`TryTake`, which is where it belongs. Two query volumes on one actor was one
too many.

**The pickup check moved above `ApplySphere`'s bounds reject**, and that ordering is not tidiness. A round
stops on the shot volume, so its impact point sits on that sphere's SURFACE - a volume-radius clear of
the visual bounds the reject tests against. Below the reject, a shot that had just visibly stopped on a
crate reported touching nothing. `IsWithinShotReach` asks the shot volume instead.

`TryTake(bIgnoreReach)` skips the server range check for a shot: a round that arrived has proved its own
reach, against the weapon's range rather than the prompt's, and applying the F radius to a bullet would
refuse every shot not taken from touching distance - the one place F already works.

**A full weapon still refuses the crate**, and that is correct rather than a bug: `AddAmmo` returns zero
at capacity and `TryTake` will not consume a pickup that gave nothing. A run starts with full ammunition,
so shooting an ammo crate in the first minute of a run does nothing at all. It is the same rule that
stops a full inventory silently eating an ore.

### Verified

`ClimpShootNearby [ammo|skill|any] [Shots]` drives it - it repeats the weapon's own object-type trace
rather than calling `FireOnce`, which is private, since what is under test is everything downstream of the
trace. **It finds its own firing position**: the tower is a field of flying rock and from wherever the
player happens to stand there is very often an island in the way, which reported MISSED three runs
running and said nothing about the feature. It tries a ring at 250, then 600, then 1,200 uu and takes the
first with a clear line to the pickup.

With a clear shot: `The trace hit 'VoxelClimpPickup_39' and STOPPED ON the pickup; 1 pickup(s) were
reported collectable; it is gone - collected.` Ammo crates report collectable and stay standing, which is
the full-ammunition refusal above.

## Ammunition is the weapon it feeds (2026-08-20)

One crate design served all four weapons. It said the right thing about the object - it is a box of
rounds - and the wrong thing about the DECISION: a player carrying an empty laser walks past three
identical boxes and finds out which one is theirs by standing on it and reading the prompt. On a spiral
of flying islands, where every pickup is a jump you commit to before you can read anything, that is the
whole cost of the detour spent on information.

An ammunition pickup is now the gun it feeds, built from the shop's own Legend geometry.

**Legend, looked up rather than written down.** Legend is the yellow tier - `GetRarityAccent` returns
`#F5C518` for it - and `GetAmmoSkinId` takes the first Legend row of each weapon's own catalogue rather
than hardcoding four ids. A renamed skin would otherwise silently produce a pickup with no geometry, and
the catalogue is a constant table so every machine picks the same one. It must: **the design's id keys
the shared mesh asset**, so a disagreement is two players seeing different guns, and a random id would be
one mesh build per pickup instead of one per weapon.

**Trimmed to what it occupies**, and that is not tidiness. A pickup centres its visual on the actor using
GRID SIZE, so an authored grid with twenty empty cells down one side hangs the model that far off its own
interaction point - the prompt, the shot volume and the sign would all be centred on air. The trim is also
what makes the size mean anything: `AmmoWorldLength` is 252 uu along the longest OCCUPIED axis, twice the
126 uu crate, and measuring the authored grid instead would make every gun arrive smaller than asked for
by a different amount each. The rifle comes out 32x5x15 cells at 7.9 uu - 252 x 39 x 118 uu - and the
grenade launcher, at 2,935 voxels against the rifle's 951, comes out the same length.

Two things the conversion has to do that the source does not: **sort the cells ascending**, because that
ordering is part of the definition format and a weapon design arrives in authoring order; and **drop
duplicates**, because two authored passes can overlap on a cell and a definition with the same index
twice renders one cube fighting another.

**The sign names the weapon** - `LASER AMMO`, not `AMMO`. The short name, not the enum's: "Grenade
Launcher Ammo" is three words on a sign read while falling past it. The shape says which gun it is to
anyone who has learned four silhouettes; the word says it to everyone else.

The rainbow needed no work - it is applied to the pickup's chunks whatever they are - and the shot volume
resized itself, since it is built from the design's half-diagonal.

### Twice the ammunition, the same number of skill orbs

"Double the rate" applied to the pickup chance would have doubled the skill orbs with it, and those are
the rarest thing on the route on purpose - they restore the one resource a run cannot get any other way.
So the rate went up by 1.67x (0.26 to 0.43 per route island, 0.045 to 0.075 per satellite) and the skill
share came down from a third to a fifth: ammunition doubles, orbs stay at the density they had. The share
is a named field now (`SkillPickupShare`) rather than a literal in the roll, because it and the two
chances have to move together or one of them silently undoes the other.

Measured: **976 planned pickups** against 596 before, 399 of them off the route.

### Every debug camera now finds its own line of sight

`FindClearViewpoint` is shared by `ClimpGotoOre`, `ClimpGotoPickup` and `ClimpShootNearby`. The tower is a
field of flying rock and everything worth looking at sits on top of one, so a camera placed by stepping
back along "the way the player came" has an island in front of it more often than not. That was
rediscovered three separate times - twice as a photograph of a pillar, once as a shot probe reporting
MISSED with the trace stopping on a rock two metres out - and each time it read as the thing under test
being broken rather than as the camera being behind cover. It rings the target at the asked distance,
then nearer, then further, and takes the first viewpoint with an unobstructed trace.

## The lighthouses, and a prompt that was printing a number it never used (2026-08-20)

### "+30" had never been added to anything

The take prompt read `Laser Ammo  +30`. That thirty is the pickup's `Amount`, and `AddAmmo` caps at the
weapon's CAPACITY - which is 10 for a rifle magazine, 3 for a grenade launcher, 5 for a physics launcher
and ten seconds of fuel for the laser. **Thirty rounds have never once been added to anything**; every
ammunition pickup in the game fully refills the weapon it feeds, and the prompt was printing the only
number in the transaction that was guaranteed not to happen.

The prompt is the name now. A figure read in the half second before a key is pressed that does not
survive arithmetic is worse than no figure at all, and the HUD's own ammo tile shows the real result a
moment later.

### The fit came down; the buildings did not

`FloorFitScale` (0.75) and `SummitFitScale` (0.50) scale the TRIGGER and the SHELL, not the lighthouse.
What shrinks is how much room around a building counts as being there - the difference between a
checkpoint you walk into and one you trip over from the next island. The two differ because the buildings
do: the summit is stamped at 3x, so the same fraction of it is three times as much world, and its shell
reached far enough out from the roof to swallow the run-in a player arrives along.

**The floor trigger's HEIGHT is deliberately not scaled with the rest.** It has to span base to roof or a
checkpoint stops being claimable by walking in at ground level, which is the one thing about these that
had to be fixed once already. Only the footprint narrows.

### The shells wear their floor's colour, and getting there meant giving up a rule

Ten identical blue shells on a tower whose entire colour language is "a floor has a hue". The most
visible object on each floor was the one object opting out of it.

Setting `Color` alone changed nothing - the shell stayed the cyan of its own hex texture on every floor.
That is this codebase's own recorded trap: **a material parameter that appears dead may be DROWNED.** At a
dense grid and a high line boost the texture's emissive swamps the tint entirely, so the tint is not so
much a property of the shell as of whatever is left after the lines.

Which means a themed shell cannot be had while the grid stays asset-owned. `SamplingScale`,
`Noise Tiling`, `Grid Amount`, `Line boost` and `Highlight` are now the stated exception to
`bBarrierLookFromAsset` - they are not "look" separate from "colour", they are the aperture the colour
comes through. Everything that decides the shell's SIZE and opacity (`Radius`, `Fade Distance`, `Alpha`,
the fresnel) still comes from `M_Barrier_Tower`, so retuning it in the editor still works.

The tint is taken from `GetStageThemes()` - the same table the rock is tinted from, so the shell and the
island it stands on cannot drift apart - and lifted towards white, because a colour that reads on opaque
rock reads as nothing through an 18% shell and the deep ones (Nightfall, Galaxy) would arrive invisible.
A claimed floor brightens rather than turning green: the colour has to keep saying WHICH floor after you
have taken it, not just that you took it.

### Nothing stands through a lighthouse any more

The stamps go down AFTER the route, onto an island that was dressed as ordinary rock - so a floor's
checkpoint routinely arrived with a boulder through its wall and a tree out of its roof. **354 placements
per run** were doing it, about thirty-five per building.

`Build` culls its own placements out of a column around each lighthouse, and two things about how:

- **From the GROUND UP, not from the building's bounds.** The rock the lighthouse stands on has to
  survive or the building is left hanging over the gap where its island was.
- **Rebuilt in order rather than removed in place.** A placement's INDEX is its identity to everything
  downstream - the streaming table, the pinned set, the run save's record of which deposits are mined -
  so a swap-remove would quietly reshuffle all of it.

The volumes are published as `FBuild::LighthouseClearances` because `Build` is not the only thing that
puts objects on a route island: the ore deposits are appended after it and the handed pickups are planned
by the game mode later still, both from the same `RouteIslandTops` a checkpoint stands on. Both now skip
them, and both test AFTER their random rolls so the stream draws the same values either way - skipping
earlier would reroll every object further up the tower.

### The last one is called LIGHT HOUSE

It read `SUMMIT - GALAXY`, which names the floor it happens to stand on. The summit is not a floor, it is
the thing the whole climb has been pointing at, and the building is the destination.

## Achievements, and the summit loses its sign (2026-08-20)

### The summit carries no label

Every other lighthouse needs one: nine identical buildings a coil apart are otherwise
indistinguishable, and the word over the roof is the only thing that says which floor you have reached.
The summit has the opposite problem. It is three times the size of the rest, it stands alone on the
tower's own axis, and it is the building a player has been looking at since the beach. It does not need
a label to be identified, and a plate over it puts text across the end of the climb.

`GetFloorLabel` still returns `LIGHT HOUSE` - the arrival announcement uses it, and "Congratulations,
you have reached LIGHT HOUSE" is the line that ends a run. Only the world sign is hidden.

### Eleven achievements, and one of them exposed a mode that had never counted anything

One per lighthouse, plus a hundred-fall tier.

| API name | Name | Gold |
| --- | --- | --- |
| `NC_CLIMP_FLOOR_1` … `NC_CLIMP_FLOOR_9` | Off the Waterline … Nightfall | 15 → 75 |
| `NC_CLIMP_SUMMIT` | Top of the Tower | 150 |
| `NC_FALL_RECOVERIES_100` | Long Way Down | 60 |

**NINE floors, not ten.** There are ten stages and the tenth IS the summit, so a tenth floor
achievement and the completion achievement would fire on the same overlap.

**A flat list, not a stat with tiers**, and the distinction is the whole point on this mode: "reach
floor 6" is not "reach a floor six times". A milestone counts events and cannot tell a player who has
climbed to the sixth floor once from one who has claimed the first floor six times across six runs.

The ids are DERIVED in `AVoxelClimpGameMode::GetStageAchievementId` rather than listed, so they cannot
drift from the stage count, and `Climp.ModeDefaults` walks all ten stages asserting each resolves to a
name the definition list actually contains. That check is the only thing in the build that would ever
notice: `UnlockAchievement` silently refuses a name it does not know, and a Steam id that was never
registered fails silently on the live build - the checkpoint still works, the sign still turns green,
and nothing at all says the unlock went nowhere.

Unlocked from `ReachCheckpoint`, downstream of `AVoxelClimpGameState::ReachCheckpoint` returning true -
which happens exactly once per stage - rather than from the overlap, which fires on every machine and
fires again on a checkpoint already claimed. Sent as `Client_UnlockAchievement` because Steam lives on
the client and the achievement belongs to whoever reached the building.

**CLIMP HAD NEVER COUNTED A SINGLE FALL.** Prop Hunt has fed `NC_STAT_FALL_RECOVERIES` since the stat
existed; Climp recovers fallen players through its own `RecoverDroppedPlayer` and reported none of them -
on the one mode in the game that is about falling. Hooked up now, into the SAME stat rather than a
Climp-specific one, because falling out of a map is falling out of a map. That also means the existing
`NC_FALL_RECOVERIES_10` starts working on this mode for the first time.

`Progression.AchievementCatalog` pins the catalogue size and had to be edited from 40 to 51. That pin is
deliberate and worth keeping: every one of these is a string agreed with Steamworks, where a person has
to create the row before it can ever unlock, and a test that read the list's own length would agree with
itself forever.

## Known defects

- `Cubiciousflage.Voxel.BaseMapRegistry` fails, and predates all of this work: `newLevel_Map_1_City`
  was resaved on 2026-08-18 at 07:06 and came back with 5697 placements instead of the expected 5698
  and its display name changed to `Map_1_City`. One placement genuinely vanished. Decide between
  re-baking the asset and updating the expectation — do not silently do the latter.
- **Fire Magma reads as scorched metal** - it borrows the junkyard and factory sets because no magma
  set exists, and `VoxelPaletteSpectrum` still has no lava or ember tones. The other half of that gap
  is closed: the object script has a `glow` keyword now and emissive works end to end
  (`CellEmissive` → `VoxelLevelSubsystem` → `Asset->VoxelEmissive`), which is what the energy ore and
  the metal ore's veins are built on.
- **The eleven new achievement ids must exist in Steamworks before they can unlock.** The build asserts
  they exist in the local definition list; nothing here can check the other half. Until the rows are
  created, `UnlockAchievement` fails silently and a claimed lighthouse looks exactly the same.
- **A run save from before a generator change points at the wrong objects.** `UVoxelClimpRunSave`
  records mined deposits as PLACEMENT INDICES, and the lighthouse cull changes how many placements a
  seed produces - so a save written before it, replayed against the same seed after it, marks deposits
  that are now different objects. Harmless (a few wrong ores missing) and self-correcting on a new
  seed, but worth knowing before trusting an old save to reproduce anything.
- **A large metal deposit is wider than a route island** - 836 uu against 650 - so one can sit across
  the step it is standing on. Deliberate (see `GetDepositScale`), unplayed, and the first thing to
  suspect if the route starts feeling blocked.
- **`Video memory has been exhausted` appears in long teleporting sessions.** Seen in two screenshot
  runs that crossed the whole tower in ninety seconds, which streams far more of it than climbing does;
  the rainbow overlay also adds a material pass per marked chunk over a few hundred deposits. Not
  measured under normal play, and not chased.
- **Pickup persistence is written on the server only.** `UVoxelClimpRunSave` is a local file and the
  game mode that writes it exists on the server, so in a listen-server session a client's own copy of
  the run's progress is not saved. Correct for single player, which is what Climp is played as today;
  a dedicated-server run would want the record to live with the run rather than with the machine.
- **A deposit is spent on the FIRST damage, not when it is empty.** That is deliberate — no voxel
  accounting, no per-deposit state, and "mine as much as you like while you are there" is the rule — but
  it does mean a player who chips one deposit and leaves has spent it. Whether that reads as fair has
  not been played.
- **The summit lighthouse's roof has never been climbed to on foot.** `ClimpSummit` teleports onto it and
  the win fires correctly from there, but whether a 3x lobby building can be climbed at all - or needs the
  paint tools, or a ramp of its own - is unknown. Everything up to its base is measured; the last seventy
  metres are not.
- **The tower has never been played, and that is the largest thing on this list.** Route metrics say it
  is spannable — 1,356 route islands, worst step 1,756 uu, worst rise 1,103 uu, never descending — but a
  1,103 uu rise means roughly 850 uu of placed cubes, and whether bridging that is satisfying is a
  question about playing rather than measuring. Everything else here is a detail next to it.

## Verification that works

Build (fails while the editor is open — Live Coding holds the lock):

```
"D:/Program Files/UE_5.8/Engine/Build/BatchFiles/Build.bat" CubiciousflageEditor Win64 Development -Project="D:/UE_Games/CubeCubeCube/Cubiciousflage.uproject" -WaitMutex
```

Headless map run, then read `Saved/Logs/Tower.log` for `Climp tower`, `Prepared streamed level`,
`initially active` and `gate released`:

```
UnrealEditor-Cmd.exe <uproject> /Game/Cube/Map/Map_Climp -game -unattended -nosplash -nullrhi -benchmark -fps=30 -benchmarkseconds=90 -log=Tower.log
```

Run that from PowerShell, not Git Bash: MSYS rewrites `/Game/Cube/Map/Map_Climp` into
`C:/Program Files/Git/Game/...`, and the editor then quietly loads the default map instead — the run
looks like it worked and reports nothing about the tower.

Streaming walk, which is the only headless way to exercise the streaming paths at all:

```
UnrealEditor-Cmd.exe <uproject> /Game/Cube/Map/Map_Climp -game -unattended -nosplash -nullrhi -benchmark -fps=30 -benchmarkseconds=150 -ExecCmds="ClimpStreamWalk 90 1200"
```

Then read the block after `ClimpStreamWalk finished` in `Saved/Logs/Cubiciousflage.log`. Note that
`-benchmark -fps=30` runs a FIXED TIME STEP: "90 s" of walk is 90 s of game time and about 146 s of wall
clock. The per-item millisecond figures are real time and are the ones that mean anything.

Tests: `Automation RunTests Cubiciousflage` — `Cubiciousflage.Climp.TowerShape` covers the spiral
angles, stage themes, ascending checkpoints, determinism, route reachability, refuses a mountain-sized
body as an island, and covers the starting island: taken whole, the standing surface arriving at Z=0
measured from the rock rather than the marker, and both size refusals.

**`-benchmark` disables audio.** Anything about music must be run without it.

The win, end to end, headless - this is what confirms the summit lighthouse's roof completes the run,
and it is one line in the log:

```
UnrealEditor-Cmd.exe <uproject> /Game/Cube/Map/Map_Climp -game -unattended -nosplash -nullrhi -benchmark -fps=30 -benchmarkseconds=60 -ExecCmds="ClimpSummit"
```

Expect `reached Climp checkpoint 10/10 and completed the run`. Under the old 1..5 clamp this was
impossible: checkpoint ten reported itself as stage five and was refused.

Console: `ClimpDrawRoute [seconds]` draws the spiral in-world coloured per stage;
`VoxelHostClimb`, `VoxelClimbQuickMatch`, `VoxelHost`, `VoxelFind`, `VoxelJoin`.

The reward commands, all of which exist because a log cannot answer the question they answer:

```
ClimpGotoOre [metal|energy] [Skip]                stand in front of a deposit and look at it
ClimpGotoPickup [ammo|skill|any] [Skip] [Dist]    stand in front of a live pickup, for its sign
ClimpTakeNearby [Radius=4000]                     take everything near you, as pressing F would
ClimpMineNearby [Radius=6000]                     break every loaded deposit near you, as shooting would
ClimpShootNearby [ammo|skill|any] [Shots=1]       shoot the nearest pickup, from a spot that can see it
```

All three of the first ones place the camera with `FindClearViewpoint`, so they will move you somewhere
other than where you asked when the direct line is blocked - and say so in the log. `Skip` walks past the
nearest, which is the same object every time from a given spot.

`ClimpGotoPickup`'s distance matters: the F prompt only appears inside `InteractionRadius`, so pass
about 150 to photograph the prompt and leave it at its default to photograph the sign.

`ClimpGotoOre` and `ClimpGotoPickup` aim well to ONE SIDE of the target on purpose: the camera is a
third-person boom, so whatever the control rotation points at is directly behind the character. Three
screenshots of the back of a character's head were spent learning that.

Pinning a seed, which anything spanning two sessions needs:

```
-ini:Game:[/Script/CubeCubeCube.VoxelClimpGameMode]:RunSeedOverride=424242
```

## Lesson worth keeping

The generator's tests asserted island count, spacing and rise — all correct — while the tower rendered
as one fused mountain, because nothing asserted that an island is *smaller than the gap between
islands*. A screenshot found it in seconds. When generating geometry, assert the relationships between
quantities, not just each quantity, and look at the thing.
