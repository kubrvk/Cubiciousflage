# Cubiciousflage — CubeKin Character Collection

Status: all five implementation phases completed 2026-08-08. All 53 CubeKin now use their authored
AnimateMesh voxel JSON sculptures; release-candidate visual review and multiplayer soak remain QA gates,
not missing systems.

## Decision

Replace the current Male/Female wardrobe concept with one unified **Characters** catalogue named **CubeKin**: 53 original voxel-built people, creatures, animals, robots and monsters. Voice remains an independent player setting.

This is intentionally a geometry collection, not another recolour pass. `FVoxelSkinCatalog` now carries
one append-only geometry key for every CubeKin, built by `FVoxelCharacterShapes` and shared by shop,
equip, save, replication and voxel-damage paths.

The four weapon catalogues remain weapon skins. The ten existing Legend identities and three Ultimate identities remain in place so their materials, auras, footsteps and future Steam DLC bundles retain value.

## Concept sheets

![Common and Rare collection A](Art/CubeKin/CubeKin_01_Common_Rare_A.png)

![Rare and Super Rare collection A](Art/CubeKin/CubeKin_02_Rare_Super_A.png)

![Super Rare collection](Art/CubeKin/CubeKin_03_SuperRare.png)

![Legend and Ultimate collection](Art/CubeKin/CubeKin_04_Legend_Ultimate.png)

These are visual targets, not literal voxel blueprints. Each production model must be simplified onto the game's 32³ grid and checked from front, side, back and the shop's real thumbnail camera.

## Visual language

CubeKin should feel like a shelf of premium toys built by one fictional civilization:

- Every visible form is made from hard, readable cubes. No smooth underlying body disguised by a pixel texture.
- A compact head, short limbs and planted feet make the characters friendly and readable at shop-tile size.
- Each character has one dominant silhouette idea: ears, hood, crown, compact tail, folded wings, horn shape or halo.
- Palettes use a dark anchor, a body family and one high-value accent. Glow is rare below Super Rare.
- Face language is shared: two dark square eyes, a minimal mouth or muzzle, and expressions carried mainly by brow and head shape.
- Surface detail supports the silhouette. It never replaces it. A dragon must read as a dragon in flat grey.
- Accessories are fused into the voxel sculpture. Avoid dangling straps, cloth cards and skeletal secondary motion.

The mood is **playful adventure with premium finish**, not parody, horror or a clone of another voxel game's character language.

## Competitive construction rules

The current body is 16 voxels tall at 10 cm per voxel inside a 32³ editing grid. CubeKin keeps that gameplay scale.

| Rule | Production target |
| --- | --- |
| Height | Z 0–15 for damageable geometry; VFX alone may exceed it |
| Width | Fit the same 10-cell shoulder/arm envelope as the current Male preset |
| Depth | Target 3–5 cells; compact tails may reach 6 only after camera/capsule review |
| Grounding | Two planted feet with the same lowest Z and comparable footprint |
| Occupancy | Target 260–340 damageable voxels; never below the existing 100-voxel character floor |
| Readability | Recognizable at 96 px in flat unlit grey and at 20 m in the lobby |
| Appendages | Horns, ears, tails and folded wings remain inside the common envelope |
| Effects | No glow bloom, aura or particles may obscure the damageable body in combat |

Equal voxel count alone does not create equal hit profiles. Before these become paid cosmetics, choose and test one fairness policy:

1. **Recommended for the prototype:** author every CubeKin on a shared combat occupancy mask. Reposition only a limited ornament budget while the torso, head, arms, legs and exposed front/side coverage remain identical.
2. Later alternative: separate a canonical damage hull from cosmetic render voxels, then explicitly revise the project's “player build is hitbox” rule and all voxel-damage feedback around it.

Do not quietly let a small paid character become harder to hit.

## The 53-character shop

### Common — three free foundations

| # | Character | Design hook |
| --- | --- | --- |
| 01 | Rookie Builder | Teal work suit, orange belt, square cap; the welcoming default mascot. |
| 02 | Moss Scout | Leaf hood, bark-brown layers and one yellow flower; nature starter. |
| 03 | Scrapjack | Patched steel maintenance bot, cyan screen face and wind-up details. |

### Rare — twenty expressive everyday creatures

| # | Character | Design hook |
| --- | --- | --- |
| 04 | Bumble Bear | Round bear beekeeper in cream and honey gold. |
| 05 | Coral Axolotl | Pink external gills, cyan overalls and compact tail. |
| 06 | Night Owl | Navy scholar coat, gold eye rims and flat mortarboard silhouette. |
| 07 | Ember Fox | Rust-red courier with green scarf and integrated satchel. |
| 08 | Frost Penguin | Blue polar jacket, square goggles and tiny orange feet. |
| 09 | Bog Frog | Bright green ranger with broad eyes and orange neckerchief. |
| 10 | Royal Corgi | Cream-and-gold corgi monarch with tiny crown and cape. |
| 11 | Bin Bandit | Charcoal raccoon scavenger with olive pack and mask stripe. |
| 12 | Deepglow Angler | Petrol-blue diver creature with a compact cyan lure. |
| 13 | Sporekin | Red mushroom cap, ivory body and herb-gathering pack. |
| 14 | Gummy Squire | Lime-and-aqua candy creature; opaque stepped jelly treatment. |
| 15 | Clockwork Mouse | Brass inventor, round ear silhouette and wind-up key. |
| 16 | Storm Ram | Slate wool armor and cyan spiral horns. |
| 17 | Paper Crane | Ivory-and-vermilion folded-cube bird warrior. |
| 18 | Candy Kaiju | Bright confection monster with gumdrop dorsal blocks. |
| 19 | Stone Gargoyle | Charcoal sentinel with folded wings forming a cape shape. |
| 20 | Pocket Rex | Small green explorer dinosaur with tan field hat. |
| 21 | Cloud Sheep | Clustered white cube fleece around a sky-blue face. |
| 22 | Cactus Bandit | Upright cactus outlaw with crown flower and red neckerchief. |
| 23 | Moon Moth | Midnight moth scholar with crescent wing markings and gold eyes. |

### Super Rare — seventeen material-forward fantasies

| # | Character | Design hook |
| --- | --- | --- |
| 24 | Neon Oni | Magenta/cyan guardian with compact horns and luminous mask seams. |
| 25 | Circuit Gecko | Black-and-lime hacker lizard with circuit bands and coiled tail. |
| 26 | Vapor Shark | Purple street-racer shark with cyan jacket and sneaker blocks. |
| 27 | Holo Kitsune | Cyan spectral fox trickster with compact layered tails. |
| 28 | Chrome Scarab | Mirror-metal beetle knight with gold joint accents. |
| 29 | Static Ghost | Monochrome noise spirit with broken antenna silhouette. |
| 30 | Skyline Raven | Black raven rogue with sunset-orange edge accents. |
| 31 | Toxic Salamander | Lime alchemist with integrated violet potion blocks. |
| 32 | Plasma Mantis | Violet monk with folded luminous forearms, never long blades. |
| 33 | Midnight Bat | Indigo detective whose folded wings read as a trench cape. |
| 34 | Arcade Mecha | Cyan cabinet-shaped robot with colorful control-panel chest motif. |
| 35 | Glitch Mimic | Black core with deliberately offset magenta/cyan cube fragments. |
| 36 | Solar Lion | Gold guardian with a radial stepped mane and blue jewel accent. |
| 37 | Voltage Hare | Yellow/cyan speedster with lightning ear cuts and compact stance. |
| 38 | Obsidian Golem | Heavy black stone body split by restrained violet cracks. |
| 39 | Crimson Drake | Deep-red dragon knight, ivory horns and folded tail. |
| 40 | Prism Peacock | Opalescent duelist with a compact geometric fan-tail. |

### Legend — ten existing identities, now with unique geometry

| # | Identity / character | Design hook | Existing bundle to retain |
| --- | --- | --- | --- |
| 41 | Aurora Stag | Arctic stag guardian with mint aurora antlers. | Aurora |
| 42 | Dragonfire Wyrm | Charcoal dragon with lava seams and ember chest. | Dragonfire |
| 43 | Starfall Knight | Ivory/cobalt astral knight with star-cut face and shoulders. | Starfall |
| 44 | Thunder Colossus | Ram-headed storm titan with cyan lightning fractures. | Thunder |
| 45 | Nebula Kitsune | Deep-violet cosmic fox mystic with starfield tails. | Nebula |
| 46 | Prism Seraph | Opalescent guardian with tightly folded crystalline wings. | Prism |
| 47 | Phoenix Ascendant | Scarlet/gold firebird warrior with flame-feather cape. | Phoenix |
| 48 | Void Reaper | Near-black horned wraith with a sparse halo of violet cube motes. | Void |
| 49 | Chrono Warden | Teal/brass clock guardian with a square halo ring. | Chrono |
| 50 | Celestial Atlas | Ivory/gold constellation lion with royal mantle. | Celestial |

### Ultimate — three flagship mascots

| # | Character | Design hook | Existing bundle to retain |
| --- | --- | --- | --- |
| 51 | NIGHTCOST Zero | Sleek black cyber guardian cut by hot-pink grid light. | NIGHTCOST |
| 52 | Cubiciousflage Prime | Charismatic gold/teal cube monarch and primary brand mascot. | Cubiciousflage |
| 53 | OMEGA Architect | White/graphite geometric construct with impossible-looking luminous cuts. | OMEGA |

## Rarity presentation

- **Common:** matte surfaces, no glow, clear workwear silhouettes.
- **Rare:** one material story or accessory; at most one tiny glow feature.
- **Super Rare:** premium surface plus a controlled emissive channel; no persistent aura by default.
- **Legend:** identity material, one aura and one footstep already associated with that identity.
- **Ultimate:** signature geometry, primary and secondary premium material, prismatic treatment and identity effects. Ultimate must look valuable when all effects are disabled.

## UE 5.8 production roadmap

### Phase 1 — prove one geometry skin

**Completed 2026-08-08:** geometry key, two code-authored sculptures, skin-specific previews,
equip/save/fresh-profile spawn integration, Development game/editor builds, headless lobby runtime
smoke check, and `Cubiciousflage.CubeKin.*` automation tests.

1. Build `Rookie Builder` and `Coral Axolotl` as code-authored `FVoxelRuntimeDesignData` on the existing 32³ grid.
2. Add front/side/back occupancy snapshots and counts to an automation test.
3. Run them through existing palette quantisation, `ConfigureAsset`, thumbnail rasterisation, capsule fitting, voxel damage and replication.
4. Test at 96 px, third-person distance, first-person shadow, lobby lighting and every map exposure range.

Exit condition: both characters survive build, runtime play, voxel destruction and `Automation RunTests Cubiciousflage` with no material or replication regression.

### Phase 2 — make geometry a first-class catalog field

**Completed 2026-08-08:** `FVoxelSkinDef::Geometry` is explicit, `FVoxelCharacterShapes` owns all
construction, and skin IDs key the session thumbnail cache. Slate never authors geometry.

Add a character geometry key or immutable design payload to the cosmetic definition. Do not overload `Surface` or infer geometry from display names. A practical shape is:

```text
FVoxelCharacterDef
    Id                 stable save id
    DisplayName
    Rarity
    GeometryId         stable code-authored geometry key
    Ramp
    Surface
    Fx / aura / footstep identity
    Price / DLC ownership
    VoicePolicy        independent; normally Any
```

Keep the store subsystem as the only ownership authority. Keep geometry construction in a dedicated registry, not in Slate. Cache preview designs by `(GeometryId, Ramp, Surface)` and invalidate when any component changes.

### Phase 3 — catalogue and save migration

**Completed 2026-08-08:** append-only `EVoxelSkinCatalogue::Character` preserves raw legacy enum keys;
Male/Female ownership and equipped selections map by original catalogue position; identity DLC app IDs
continue to grant the matching Legend/Ultimate; voice no longer changes with a body preset.

1. Introduce the unified `Character` catalogue without reordering existing raw material/effect enums.
2. Map owned Male/Female skin IDs to equivalent CubeKin grants or compensate with Gold.
3. Migrate equipped Male/Female selection to one equipped Character ID.
4. Move voice selection out of preset choice; existing Male/Female voice choice becomes an independent saved field.
5. Retain all 13 identity DLC mappings and gift bundles.

### Phase 4 — production batches

**Completed 2026-08-08:** Commons 01–03, Rares 04–23, Super Rares 24–40, Legends 41–50 and Ultimates
51–53 all have distinct immutable geometry and authored ramps. Every sculpture includes the same
270-cell combat core and stays inside the automated 270–340 voxel budget.

Build in visual/risk order:

1. Commons 01–03: establish grid language and shared occupancy mask.
2. Rares 04–13: animal faces, tails and hats.
3. Rares 14–23: unusual materials and folded silhouettes.
4. Super Rares 24–40: emissive and premium material validation.
5. Legends 41–50: identity bundle integration.
6. Ultimates 51–53: signature VFX, marketing renders and final polish.

Each batch needs code review, a Development build, a runtime walk/jump/fire/damage/equip check, multiplayer replication, thumbnail review, and `Automation RunTests Cubiciousflage`.

### Phase 5 — shop presentation

**Completed 2026-08-08:** the live shop exposes one Characters tab, fixed 96 px axonometric cached
thumbnails, rarity-neutral dark preview plates, and a `FLAT SILHOUETTE` accessibility toggle. The grid
contains no live voxel actors and therefore adds no Tick cost.

- Use a fixed camera and turntable, not 53 manually composed thumbnails.
- Render on rarity-tinted neutral pedestals; never let the background compete with the silhouette.
- On focus, show name, rarity, material, aura and footstep; do not animate every off-screen tile.
- Rotate only the selected preview. A grid of dozens of live voxel components is unnecessary GPU and Slate work.
- Add a “flat material” preview toggle for honest silhouette inspection and accessibility.

## Acceptance checklist for every character

- Recognizable as a grey silhouette with all materials and VFX disabled.
- Fully contained in the approved damageable envelope.
- Occupied voxel count and front/side exposure fall within the chosen fairness tolerance.
- No one-cell whiskers, fingers or cloth details that flicker or vanish at distance.
- Face reads in the shop thumbnail and does not rely on antialiasing.
- Premium material preserves palette hierarchy instead of flattening the character.
- Aura and footstep match the identity and remain readable without hiding combat state.
- No hard references to external character brands or copied voxel designs.
- Builds, cooks, replicates, takes voxel damage and passes the Cubiciousflage automation suite.

## Art-source note

The four sheets were generated with the built-in image-generation workflow and copied into the project. They are original visual references. Exact generation prompts are stored in `Docs/Art/CubeKin/PROMPTS.md`.
