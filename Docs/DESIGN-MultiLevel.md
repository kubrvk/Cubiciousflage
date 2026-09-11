# Multi-Level System — Design

**Status: phases 1–6 are built. Phase 7 (level transfer to clients) is not.**

What survives here is the *reasoning and the measurements* — why a level is objects-and-placements
rather than one grid, and the numbers that decided it. What was built from it is recorded in
[`PROJECT-VoxelPropHunt.md`](PROJECT-VoxelPropHunt.md), which is the record of what exists.

Where the built code differs from what is written below, the code is right and this is history:
`FVoxelLevelObject` became **`FVoxelLevelObjectDef`** (UHT strips prefixes, so the struct collided with
the `AVoxelLevelObject` actor), voxel size settled at **10** to match the character rather than 20, and
spawn markers are placed objects with authored yaw rather than reserved palette slots.

Decisions taken as given (chosen by the project owner):

| Question | Answer |
| --- | --- |
| Level representation | **A library of authored objects, placed as instances** |
| Level editor | **Two features: create an object, add an object to the scene** |
| Object size | **Chosen by the player when the object is created** |
| Client gets the host's custom level | **Chunked transfer on join** |

> **This document was rewritten once.** The first version proposed a custom level as one large voxel
> grid, with spawn points as reserved palette slots. §2 is why that was abandoned. Where the old
> decisions still show through, they are marked.

---

## 1. Shape of the system

Nine playable maps, of which eight are ordinary Unreal levels and one is a container.

**The eight base levels** are built the normal way: real geometry, a `VoxelPropGameMode` override in
World Settings, hand-placed `AVoxelMatchSpawnPoint` actors. They need no new code — they are already
what `VoxelPropGameMode` and `UVoxelMapLibrary` were written for. The only work is authoring them and
filling in a `UVoxelMapLibrary` asset.

**The custom level** is one map, `Voxel_Custom.umap`, empty but for lighting and a single
`AVoxelLevelDirector`. What fills it is a save file: a **library of voxel objects the player authored**
and a **list of where those objects are placed**. The same map serves every custom level anyone builds,
under two different game modes depending on why it was opened.

A player-authored level cannot be a `.umap` — a packaged game has no map cooker and the runtime cannot
write one. A data payload applied to a fixed container is the only route that survives packaging, which
is the same constraint that forced the runtime-edit design in the first place.

---

## 2. Why the level is not one big voxel grid

The first version of this design made a custom level a single `UVoxelSculptComponent` at 128×128×48.
It was measured before anything was built. It does not work, and the reason is worth recording because
it is a property of the plugin, not of the grid size.

### The measurement

`VoxelLevelStress.cpp`, a hollow arena at each grid, timing a forced-synchronous rebuild. The
character's own sculpt is `VoxelPresetGridSize` = 32×32×32, and it ships and feels fine, so it is the
calibration point.

| Grid | Cells | Occupied | Rebuild per **one** edit | Static meshes |
| --- | --- | --- | --- | --- |
| 16×16×24 | 6,144 | 1,636 | 31 ms | 1 |
| **32×32×32** *(character — known good)* | 32,768 | 5,084 | **101 ms** | 1 |
| 48×48×32 | 73,728 | 8,996 | 169 ms | 1 |
| 64×64×32 | 131,072 | 13,852 | 260 ms | 1 |
| 64×64×32 **solid** | 131,072 | **131,072** | **196 ms** | 1 |
| 96×96×48 | 442,368 | 32,476 | 602 ms | 4 |
| 128×128×48 *(first proposal)* | 786,432 | 50,844 | **872 ms** | 4 |

Development editor binary, headless, synchronous path — the absolute numbers are pessimistic, the
relative scaling is the trustworthy part.

Two things fall straight out. **Cost tracks grid cell count, not voxel count** — the solid 64³ with
131,072 voxels rebuilt *faster* than the hollow one with 13,852. And **the static-mesh-component
explosion never happened**: four components at 128×128×48, not hundreds. That worry is dead.

### The actual cause: Invalidate throws away the shared build

`FVoxelRuntimeTemplateCache::GetOrCreate` is keyed by `{Asset, VoxelSize}`. In a game world
`UVoxelComponent` calls it with `bRequireSharedVisual = true` and
`bRequireSharedCollision = bGenerateCollision`, and it produces `FVoxelRuntimeSharedCluster`s —
`TStrongObjectPtr<UStaticMesh>` **visual and collision meshes built once per asset and referenced by
every instance**. The plugin says so itself:

> *Shared visual/collision cluster generated once per voxel asset from sparse occupied cells.
> Instances reference these meshes while pristine and replace only edited clusters locally.*
> — `Runtime/VoxelRuntimeTemplate.h`

So a hundred placed copies of one object cost **one** template build and a hundred pointer
attachments. That is why `Voxel_Level` loads instantly with large voxel objects all over it, and it is
a runtime path — not an editor bake. The disk asset is still dense arrays; the conversion happens on
first use and is then shared.

The sculpt destroys exactly this. `UVoxelSculptComponent::FlushPendingRebuild` calls
`FVoxelRuntimeTemplateCache::Invalidate` on **every edit**, which drops the whole template — sparse
data, every chunk mesh, every shared cluster, and the collision cook — so the next refresh rebuilds all
786,432 cells from nothing. That one line is load-bearing (without it edits are silently invisible) and
it is also the entire cost.

The plugin even carries copy-on-write machinery for replacing single edited clusters
(`Runtime/VoxelRuntimeEditOperations.h`) that the sculpt bypasses.

### The conclusion

Editing a big grid fights the template cache. Placing many small objects rides it. So the level is a
**library of small authored objects plus a placement list** — which is both the faster architecture and
the one the plugin was built for.

---

## 3. The model

Two kinds of thing, and the distinction is the whole design.

**An object definition** is authored voxel geometry: a crate, a tree, a wall segment. It owns exactly
one transient `UVoxelMeshAsset`. Editing one is a sculpt operation and costs what its own grid costs —
~100 ms at 32³, the character's price.

**A placement** is an instance: an object id plus a transform. Spawning one is an `AVoxelLevelObject`
(a thin `AVoxelActor`) pointed at that object's shared asset. It builds no geometry of its own — it
attaches the shared cluster meshes. Placing the two-hundredth crate rebuilds nothing.

| Piece | Cost |
| --- | --- |
| Author one edit on a 32³ object | ~100 ms (same as a character edit today) |
| Place an instance | ~free — shares the template's meshes and collision |
| Load a level of N objects, M placements | N template builds, M attachments |

### The rule that keeps it fast

**Objects are edited on a workbench copy, never on the shared asset.** Placed instances hold the asset
pointer, so editing it in place would `Invalidate` the template and rebuild every instance on every
click — the abandoned design wearing a new hat. Authoring happens on `AVoxelLevelWorkbench`, and the
result is committed into the shared asset once, on leaving object-edit mode.

---

## 4. New files

| File | Role |
| --- | --- |
| `VoxelLevelTypes.h` | `FVoxelLevelObjectDef`, `FVoxelLevelPlacement`, `FVoxelLevelMarker`, `FVoxelLevelData`, `UVoxelLevelSave`, `UVoxelLevelIndex`, `UVoxelLevelPrefs` |
| `VoxelLevelSubsystem.h/.cpp` | Save/load/index/delete, selected level, lobby-override preference, and the transient `UVoxelMeshAsset` per object definition |
| `VoxelLevelDirector.h/.cpp` | The one actor in `Voxel_Custom`. Builds the object assets, spawns placements and markers |
| `VoxelLevelObject.h/.cpp` | `AVoxelLevelObject` — one placed instance. A thin `AVoxelActor` |
| `VoxelLevelWorkbench.h/.cpp` | Where a single object is authored, in isolation from its instances |
| `SVoxelLevelPlacementPanel.h/.cpp` | Placement UI: object palette, rotate, snap, delete, save level |
| `SVoxelLevelBrowser.h/.cpp` | **My Levels** menu page: list, New, Edit, Rename, Delete, Set as lobby, Host |

### Files that change

| File | Change |
| --- | --- |
| `VoxelRuntimeEditorComponent` | Edit-target switch (body ↔ workbench), save-domain switch |
| `SVoxelRuntimeEditorPanel` | A **Target** line in the header. Otherwise untouched |
| `SVoxelMainMenu` | Ninth "Custom Level" tile on the host page; **My Levels** page |
| `VoxelSessionSubsystem` | Travel to `Voxel_Custom` carrying the level identity |
| `VoxelPropCharacter` | Build-mode flight, gated to level editing |
| `VoxelWorldDamage` | Honour a placed object's destructible flag |
| `VoxelMapLibrary` | The eight base entries |

---

## 5. The save format

```cpp
USTRUCT()
struct FVoxelLevelObjectDef          // one authored definition
{
    FGuid         ObjectId;
    FString       DisplayName;
    FIntVector    GridSize  = FIntVector(32, 32, 32);
    float         VoxelSize = 20.0f;
    int32         PalettePage = 0;
    TArray<int32> CellIndices;    // ascending, so deltas stay possible
    TArray<uint8> CellSlots;      // parallel
    bool          bDestructible = true;
};

USTRUCT()
struct FVoxelLevelPlacement       // one instance of one definition
{
    FGuid   ObjectId;
    FVector Location;
    int32   YawSteps = 0;         // 0-3, ninety degrees each
};

USTRUCT()
struct FVoxelLevelMarker          // a spawn point. Carries no geometry
{
    EVoxelSpawnRole Role = EVoxelSpawnRole::Lobby;
    FVector Location;
    int32   YawSteps = 0;
};

USTRUCT()
struct FVoxelLevelData
{
    static constexpr int32 CurrentFormatVersion = 1;

    int32     FormatVersion = CurrentFormatVersion;
    FGuid     LevelGuid;
    uint32    ContentHash = 0;    // the transfer cache key
    FString   DisplayName;
    FString   AuthorName;
    FDateTime SavedAtUtc;

    TArray<FVoxelLevelObjectDef>    Objects;
    TArray<FVoxelLevelPlacement> Placements;
    TArray<FVoxelLevelMarker>    Markers;
};
```

Deliberately not a reuse of `FVoxelRuntimeDesignData`. That struct is a replicated property on the
character, size-capped at 8000 voxels, and a level must never accidentally reach that path.

**Rotation is in ninety-degree steps, not a quaternion.** Voxel geometry is axis-aligned; an arbitrary
rotation puts cells off-grid and makes the placement cursor lie about where a thing will land.

**Markers are their own array, not placements with a role.** A marker has no voxels, so giving it an
`ObjectId` would mean inventing an empty object for it to point at.

Slots: `__VoxelLevel_<slotname>`, indexed by `__VoxelLevelIndex` — same pattern and same reason as
`UVoxelRuntimeDesignIndex`, because `UGameplayStatics` cannot enumerate save slots. A separate
namespace from character designs, so a player can have a character and a level both called "Castle".
Preferences (`SelectedLevelSlot`, `bUseCustomLevelAsLobby`) live in `__VoxelLevelPrefs`.

### Size

A level of 20 objects averaging 3,000 voxels is ~300 KB of definitions, and 400 placements is ~16 KB.
Two hundred crates cost **one** definition and 200 × ~40 bytes — the old single-grid format would have
paid for all two hundred separately. This is the format's biggest win and it is not a small one.

---

## 6. The editor: two features, two panels

Reached from **My Levels → Edit**, which travels to `Voxel_Custom` in standalone under
`VoxelLobbyGameMode` with `?edit=1`. The lobby mode because editing is a home-screen activity: no
teams, no phases, no clock — and the guns work, which is how you find out whether your cover is
actually shootable.

### Creating an object — the existing Cube Builder, retargeted

`UVoxelRuntimeEditorComponent::SetSculpt` is already public and `bAutoFindSculptOnOwner` is only a
fallback. Point it at the workbench's sculpt and **the entire Cube Builder works on a level object**:
every shape tool, mirror, fill, extrude, the colour wheel. No new editing code. This is the largest
reuse in the design and the reason the rest is affordable.

The panel gains one thing: a **Target** line in the header saying whether you are editing *your
character* or *an object*, so a player can still fix their body while standing in their level.

**Size is chosen when the object is created**, as in the plugin's own asset editor. Default 32×32×32 —
the character's grid, and a known-good ~100 ms per edit. Sizes past 64³ carry a warning: 64×64×32 is
already 260 ms per click, and the plugin's 256³ ceiling is far past the point where runtime authoring
stops being usable. The limit is *advisory*, not enforced; the number is shown next to the size picker.

### Adding an object to the scene — a new, small panel

`SVoxelLevelPlacementPanel`, deliberately separate. In placement mode roughly eighty per cent of the
Cube Builder means nothing — Add/Erase/Paint/Pick, eight shape tools, mirror, the wheel, 256 swatches —
and hiding most of a panel behind a mode toggle is worse than a second purpose-built one.

It holds: the object palette (with the marker roles as their own section), the placement cursor, `Q`/`E`
to rotate in ninety-degree steps, delete, grid snap, and the level's own Save / Save As / name field.

### Two things must change or level building is unworkable

- **`MaxEditDistance` is 2000 uu.** Fine for a 32³ object at arm's length; useless for placing an
  object across a room. Raised to 6000 in placement mode.
- **Flight.** A level is built upward and the character has gravity and a ~96 uu jump. Build mode in
  the level editor switches to `MOVE_Flying` with Space/Ctrl, and back on close. Gated to level
  editing — a flying player in a match is a different feature entirely.

---

## 7. Spawn points

**Placed markers, not reserved palette slots.** *(This replaces the earlier reserved-slot decision.)*

The old design had no way to author a rotation, so a marker's facing had to be derived by pointing it
at the grid centre. Placement carries a yaw, so **facing is authored** and that limitation disappears.
Reserved slots also stop meaning anything once there is no single grid to reserve them in, and the
accident risk they carried — painting near the top of the swatch grid silently creating spawn points —
goes with them.

At load, `AVoxelLevelDirector` spawns a real `AVoxelMatchSpawnPoint` per marker, with its role and yaw.
So `AVoxelPropGameMode` and `AVoxelLobbyGameMode` need **no changes at all**: they already collect
points through `AVoxelMatchSpawnPoint::CollectPoints`, and they now find them in a custom level exactly
as they do in a hand-built one. Every existing rule — props scattered at hiding, hunters released at
the hunt, everyone returned to Lobby points on reset — works untouched.

The runtime level editor treats custom-level markers as **one marker per role**. Dropping a second
`Lobby`, `Prop`, or `Hunter` marker is a move/replace operation, but it must go through the in-game
`SVoxelLevelEditorPanel` confirmation popup first. Removing a marker also uses that runtime popup.
Do not use `FMessageDialog` or other Unreal editor UI for this path; the level editor is available in
game/runtime contexts.

**Ordering matters.** The game mode's `ChoosePlayerStart` runs on the first login, so the director must
have spawned its markers before then. It loads in `PostInitializeComponents`, which runs during level
load and before any `PostLogin`.

**Fallbacks.** A role with no markers keeps today's behaviour: those players stay where they are, logged
once. A level with *no markers at all* gets one synthesised Lobby point above the highest placed object,
because the alternative is spawning everyone at world origin, possibly inside geometry.

---

## 8. Destructibility

The old design needed a special rule here, because a whole level being one voxel object created a
gameplay tell: with the level immune and props not, one rifle round at anything answered "is this a
player?" — the rifle does exactly `VoxelDefaultCellHealth`, so a landed shot always removes a cell.

**Objects-and-placements dissolves that problem rather than solving it.** Every placed object is its own
voxel object with its own template, which is precisely what a level prop already is, and
`FVoxelWorldDamage` already routes damage per component. A level object chips exactly like the crate a
prop is disguised as, because it *is* the same kind of thing.

What remains is one flag: **`bDestructible` per object definition, default true.** Set it false on
floors, outer walls, and anything structural, so players cannot shoot the ground out and fall through
their own map. `FVoxelWorldDamage` gains a check for it. The placement panel marks indestructible
objects distinctly, and a floor object defaults to false when created from the floor preset.

Still open, and pre-existing: `NotifyVoxelDamage` measures player bodies only, so destroyed level voxels
are accounted nowhere. Fine for the match rules — level damage is meant to be free — but scoring will
eventually want it.

---

## 9. Networking

The format did most of this work already. A level is now ~300 KB of definitions rather than a
50,000-voxel grid, and identical objects are stored once no matter how often they are placed.

- **Definitions** travel as chunked reliable RPCs, ~4000 voxels per chunk, **paced from a server timer**
  rather than sent in one burst — thirty reliable RPCs queued in a single frame is the oversized-bunch
  trap wearing a different hat.
- **Placements and markers** are tiny and travel in one payload.
- **A client cache keyed by `ContentHash`** (`__VoxelLevelCache_<hash>`) means the transfer header
  arrives first so a client can answer *"I already have this"* and skip every chunk. Joining a friend's
  server a second time downloads nothing.
- **Progress must be visible** — a client sitting in an empty level with no explanation has, from where
  they are standing, joined a broken server.
- **Apply once, at the end.** Applying per chunk would rebuild templates repeatedly, which is the very
  cost §2 exists to avoid.
- **Verify count and hash** at the end, and on mismatch return the player to the lobby with a reason
  rather than dropping them into half a world.

Per-object dedup makes a v1 without delta encoding comfortable. `CellIndices` is specified ascending so
delta encoding stays available if it is ever needed.

---

## 10. Selection, hosting, and the custom lobby

`UVoxelLevelSubsystem` holds `SelectedLevelSlot` and `bUseCustomLevelAsLobby`, persisted to
`__VoxelLevelPrefs`.

### Presets are save slots, not a special case

`FVoxelLevelPresets` builds shipped levels in code, and `EnsurePresetsSeeded` writes them into ordinary
save slots the first time the game runs — on an empty index only, so deleting one means it stays
deleted.

Seeding them as real slots is what lets everything downstream stop knowing presets exist. The browser
lists slots, hosting selects a slot, the transfer sends a slot, and a preset **is** a slot. The
alternative was a parallel "is this a preset?" branch through all three.

Their object ids and level guid are **deterministic** (`MakeDeterministicGuid`), not random: a preset
rebuilt on two machines has to hash identically, or a client re-downloads a level it already has and
the transfer cache never hits for the case it matters most for.

### The front end

The root menu is **Start Game** and **Join Game**. Start Game shows the eight base maps four to a row,
the player's custom levels beneath them, a **Create custom level** button, and the match rules — max
players, hunt minutes, hiding seconds. Exactly one map is selected across the two lists.

Base maps come from `UVoxelMapLibrary::GetAvailableMaps()`: the library asset when the project has one,
and eight built-in entries when it does not. A data asset is then what adds *thumbnails*, not what makes
the picker work at all.

Picking a custom level commits it to `UVoxelLevelSubsystem` and travels to the container map. The level
does not ride the URL — it is orders of magnitude too big for a travel option — so what crosses the map
change is the *selection*, and the director reads it on the far side.

**Match rules must ride the URL.** The game mode's tunables are `EditDefaultsOnly`, baked into the class
default object, and the game mode is constructed fresh on the server after travel. A value set on the
menu's side of the travel reaches nothing. `FVoxelMatchSettings` renders to `?VoxelHuntMinutes=…` and
`AVoxelPropGameMode::InitGame` reads it back — the same reason `?game=` is already stated explicitly.

Option names are prefixed because travel options are one flat namespace shared with the engine's own:
`MaxPlayers` is a plausible collision, `VoxelMaxPlayers` is not.

`Voxel_Custom` must have **no GameMode override in World Settings** — the URL decides, and the URL is
always stated explicitly, exactly as `VoxelSessionSubsystem::MatchGameModeClass` already does and for
the same reason. One map, two modes:

| Opened as | Game mode | What it is |
| --- | --- | --- |
| Lobby / Edit | `VoxelLobbyGameMode` | Home screen, menu on screen, guns work, no rules |
| Match | `VoxelPropGameMode` | A round of prop hunt |

With `bUseCustomLevelAsLobby` set, the lobby controller in `Voxel_Lobby` travels to
`Voxel_Custom?game=…VoxelLobbyGameMode` on entry. **Guard on the current map name** or `Voxel_Custom`
re-triggers the same travel on arrival and the game becomes an infinite load screen.

The project's default map stays `Voxel_Lobby`: it is what runs when preferences are missing, corrupt, or
point at a deleted level. A startup path that depends on a save file being valid is one that can brick.

---

## 11. Unknowns worth naming

### Measured: the sharing claim holds

`VoxelLevelBuildTest`, two definitions (a 32×32×1 floor tile and an 8×8×8 crate) placed many times:

| Placements | Whole-level build | Marginal cost per placement |
| --- | --- | --- |
| 164 | 129 ms | — |
| 564 | 234 ms | 0.26 ms |
| 1,256 | 447 ms | 0.31 ms |

A fixed ~100 ms for the template builds, then **~0.3 ms per placed instance**. The abandoned design
paid 872 ms for a single click; placing an object here costs 0.3 ms. Cost scales with the number of
*definitions*, not the number of placements, which is exactly what §2 predicted.

### Still open

- **Steady-state frame cost.** The table above is *build* time. Whether 1,256 actors with collision are
  affordable to tick and render has not been measured, and a level that loads in 447 ms and then runs
  at 20 fps is no use. Measure before the placement panel is polished.
- **Committing an edited object** invalidates its template and rebuilds every instance of it. Correct,
  and the reason edits are committed on exit rather than per click — but a 200-instance object will
  visibly hitch on commit. Not yet measured.
- **Palette sharing.** Every object takes its palette from the same donor asset, so objects cannot yet
  carry different palette pages. Probably fine; unverified.
- **No level thumbnails.** The base-level tiles have artist screenshots; a custom level would want a
  captured one, which is a scene-capture feature this project does not have.

---

## 12. Build order

Each phase is independently useful and independently testable.

1. ~~**Data and runtime**~~ — `VoxelLevelTypes`, `UVoxelLevelSubsystem`, `AVoxelLevelDirector`,
   `AVoxelLevelObject`. **Done.** Instance scaling measured; see §11.
2. ~~**Object authoring**~~ — the workbench is an `AVoxelRuntimeCanvas`, driven by the editor pawn's own
   Cube Builder rather than the character's. **Done.**
3. ~~**Placement**~~ — `SVoxelLevelEditorPanel`, ghost preview, hover outline, place/rotate/delete/snap,
   level save. **Done.**
4. ~~**Markers**~~ — placed markers become real `AVoxelMatchSpawnPoint`s with runtime visuals and labels,
   plus fallbacks. **Done**, except the `bDestructible` check in `FVoxelWorldDamage` (§8), which is
   carried in the save format but not yet enforced.
5. ~~**Menu**~~ — Start Game / Join Game, level list, selection, lobby preview override. **Done.**
6. ~~**Base levels**~~ — the eight maps exist and are listed in code, with the Custom entry.
   **Done**, except thumbnails, which need a `UVoxelMapLibrary` asset.
7. **Networking** — chunked definition transfer, cache, progress UI. **Not built.**
   *Outcome: friends can play your level.*

Until 7 exists a client builds from its own save, which is only safe while both machines agree — true
in PIE, and true for the presets, whose deterministic GUIDs were chosen for exactly this.
