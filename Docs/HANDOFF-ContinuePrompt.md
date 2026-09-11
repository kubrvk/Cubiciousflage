I'm building Cubiciousflage, a UE 5.8 C++ voxel prop-hunt/climbing game at `D:\UE_Games\CubeCubeCube`.

Read `Docs/HANDOFF-ClimpTower.md` first — it is the authority on the generated Climp climb and has been
kept current: what the tower is, why each decision was made, the verification commands, and the known
defects. Also read `CLAUDE.md` and `Docs/DESIGN-Climp.md` for conventions.

## Where things are

`Map_Climp` is procedural. `VoxelClimpTower::BuildAndApply` generates a ten-turn conical spiral from the
run seed; each floor draws its geometry from a matching base-map library and is tinted to one of six
colour bands. The ground floor is the converted ZhangJiajie map. Distant objects are stood in for by
instanced two-LOD coarse silhouettes.

Recent sessions closed out, all built and verified:

- **Streaming.** The mesh build moved to worker threads (`VoxelDefinitionMeshBuilder`) — 83% of it left
  the game thread. Mean streaming pulse went 38.4 ms → ~4 ms over a 90 s walk; synchronous cold spawns
  went to zero. The startup freeze in `BuildFrom` went 1,535 ms → 14 ms.
- **Spawn/fall loop.** On a procedural map the level arrives after `ChoosePlayerStart`, so the pawn was
  placed at the world origin 34,775 uu from the Lobby marker, and the recovery point was recorded there
  too — an inescapable fall loop. Fixed three ways: `AVoxelLevelDirector::OnLevelReady`,
  `PlacePlayersAtLobbyMarker` (which also rewrites the recovery transform), and holding gravity while
  the startup gate is shut.
- **Flat squares.** The island size band said nothing about SHAPE, so modular floor tiles were being used
  as flying islands. `MeasureShape`/`IsSlab` reject them; 0 of ~510 body-band definitions are slabs now.
  Small rocks are scaled up to fill the gap; satellites go up to 5x.
- **Pickups.** Two systems: ore deposits are level geometry, mined by shooting through the existing
  harvest path; ammo and skill orbs are actors taken with F, with a world-space sign and icon.

## Build both targets — the game target is the one that's easy to forget

```
"D:/Program Files/UE_5.8/Engine/Build/BatchFiles/Build.bat" CubiciousflageEditor Win64 Development -Project="D:/UE_Games/CubeCubeCube/Cubiciousflage.uproject" -WaitMutex
```

Then the same command with `Cubiciousflage` instead of `CubiciousflageEditor`.

Builds fail while the editor is open (Live Coding holds the lock) — ask me to close it rather than
working around it. If no Unreal process is running and it still says Live Coding is active, that is a
stale lock: just retry the build once before asking me.

## Traps that have each cost real time

* Run headless commands from PowerShell, not Git Bash. MSYS rewrites `/Game/Cube/Map/Map_Climp` into
  `C:/Program Files/Git/Game/...`, the editor silently loads the default map, and the run looks like it
  worked.
* The first run after any build shows a huge startup time — cold shader compilation, not a regression.
* **Measure the ACTORS, not the plan.** Two rounds of "it is fixed now" were spent on pickups whose
  planned positions were spread over 60 km while every spawned actor sat in one heap. A plan being right
  and the world being wrong are entirely compatible.
* **`AVoxelActor`'s root IS its `VoxelComponent`.** `VoxelComponent->SetRelativeLocation(...)` therefore
  moves the ACTOR. That is what piled every pickup at the world origin.
* **Do not re-derive the tower.** It is deterministic from the seed only for identical INPUTS, and
  `BuildAndApply` passes resolvers a bare `Build()` call does not. Read `AVoxelClimpGameState::
  GetRouteIslandTops()` / `GetStageCheckpoints()`, which keep the build that actually ran.

## Verification that works

Tests: `Automation RunTests Cubiciousflage`. Expect **16 pass, 1 fail** —
`Cubiciousflage.Voxel.BaseMapRegistry` is a known pre-existing defect (see the handoff), not a
regression. Anything else failing is yours.

Headless streaming walk, which is the only way to exercise the streaming paths without a player:

```
UnrealEditor-Cmd.exe <uproject> /Game/Cube/Map/Map_Climp -game -nosplash -unattended -benchmark -fps=30 -benchmarkseconds=200 -ExecCmds="ClimpStreamWalk 90 1200"
```

Then read the block after `ClimpStreamWalk finished` in `Saved/Logs/Cubiciousflage.log`. Note
`-benchmark -fps=30` is a FIXED TIME STEP, so "90 s" of walk is about 146 s of wall clock; the per-item
millisecond figures are real time and are the ones that mean anything.

Console commands added for this work: `ClimpStreamWalk` / `ClimpStreamWalkStop`, `VoxelStreamStats` /
`VoxelStreamStatsReset`, `ClimpShapeReport`, `VoxelFly`, `ClimpGoto`, `ClimpDrawRoute`,
`ClimpPreviewTower`.

## What I'd like to work on next

1. **Whatever is still visually wrong with the pickups.** I have been checking these by eye each round.
   Outstanding from my side: a widget that appeared when hovering over metal ore deposits, which could
   not be reproduced from the code (deposits are plain level geometry with no widget or interaction) —
   the best guess was an F prompt from a nearby pickup. If I report it again, chase it properly rather
   than guessing.

2. **The remaining streaming cost, which is now a content problem rather than a scheduling one.**
   `BuildFromMeshDescriptions` cannot leave the game thread and is ~8-13 s per 90 s walk. Two levers,
   both measured and both in the handoff:
   - **42% of definitions are duplicate geometry.** `TintDefinition` and `DarkenDefinition` return a new
     definition with the same cells, differing only in palette slot — and the plugin bakes slot into
     vertex colour, so every recolour is a full mesh build of geometry already built. Tinting through a
     per-stage palette texture instead would collapse ~1,400 definitions to ~810. Note the code carries
     scars from prior palette bugs; read `FVoxelCubeMaterial` before touching it.
   - **Triangle counts are very high.** `SM_tree_01` is 102k voxels and 235k triangles — roughly two
     triangles per exposed face with no merging across faces of the same slot. A greedy merge in the
     plugin's chunk mesher, or a coarser voxel size for large definitions, is the bigger lever.

3. **Distant proxies now cost ~5 s of the startup gate** (259 definitions x 2 LODs = 518
   `BuildFromMeshDescriptions` calls, synchronously, in `BuildDistantProxies`). It sits behind the
   loading card so it is not a spike, but the same worker-thread split `VoxelDefinitionMeshBuilder` uses
   would apply directly if that loading time is worth attacking.

4. **The tower has still never been played end to end.** Route metrics say it is spannable, but whether
   the climb is satisfying is unknown.
