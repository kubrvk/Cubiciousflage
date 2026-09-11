# Automatic Level Voxel Conversion

## Decision

Cubiciousflage converts a conventional level **per unique static-mesh variant**, then stores lightweight
placements in a `UVoxelLevelAsset`. A variant is identified by static mesh and material overrides.
Rotation, location, and scale stay on each placement, allowing randomized foliage instances to share
one voxel definition instead of creating thousands of nearly identical definitions.

This is preferable to one world-sized voxel object:

- repeated meshes share one runtime voxel template and collision build;
- one oversized edit cannot invalidate the entire map;
- individual props can remain destructible while floors, walls, houses, roofs, roads, and similar
  structural meshes are marked non-destructible by name;
- one failed or effect-only component can remain native without invalidating the rest of the map;
- the source marketplace map is never modified.

Lights, fog, sky, post-process, cameras, audio, particles, gameplay actors, and volumes remain native.
They do not represent solid geometry and converting them would either be meaningless or destructive.

## Editor workflow

1. Open the source map.
2. Choose **Tools > Voxel Objects > Convert Current Level to Voxel Copy**.
3. Confirm the derived output paths.
4. Open the generated `<SourceName>_Voxel` map and test Play In Editor.

The console equivalent is:

```text
VoxelConvertCurrentLevel [/Game/Path/OutputMap] [/Game/Cube/Levels/OutputLevelAsset] [WorldScale] [InclusionVolume|none] [MinOccupiedVoxels]
```

With no arguments, both paths are derived from the loaded map. Existing output assets are updated,
so the operation is repeatable after changes to the source level or converter.

The same command also compiles reusable runtime proxies. On a large map (512 or more placements),
only the 24 definitions with the greatest player-near/reuse value are prebuilt; unchanged proxies are
reused by content hash on later conversions. The remaining definitions are valid voxel payloads and
are compiled lazily when distance streaming first requests them. This keeps the one-click workflow
bounded instead of turning hundreds of one-time mesh builds into a multi-minute editor stall.

### Inclusion volumes and uniform world reduction

If the source contains an `AVolume` whose actor label or object name is `cutvoleme`, conversion treats
its actual brush as an inclusion mask. Static-mesh instances and Landscape components whose bounds do
not intersect the volume are rejected before mesh export, material baking, or voxelisation. For a
placement crossing the boundary, each occupied voxel centre is tested with `AVolume::EncompassesPoint`;
only retained cells are stored. The generated copy hides outside source geometry and disables the
marker volume's visibility/collision. The source map is never saved. Pass `none` in the fourth console
argument to force whole-world conversion.

Existing shipped voxel payloads can be trimmed and redressed without repeating mesh/material conversion.
Load the paired `/Game/Cube/Map/Map_*` travel map and run `VoxelDressShippedLevels <map>` for a dry run or
`VoxelDressShippedLevels Apply <map>` to save. A `cuthere` volume clips occupied voxel centres and removes
outside placements; Factory and Junkyard are deliberately excluded. `spamhere` and `canvoxelspam` volumes
target 100 themed props with one-voxel relaxed clearance, while ordinary placement retains the standard
20 cm clearance. Props must fit fully inside the volume. Port's volume tests account for its 0.7 converted
world scale. Each first save creates a `.before-density-cut-backup` beside the level asset.

`WorldScale` uniformly scales a generated world about world origin without changing voxel size.
Geometry is re-voxelised at that scale, placement locations receive the same factor, and retained
native root actors in the generated copy are transformed with it. `Placement.Scale` deliberately does
not receive this factor: applying it there would create half-sized voxel cubes instead of reducing a
20x50-voxel object to approximately 10x25 cells. This is an offline data reduction and adds no runtime
per-frame work. `MinOccupiedVoxels` optionally omits converted fragments below a conservative cell
count; zero disables that filter.

## Material colour and emissive

Colour conversion uses Unreal's `MaterialBaking` evaluator, not texture-name guesses. Each used
material slot is evaluated to a 512x512 final Base Color and Emissive texture, including parent
material functions and material-instance overrides. The voxelizer samples those baked results at the
mesh UV and quantizes the result to the active Cubiciousflage palette.

The baker derives the UV0 bounds actually used by every material slot and remaps that rectangle into
the bake target. This is important for marketplace materials with tiled, scaled, or negative UVs:
baking only Unreal's default 0..1 rectangle clips those surfaces and turns the uncovered texels white.

Run conversion in a normal editor session. UE 5.8's material baker crashes in commandlet/`-nullrhi`
rendering, while the editor workflow above supplies the renderer the API expects. Materials that cannot
be baked retain the older texture/constant fallback rather than aborting the entire level.

Emissive is stored as an optional byte parallel to each sparse voxel definition and copied into the
runtime template. The shared cube material reads that mask from vertex blue and emits the voxel's
palette hue. Assets created before this field was added keep an empty array and render exactly as before.
The conversion remains offline/editor-only; runtime CPU cost is unchanged, while an emissive voxel adds
one byte of authored data and the shared material adds one palette sample per rendered voxel pixel.
Only a genuinely connected Emissive Color graph is baked, and only HDR energy above the ordinary 0..1
colour range authors a voxel glow. This excludes UE's white default property buffer and foliage
materials that route Base Color through Emissive for unlit fill. Components made entirely from
translucent/additive materials remain native because converting an effect plane into opaque cubes is
not a meaningful representation of its blend mode.

The mesh-backed bake reads the final render LOD rather than only the imported source mesh. This is
required because Unreal stores painted vertex colours on each placed `UStaticMeshComponent`; they are
not necessarily present in the mesh asset. The component paint hash is part of both the level variant
key and the material-bake cache key, so differently painted instances do not share the first instance's
snow/brick blend. Invalid bright-magenta MaterialBaking output is recognized as Unreal's missing-texture
sentinel and discarded in favor of the authored texture/constant fallback. Fallback ranking uses the
material parameter name as well as the texture name: packed ORM/roughness/AO/mask inputs are never
interpreted as RGB merely because a marketplace texture has an incorrect sRGB flag. If the material
exposes only Unreal's default placeholder texture, its primary authored colour vector wins.

## Landscape terrain

Landscape conversion uses `ALandscapeProxy::ExportToRawMesh` once per `ULandscapeComponent`. Each
component becomes an independent structural voxel definition and placement; successfully converted
native Landscape components are hidden and have collision disabled only in the duplicate map. The
component's generated material instance, including its painted weight maps, goes through the final
Base Color baker. A normal interactive editor may additionally attempt an isolated deferred BaseColor
capture of the real Landscape component. Unattended conversion deliberately skips that renderer-only
capture because UE commonly supplies a blank or gray WorldGrid fallback there. Covered-pixel checks
also reject an empty interactive capture rather than writing it into the voxel palette. The evaluated
component bake is retained because it preserves the final tint and painted spatial variation. Before
palette matching, only underexposed fallback bakes receive a bounded adaptive exposure toward 0.10
linear luminance. Sampling raw `Color A` was tested and rejected because that input alone rendered the
Cemetery terrain white; it is not the final Landscape output.

Terrain uses half the Landscape's authored horizontal quad spacing (50 uu in CemeteryCrypt), not the
10 uu prop resolution. This doubles resolution on each horizontal axis and removes the severe 100 uu
terracing while keeping each chunk below the safe dense-grid ceiling. Using 10 uu would create roughly
100 times the authored surface sample count. This is an offline editor cost and adds no per-frame
conversion work.

Ordinary scenery begins at the requested 10 uu resolution, but a mesh whose predicted voxel grid has
an axis longer than 64 cells automatically receives the smallest coarser voxel size that fits that
limit. This is what allows `Big_mountain` combinations to convert: rejecting an enormous dense grid
left the original smooth mesh visible, while attempting to build it at 10 uu consumed excessive time
and memory. The adaptive result remains voxel geometry, preserves the actor transform, material slots,
damage policy, and occupied silhouette, and records its actual voxel size in the definition.

Instanced components above 2,048 instances remain native. Converting dense grass into one voxel actor
per blade is not a viable runtime representation; ordinary instanced props below the limit are converted
and preserve their per-instance transform and scale.

## Large-level runtime streaming

Baked levels with fewer than 512 placements retain the eager path. Larger non-creator levels keep the
complete placement metadata resident but materialize only player-near voxel actors. A bounded collision
bubble is created around spawn markers before possession; after that, a timer requests actors nearest
first within 12,000 uu and destroys them beyond 16,000 uu. The separate unload radius prevents boundary
churn. At most 2,500 scenery actors may be live, construction is limited to eight placements per 25 ms
pulse, and destruction to 64 per pulse. All player pawns are streaming sources and a short velocity
look-ahead reduces visible pop-in during fast movement.

The 13k-entry bounds table is evaluated four times per second; there is no per-frame actor or placement
scan. Empty timer pulses only inspect the pending queues. A damaged placement is pinned resident because
unloading a locally edited actor would otherwise recreate its original baked voxel data. Gameplay that
needs the mountain extent, including Climp's five checkpoint bands, reads immutable baked bounds and
does not depend on far actors being spawned.

Destroying streamed-out actors is materially better than only hiding them: visibility culling saves draw
calls but retains every UObject, voxel mesh, collision body, and the full startup construction cost.
Distance materialization removes those CPU/memory costs as well as rendering cost. World Partition or
HLOD can still stream retained native marketplace content independently; it does not replace voxel
placement streaming.

One source PlayerStart becomes the generated level's lobby marker. A start carrying the actor tag
`VoxelConversionStart` wins; otherwise the lowest eligible start is selected deterministically. This
keeps vertical maps at their waterline entrance instead of depending on actor iteration order and
occasionally starting at the summit. If a showcase map has no PlayerStart, the converter uses the
median converted-placement coordinates (robust against distant sky and effect meshes), traces down to
source collision, and writes a ground-relative fallback marker.

## Christmas Town result

Source:

```text
/Game/ChristmasTown/Levels/Scene_compositing
```

Generated outputs:

```text
/Game/ChristmasTown/Levels/Scene_compositing_Voxel_Final
/Game/Cube/Levels/Level_Scene_compositing_Voxel_Final
```

The final conversion produced 253 reusable definitions, 1,160 placements, 2,171,865 occupied voxels,
and 11,566 authored emissive voxels. The higher definition count is intentional: component-level vertex
paint is now preserved, so buildings with different brown/brick/snow blends remain distinct.
The only visible native static mesh left in the output is Unreal's 400x sky sphere. Its requested voxel
grid would contain roughly 35 quadrillion cells; it is an atmospheric backdrop with no gameplay
geometry and is intentionally preserved.

Surface voxelization is used by default. Filling every mesh interior would multiply storage and rebuild
cost for no visible benefit until geometry is cut open.

## Cemetery Crypt result

Source and generated outputs:

```text
/Game/CemeteryCrypt/CemeteryMap
/Game/CemeteryCrypt/CemeteryMap_Voxel_Final4
/Game/Cube/Levels/Level_CemeteryMap_Voxel_Final4
```

The conversion produced 183 reusable definitions, 1,701 voxel placements, 1,812,935 occupied voxels,
and 16 authored emissive voxels. This includes all 64 Landscape components, normally about
`126x126` cells horizontally. Three extremely dense foliage components
and 47 translucent/effect components remain native. The source map has no PlayerStart, so the generated
asset contains one traced median fallback marker. Runtime level construction completed in 34.2 seconds
and spawned `VoxelPropCharacter_0` in the offscreen development-editor validation run.

## Marketplace map batch result

The finalized colour/emissive and Landscape pipeline was also applied to these source maps. Each
source remains unchanged; the generated map uses the `_Voxel` suffix and its data asset is stored in
`/Game/Cube/Levels/Level_<MapName>_Voxel`.

| Generated voxel map | Reusable objects | Placements |
| --- | ---: | ---: |
| `/Game/AlienBiomass/alienmap_Voxel` | 97 | 1,045 |
| `/Game/Docks/dockmap_Voxel` | 546 | 1,197 |
| `/Game/Factory/factorymap_Voxel` | 272 | 2,750 |
| `/Game/GreenwoodFantasyVillage/GreenwoodVillagemap_Voxel` | 56 | 886 |
| `/Game/Horror_Mansion_1/horrormansionmap_Voxel` | 504 | 5,420 |
| `/Game/Japanese_Temple/JapaneseTemplemap_Voxel` | 210 | 3,770 |
| `/Game/Junkyard/junkyardmap_Voxel` | 198 | 665 |
| `/Game/ParisStreet/parisstreetmap_Voxel` | 355 | 1,846 |
| `/Game/Port/portmap_Voxel` | 368 | 3,443 |
| `/Game/TokyoStylizedEnvironment/tokyomap_Voxel` | 586 | 5,539 |

After the palette-identity fix, the marketplace maps were regenerated on 2026-08-10. Every current
conversion established `256/256` palette-slot identity with worst Oklab distance `0.000000` and wrote
its `/Game/Cube/Levels/Level_*_Voxel` asset. Factory, Junkyard, Greenwood, and Port were subsequently
rebuilt from their updated source maps using the reduction policies below; the table shows their latest
object and placement counts. Source maps remained unchanged by conversion.

All ten generated maps completed a clean offscreen runtime load. Before inclusion clipping, Junkyard
took about nine minutes to construct 2,334 placements and 1,025 definitions; Port previously took close
to five minutes with 9,353,011 stored voxels. The reduced heavy-map results are recorded below.

| Map | Policy | Stored voxels | Level asset | Real-renderer map load |
| --- | --- | ---: | ---: | ---: |
| Factory | `cutvoleme` | 2,608,345 | 16.7 MB | 35.4 s |
| Junkyard | updated `cutvoleme`, minimum 8 cells | 1,607,983 | 9.9 MB | 38.6 s |
| Greenwood Village | `cutvoleme` | 283,677 | 2.0 MB | 9.1 s |
| Port | 0.7 world scale | 4,540,888 | 28.5 MB | 89.2 s |
| Tokyo | `cutvoleme` | 1,176,520 | 9.1 MB | 28.1 s |

Factory rejected 2,199 source placements and removed 859,844 cells from 106 boundary placements.
Junkyard's enlarged volume retained 665 placements, rejected 1,736, removed 1,207,569 cells from 80
boundary placements, and omitted one sub-eight-cell variant. Greenwood retained 886 of 4,600 source
placements, rejected 3,714, and removed 4,337 cells from its one boundary placement. Port retained all
3,443 placements but reduced stored voxels by 51.5% from 9,353,011; its level asset fell from 57.4 MB
to 28.5 MB. Tokyo's broad inclusion volume rejected 2 of 5,541 source placements and removed 29,320
cells from 4 boundary placements, leaving 5,539 placements. All five generated maps were rebuilt from
their source worlds and reached
`UEngine::LoadMap Load map complete` under D3D12.

### World colour fidelity root cause and fix (2026-08-10)

The generic green/olive/beige and blue-grey cast was a palette-identity bug, not primarily a
`MaterialBaking` bug. Full-level conversion called `BuildEffectivePalette` while the donor still held
its marketplace-authored palette. Runtime `UVoxelLevelSubsystem::Initialize` later called
`EnsureGeneratedRampApplied` and replaced that texture row with the Cubiciousflage spectrum. Historical
runtime logs measured **251 of 256 slots different** between those tables. Catalog objects had a legacy
donor-to-ramp translation; `FVoxelLevelData` did not. A valid stored index therefore selected an
unrelated runtime colour even when the evaluated Base Color was correct.

Both full-level and bulk-object conversion now establish the generated ramp first, require a successful
GPU texture write, read the effective palette back through the same donor/page path, and compare all 256
entries with the generated spectrum. Conversion fails before writing output unless the worst slot Oklab
distance is at most 0.002. A healthy run logs `256/256` agreement and distance `0.000000` before its first
material bake. This removes asset-pack and material-parameter assumptions; any future asset is quantized
against exactly what the runtime material samples.

The voxelizer also reports per-object mean/max Oklab quantization error plus the worst source linear RGB,
chosen slot, and actual slot RGB. The August 10 reconversions measured:

| Map | Definitions | Voxels | Weighted mean Oklab | Worst cell Oklab | Emissive voxels |
| --- | ---: | ---: | ---: | ---: | ---: |
| Christmas Town | 253 | 2,171,865 | 0.02097 | 0.1159 | 11,566 |
| Cemetery Crypt | 183 | 1,812,935 | 0.02286 | 0.1209 | 16 |

Both generated maps completed real D3D12 runtime loads after reconversion (1,160 and 1,701 placements).
`Automation RunTests Cubiciousflage` passed all three discovered tests, including the palette self-quantization
and vertex-red byte round-trip regression. Cemetery's level asset saved successfully;
the duplicate `.umap` rewrite was refused because Unreal could not delete the already-loaded destination
world, but the existing map already points to that level asset, and its subsequent runtime load used the
new 183-definition payload.

Remaining colour error is ordinary 256-slot quantization and material-bake/fallback quality. Some highly
saturated or very dark samples still exceed 0.10 Oklab, and visual art approval is still required. The
fix intentionally leaves emissive, component vertex paint, render-LOD keys, invalid-magenta rejection,
packed-ORM exclusion, and evaluated Landscape baking unchanged.

## VoxelPro incompatibility

`Plugins/VoxelPro` and the established `Plugins/VoxelEditor` both declare a module named
`VoxelEditor` and a Build.cs rule class with that same name. Unreal cannot load or even generate build
rules for both plugins in one project.

VoxelPro is quarantined by naming its descriptor `VoxelPro.uplugin.disabled`, and it is also explicitly
disabled in `Cubiciousflage.uproject`. Its source remains available for inspection. Renaming the descriptor
back is only safe after moving it to a separate host project or renaming every conflicting module and
updating the plugin's internal dependencies; it must not be enabled alongside the current plugin as-is.
