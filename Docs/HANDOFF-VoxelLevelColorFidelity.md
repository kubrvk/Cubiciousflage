# New-chat prompt: voxel level colour fidelity

Copy the prompt below into a new Codex chat.

```text
You are continuing work on Cubiciousflage, a UE 5.8 C++ voxel prop-hunt prototype at:
D:\UE_Games\CubeCubeCube

Read these first:
- D:\UE_Games\CubeCubeCube\AGENTS.md
- D:\UE_Games\CubeCubeCube\Docs\PROJECT-VoxelPropHunt.md
- D:\UE_Games\CubeCubeCube\Docs\WORKFLOW-LevelVoxelConversion.md
- D:\UE_Games\CubeCubeCube\Docs\DESIGN-ObjectCatalog.md
- D:\UE_Games\CubeCubeCube\Docs\PLUGIN_PATCHES.md

Goal:
Diagnose and fix the generic colour-fidelity problem in automatic full-level voxel conversion. The
maps convert structurally and load, but most voxel objects across the world still appear in shades of
green, olive, beige, or blue-grey instead of their original rendered/unlit colours. This is visible
across multiple unrelated marketplace packs, so do not hard-code fixes for Christmas Town, Cemetery,
or specific material parameter names. The converter must work with arbitrary future assets.

Important history:
- Conversion is per unique static-mesh/material/vertex-paint variant, with lightweight placements.
- Source maps are never modified; generated maps use `_Voxel`.
- Base Color and Emissive use UE MaterialBaking at 512x512.
- Component vertex paint and render LOD are included in variant/bake keys.
- Bright-magenta missing-texture bake output is rejected.
- Packed ORM/roughness/AO/mask textures are excluded from RGB fallback selection.
- Emissive works and must remain working.
- Landscapes use exported raw meshes at half the authored quad spacing and retain evaluated material
  bake. Raw Landscape `Color A` was tried and made Cemetery ground white, so do not restore it.
- VoxelPro is disabled because it conflicts with the existing VoxelEditor module. Do not enable it.

Previous visible failures:
- Christmas Town brown walls became white.
- Cemetery trees/walls appeared white.
- Cemetery rocks became orange because an ORM texture was selected; this specific bug was fixed.
- Cemetery terrain became white when raw `Color A` was used; this was reverted.
- After those corrections, the broader result still has most objects clustered into green/beige or
  blue-grey rather than their source colours.

Primary code to inspect:
- Source\CubeCubeCubeEditor\Private\VoxelMeshVoxelizer.cpp
- Source\CubeCubeCubeEditor\Private\VoxelLevelMapConverter.cpp
- Source\CubeCubeCubeEditor\Public\VoxelMeshVoxelizer.h
- Source\CubeCubeCube\VoxelRuntimeEditor\VoxelLevelTypes.*
- Source\CubeCubeCube\VoxelRuntimeEditor\VoxelActor.* and voxel runtime mesh/material code
- Source\CubeCubeCubeEditor\Private\VoxelCubeMaterialBuilder.cpp
- Source\CubeCubeCube\VoxelRuntimeEditor\VoxelPaletteSpectrum.*

Strong diagnostic direction:
Do not assume MaterialBaking is still the main fault. Existing logs often show plausible evaluated
Base Color averages. Trace representative colours end to end and produce evidence at every boundary:
1. source material evaluated Base Color / editor unlit reference;
2. baked texel RGB before quantization;
3. chosen palette index and Oklab distance;
4. actual RGB in that exact slot of the palette texture saved/bound to the generated voxel asset;
5. encoded vertex-color red value in the runtime mesh;
6. final unlit rendered voxel pixel.

Check especially whether conversion quantizes against one palette but runtime renders with another.
The project documentation notes that a palette donor can be auto-picked and colour identity may depend
on asset-registry order. Treat this as a primary hypothesis, not a proven cause. Also verify sRGB/linear
conversion and byte/index normalization exactly once at each boundary.

Useful generated test maps:
- /Game/ChristmasTown/Levels/Scene_compositing_Voxel_Final
- /Game/CemeteryCrypt/CemeteryMap_Voxel_Final4
- /Game/AlienBiomass/alienmap_Voxel
- /Game/Docks/dockmap_Voxel
- /Game/Factory/factorymap_Voxel
- /Game/GreenwoodFantasyVillage/GreenwoodVillagemap_Voxel
- /Game/Horror_Mansion_1/horrormansionmap_Voxel
- /Game/Japanese_Temple/JapaneseTemplemap_Voxel
- /Game/Junkyard/junkyardmap_Voxel
- /Game/ParisStreet/parisstreetmap_Voxel
- /Game/Port/portmap_Voxel
- /Game/TokyoStylizedEnvironment/tokyomap_Voxel

Implement a generic production-quality fix, reconvert at least two visually different maps, and make
an objective before/after comparison. Preserve existing correct behavior. Validate with an editor
build, a real runtime map load, and `Automation RunTests Cubiciousflage`. Document the confirmed root cause,
the fix, performance/data impact, test evidence, and any remaining limitations.
```
