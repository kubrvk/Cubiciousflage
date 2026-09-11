# Object Catalog — Design

How objects get made, where they live, and why they are one format and not three.

Companion to [`DESIGN-MultiLevel.md`](DESIGN-MultiLevel.md), which explains why a level is a library of
objects plus a list of placements. This document is about the *library* half: the part that was
missing.

---

## 1. The problem: nothing accumulated

A level's object definitions lived inside the `FVoxelLevelData` that used them, and
`AVoxelLevelDirector::SaveWorkingLevel` dropped every definition nothing placed — deliberately, so
unused geometry stayed off disk and off the wire.

The consequence was not deliberate. **An object a player sculpted and did not immediately place was
destroyed by the next save.** There was no library to put it in, so every level began from an empty
palette, and building a room meant sculpting a crate again. That is the real reason level editing felt
like starting over, and it is not fixed by making authoring faster.

A converter for static meshes runs into the same wall from the other side: convert two hundred meshes
and there is nowhere to put them.

So the catalog came first, and everything else feeds it.

---

## 2. One struct, three producers

Everything that makes an object produces `FVoxelLevelObjectDef` — unchanged, the same struct a
player's sculpt commits and the same struct a level embeds:

```cpp
FVoxelLevelObjectDef {
  FGuid ObjectId; FString DisplayName;
  FIntVector GridSize; float VoxelSize;   // 10.0f, always
  int32 PalettePage;
  TArray<int32> CellIndices;  // linear, ASCENDING
  TArray<uint8> CellSlots;    // parallel, never 255
  bool bDestructible;
}
```

| Producer | Where | Path |
| --- | --- | --- |
| Player sculpting | Runtime, level editor | Cube Builder → `CommitObject` → player library |
| Object script | Either | `FVoxelObjectScript::Parse` |
| Static mesh | Editor only | `FVoxelMeshVoxelizer::Convert` |
| MagicaVoxel `.vox` | Editor only | `FVoxelObjectImport::ImportVox` |
| Plugin grid JSON | Editor only | `FVoxelObjectImport::ImportPluginJson` |

There is deliberately **no catalog-specific geometry format**. A second format would need converting
at every boundary — the level save, the shared asset cache, the transfer, the browser — and three
producers with three formats is three subtly different pipelines that drift.

`FVoxelCatalogEntry` wraps the definition with a category, tags and a source, and nothing else. The
wrapper is metadata; the geometry is untouched.

---

## 3. Three tiers, merged

`UVoxelObjectCatalogSubsystem` merges three sources into one list. Load order **is** the precedence
rule — a later tier replaces an earlier entry with the same id:

| Tier | Storage | Editable | Why it exists |
| --- | --- | --- | --- |
| `BuiltIn` | Compiled-in object script | No | A fresh clone must have content with nothing wired up |
| `Library` | `UVoxelObjectLibrary` data assets | In the editor | Where the mesh converter writes; a cooked asset packages itself |
| `Player` | Save slots | Yes | Survives the level it was authored in |

Reversing the order would make built-ins un-overridable, and a player re-sculpting the built-in crate
would find their version silently ignored.

**Built-ins are code-authored** because that is the rule the cue table, the effect table, the map list
and the level presets all follow: a checkout that has never created a data asset would otherwise have
an empty object browser, and an empty browser is indistinguishable from a broken one. A data asset is
the more editable home and `UVoxelObjectLibrary` exists for exactly that — it just cannot be the only
home.

**Player objects use save slots keyed on the object's id**, not its display name, so saving an object
twice overwrites rather than leaving a stale copy behind under the old name. Same index pattern and
same reason as `UVoxelLevelIndex`: `UGameplayStatics` cannot enumerate save slots.

### Levels stay self-contained

A level still embeds every definition it uses, and that does not change. A joining client may not have
the host's catalog, and the chunked transfer sends a payload rather than a bill of materials. **The
catalog is where objects are drawn from, never a dependency a saved level acquires.**

`SaveWorkingLevel` still drops unplaced definitions. That was never the bug — the missing library was.

---

## 4. Object scripts

An object is a short program of shape operations, not a list of coordinates:

```
object "Park Bench"
  grid 24 8 9
  category Furniture
  tags bench seat park
  destructible true

  box 1 1 0   2 6 3    dark_gray   # one leg pair
  mirror x                          # the other
  box 0 0 4  23 7 5    orange       # seat, spanning both
  box 0 6 6  23 7 8    orange       # back
```

Fifteen lines against several hundred coordinates, and a wrong line is visible as a wrong line. That
property is what makes it practical to author objects in bulk and still know what they are.

| Command | Arguments |
| --- | --- |
| `box` / `shell` / `line` | `x0 y0 z0  x1 y1 z1  <colour>` |
| `erase` | `x0 y0 z0  x1 y1 z1` |
| `sphere` | `cx cy cz  r  <colour>` |
| `cyl` | `<x\|y\|z>  cx cy cz  r  length  <colour>` |
| `mirror` | `<x\|y\|z>` |

Headers: `object "Name"`, `grid X Y Z`, `category`, `tags`, `destructible`, `id`.

A colour is a name (`orange`, `dark_gray`), a raw slot `0`–`254`, or `#RRGGBB` quantised to the nearest
slot. Operations apply in order and the last to touch a cell wins.

**`mirror` reflects what has been placed so far and then stops applying.** A mode that stayed on could
not express "mirror the legs, then lay one seat across both" — which is most of what mirroring is for.

**Ids are deterministic**, hashed from category and name. A random `FGuid` would change a level's
`ContentHash` between host and client and the joining player would re-download geometry they already
had — the same failure the shipped level presets use deterministic ids to avoid.

---

## 5. Static mesh conversion

Editor-only, and not by choice: reading a texture's pixels needs `FTextureSource`, which does not
exist in a packaged game.

1. **Geometry** — walk `FMeshDescription` triangles and **sample each triangle's own surface**, one
   sample every 0.4 of a voxel, dropping each into the cell it lands in.

   This started as a triangle/box separating-axis test, which is the clever answer and was wrong: one
   of the two projections on each of the nine edge axes multiplied two vertex components together
   instead of edge-cross-vertex. The first of each pair was correct, so it was never obviously broken
   — it simply accepted and rejected cells on a meaningless number, producing holes, stray blocks and
   colours on cells no triangle touched. That read as a *colour* fault and cost three rounds of
   texture work before the geometry was suspected.

   Surface sampling has no geometry predicate to get wrong. A point lands in a cell or it does not,
   and `floor` decides. The only correctness condition is that the spacing is under one voxel, which
   is one number rather than nine axis tests. It is also strictly better for colour: a sample is on
   the triangle by construction, so its barycentric weights are exact and known *before* the position
   is computed — where the old version had to recover them from a cell centre that was usually
   outside the triangle, and clamping that guess was a patch on a self-inflicted problem.

   Cost scales with surface **area**, not triangle count: dense meshes have sub-voxel triangles that
   take one or two samples each.
2. **Interior** (optional, off by default) — flood from *outside* the grid inward; anything unreached
   is interior. Flooding outward from a centre seed needs a known-interior cell, and an open mesh has
   none, so it fills the whole grid solid. This way an open mesh simply stays hollow.
3. **Colour** — base-colour texture at the barycentric UV, then vertex colours, then the material's
   constant, then a fallback slot.
4. **Quantise** — nearest slot in **Oklab**. RGB distance is dominated by luminance, so a mid brown
   goes to a dark grey rather than the nearest orange, and skin and wood — the two things a converted
   mesh is most likely to be made of — come out as the two worst available choices.

### Resolution is not an option, and this is the important part

A voxel is `VoxelLevelVoxelSize` = 10 uu = 10 cm, and **the player is 16 cells tall**. The grid falls
out of the mesh's bounds:

```
chair (90 cm)  ->   9 cells tall
door  (2 m)    ->  20 cells
car   (4.5 m)  ->  45 cells long
```

One subtlety that cost a full round: the grid is measured from the mesh's **source** triangles, which
do not carry `BuildSettings.BuildScale3D` — that is applied by the mesh build, and only shows up in the
post-build render bounds. `GatherTriangles` folds it in explicitly; `PredictGridSize` deliberately does
not, because the bounds it reads already have it. Getting this wrong scales an object by whatever the
asset was imported at, and `SM_Barrier_02A` at 0.125 converted eight times too large.

A converted chair is nine cells tall and **nearly all surface detail is gone**. What survives is
silhouette, which is also all that reads at prop-hunt distance, so this is the right trade — but it
means the converter earns its place on *large, awkward* geometry (buildings, vehicles, terrain, rock
formations) far more than on small props, where nine cells is about as much work to sculpt by hand.

Offering a resolution slider would let someone import a chair at 64 cells that then stands three times
a player's height, and the mistake would not be visible until it was placed. Scale the source mesh
instead; that is the honest control. `Tools → Preview Voxel Conversion` shows the grid before a batch.

### The palette could not express muted colour — it can now

**This section used to describe a permanent limitation. It is fixed, and the fix was neither a per-page
authored palette nor a better nearest-colour search.**

What it said was true: every spectrum slot was at least 0.8 saturation and 0.85 value, 192 vivid hues and
a 64-step grey ramp with nothing in between, so browns, olives, navies and skin tones quantised to
whichever grey matched their lightness. A converted wooden chair came back orange, and a mid brown came
back concrete.

The mistake was in what the palette was *spending* its slots on. 192 hues is one every 1.875° — a spacing
no eye resolves. The entire chromatic half of the palette went on distinctions nobody can see, leaving
nothing for the ones everybody can. `VoxelPaletteSpectrum` is now **24 hue bands × 8 tones**: still one
hue every 15°, with each hue available vivid, tinted-white, pale, light, mid, dark-mid, dark and very
dark.

Measured against 72 real material colours: mean **0.024** Oklab, worst **0.058**, 69 of 72 within 0.05 and
all of them within 0.058, with **no chromatic material landing on a grey** — against 20 of the first 24
collapsing to grey before. Wood is brown, foliage is dark green, rust is rust. The full reasoning, the
fitted tone table and the two properties preserved for existing content are in
[`PROJECT-VoxelPropHunt.md`](PROJECT-VoxelPropHunt.md) §3.

Two consequences for this document:

- **Conversion colour should now be judged again.** Every note here about converted meshes coming back in
  poster-paint colours was describing the palette, and the palette changed.

- **Already-converted objects are translated, not re-converted.** A stored slot means "the colour at index
  N *in the palette it was quantised against*", and the converter used the project's donor — the demo
  brick building. Replacing that palette with the ramp made 560 objects render dark purple. The fix maps
  each old slot through the original palette to the nearest ramp slot, in memory as the library loads:
  exact to the ramp's resolution, no source mesh needed, and **the converter's `.uasset` is never
  rewritten**, so there is no migration to run and no half-migrated state to land in. The table is
  captured before the ramp is written, which is the only moment the original palette still exists.
- **`IsMutedColor` still exists and still counts what cannot be reproduced**, but it measures the actual
  nearest-slot distance rather than testing saturation. The old rule would now report the browns and
  olives the palette handles *well* as the ones it cannot handle at all.

And the step that makes any of it visible: slot colours come from the **donor's texture**, not from
`GetSlotColor`, so `EnsureGeneratedRampApplied` writes the ramp into the donor once per process.
`VoxelPaletteCheck` reports whether it took. Without that, a converted mesh is still quantised against one
table and rendered through another — the trap this project hit from three separate directions.

---

## 6. Why the plugin's own importers were copied, not called

`FVoxelGridVoxIO` and `FVoxelGridJsonIO` live in `VoxelEditorEditor` and do exactly what is needed.
They are declared **with no `VOXELEDITOREDITOR_API` export macro**, so their symbols are not visible
outside that DLL and referencing them is an unresolved external at link time — not a compile error,
which is a slow way to find out.

Both formats are re-read in `VoxelObjectImport.cpp` instead:

- The `.vox` chunk walk follows the same published spec and **keeps the plugin's Y-flip**
  (`Y = SizeY - 1 - Y`), so a model imported here lands the same way round as one opened in the Voxel
  Asset Editor. Getting that wrong produces objects mirrored relative to everything else in the
  project — subtle enough to survive review, obvious the moment two are placed side by side.
- The plugin's JSON is read straight into a definition rather than through a `UVoxelMeshAsset` and back
  out, which would be three copies of the data to read a file whose format is four fields.

This also preserves the project's standing rule: nothing outside the plugin depends on anything but its
exported **runtime** API, so a plugin update cannot break this.

---

## 7. Editor entry points

Under **Tools → Voxel Objects**:

| Entry | Does |
| --- | --- |
| Preview Voxel Conversion | Grid size the selected static meshes would produce. Writes nothing |
| Convert Selected Meshes to Voxel Objects | Batch converts into `/Game/Cube/VoxelObjects/ConvertedObjects` |
| Import Voxel Objects | `.vox`, plugin JSON, or object scripts into the same library |

Writing **adds to** an existing library rather than replacing it. Creating a fresh object over the top
would silently discard every previously converted mesh, and the loss would not be noticed until
someone went looking for one of them.

---

## 8. In game

The level editor panel gained a **Library** section under the level's own object list: a search box, a
category filter row, and the merged catalog. Clicking an object copies its definition into the standing
level and selects it for placing.

Two lists rather than one, because they answer different questions. "Objects" is what this level is
made of and where you go to re-edit something; the Library is where you go to find something new.
Merging them would put four hundred objects into the list a player scans to find the crate they are
halfway through editing.

Delete is offered only on `Player` entries. A built-in is compiled in and a library entry lives in an
asset on disk, so deleting either would remove it from the list and have it return on the next refresh
— which reads as the button being broken rather than as a rule.

**Committing an authored object writes it to the player's library automatically**
(`bAutoSaveAuthoredObjects`). Asking a player to remember a second button after every commit is asking
them to lose work until they learn the habit.

---

## 9. What ships

**300 compiled-in objects**: a 38-object general kit plus **262 across eight themed sets**, one per base
map. Verified by `VoxelObjectList`.

### The general kit — 38 objects across five categories

| Category | Objects |
| --- | --- |
| Structure (12) | Floor Tile, Floor Tile Small, Wall Panel, Wall With Doorway, Wall With Window, Pillar, Round Pillar, Stairs, Catwalk, Ramp, Doorframe, Fence Section |
| Furniture (7) | Table, Chair, Stool, Bench, Shelf Unit, Bed, Cabinet |
| Container (7) | Crate, Crate Small, Crate Tall, Barrel, Dumpster, Pallet, Sack Pile |
| Nature (6) | Tree, Tree Tall, Bush, Rock, Log, Planter |
| Street (6) | Street Lamp, Bollard, Traffic Cone, Sign Post, Crate Barricade, Vent Unit |

58,550 voxels total. `bDestructible` is false only for things a player stands on, walks through, or
would fall out of the level without — a prop hunt whose scenery cannot be chipped is one where a
hunter identifies every disguised player with a single round, because a landed shot always removes a
cell from a player body and would remove nothing from a crate.

### The themed sets — 262 objects, one file per base map

| File | Map | Objects | Sample |
| --- | --- | --- | --- |
| `VoxelThemeCity.cpp` | Map_1_City | 36 | City Bench, Post Box, Traffic Light, Bus Stop Shelter, Fire Escape Stair, Roof Water Tank, Skip Container |
| `VoxelThemeWoodlands.cpp` | Map_2_Cemetery nature fallback | 35 | Oak Tree, Pine Tree, Birch Tree, Hollow Trunk, Camp Tent, Well, Hunting Blind, Picnic Table |
| `VoxelThemeJunkyard.cpp` | Map_3_Junkyard | 32 | Car Wreck, Crushed Cube, Tyre Stack, Bus Wreck, Junk Press, Scrap Tower, Forklift Wreck |
| `VoxelThemeAlienWorld.cpp` | Map_4_AlienWorld | 30 | Spore Pod, Fungal Tower, Crystal Spire, Egg Nest, Rib Cage, Hive Entrance, Obelisk |
| `VoxelThemeFactory.cpp` | Map_5_Factory | 32 | Machine Press, Conveyor Belt, Industrial Robot Arm, Storage Tank, Silo Tall, Gantry Walkway, Pallet Racking |
| `VoxelThemeBazaar.cpp` | Map_6_Temple architecture fallback | 31 | Market Stall, Rug Stack, Spice Sacks, Amphora, Brass Lantern, Carpet Loom, Arched Doorway |
| `VoxelThemeFrozen.cpp` | Legacy / unassigned | 31 | Snow Drift, Ice Spike, Frozen Tree, Ice Fishing Hut, Snow Shelter, Snowmobile, Crevasse Edge |
| `VoxelThemeSea.cpp` | Map_8_Sea | 35 | Rowing Boat, Fishing Boat, Jetty Section, Lobster Trap, Anchor, Harbour Crane, Cargo Container |

**Filed by what things are, not by which map they are for.** Category stays the established set, so a
bench is Furniture whichever map it was drawn for — the picker's filter row is how you find a bench, and a
per-theme category means looking in eight places for one. The theme rides in **tags**, as the map's short
legacy theme name (`city`, `woodlands`, `junkyard`, `alien`, `factory`, `bazaar`, `frozen`, `sea`), and
tags are searched. Current map-name routing maps Cemetery to the woodland fallback, Temple to the bazaar
fallback, and Mansion to the general city fallback. Converted levels normally provide their own local
objects, so these sets are only catalog/bot fallbacks. Same reasoning as §5's refusal of a "Converted"
category: where an object came from is metadata, not a shelf.

**Load order is load-bearing.** `AddOrReplace` keys on the object id and ids are hashed from the **name
alone**, so two objects with the same name are one object. The general kit loads first, so a collision
resolves identically on every machine rather than by file order. Checked: 300 names, no duplicates.

These are the first objects in the project authored against a palette that has realistic colour, and the
scripts use **material names** (`wood_dark`, `foliage_deep`, `rust`, `asphalt`, `sea_deep`, `ice`) rather
than slot numbers throughout — a wrong name is visible in the text, and retuning the tone table moves
everything that asked for wood together.

---

## 10. Verified, and not

### Verified
- Both targets compile and link: `CubeCubeCubeEditor` and the packaged `CubeCubeCube`, **re-run after the
  palette rework and the themed sets**, with zero warnings and the editor DLL confirmed relinked by
  timestamp and size.
- Headless from `Voxel_Lobby`: **`Compiled-in catalog: 300 objects from 9 scripts`** with zero script
  errors and zero "almost empty" flags, per-file counts matching the offline validator exactly; and
  **`VoxelPaletteCheck` → PASS, all 255 slots agree**, with `wood` drawing `#946B43` and `asphalt`
  `#323232`. The palette check needs a real RHI — under `-nullrhi` there is no platform data to write, so
  it correctly reports FAIL.
- All 38 general-kit objects parse with **zero** script errors, headless (`Object catalog ready: 38
  entries`).
- Geometry checked by `VoxelObjectList` and by arithmetic: Chair is 82 voxels (36 seat + 30 back + 16
  legs); Wall With Doorway is exactly 640 fewer voxels than Wall Panel, which is its 8×4×20 hole; Floor
  Tile is 4,096 at 64×64×1. Every object's height is plausible against the player's 1.60 m.
- **All 300 objects parse offline**, against a reimplementation of `FVoxelObjectScript::Parse` and
  `FVoxelObjectBuilder` that mirrors the token splitting, per-command colour-token index, clamping and
  last-write-wins cell map. Every colour token resolves against `GNamedSlots` read directly out of
  `VoxelPaletteSpectrum.cpp`; no empty objects; no duplicate names. It caught two colour names used in
  scripts but missing from the table (`amber`, `cobalt`, both since added) and one `box` reaching Z 12 in
  a grid 12 tall, which the builder would have silently clamped away.

### Not verified — nobody has looked at these
- **What any object looks like.** Voxel counts prove the arithmetic, not the shape. A well-formed object
  can still be an ugly one, and this now covers 262 objects nobody has placed.
- **Whether the fitted tone table reads well.** The palette is measured arithmetic — mean 0.024 Oklab
  against 72 real materials — and whether that *looks* right is a judgement nobody has made. Run
  `VoxelPaletteCheck` first: it reports whether the renderer is even using the ramp, which has to be true
  before any opinion about the colours means anything.
- **Whether the 560 converted objects still look the way they did.** Their greys are byte-identical; their
  hue slots are re-toned. No before/after comparison has been made — though "before" was rendering through
  a brick building's palette, so it was not correct either.
- ~~**The mesh converter against a real mesh.**~~ — **measured against arithmetic.**
  `/Engine/BasicShapes/Cube` converts to a 10×10×10 grid and **488** voxels, which is exactly its
  surface shell (10³−8³); cylinder 400 and sphere 392 fall below it in the right order. Run
  `VoxelConvertTest` to re-check. Colour on a textured asset is still unproven.

### Fixed after first use
- **A converted library was invisible outside the editor.** `LoadLibraryAssets` queried the asset
  registry, which outside the editor holds only assets something already loaded or a cooked registry
  listed — and a freshly written library referenced by nothing is neither. The converter reported
  success, the `.uasset` was on disk, and the object was simply absent. `ScanPathsSynchronous` on
  `/Game/Cube/VoxelObjects` runs before the query now. Scoped to one folder, because `SearchAllAssets`
  would scan the whole project at every startup.
- **`.vox` and plugin-JSON import.** No file of either kind has been imported.
- **The Library panel on screen.** The section is built and bound; nobody has opened the level editor
  and scrolled it.
- ~~**Quantisation quality against a palette with no muted colours.**~~ — the premise is gone. The palette
  now has eight tones per hue; see §5. Whether the result reads well is still unobserved, but it is no
  longer a question about a limitation.
