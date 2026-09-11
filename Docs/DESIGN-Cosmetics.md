# DESIGN — Cosmetics, Shop, Materials, Effects, Steam Monetisation

> **CubeKin migration note (2026-08-08):** `Docs/DESIGN-CubeKin-Characters.md` defines the live
> geometry-bearing character direction. The paint-only rules below remain authoritative for weapons
> and hidden legacy character rows. All 53 CubeKin rows select immutable code-authored sparse geometry
> through `EVoxelCharacterGeometry`; `ApplySkinToDesign` is still only the recolour stage.

Scope: the skin catalogue, the material system built on the purchased asset packs, the effect system,
the shop and library UI, the Steam DLC / Gold plan, and how all of it is tested.

Companion to `PROJECT-VoxelPropHunt.md`. Nothing here overrides the rules in that file; where the two
touch (palette single-source, material assignment order, Shipping-safe visuals) this document explains
how cosmetics fit *inside* those rules rather than around them.

---

## 0. Status

| Piece | State |
| --- | --- |
| Original catalogue (6 catalogues x 53) | **built; 318 definitions retained** |
| Ownership, purchase, gifting, DLC, equip | **built** |
| Dev console commands (7) | **built** |
| Shop: catalogue tabs, Materials, Effects with Footsteps/Aura sub-tabs | **built** |
| Library: owned skins in Characters / Weapons | **built** |
| Skin recolour applied in-world (body + viewmodel) | **built** |
| 58 material instances harvested from the purchased packs | **built** - run `VoxelBuildSkinMaterials` |
| Cube Builder: MATERIAL, AURA and EFFECT dropdowns | **built** |
| 65 authored Niagara effects, two kinds worn together | **built** |
| Levels -> Materials category | **built** |
| FX overlays (Sheen / Pulse / Rgb / Prismatic) | **not built** - `GetFxOverlayPath` names assets that do not exist |
| Unified CubeKin catalogue | **built** - 53/53 unique geometry entries, migration, preview/equip/fresh-spawn paths and automation |
| Footstep/Aura sub-buttons in the Characters window | **not built** |
| Steam Inventory Service (Gold packs) | **not built** |
| Real Steam DLC appids | **placeholders** |

### File map

| File | Role |
| --- | --- |
| `VoxelSkinCatalog.h/.cpp` | 371 definitions: 318 original/weapon plus 53 live CubeKin, alongside 58 materials and 65 effects. Every table, price, tier colour, texture path, technique value, migration and Niagara path. `ApplySkinToDesign` lives here. |
| `VoxelStoreSubsystem.h/.cpp` | Ownership, purchases, gifting, DLC resolution, equipped skin / material / footstep / aura. |
| `VoxelStoreDiagnostics.cpp` | `VoxelSkinList`, `VoxelSkinStatus`, `VoxelSkinVerify`, `VoxelForceMaterial`, `VoxelGrantDlc`, `VoxelGrantGold`, `VoxelEquipSkin`. Development only. |
| `VoxelSkinMaterialBuilder.cpp` (editor) | `VoxelBuildSkinMaterials`: generates `M_VoxelCubeSkin` and its 58 instances. Holds the shader. |
| `VoxelCubeMaterial.h/.cpp` | Three-argument `ConfigureAsset` binds a skin surface before the first chunk exists; skin instance cache. |
| `VoxelSculptComponent.h/.cpp` | `ResolveEquippedSurface`, `RefreshSkinSurface`, `ForceSkinSurface`, `DescribeBoundMaterial`. |
| `VoxelPropCharacter.h/.cpp` | Footstep timer, attached aura, the hidden skeletal driver mesh. |
| `SVoxelMainMenu.cpp` | Shop and library UI. |
| `SVoxelRuntimeEditorPanel.cpp` | Cube Builder MATERIAL / AURA / EFFECT dropdowns. |

---

## 1. Catalogue shape

Six **catalogues**, one per skinnable thing:

| Catalogue | Source enum | Notes |
| --- | --- | --- |
| Male | `EVoxelBuiltInPreset::Male` | character |
| Female | `EVoxelBuiltInPreset::Female` | character |
| Rifle | `EVoxelWeaponType::Rifle` | weapon |
| Grenade Launcher | `EVoxelWeaponType::GrenadeLauncher` | weapon |
| Physics Launcher | `EVoxelWeaponType::PhysicsLauncher` | weapon |
| Laser | `EVoxelWeaponType::Laser` | weapon |

Each original catalogue holds **53 skins**: 50 earnable, plus 3 Ultimate that are DLC only. The
append-only unified Character catalogue adds 53 CubeKin, for **371 stored definitions**. Male/Female
remain loadable for migration and are not shown in the live shop.

**The Cube catalogue was removed.** The crate is what `EnforceMinimumPropModels` forces a prop into
below the minimum voxel count - a failure state players work to avoid - so a 53-skin wardrobe for it
was fifty tiles for a body nobody chooses to wear, cluttering both the shop and the library. `Crate`,
`Dot` and `None` return false from `CatalogueForPreset`.

### 1.1 Rarity tiers

| Tier | Per catalogue | Accent | Tile background | Cube surface |
| --- | --- | --- | --- | --- |
| Common | 3 | Grey `#9AA0A8` | flat slate, no sheen | Matte |
| Rare | 20 | Blue `#3B82F6` | blue vertical gradient | Matte, faint gloss |
| Super Rare | 17 | Purple `#A855F7` | purple gradient + static corner sheen | **Shiny**, emissive accent voxels |
| Legend | 10 | Gold `#F5C518` | gold gradient + animated sweep | **Shiny + emissive + RGB cycle + VFX** |
| Ultimate | 3 | Prismatic `#22D3EE` → white | prismatic animated, "DLC ONLY" ribbon | everything Legend has, plus a signature VFX |

3 + 20 + 17 + 10 = 50 earnable, + 3 Ultimate = 53.

The tier accent colour is one value used in five places, so the shop reads as a single system: the tile
background, the tile border, the name text, the rarity label, and the tint of the cube-burst VFX played
on purchase. It is defined once in `FVoxelSkinRarity::GetAccent`.

### 1.2 The identity axis — why this is what makes the DLC tractable

Legend and Ultimate skins are not 70 and 21 unrelated things. They are **10 Legend identities** and
**3 Ultimate identities**, each rendered once per catalogue:

> Aurora, Dragonfire, Starfall, Thunder, Nebula, Prism, Phoenix, Void, Chrono, Celestial
> — and NIGHTCOST, Cubiciousflage, OMEGA.

So "Aurora" is a set of six skins: Aurora Shogun (Male), Aurora Empress (Female), Aurora Lance
(Rifle), Aurora Bloom (Grenade), Aurora Grasp (Physics), Aurora Ray (Laser).

An identity also carries a **material**, an **aura** and a **footstep**, all gifted together - see 4.3.

This is the reason the Steam plan in section 5 works. **One identity = one DLC appid = six skins plus
their material and effects.** Ten Legend DLCs and three Ultimate DLCs is a manageable store page.
Sixty and eighteen separate SKUs is not.

---

## 2. The original 318 skins and 53 CubeKin

Common skins are owned from the start (price 0) and are what a fresh save equips.

The Male, Female and four weapon lists below are unchanged from the original design; only the Cube
catalogue was dropped.

### 2.1 Male

**Common (3)** - Standard Issue - Denim Grey - Work Crew

**Rare (20, 150g)** - Track Runner - Hoodie Night - Varsity Blue - Diver Suit - Ranger Green -
Postal Orange - Chef Whites - Mechanic Grease - Surf Break - Snowfield - Desert Patrol -
Pixel Tourist - Referee - Lifeguard - Courier Red - Ice Fisher - Camp Counselor - Grid Worker -
Rooftop Runner - Subway Busker

**Super Rare (17, 400g)** - Neon Sensei - Circuit Samurai - Vaporwave Prince - Bladerunner Coat -
Chrome Boxer - Hologram Detective - Static Monk - Skyline Ronin - Toxic Racer - Plasma Coach -
Midnight Fencer - Arcade Champion - Glitch Gambler - Solar Nomad - Voltage Punk - Obsidian Envoy -
Crimson Vanguard

**Legend (10, 1200g or DLC)** - Aurora Shogun - Dragonfire Cube - Starfall Paladin - Thunderking -
Nebula Ronin - Prism Sovereign - Phoenix Ascendant - Void Emperor - Chrono Warden - Celestial Titan

**Ultimate (3, DLC only)** - NIGHTCOST Zero - Cubiciousflage Prime - OMEGA Architect

### 2.2 Female

**Common (3)** - Standard Issue - Denim Dusk - Field Crew

**Rare (20, 150g)** - Track Sprinter - Hoodie Dawn - Varsity Rose - Diver Marine - Ranger Fern -
Postal Amber - Patisserie - Engineer Ash - Surf Coral - Snowdrift - Dune Scout - Pixel Traveler -
Umpire - Lifeguard Sun - Courier Magenta - Frost Angler - Camp Guide - Grid Technician -
Rooftop Sprinter - Subway Violinist

**Super Rare (17, 400g)** - Neon Kunoichi - Circuit Maiden - Vaporwave Idol - Bladerunner Trench -
Chrome Duelist - Hologram Diva - Static Priestess - Skyline Assassin - Toxic Pilot - Plasma Captain -
Midnight Ballerina - Arcade Queen - Glitch Fortune - Solar Wanderer - Voltage Riot - Obsidian Herald -
Crimson Lancer

**Legend (10, 1200g or DLC)** - Aurora Empress - Dragonfire Sakura - Starfall Valkyrie - Stormqueen -
Nebula Kitsune - Prism Matriarch - Phoenix Reborn - Void Sovereign - Chrono Oracle - Celestial Seraph

**Ultimate (3, DLC only)** - NIGHTCOST Nova - Cubiciousflage Diva - OMEGA Muse

### 2.3 Rifle

**Common (3)** — Classic · Field Grey · Sandline

**Rare (20, 150g)** — Arcade · Elite · Woodland · Urban Splinter · Arctic · Desert Tan · Naval Blue ·
Racing Stripe · Copper Works · Riot Yellow · Carbon Line · Jungle Fern · Volcanic Ash · Steel Rain ·
Ranger · Marksman · Bluecap · Redwood · Nightwatch · Hazard

**Super Rare (17, 400g)** — Neon Recoil · Circuit Breaker · Vaporwave Volley · Holo Sight ·
Chrome Kick · Static Burst · Skyline Sniper · Toxic Spray · Plasma Feed · Midnight Ripper ·
Arcade Blaster · Glitch Round · Solar Flare · Voltage Coil · Obsidian Barrel · Crimson Fang · Prismshot

**Legend (10, 1200g or DLC)** — Aurora Lance · Dragonfire Repeater · Starfall Rail · Thunderclap ·
Nebula Storm · Prism Eye · Phoenix Roar · Void Piercer · Chrono Burst · Celestial Judgement

**Ultimate (3, DLC only)** — NIGHTCOST Verdict · Cubiciousflage Goldbore · OMEGA Rail

### 2.4 Grenade Launcher

**Common (3)** — Classic · Olive Tube · Rust Drum

**Rare (20, 150g)** — Arcade · Elite · Woodland Drum · Urban Thumper · Arctic Lob · Desert Mortar ·
Naval Popper · Racing Barrel · Copper Kettle · Riot Cannon · Carbon Drum · Jungle Boomer ·
Volcanic Pot · Steel Hopper · Ranger Lob · Bombardier · Bluecap Drum · Redwood Thumper ·
Nightwatch Mortar · Hazard Drum

**Super Rare (17, 400g)** — Neon Boom · Circuit Mortar · Vaporwave Lob · Holo Arc · Chrome Kettle ·
Static Shell · Skyline Bombard · Toxic Canister · Plasma Pod · Midnight Howler · Arcade Popper ·
Glitch Shell · Solar Burst · Voltage Drum · Obsidian Mortar · Crimson Shell · Prism Arc

**Legend (10, 1200g or DLC)** — Aurora Bloom · Dragonfire Mortar · Starfall Shell · Thunder Drum ·
Nebula Bloom · Prism Detonator · Phoenix Egg · Void Collapse · Chrono Charge · Celestial Bombard

**Ultimate (3, DLC only)** — NIGHTCOST Doomdrum · Cubiciousflage Goldshell · OMEGA Bloom

### 2.5 Physics Launcher

**Common (3)** — Classic · Grey Prongs · Iron Fork

**Rare (20, 150g)** — Arcade · Elite · Woodland Fork · Urban Prongs · Arctic Tine · Desert Rake ·
Naval Claw · Racing Prong · Copper Tine · Riot Grabber · Carbon Fork · Jungle Claw · Volcanic Tine ·
Steel Jaw · Ranger Fork · Wrecker · Bluecap Claw · Redwood Fork · Nightwatch Prong · Hazard Jaw

**Super Rare (17, 400g)** — Neon Kinetic · Circuit Puller · Vaporwave Shove · Holo Grip ·
Chrome Jaws · Static Tug · Skyline Wrecker · Toxic Grip · Plasma Tine · Midnight Claw · Arcade Slam ·
Glitch Pull · Solar Shove · Voltage Jaws · Obsidian Claw · Crimson Talon · Prism Fork

**Legend (10, 1200g or DLC)** — Aurora Grasp · Dragonfire Talon · Starfall Impact · Thunder Grip ·
Nebula Pull · Prism Kinetic · Phoenix Claw · Void Ripper · Chrono Shove · Celestial Wrecker

**Ultimate (3, DLC only)** — NIGHTCOST Shatterhand · Cubiciousflage Goldclaw · OMEGA Kinetic

### 2.6 Laser

**Common (3)** — Classic · White Lens · Grey Emitter

**Rare (20, 150g)** — Arcade · Elite · Woodland Emitter · Urban Beam · Arctic Lens · Desert Ray ·
Naval Beam · Racing Lens · Copper Emitter · Riot Ray · Carbon Lens · Jungle Beam · Volcanic Ray ·
Steel Lens · Ranger Beam · Cutter · Bluecap Beam · Redwood Ray · Nightwatch Lens · Hazard Beam

**Super Rare (17, 400g)** — Neon Cutter · Circuit Beam · Vaporwave Ray · Holo Lens · Chrome Emitter ·
Static Ray · Skyline Cutter · Toxic Beam · Plasma Lens · Midnight Ray · Arcade Beam · Glitch Lens ·
Solar Ray · Voltage Beam · Obsidian Lens · Crimson Cutter · Prism Beam

**Legend (10, 1200g or DLC)** — Aurora Ray · Dragonfire Beam · Starfall Lance · Thunder Lens ·
Nebula Ray · Prism Divider · Phoenix Beam · Void Lens · Chrono Ray · Celestial Cutter

**Ultimate (3, DLC only)** — NIGHTCOST Eclipse · Cubiciousflage Goldbeam · OMEGA Ray

---

## 3. What a skin actually is

A skin is **not** a static or skeletal mesh. Weapon and legacy character skins are a recolour plus a
surface treatment applied to an existing design. A CubeKin character skin may additionally select an
immutable code-authored voxel sculpture; that sculpture is still ordinary `FVoxelRuntimeDesignData`
and uses the same save, replication, damage and material paths:

```
FVoxelSkinDef
    Id              "Skin.Male.Legend.Aurora"     stable, save-safe
    Catalogue       Male | Female | Rifle | Grenade | Physics | Laser
    Rarity          Common | Rare | SuperRare | Legend | Ultimate
    Identity        None | Aurora | ... | Omega     (Legend/Ultimate only)
    DisplayName     "Aurora Shogun"
    PriceGold       0 | 150 | 400 | 1200 | -1 (DLC only)
    DlcAppId        0, or the Steam appid that also unlocks it
    Ramp            FVoxelSkinRamp   -- 4 palette slots: Body, Accent, Dark, Glow
    Surface         EVoxelSkinMaterial     -- which MIC the voxel asset is given
    Fx              EVoxelSkinFx           -- overlay material; NOT BUILT, see section 9
    Geometry        EVoxelCharacterGeometry -- optional CubeKin sparse sculpture
    AmbientVfx      EVoxelVfx              -- retired; effects are Niagara, see 3.2
    GrantsMaterials TArray<EVoxelSkinMaterial>   -- the gift bundle, §4.3
```

The **Ramp** is four palette slot indices, resolved through `VoxelPaletteSpectrum` — the same single
palette source everything else uses. A skin never carries raw colours, because *rendered colour comes
from the palette texture, not from stored colour values* (`PROJECT-VoxelPropHunt.md` § Palette /
Rendering). `ApplySkinToDesign` rewrites the slot indices of the design's Body / Accent / Dark voxels
and never changes geometry. `BuildCharacterGeometry` is the separate CubeKin geometry stage; because
that stage can change voxel count, bounds, capsule fit and replicated payload size, every geometry row
requires the explicit invariant and fairness tests defined in `DESIGN-CubeKin-Characters.md`.

### 3.1 Surfaces — 58 materials harvested from the purchased packs

`UVoxelMeshAsset::VoxelMaterialInstance` is a `UMaterialInstanceConstant`. A MIC cannot be given
parameter overrides at runtime, and every route around that races the plugin's chunk building -
`VoxelCubeMaterial.h` records the five attempts that proved it. So the surfaces are **assets on disk**,
generated by the editor command `VoxelBuildSkinMaterials`, and a skin picks one **by pointer** at the
moment `ConfigureAsset` already runs. There is no window in which the wrong material is on screen.

**One master, 58 instances.** `Matte` has no instance at all - it *is* `MI_VoxelCube`, which is what
keeps "no skin" and "Matte skin" the same code path. `Gloss` is free. The other 56 are premium.

#### The packs cannot be used directly, and that is the central constraint

The purchased materials (CrystalVFX3, Stylized_LavaV1, GeometryMaterial, GlitterShader) are ordinary
PBR / Substrate materials. **None of them sample the palette texture through vertex-colour red**, which
is how every colour in this game is decided. Assigning one to a voxel body renders the whole model in a
single flat colour and discards everything the player painted.

What is reusable is their **art**, and their **techniques**:

- **Textures** are harvested into `M_VoxelCubeSkin`'s triplanar slots. 12 crystal families, 14 lava
  sets, 11 geometric grids, 10 noise/stone, 4 specials.
- **Techniques** are reimplemented in HLSL, read off their node graphs: `BumpOffset` parallax,
  `Fresnel` rim, thin-film iridescence, `Panner` scrolling. GlitterShader ships **zero textures** - its
  effect is a MaterialFunction - so glitter is implemented, not bound.

#### Triplanar, because voxel chunks have no usable UVs

The plugin builds chunk UVs for the palette lookup, not for surface detail. So every texture is
projected from **object-local position** on all three axes and blended by the face normal. Local space
and not world, for the same reason the base cube mask uses it: a body that moves and turns would
otherwise swim through its own texture.

Normals use **whiteout blending** - the cheapest triplanar normal that produces no seams, and seams on
a cube would be unmissable. Normals are output in **world space** (`bTangentSpaceNormal = false`),
because there is no meaningful tangent frame to build.

#### The five map slots

| Slot | Parameter | Purpose |
| --- | --- | --- |
| Colour | `DetailTexture` | albedo pattern |
| Normal | `DetailNormalTexture` | relief - this is what makes a material look authored rather than painted |
| Mask | `MaskTexture` | the glow image; **is** the emissive, see below |
| Roughness | `RoughnessTexture` | per-pixel roughness |
| — | `DetailTintScale` | scales the colour map before it is blended |

A material may bind any subset. Crystal binds only a normal, keeping the player's colour and adding
pure relief; lava binds all four.

#### Two lessons that cost five rounds each

**1. Emissive and albedo are not the same pixels.** A lava texture's bright regions are bright because
they *emit*. Putting the same pixels in the albedo *and* the emissive double-counts them: the surface
is lit as white rock and then has glow added, saturating to white at **any** emissive strength.
Lowering the strength only ever produced a dimmer white. The glow is now **subtracted from the albedo**
(`Albedo *= 1 - GlowLum`), so the two sum to the authored image instead of each being it.

**2. A pack's BaseColor is not our albedo.** A stylized lava BaseColor is a picture of *glowing* rock -
bright across its whole area, because in the source material the brightness is meant to come from the
albedo. Blending 80% of that in made the body white **before any emissive maths ran**. `DetailTintScale`
(0.16 for lava) turns the same art into dark crust. This was the actual root cause of "magma is white",
after five rounds of fixing the emissive - which was working the whole time.

There is a third, smaller one: the contrast multiply that fakes relief on mapless materials reaches
1.75, and on a material whose albedo is authored art it clips to white. It now fades out in proportion
to how much a texture supplies the colour, and the albedo is clamped afterwards.

#### Palette-driven hue

`PaletteTint` (1.0 on lava) makes the texture contribute only its **luminance** and takes the hue from
the player's voxel colour - so a green torso is green crust with green-hot cracks, and the model the
player built stays legible. Without it every lava body was the same orange.

#### The technique dials

All default to **off**, so a material that does not ask pays nothing - every branch is uniform per
instance.

| Parameter | Effect |
| --- | --- |
| `ParallaxDepth` | BumpOffset. The single biggest reason the packs' crystals read as deep. |
| `FresnelBoost` / `FresnelPower` | Grazing-angle brightening. Reads under flat lobby lighting, where roughness alone says nothing. |
| `Iridescence` | Thin-film hue shift by view angle, as a three-cosine palette. |
| `PanSpeed` | Time-scrolled UVs. On lava, the difference between a picture of lava and lava. |
| `GlitterAmount` / `GlitterDensity` / `GlitterColour` | See below. |

#### Glitter

Round flakes on the face plane - a 3D cell with a hard step produces *cubes*, and cubes on a cube read
as a checkerboard. **Inside each disc is the player's voxel colour; the metal is the surface between
them.** The first version had this reversed, which read as glitter sprinkled *on* the model; inverted,
the metal is the setting and each disc is a window onto the build underneath. The specular highlight
rides on the metal, not the discs, or it would put a sheen over the voxel colour and muddy exactly what
the swap reveals.

Six variants, differing by background metal: black (Fine), dark gold (Coarse), gold (Gold), pale ice
(Frost), near-black violet (Nebula), near-black (Diamond).

### 3.2 Effects — 65 authored Niagara systems, two kinds worn together

Effects come from two packs and are **two different kinds**, worn at the same time:

| Kind | Source | Count | Trigger |
| --- | --- | --- | --- |
| Footstep | MagicalFootstepFX | 22 (16 footprints, 6 glyphs) | distance walked, ~90 uu apart, ground-placed |
| Aura | NiagaraDashEffects | 43 | attached to the body, continuous |

**Two save slots, not one.** A footstep is left on the ground behind you and an aura plays around your
body - they occupy different space, so there is no design reason wearing one should cost the other. The
reader also rejects mismatched kinds, so an aura saved before the split cannot come back in the
footstep slot and play at ankle height on a distance timer.

**The auras work unedited, via a hidden driver mesh.** All 43 sample an emitter-scoped
`Emitter_SkeletalMesh` data interface, which resolves from the owning actor. `AVoxelPropCharacter`
already carried a hidden `USkeletalMeshComponent` with no mesh; `SK_Mannequin` ships inside the dash
pack itself - the very mesh they were authored against. It is bound as a **shape source, never drawn**,
with no animation blueprint and no pose ticking: the data interface wants triangles and a transform,
not a walk cycle.

**Names are derived from content, not invented.** Each `NS_Dash_*` asset was read for its imports and
named from what it actually references - `SM_Crow` plus fire trails became *Crow Flight*, `SM_Coin`
became *Coin Rush*, `M_Lightning`/`M_ThunderTrail` became *Thunderstrike*, `SM_Skull` plus ghost trails
became *Skullfire*, and so on through Butterfly Drift, Frostspike, Sandstorm, Inferno, Arcane Bloom,
Wraith and Boulder Dash.

Paths live in `FVoxelSkinCatalog::GetEffectNiagaraPath` rather than as 65 new `EVoxelVfx` rows. The
enum earns its keep for *gameplay* effects, where a call site names an intent and the library decides
what it is made of; these are fully authored assets, and the catalogue is already the naming layer.
`UVoxelVfxSubsystem::PlayNiagaraAt` / `AttachNiagara` take a path and reuse the same resolve cache,
tint and per-frame budget.

**Suppressed for props during Hunting.** A prop trailing particles is not hiding. Gameplay rule, not a
client preference.

---

## 4. Materials and effects as owned items

### 4.1 What they are

A skin is one outfit; a material and an effect are **tools**, usable on any design the player authors.
That is what makes them worth selling separately.

### 4.2 Where they live

- **Shop → Materials tab** — 800 gold each, deliberately under the 1200 Legend that gifts one, so the
  gift bundle stays the better deal.
- **Shop → Effects tab**, with **Footsteps** / **Aura** sub-buttons — the two kinds fill different
  slots, so a single grid gave no way to tell which one a tile would replace.
- **Cube Builder** — `MATERIAL`, `AURA` and `EFFECT` dropdowns. This is where they are *used*.
- **Levels window → Materials category** — a display of what you own.

**Owned items leave the shop**, exactly as skins do; a tab with nothing left says so rather than
showing an empty grid. This only became safe once equipping moved into the Cube Builder - while the
shop was the only place to wear an effect, removing a bought one took away the only way to equip it.

### 4.3 The gift bundle

Buying a Legend or Ultimate skin grants its **material, aura and footstep**. All three are matched by
theme: Dragonfire gets Inferno with Dinosaur steps, Thunder gets Thunderstrike with Bull steps,
Cubiciousflage gets Coin Rush with Smart Shoe steps, Void gets Wraith with the Villain glyph.

Ownership and DLC resolution check **both** effect slots. Checking only the aura left every pack owner
without the footstep they had also paid for.

Granting is idempotent and additive: a second Legend sharing a material grants nothing new and must not
fail, refund or warn.

---

## 4A. Shop and library: buying is not wearing

- **The shop lists only what you do not own** - skins, materials and effects alike. A tab with nothing
  left says so rather than showing an empty grid.
- **Buying never equips.** Kitting out means buying several things in a row, and auto-equipping each
  leaves you wearing whatever you clicked last.
- **Owned skins appear in the Characters and Weapons libraries**, Commons included - they are what a
  player reverts to.
- **One character tab**, following the equipped body, labelled "Characters".
- **The old built-in weapon skins are gone.** `BuildDefaultDesign` builds identical geometry for
  Classic / Arcade / Elite and varies only the palette - they were colour variants, and leaving them
  alongside the catalogue put two systems in charge of the same pixels. **One colour authority.**
- **Custom weapon designs are never repainted.** The player painted them; the catalogue skin supplies
  only the surface. Repainting made the weapon editor look broken, since every colour chosen in it was
  overwritten the moment the gun was drawn.
- **A skin only repaints the body it belongs to**, and **equipping repaints the body already worn**
  rather than reapplying a preset - a character the player built has to survive buying a shirt.
- **Equipping a skin adopts its material**, clearing any override. The override is a per-skin
  deviation, not a permanent setting; left sticky, every skin after the first looked identical.

Previews are the real voxel model, repainted through `ApplySkinToDesign` and rasterised by
`FVoxelObjectThumbnail`, cached against the skin id and trimmed to their occupied bounds first - the
rasteriser frames the *grid*, so an untrimmed preview is a postage stamp in an empty square.

### Save compatibility

`ActiveMaterialOverride` and `OwnedMaterials` store **raw uint8 indices**, so reordering the material
enum silently repoints every stored index at a different material - a player who picked Chrome comes
back wearing whatever now sits at that index, with nothing in the game able to tell. `MaterialSetVersion`
guards this: a mismatch clears the stored indices rather than honouring numbers that now mean something
else. **Bump it whenever the enum is reordered.**

---

## 5. Steam monetisation

### 5.1 How Steam DLC actually works, and what it means here

Steam DLC is **not** like an Unreal plugin or a chunked pak. A DLC is a separate **appid** that Valve
allocates, listed under your base app. Buying it adds a licence to the customer's account. That is all
it is — an ownership bit.

The DLC does **not** have to ship any files. It can, via its own depot, but it does not have to. And
for this project it should not: every skin is code-authored voxel data and generated materials that are
already in the base build. Shipping content in a DLC depot would mean re-uploading and re-cooking for
every skin change, and would let anyone read the manifest to see unreleased skins.

So: **all 371 definitions ship in the base game, always. DLC ownership only unlocks them.** The client asks
`USteamProApps::BIsSubscribedApp(AppId)`, and the skin flips from locked to owned. This is the standard
approach for cosmetic DLC and it is why the plan below is cheap to run.

It also means a determined player can patch the check out locally. That is acceptable for cosmetics in
a game with no competitive economy. Do not build anti-tamper for this; it is not worth the complexity,
and any real fix requires a server that this game does not have.

### 5.2 The SKUs

Base app is **5041030 (Cubiciousflage)**. Two entitlement-only DLC products contain no separate
payload; all multiplayer-visible assets remain in the base depot. The Deluxe Edition is a Steam
**Complete the Set** bundle, not a third DLC app.

| App ID | SKU | Contains |
| --- | --- | --- |
| 5102500 | Character Pack | 13 Legend/Ultimate identities across Male, Female and unified Character: 39 skins plus their gifts |
| 5102510 | Weapon Pack | 13 Legend/Ultimate identities across Rifle, Grenade, Physics and Laser: 52 skins plus their gifts |

The **Cubiciousflage Deluxe Edition** bundle contains the base-game package, Character Pack and Weapon
Pack. Together those packs unlock the entire premium shop: all included skins and their special effects
and materials. A base-game owner purchasing that Complete-the-Set bundle pays only for the two missing
DLC packages (with the configured bundle discount); Steam does not charge for the base game twice.

`UVoxelStoreSubsystem::IsSkinUnlockedByDlc` checks the skin catalogue's pack appid. There is no Deluxe
entitlement bit: a Deluxe buyer owns both underlying packs. Premium materials/effects resolve from the
skins that gift them. Gold are not included in any DLC or edition.

### 5.3 Gold — Inventory Service, not DLC

Selling currency as DLC is wrong: DLC is a permanent one-time licence, so a player could buy a 5000-gold
DLC exactly once. Currency must be repeatable and consumable.

The correct mechanism is the **Steam Inventory Service**, and SteamCore PRO exposes all of it:

| Need | SteamCore PRO call |
| --- | --- |
| Load the item definitions | `USteamInventory::LoadItemDefinitions()` |
| Fetch localised prices | `RequestPrices(callback)` then `GetItemsWithPrices` / `GetItemPrice` |
| Open Steam's purchase overlay | `StartPurchase(callback, ItemDefs, Quantities)` |
| Read what the player owns | `GetAllItems(handle)` → `GetResultItems(handle, items)` |
| Spend it | `ConsumeItem(result, instanceId, quantity)` |

Item definitions are authored as JSON on the Steamworks partner site (Inventory Service → Item
Definitions), typed `item` with `"price"` set. Four SKUs:

| Item def | Grants | Price |
| --- | --- | --- |
| 1001 Pouch of Gold | 1,000 | $1.99 |
| 1002 Sack of Gold | 5,500 | $9.99 |
| 1003 Chest of Gold | 12,000 | $19.99 |
| 1004 Vault of Gold | 30,000 | $44.99 |

Flow, all client-side, no backend server:

1. Player clicks a currency tile → `StartPurchase`. Steam's own overlay handles payment; the game never
   sees a card number and must never ask for one.
2. On callback success, `GetAllItems` → `GetResultItems`.
3. For each owned currency item, `ConsumeItem` it, and on success credit
   `UVoxelWeaponCosmeticSave::GoldCubes` by the item's grant amount and save.
4. **Credit only after `ConsumeItem` reports success**, never on the purchase callback alone. If the
   game is killed between purchase and consume, the item is still in the Steam inventory and step 2–3
   run again at next launch — so reconcile on every startup, not only after a purchase. That reconcile
   pass is what makes this crash-safe without a server.

`GoldCubes` is spendable currency and must **not** touch `TotalGoldEarned`, which drives rank. Buying
gold must never buy rank — that is a `PROJECT-VoxelPropHunt.md` rule and this is exactly the change
that would quietly break it.

Note that Steam Inventory Service requires the partner-site configuration to exist before any of these
calls return anything useful, and that Valve reviews the item definitions before they go live. Budget
for that; it is not instant.

### 5.4 What this needs from Valve, in order

1. The two DLC appids are allocated: Character `5102500` and Weapon `5102510`. Complete each store
   page, pricing, art, review and release checklist. Leave the mistakenly allocated `5102520`
   unreleased; do not reuse it as a Deluxe entitlement.
2. Each DLC appid must be attached to a **package**, and the package to the store item, or it cannot
   be bought or granted.
3. Steamworks → **Inventory Service** → enable, author the four item definitions, publish to sandbox,
   then to production after review.
4. Create a **Complete the Set** Steam bundle containing the base-game store package and both DLC store
   packages. Name and present it as `Cubiciousflage Deluxe Edition`; mention that it includes every
   shop skin plus the associated special effects and materials.
5. No new depots and no new uploads for any of it — the base build already contains the content.

---

## 6. Testing

### 6.1 The part that matters most: test without Steam at all

The great majority of this is game-side logic — ownership tables, gifting, tile states, material
application, save round-trips. None of that needs Steam, and waiting on partner-site review to test it
would be a mistake.

**Built.** Four Development-only console commands (`!UE_BUILD_SHIPPING`), alongside the existing
`VoxelSessionStatus` / `VoxelObjectList` family, in `VoxelStoreDiagnostics.cpp`:

| Command | Effect |
| --- | --- |
| `VoxelGrantDlc <character \| weapon \| deluxe \| all \| none \| clear \| AppId>` | Fakes DLC ownership in `UVoxelStoreSubsystem`. Every unlock path reads the subsystem, never `BIsSubscribedApp` directly, so this exercises the real code. `none` forces OFF in both directions, so the locked path can be tested on a machine that genuinely owns the DLC. |
| `VoxelGrantGold <Amount>` | Credits `GoldCubes` only, through `AddPurchasedGold`. Must stay visibly separate from `GrantGold`, which also moves `TotalGoldEarned` - using the wrong one is how rank silently inflates. |
| `VoxelSkinList [catalogue]` | Prints every skin with rarity, price, owned flag, DLC appid, surface and granted materials, plus tier counts and owned premium materials. Catches table mistakes a compile cannot: a Rare priced at 0, a Legend that gifts nothing, an identity present in six catalogues out of seven. |
| `VoxelEquipSkin <SkinId>` | Equips regardless of ownership, for looking at a skin without buying it. |
| `VoxelSkinStatus` | What resolves: master, how many surface instances are built vs missing, how many of the 65 Niagara systems load, what is equipped per catalogue, and whether the skeletal driver is present. |
| `VoxelSkinVerify [filter]` | Reads baked parameter values and bound texture names **back off the assets on disk**. Whatever this prints is what the renderer uses. |
| `VoxelForceMaterial <name>` | Binds a surface straight onto the local body, bypassing store, ownership, override and equipped skin, then reports what the asset ended up carrying. |

The last three exist because of a specific failure: five rounds of shader changes were made while the
real cause was elsewhere, and "it still looks wrong" cannot distinguish an unbuilt asset, a failed
load, a wrong texture in a slot, a stale material index, or a genuinely wrong shader. Each needs a
different fix and none is visible on screen.

**The lesson, recorded because it cost the most time in this whole system:** when something looks
wrong, read the asset on disk before changing the shader. `VoxelSkinVerify` answers in one pass what
screenshots cannot answer at all. The editor can also be driven headlessly to do it:

```
UnrealEditor-Cmd.exe Cubiciousflage.uproject -unattended -nosplash -nullrhi -abslog=V.log
    -ExecCmds="VoxelBuildSkinMaterials, VoxelSkinVerify Magma, quit"
```

One editor command must be run once before any surface renders:

    VoxelBuildSkinMaterials

Checklist these cover, all offline:

Checklist these cover, all offline:

Checklist these cover, all offline:

- A fresh save owns exactly 3 skins per catalogue and equips a Common.
- Buying at exactly the price leaves 0 gold; one short refuses and changes nothing.
- Buying a Legend grants its materials, and they appear in the Levels → Materials category.
- Buying a second Legend sharing a material is a no-op for that material, not an error.
- Ultimate tiles show "DLC ONLY" and no gold price, and cannot be bought with any amount of gold.
- `VoxelGrantDlc` flips them to owned; `none` flips them back and unequips anything now locked
  **without** falling back to a broken or empty design.
- Every one of the 14 surfaces renders on a player body and on a viewmodel, and survives being shot —
  a damaged chunk must keep the skin's material, not revert to `MI_VoxelCube` or go grey. This is the
  regression `ConfigureAsset` exists to prevent, and skins are a new way to reintroduce it.
- Legend ambient VFX stop for a prop when Hunting begins and resume at round end.
- Save round-trip: equip, quit, relaunch, still equipped.
- Buying does **not** equip, and the bought skin appears in the matching library window.
- A bought skin disappears from the shop grid and does not come back.
- Equipping a Cube skin while standing as a humanoid does not repaint the humanoid.
- Equipping a legacy paint-only skin over a **custom** character keeps its geometry. Equipping a
  CubeKin skin deliberately replaces it with that character's immutable sculpture, then saves and
  replicates the result through the same runtime-design path.
- A Matte-surfaced skin and no skin at all are indistinguishable. If they differ, `M_VoxelCubeSkin`
  has drifted from `M_VoxelCube` and the free Common skins will visibly alter every body.

### 6.2 Testing real DLC ownership

You do not have to buy your own DLC.

- As a Steamworks admin you can grant your own account a licence: **Steamworks → Users & Permissions →
  Manage Packages**, or from the app's page, add the DLC package to your account. Ownership then
  appears exactly as a customer's would, and `BIsSubscribedApp` returns true.
- The steam client must be running and the app launched **through Steam**, not from the archive folder.
  Shipping is the only config Steam accepts, and Shipping has no console — so `VoxelGrantDlc` is not
  available there. Any bug that only appears in Shipping has to be diagnosed through logging.
- Watch for the `steam_appid.txt` trap already documented in `PROJECT-VoxelPropHunt.md`: the plugin
  writes that file next to the executable, it makes Steam skip the ownership check, and a folder the
  game has already been run from must never be uploaded. It does not grant DLC, so it will not mask a
  DLC bug — but it will mask a *base ownership* bug.
- Test the negative case on a second Steam account that owns the base game and no DLC. This is the case
  that ships to most players and the one least likely to have been exercised.

### 6.3 Testing Inventory Service purchases

- Steamworks provides a **sandbox** for item definitions: publish the defs to sandbox first and point
  the client at it, so `StartPurchase` runs the whole flow without real money. Confirm the current
  sandbox switch against Valve's Inventory Service documentation before wiring it — the mechanism has
  changed over the years and it is not worth guessing.
- Force-quit the game between the purchase callback and `ConsumeItem`, then relaunch. The startup
  reconcile in §5.3 step 4 must find the unconsumed item and credit it exactly once. Run it twice; a
  double-credit here is the bug that costs real money to fix.
- Verify `TotalGoldEarned` is unchanged after a purchase, and that rank did not move.

### 6.4 What cannot be verified headlessly

Per the existing caveats: `-nullrhi` cannot validate the 14 material instances, and a successful build
proves nothing about whether the generated MICs made it into the package. After
`VoxelBuildSkinMaterials`, check the staged output for all 14 under `/Game/Cube/Materials`, and look at
a Chrome and a LavaCrust skin in a real packaged run. `BUILD SUCCESSFUL` does not mean the files you
wanted are in the build.

---

## 7. Economy check

Per catalogue, everything earnable costs 20x150 + 17x400 + 10x1200 = **21,800 gold**; across six
catalogues, **130,800**. Materials add 56 x 800 and effects are priced off the same rarity table.

At 5 gold per hunter kill and 10 per survival minute, capped at 100 per match, a good match is ~100
gold. Completing everything is a chase rather than a grind wall provided the *first* purchase comes
fast: a 150-gold Rare is roughly two matches, and the 100 starting gold puts a new player two thirds of
the way there before they play. That first purchase is the conversion moment and the numbers should
protect it.

---

## 8. Decisions

- **Legend skins are dual-path** - 1200 gold *or* the matching Character/Weapon Pack.
- **Common skins never appear in the shop**; they are owned from launch and shown in the libraries.
- **Owned skins, materials and effects all leave the shop.** Equipping happens in the libraries and the
  Cube Builder.
- **The Cube catalogue was removed** - a wardrobe for a failure-state body.
- **The procedural material set was replaced entirely** by texture-backed materials from the packs.
  Chrome, Camouflage, Polka Dot and the rest were arithmetic standing in for art, and next to the packs
  they read as exactly that. This broke the append-only rule once, deliberately, guarded by
  `MaterialSetVersion`.
- **Footstep and aura are separate slots**, worn together.
- **Materials 800 gold**, under the 1200 Legend that gifts one.

**Open:**

1. **Premium surfaces during Hunting.** Ambient effects are suppressed for hidden props; should a
   Chrome finish be too? Arguably the player's own choice, but it may need a balance pass.
2. **Regional pricing** for the Gold tiers is Valve's suggested table by default.
3. **Grid materials share one roughness/metallic pair**, so Hex Shell and Torus Lattice read more alike
   than they should. Needs an eye, not a guess.

---

## 9. What is left

1. **FX overlay materials** - `M_SkinFxSheen`, `Pulse`, `Rgb`, `Prismatic`. `GetFxOverlayPath` names
   assets that do not exist; nothing crashes, the overlay simply never applies. This is the visible
   difference between a Super Rare and a Legend.
2. **Footstep / Aura sub-buttons in the Characters window.** The shop and the Cube Builder have them.
3. **Steam Inventory Service** for the Gold packs, including the startup reconcile in 5.3.
4. **Real DLC appids**, replacing the placeholders in `VoxelSkinCatalog.cpp`. One edit; do not scatter
   appid literals elsewhere.
5. **A lighting pass on the material numbers.** Roughness, metallic, parallax and fresnel were chosen
   by reasoning. They are all one table, `GetMaterialSurfaceParams` and `GetMaterialDetailTexture`, but
   tuning them needs eyes on a lit scene - a headless build cannot say anything about it.
