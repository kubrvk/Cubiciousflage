# Cubiciousflage Project Context

Evergreen AI context for Cubiciousflage, a UE 5.8 C++ voxel prop-hunt prototype.

## Identity

| Item | Value |
| --- | --- |
| Project | Cubiciousflage |
| `.uproject` | `Cubiciousflage.uproject` |
| Game target / exe | `Cubiciousflage` / `Cubiciousflage.exe` |
| Editor target | `CubiciousflageEditor` |
| Runtime module | `CubeCubeCube` |
| Editor module | `CubeCubeCubeEditor` |
| Engine | `D:\Program Files\UE_5.8` |

- Do **not** rename `CubeCubeCube`: `/Script/CubeCubeCube` is referenced by maps/assets in `Content/`.
- C++ only for gameplay/animation logic. No gameplay Blueprints or AnimBlueprints.

## Architecture Boundaries

| Path | Scope | Rules |
| --- | --- | --- |
| `Source/CubeCubeCube/VoxelRuntimeEditor/` | Runtime/game systems | Must package. May depend on plugin runtime module `VoxelEditor`; never `VoxelEditorEditor`. |
| `Source/CubeCubeCubeEditor/` | Editor tools | Import, conversion, baking, asset generation, saving packages, `FTextureSource`, `FMeshDescription`, content browser/menu tools. |
| `Plugins/VoxelEditor/` | Third-party voxel runtime/editor plugin | Do not modify for game features. If blocked, patch with `[Cubiciousflage patch NC-xxx]` and log in `Docs/PLUGIN_PATCHES.md`. |
| `Docs/DESIGN-MultiLevel.md` | Custom level rationale | Read before level architecture changes. |
| `Docs/DESIGN-Climp.md` | Climp mode contract and delivery order | Read before Climp gameplay or generation changes. |
| `Docs/HANDOFF-ClimpTower.md` | The generated tower: shape, libraries, proxies, streaming, pickups, lighthouses | Read before touching `VoxelClimpTower` or `AVoxelLevelDirector` streaming. It is the authority on the generator and carries the reasoning behind every decision in it, including the ones that were wrong first. `DESIGN-Climp.md` is the mode contract; this is how it is built. |
| `Docs/DESIGN-ObjectCatalog.md` | Object catalog design | Read before catalog/library changes. |
| `Docs/DESIGN-Cosmetics.md` | Cosmetics design | Read before store/skin/material/effect work. |

- `CubeCubeCube.Build.cs` exposes `CubeCubeCube/VoxelRuntimeEditor` publicly so `CubeCubeCubeEditor` can reuse runtime data types.
- Runtime deps now include `MeshDescription` and `StaticMeshDescription`: `UStaticMesh::BuildFromMeshDescriptions` is the only runtime-safe route to a multi-LOD static mesh, which is what lets distant-object proxies carry both detail levels and leave the choice to the renderer.
- **Two build targets.** `CubiciousflageEditor` is what headless verification loads; `Cubiciousflage` is what the game launches. Building only the first and then measuring through `UnrealEditor-Cmd.exe` proves nothing about the binary a player runs - that mistake cost most of a session.
- Runtime deps include `VoxelEditor`, `AssetRegistry`, `ImageCore`, `Slate`, `SlateCore`, `AppFramework`, `UMG`, `OnlineSubsystem`, `OnlineSubsystemUtils`, `Niagara`.
- Editor-only APIs stay in `CubeCubeCubeEditor`.

## Critical Runtime Voxel Rule

Plugin runtime is subtractive-only:

- `UVoxelComponent` delete/damage = sparse overrides on immutable shared template.
- No runtime plugin API adds voxels back.
- Plugin placement/painting lives in editor-only `VoxelEditorEditor`, unavailable in packaged builds.

Runtime sculpting path:

1. Own/create transient `UVoxelMeshAsset`.
2. Mutate dense voxel data.
3. Invalidate template cache.
4. Refresh `UVoxelComponent`.

Always after dense asset edit:

```cpp
FVoxelRuntimeTemplateCache::Invalidate(SculptAsset);
VoxelBody->RequestFullVoxelVisualRefresh(false);
```

Reason: game-world refresh uses cache with `bValidateSourceHash = false`; without invalidation, stale pre-edit templates can be reused and new voxels do not appear.

## Source Map

### Player Sculpt / Cube Builder

| File | Responsibility |
| --- | --- |
| `VoxelPropCharacter.h/.cpp` | Main pawn: voxel body, movement, camera, build mode, input, capsule fit, HUD, weapons, lobby/match gates, nameplates, post-process. |
| `VoxelSculptComponent.h/.cpp` | Core editable sculpt: transient asset, 256 palette slots, GPU palette updates, voxel mutations, sparse payloads, grid raycasts, save/load, bulk ops, replication publish. |
| `VoxelRuntimeEditorComponent.h/.cpp` | Cube Builder state: cursor, tools, drag/two-click previews, mirror, presets, save/load UI, panel lifetime. |
| `SVoxelRuntimeEditorPanel.h/.cpp` | In-game Cube Builder Slate panel. |
| `VoxelRuntimeEditTypes.h` | Tool/shape/mirror enums, design payloads, `USaveGame` classes. |
| `VoxelPresetShapes.h/.cpp` | Code-authored starting models/templates. |
| `VoxelDesignLibrary.h/.cpp` | Data asset for authored starting designs. |
| `VoxelRuntimeCanvas.h/.cpp` | Editable level prop/workbench case. |

### Palette / Materials / Object Catalog

| File | Responsibility |
| --- | --- |
| `VoxelPaletteSpectrum.h/.cpp` | Single palette source: 24 hue bands x 8 tones + 64 greys, named colors, Oklab quantizer, donor ramp writer. |
| `VoxelCubeMaterial.h/.cpp` | Runtime cube material apply, generated material usage, PSO warm-up, `VoxelCubeTexture` toggle, skin material binding. |
| `VoxelObjectScript.h/.cpp` | Object script format and `FVoxelObjectBuilder`: box, shell, erase, sphere, cylinder, line, mirror. |
| `VoxelObjectCatalog.h/.cpp` | `FVoxelCatalogEntry`, `UVoxelObjectLibrary`, save classes, `UVoxelObjectCatalogSubsystem`. |
| `VoxelObjectCatalogDefaults.h/.cpp` | General-kit built-in objects. |
| `VoxelObjectCatalogThemes.h/.cpp` | Theme registry in base-map order. |
| `VoxelTheme*.cpp` | Theme object scripts and hideable/clutter/backdrop entries. |
| `VoxelObjectCatalogDiagnostics.cpp` | Catalog/palette console diagnostics. |
| `VoxelObjectThumbnail.h/.cpp` | Code-rasterized thumbnails cached by object id. |
| `SVoxelObjectPicker.h/.cpp` | Full-screen add-object browser with search/category filter. |

### Custom Levels

| File | Responsibility |
| --- | --- |
| `VoxelLevelTypes.h/.cpp` | `FVoxelLevelObjectDef`, `FVoxelLevelPlacement`, `FVoxelLevelMarker`, `FVoxelLevelData`, save classes, `UVoxelLevelAsset`, `VoxelLevelVoxelSize`. |
| `VoxelLevelSubsystem.h/.cpp` | Save/load/index/delete, current selection, lobby override, preset seeding, shared asset cache by object def. |
| `VoxelLevelDirector.h/.cpp` | Builds level payload into world actors; owns level editing ops and placement transforms. |
| `VoxelLevelObject.h/.cpp` | One placed instance; thin `AVoxelActor` pointing at shared asset. |
| `VoxelLevelPresets.h/.cpp` | Code-authored starting levels. |
| `VoxelLevelEditorPawn.h/.cpp` | Invisible flying level-editor pawn with separate controls and Cube Builder. |
| `VoxelLevelEditorComponent.h/.cpp` | Edit modes, placement cursor, ghost preview, hover, selection, undo/redo, panel lifetime. |
| `SVoxelLevelEditorPanel.h/.cpp` | Level editor UI: object library, spawn markers, save, confirm popups. |
| `VoxelPlacementGizmo.h/.cpp` | Three-axis move gizmo. |
| `VoxelLevelStress.cpp` | Level build/stress console diagnostics. |
| `VoxelMapLibrary.h/.cpp` | Authoritative travel-map to shipped voxel-level pairing. |

### Match / Lobby / Combat / Presentation

| File | Responsibility |
| --- | --- |
| `VoxelMatchTypes.h/.cpp` | Phase/team/result enums. |
| `VoxelPropGameMode.h/.cpp` | Server-only match rules, phases, teams, ready-up, conversion, bot spawn/elimination, scoring hooks. |
| `VoxelPropGameState.h/.cpp` | Replicated phase, clock, team counts, conversion threshold, client-visible rules. |
| `VoxelPropPlayerState.h/.cpp` | Replicated team, ready flag, integrity baseline, destroyed voxels, display integrity. |
| `VoxelPropBot.h/.cpp` | `AVoxelPropBotController`: prop-only cube bots, disguise choice, wander/hide behavior. |
| `VoxelMatchSpawnPoint.h/.cpp` | Lobby/Prop/Hunter spawn markers. |
| `VoxelMatchPlayerController.h/.cpp` | Match controller, pause menu, phase watch, end-round UI. |
| `SVoxelMatchHud.h/.cpp` | Phase/clock/role banner. |
| `SVoxelPartyRoster.h/.cpp` | Top-left roster, in both modes. Hunt: hunters/props split with per-prop integrity. Climp: one PARTY block of names, hidden below two players. Reads `AGameStateBase::PlayerArray`, not `AVoxelPropGameState` - `AVoxelClimpGameState` derives from the base directly, which is why the panel used to be invisible in Climp. |
| `SVoxelChatPanel.h/.cpp`, `VoxelChatPlayerController.h/.cpp` | Text chat and push-to-talk path. |
| `SVoxelRankHud.h/.cpp`, `VoxelLeaderboardSubsystem.h/.cpp` | Lifetime-gold rank HUD and Steam leaderboard. |
| `VoxelWeaponComponent.h/.cpp` | Loadout, server fire, ammo, reload, resupply. |
| `VoxelWeaponVisual.h/.cpp`, `VoxelWeaponCosmetics.h/.cpp` | Cube-built first-person viewmodel and weapon cosmetics. |
| `VoxelProjectile.h/.cpp` | Server-spawned replicated projectiles. |
| `VoxelWorldDamage.h/.cpp` | Damage router: player sculpt vs level prop. |
| `SVoxelWeaponHud.h/.cpp`, `SVoxelCrosshair.h/.cpp` | Weapon strip and first-person reticle. |
| `VoxelLobbyGameMode.h/.cpp`, `SVoxelMainMenu.h/.cpp` | Lobby behavior, start/join/map picker/settings/server browser/library/shop windows. |
| `VoxelClimpGameMode.h/.cpp`, `VoxelClimpGameState.h/.cpp` | Shared build-capable controller and authoritative Q/E placement/carry path, plus dedicated Climp loading, ten-stage run state, checkpoints, recovery/refills, distance-streamed pickup planning, and the per-floor achievement unlocks. The game state keeps the route tops, satellite tops, checkpoints and lighthouse volumes from the build that actually ran - read those rather than re-deriving the tower. |
| `VoxelBuildTypes.h`, `VoxelPropPlayerState.h/.cpp`, `VoxelClimpPlayerState.h/.cpp`, `SVoxelClimpHud.h/.cpp` | General replicated Block/Metal/Energy inventory, Climp pattern extension, shared bottom-left icon tool/resource stack, and Climp-only middle-right height rail. |
| `VoxelClimpCheckpoint.h/.cpp` | Server-authoritative stage flags, placed from the generator's own checkpoints. One lighthouse per floor: a trigger inside the building, a floor-tinted barrier shell around it, an entry sound and a VFX burst. **The floor-name plate over the roof is off** - see `bShowFloorNamePlate`; the component and all of its placement survive behind that flag, so restoring the signs is one value. Which floor a climber is on is carried by the shell colour, the height rail and the arrival banner instead. |
| `VoxelClimpRoute.h/.cpp`, `VoxelClimpAmmoPickup.h/.cpp`, `VoxelClimpSkillPickup.h/.cpp` | Seeded reachable fallback platforms and the earlier always-resident pickup actors, kept for baked maps. The generated tower uses `VoxelClimpPickup` instead. |
| `VoxelClimpPickup.h/.cpp`, `VoxelClimpPickupDesigns.h/.cpp` | The tower's pickups: ammunition shaped like the weapon it feeds (from that weapon's Legend shop geometry), skill orbs, and the six ore-deposit definitions the level places as ordinary geometry. Taken with F or shot from range; distance-streamed by the game mode, never all resident. |
| `VoxelClimpRunSave.h/.cpp` | What a run has already spent - pickups taken, deposits mined - keyed on the run seed so the indices only mean anything within one tower. |
| `SVoxelClimpBuilderPanel.h/.cpp`, `VoxelClimpPlacedObject.h/.cpp` | Legacy pattern-panel compatibility plus replicated paint groups, seeded Rainbow color, strong direct-color Metal/Energy response, rainbow held overlay, connected movement, and total-lifetime LIFO decay. |
| `VoxelClimpTests.cpp` | Climp class wiring, tower shape and reachability, ore and ammunition design contracts, and the achievement ids. Expect **16 pass / 1 fail** from `Automation RunTests Cubiciousflage`; the failure is the pre-existing `Voxel.BaseMapRegistry` defect. |
| `VoxelClimpTower.h/.cpp` | The generated Climp tower: a seeded ten-turn cone of themed islands built as ordinary level data. Pure functions plus one `BuildAndApply` that knows where content lives. `RoundStampFootprint` cuts the four vertical corner edges off the lighthouse stamp before it is tinted, at `LighthouseCornerRounding` of its half-width - a per-VOXEL cut, because a placement is several metres of wall and dropping whole ones bites lumps out of the corners rather than rounding them. See `Docs/HANDOFF-ClimpTower.md`. |
| `VoxelDistantProxy.h/.cpp` | Coarse instanced silhouettes for objects past the streaming radius, so a 1 km climb is legible from the ground. Two detail levels in one static mesh; the renderer picks per instance. |
| `VoxelSessionSubsystem.h/.cpp` | Host/find/join/direct connect/quick matchmaking/travel URL, plus Steam invites: `InviteFriends` opens Steam's own friend picker through `IOnlineExternalUI::ShowInviteUI`, and `HandleInviteAccepted` answers both a friend's invite and Join Game from the Steam friends list. Binding that delegate is also what makes a cold-start invite work - SteamCore holds a `+connect_lobby` launch until something is listening. |
| `VoxelMapLibrary.h/.cpp` | Hostable map data asset + code defaults. |
| `VoxelUserSettings.h/.cpp`, `SVoxelSettingsPanel.h/.cpp` | Audio, controls, keybinds, startup, builder prefs, shared settings UI. The Audio page also carries the device controls below. |
| `VoxelAudioDevices.h/.cpp` | Which hardware the game talks to: microphone for voice chat, output endpoint for everything, and a voice-chat mute. Enumerates with `Audio::FAudioCapture` and `UAudioMixerBlueprintLibrary`; applies through `SwapAudioOutputDevice`, the `voice.MuteAudioEngineOutput` CVar, and plugin patch NC-005. A GameInstance subsystem, so the choices survive lobby -> match travel. Not `UVoxelAudioSubsystem`, which owns what the game sounds like rather than what it talks to. |
| `VoxelCursorRequests.h/.cpp` | Single mouse cursor owner/arbitrator. |
| `VoxelGameCues.h/.cpp`, `VoxelAudioSubsystem.h/.cpp` | Code-authored sound cue table and centralized sound playback. Climp carries `ClimpPickupAmmo`, `ClimpPickupSkill` and `ClimpOreMined` on top of the two checkpoint cues; ore taken whole shares the mined cue, because one substance should not have two voices. |
| `VoxelVfxLibrary.h/.cpp`, `VoxelVfxSubsystem.h/.cpp` | Code-authored effect table and centralized VFX playback. |
| `VoxelCubeBurst.h/.cpp`, `VoxelCubeBeam.h/.cpp` | Shipping-safe cube effects, debris, laser/tracer. |
| `VoxelUIStyle.h/.cpp` | Shared Slate style set. |
| `VoxelLoadingTips.h/.cpp`, `SVoxelLoadingScreen.h/.cpp`, `VoxelLoadingScreenSubsystem.h/.cpp`, `VoxelGameInstance.h/.cpp` | Travel/boot loading card, tips, MoviePlayer hold, fade flow, and bounded post-load runtime holds. |
| `VoxelFallRecoveryVolume.h/.cpp` | Fall recovery volume. |
| `VoxelAchievementSubsystem.h/.cpp` | SteamCore achievement/stat integration entry point. |

### Cosmetics / Shop

| File | Responsibility |
| --- | --- |
| `VoxelSkinCatalog.h/.cpp` | Catalog: 371 definitions (106 legacy character + 212 weapon + 53 unified CubeKin), 58 materials, 65 effects; migration, price, tier color, texture path, technique, Niagara path; `ApplySkinToDesign`. |
| `VoxelCharacterShapes.h/.cpp` | All 53 code-authored immutable CubeKin sculptures on a shared competitive occupancy mask. |
| `VoxelStoreSubsystem.h/.cpp` | Ownership, purchases, gifting, DLC resolution, equipped skin/material/footstep/aura; only authority for "can this player use this". |
| `VoxelStoreDiagnostics.cpp` | Development console commands. |
| `VoxelSkinMaterialBuilder.cpp` | Editor command `VoxelBuildSkinMaterials`; generates `M_VoxelCubeSkin` and 58 instances; shader HLSL is chunked. |

#### Weapon skin concept assets

The live weapon catalog contains **212 weapon skins**: 53 each for Rifle, Grenade Launcher,
Physics Launcher, and Laser. The visual concept pass is organized into 53 aligned theme families
and 19 bright shop-catalog sheets (Common 12, Rare 80, Super Rare 68, Legend 40, Ultimate 12).

Each skin also has an individual 1024 x 768 PNG export, using the exact live display name as its
filename under `Docs/Art/CubeKin/Weapons/AllSkins/Individual/<WeaponType>/<Rarity>/`. The complete
ID/name/theme/path index is `Docs/Art/CubeKin/Weapons/AllSkins/Individual/individual_weapon_pngs.json`.
The catalog sheets are in `Docs/Art/CubeKin/Weapons/AllSkins/CatalogSheets/`, and the reusable
extraction/prompt manifest is `Docs/Art/CubeKin/Weapons/AllSkins/weapon_skin_concepts.json`.

These PNGs are art references for the 2D-to-3D/voxel conversion pipeline; they do not replace the
runtime `FVoxelRuntimeDesignData` weapon designs until an individual conversion is approved and
integrated into `FVoxelWeaponCosmetics`.

### Editor Module

| File | Responsibility |
| --- | --- |
| `CubeCubeCubeEditorModule.h/.cpp` | Editor module entry + Tools > Voxel Objects menu; editor FPS cap on `OnPostEngineInit`. |
| `VoxelCubeMaterialBuilder.cpp` | `VoxelBuildCubeMaterial`; generates `M_VoxelCube` + `MI_VoxelCube`. |
| `VoxelMeshVoxelizer.h/.cpp` | Static mesh -> `FVoxelLevelObjectDef`. |
| `VoxelObjectImport.h/.cpp` | `.vox`, plugin grid JSON, object script import, object library writing, level bake. |
| `VoxelBulkConvert.h/.cpp` | Batch convert prop-suitable meshes from content folders. |
| `VoxelLevelMapConverter.h/.cpp`, `VoxelMeshVoxelizer.h/.cpp` | Convert source levels and static-mesh variants into shipped `UVoxelLevelAsset` payloads. |
| `VoxelPostProcessBuilder.cpp` | `VoxelBuildPostProcessMaterials`; legacy/generated PP material support. |

## System Rules

### The Currency Is Called Gold

Renamed from "Golden Cube" throughout the player-facing text on 2026-08-20. **Identifiers were not
renamed and must not be**: the Steam leaderboard API name is still `GoldenCube`, the achievement API
names are still `NC_GOLD_*`, and the C++ symbols (`RefreshGoldenCubeLeaderboard`, `FVoxelLeaderboardEntry::GoldenCubes`,
the `GoldCubes` save field) are unchanged. Those name things Valve and existing save files already
hold; the display name is the only part that was ever the player's.

The rename went through the localization pipeline rather than the source alone. A `LOCTEXT` is looked
up by namespace and key, so changing the English string in C++ changes nothing a player sees while a
compiled `.locres` still carries the old translation for that key - including the English one. Every
PO catalog was rewritten (msgid to the new source, msgstr to the literal word "Gold" in all 26
cultures) and `-run=GatherText -config=Config/Localization/Cubiciousflage.ini` re-imported and
recompiled them. **The currency name is deliberately untranslated**: it is a short proper noun now,
and several of the machine-translated forms were wrong anyway - Korean read `Mg를 가진`.

Still outstanding: three achievement descriptions name the currency in the **Steamworks partner
backend**, which no code change reaches. See `Docs/Steam/ACHIEVEMENTS.md`.

### The Climp Run Clock

`AVoxelClimpGameState` replicates `RunStartServerSeconds` and `RunEndServerSeconds` - two timestamps,
not a counting number, so a run clock costs no per-second property update per client.
`GetRunElapsedSeconds` derives the rest from `GetServerWorldTimeSeconds`, which is the server's clock
corrected for each client's offset: two players on one run read the same elapsed time rather than each
timing their own session. The clock starts with `InitializeRun` and freezes at the summit.

Drawn at the head of the Climp height rail by `SVoxelClimpHud::GetRunTimerText`, dimmer than the
height under it - a climber steers by metres and glances at the time. Collapsed where there is no
generated run, so a baked Climp map shows no clock rather than one stuck at zero.

### Climp Body Size, Pickup Sound and the Lighthouse Plan

**Body size is fixed in Climp.** `AVoxelPropCharacter::IsClimpMode` forces `MinimumCharacterScale` and
refuses `CanChangeCharacterScale`, which also removes the Cube Builder's size slider there. Not a
balance rule like the hunt's prop lock - the generator sized every ledge, gap and doorway on a
kilometre of procedural parkour against one body, so a larger character does not make the climb harder,
it makes parts of it impossible. Note the two locks pull in OPPOSITE directions: a round-active prop is
forced up to 1.0, a climber down to 0.5.

**Pickups announce themselves with a multicast, and mining with a client RPC.** These are the two
deliberate exceptions to `UVoxelAudioSubsystem`'s rule that each machine derives its own audio from an
event already on the wire:

- A pickup's only replicated event is the actor being destroyed, and the distance streaming destroys
  pickups just as often as taking them does - so playing on destroy would fire a collect chime for
  every crate that scrolled out of range. `AVoxelClimpPickup::Multicast_PlayTakenCue` is one unreliable
  RPC per pickup actually taken, of which a run has a few dozen.
- What a shot banked is a fact about the shooter's inventory, decided on the server from the damage
  report. `AVoxelClimpPlayerController::Client_PlayHarvestCue` goes to that player only, throttled
  server-side at `HarvestCueIntervalSeconds` so a held laser rattles rather than buzzes.

### Steam Invites and the Lobby Password

Sessions are created as Steam **lobbies** (`bUseLobbiesIfAvailable`, `bUsesPresence`,
`bAllowJoinViaPresence`, `bAllowInvites`), which with `bAllowJoinViaPresenceFriendsOnly` left false
makes them `k_ELobbyTypePublic` - invitable, and joinable from the Steam friends list.

An invite bypasses the lobby password on purpose. The invite carries the host's advertised settings,
so `TryConsumePendingInvite` reads the verifier off the invite and sends it as the join response. An
invite IS the permission: a host who sent one has already decided that player is allowed in.

**The password is currently a client-side gate only.** The host advertises `VOXELPASSWORDVERIFIER`
alongside the salt, and `AVoxelPropGameMode::PreLogin` compares the client's response against that
same advertised value - so anything that can see the session in the browser can read the verifier and
join without knowing the password. `JoinGameByIndex` refusing a wrong password is a courtesy, not
enforcement.

The fix is to advertise the salt only and let the server compare against its own verifier, which it
already holds from the travel URL. **That change and the invite bypass are coupled**: once the
verifier is private an invited player cannot derive it, so invites would then need a host-side
allowlist of Steam IDs - which in turn only covers invites the game sent itself, because the Steam
overlay's invite dialog reports nothing back about who was invited. Do not do one without deciding
about the other.

Invites need the game to be running **under the Steam client** with the overlay enabled.
`bRelaunchInSteam=False` means launching the exe directly does not bounce through Steam, which is
what makes local testing possible - and also why `ShowInviteUI` can succeed and display nothing on a
developer machine. Check the `Session status: OSS=...` log line: if it reads `NULL`, Steam never
initialised and no part of this applies.

### Player Sculpt

- Player voxel body is a component on `AVoxelPropCharacter`, not a separate actor.
- Pawn, hitbox, camera, match role, save identity stay together.
- Cursor picking uses Amanatides-Woo traversal in sculpt local grid space, not collision trace.
- Voxel body has no editing collision; capsule would block camera-to-voxel traces.
- Runtime sculpt size comes from `VoxelSculpt` component fields, not plugin Project Settings.
- UE units are cm; `16 x 16 x 18` at voxel size `10` reads roughly human-scale.
- Character size is presentation/collision scale, not a voxel-size or saved-design rewrite. The voxel
  body and fitted capsule scale together from `0.5` to `1.0`; authored cells remain 10 cm.
- Lobby characters and hunters use the locally saved preferred size (fresh-profile default `0.5`).
  Match props are always forced to `1.0`, while retaining the preference for a later role/map change.
- Save payload = sparse occupied cells + palette indices.
- Home character save slot: `__VoxelHomeCharacter`; save only outside matches.
- Named design index slot: `__VoxelDesignIndex`; `UGameplayStatics` cannot enumerate save slots.
- Player build is hitbox: refit capsule after edits.
- Capsule height is measured from grid floor so erasing base voxels does not drop actor.
- Capsule radius is measured about sculpt/grid center so asymmetry widens capsule instead of sliding body.

### Palette / Rendering

- Rendered color = vertex color red channel stores `PaletteIndex / 255`; material samples palette texture.
- `UVoxelMeshAsset::Palette` is not authoritative for rendered colors; palette texture is.
- Set `UVoxelMeshAsset::VoxelMaterialInstance` before geometry exists so plugin uses it during chunk builds.
- Runtime assets cannot author/parameterize required `UMaterialInstanceConstant`; editor creates MIC assets.
- Prefer generated palette/ramp with valid authored source over arbitrary donor palettes.
- Quantization must use the same effective generated palette that the renderer samples.
- Headless `-nullrhi` cannot validate RHI/platform texture data.
- `DefaultEngine.ini`: `r.PSOPrecache.ProxyCreationStrategy=0` avoids first-hit grey-material flashes when runtime damage creates mesh components before PSOs are ready.
- Cooked texture rule: writing CPU mip + `UpdateResource()` works in editor but cooked builds reload from pak and discard the write. `ApplyGeneratedRampToDonor` must also use `UpdateTextureRegions` and set `NeverStream`.
- Palette donor is auto-picked by asset registry when no `PaletteSourceAsset` is assigned and is only process-stable. Explicit donor assignment remains preferred.

### Object Catalog

- Reusable objects persist in `UVoxelObjectCatalogSubsystem`, not just level saves.
- `SaveWorkingLevel` may drop unplaced definitions from level payloads.
- Built-ins are code-authored script text, categorized by object identity and theme.
- Static mesh conversions are gameplay voxel silhouettes, not full-fidelity preservation.

### Custom Levels

- Level data = object definitions + placements + markers; never one giant editable grid.
- Large-grid sculpt invalidation rebuilds shared visual/collision meshes per edit and is too expensive.
- `UVoxelLevelSubsystem::GetOrCreate` caches shared mesh assets by `{Asset, VoxelSize}`.
- Many placements of one object share one asset/template; per-placement unique assets are explicit memory/rebuild cost.
- Author object geometry on workbench copy; commit at boundaries. Live-editing shared defs invalidates all instances per stroke.
- `UVoxelLevelAsset` bakes `FVoxelLevelData` into content for base-map dressing.
- A director's editor-only **Build Base Level** action compiles supported definitions into cooked voxel
  assets with pristine render/collision proxies. Runtime uses a proxy only while the stored content hash
  and voxel size match; actors retain sparse voxel state and the normal damage path replaces the proxy's
  affected cluster on the first edit. Oversized definitions and all custom levels safely use runtime generation.
- Director fallback baked asset path: `/Game/Cube/Levels/Level_<MapName>`.
- `DefaultGame.ini` cooks `/Game/Cube/Levels` because fallback loads by name.
- Level editor uses its own flying pawn; match controls and authoring controls are separate.
- Spawn markers use `FVoxelLevelMarker`.
- Custom levels carry one editable marker per role: `Lobby`, `Prop`, `Hunter`.
- Replacing/deleting existing markers requires in-game `SVoxelLevelEditorPanel` confirmation via `UVoxelLevelEditorComponent` pending actions.
- Creator/base-map authoring should isolate authoring space above the map, not hide real level.
- Base objects are read-only to players; deletion hides/removes placement, not source definition.
- Runtime level chunks are local implementation details, not replicated subobjects.
- `AVoxelLevelObject` is non-replicated; each peer builds chunks locally.
- Generated `RuntimeSharedCluster_*` / `RuntimeOverrideEditChunk_*` components must be non-replicated.
- Level-object voxel components become static mobility in `AVoxelLevelObject::BeginPlay`, not constructor, so preview/editor instances can be moved.
- Moving level objects at runtime must temporarily set Movable, move, then restore Static. Existing paths: `AVoxelLevelDirector::SetPlacementTransform`, level editor gizmo drag/duplicate.

### Match / Gameplay

- `AVoxelPropCharacter` is used in lobby and match maps.
- Client-safe match test: `AVoxelPropCharacter::IsInMatch` checks replicated `AVoxelPropGameState`, not `GameMode`.
- Match rules belong in server-only `AVoxelPropGameMode`.
- HUD visibility must be bound/reactive; client `GameState` can arrive after pawn/widget construction.
- Match-only UI: phase banner, party roster, hunters-only weapon strip.
- Combat, reticle, and viewmodel are not globally disabled in lobby.
- Conversion is health-based: props default to fixed 100 HP. Voxel destruction is visual feedback and does not scale health.
- At match start and hunt start, prop health resets independently from occupied voxel count.
- Every surviving prop starts the hiding phase as the built-in Dot: one occupied cube rendered at the
  prop role's full `1.0x` scale. Props build the disguise during hiding and must reach `1,000` occupied
  voxels before hunting begins; an undersized model is replaced with the built-in Cube.
- During hunt, props under `1,000` voxels get 30 s grace and a bottom-screen Cube Builder warning; still under the limit at expiry is replaced with the built-in Cube.
- Hunters under `1,000` voxels get the same 30 s grace. At expiry they restore the selected character
  design captured when they entered the match; if unavailable, they fall back to the first free base
  CubeKin. Prop death/conversion uses the same selected-character restore path.
- General custom character/hunter designs require at least `1,000` occupied voxels to save/keep after
  builder close; undersized local designs outside a match fall back to the selected/free base CubeKin.
- Player bodies use dense sculpt damage only and set `bCanBeDestroyed = false` on their voxel components. Letting plugin world damage also write sparse removals desynchronizes occupied count and creates invisible-but-occupied Cube Builder cells.
- Built-in "Start as" shapes are clean templates, not save slots. Applying one replaces dense data and clears sparse runtime damage/removal state.
- In match maps, prop model choice must be authored in builder, not loaded from saved character builds after round start.
- Match prop edits publish with short debounce while editor is open so hunters/server damage use current body. Outside matches, close remains normal commit point.
- Weapons have separate voxel-cell damage and match-health damage.
- Grenade projectiles use an authority-only hit/stop path with gravity sub-stepping and a collider
  matching the visible cluster. Their instant dense carve is `36 uu` (the earlier `140 uu` radius
  could erase a minimum prop's entire dense model behind a stale rendered shell); match damage is a
  separate 40-point payload. An empty sculpt explicitly hides its previous runtime representation.
- Display integrity = replicated health percent; reaches 0% on conversion.
- Party roster rebuilds only on join/leave/conversion; percentages are bound attributes; sort hunters, then props, then name.
- Ready-up auto-start is server-only:
  - 2+ players starts ReadyUp timers.
  - New join during ReadyUp restarts 30 s soft timer and 60 s force timer.
  - Soft timer starts if at least one player is ready.
  - Force timer starts even if nobody readies.
  - Unanimous ready starts immediately.
- Nameplates are `UTextRenderComponent`s on `AVoxelPropCharacter`, not viewport widgets.
- Waiting/ReadyUp: names visible. Active rounds: hunters always show names; props hide names unless server reveals for 3 s after valid hunter match damage.
- Rank progression uses lifetime earned gold (`TotalGoldEarned`), not current spendable `GoldCubes`.
- Rank spans: Bronze 100, Silver 200, Gold 300, Platinum 400, Diamond 500, Master 1000.
- Gold leaderboard: `UVoxelLeaderboardSubsystem` + SteamCore PRO `UserStats`; Steam leaderboard API name `GoldenCube`; upload `TotalGoldEarned` with `KeepBest`. **The API name keeps the old spelling on purpose** - it identifies the board Valve already holds scores on, and renaming it would orphan every posted score.
- Achievements: `UVoxelAchievementSubsystem` defines 40 curated Steam achievements. Thirty-seven
  use persistent Steam integer stats and repeated thresholds (no menu-open or single-input awards),
  while three use lifetime Gold totals. Steamworks 1.61+ synchronizes current-user stats before process launch;
  SteamCore PRO's deprecated `RequestCurrentStats()` wrapper is a no-op, so Cubiciousflage reads the
  synchronized state directly on subsystem initialization and menu refresh. An unlock calls
  `SetAchievement(ApiName)` followed by `StoreStats()`; achievement rewards remain manually claimed
  and are protected by Steam integer stats `NC_ACH_CLAIM_MASK_0` / `NC_ACH_CLAIM_MASK_1`.
- Steam progress achievements link their Client INT stat to the achievement in Steamworks. Runtime
  calls `IndicateAchievementProgress` at 25/50/75 percent, immediately stores completed unlocks, and
  performs an explicit client-side progress flush at match end; non-critical changes are debounced.
- The exact achievement IDs, thresholds, reward values, and required integer stats are maintained in
  `Docs/Steam/ACHIEVEMENTS.md`. Locked milestone rows show current/required Steam progress in-game.
- `VoxelExportShopSkinIcons [OutputDirectory] [Size]` exports deterministic shop preview geometry
  with colors resolved directly from the authored Cubiciousflage spectrum as square PNGs plus
  `ShopSkinIcons.csv`. Default output is
  `Docs/Steam/ShopSkinIcons`, default size is 256. It includes Character, Rifle, Grenade, Physics and
  Laser tabs (252 files: 52 character entries including two free bases, plus 50 per weapon).

### Cube Bots

- Bots are props only; no hunter AI.
- `AVoxelPropBotController` is `AController`, not `AAIController`; project has no navmesh.
- Bot movement: `AddMovementInput` plus wall/ledge line traces on 0.35 s timer.
- Bot controller calls `InitPlayerState()` itself; `AController` does not.
- Bot `AVoxelPropPlayerState` participates in team counts, roster, prop health, spawn moves, conversion paths.
- Bots count toward `MinPlayersToStart`.
- Bots do **not** count in first-hunter draft (`StartMatch` filters `IsABot`) or `GetHumanPlayerCount`.
- Bot at zero health is eliminated, not converted: `ConvertToHunter` branches into `EliminateBot`, multicasts shatter, destroys pawn/controller, win-checks next tick.
- `AController::Destroyed` calls `GameMode->Logout`; `Logout` must early-return for bots after recount or one-player/one-bot wins can be voided.
- Disguise chosen once at start of hiding after `MoveTeamToSpawnPoints`.
- Disguise priority: placed level objects within 3000 uu, widen to 12000 uu if fewer than 5, top up from theme-tagged catalog entries tagged `hide`, then built-in crate.
- Shared taken-id set avoids duplicate bot disguises in same round.
- Theme tags come from `VoxelObjectCatalogThemes`; avoid cross-theme disguises.
- Bot disguises must satisfy `MinimumPropVoxelCount` and stay under `MaxReplicatedVoxels`.
- Bots use `UVoxelSculptComponent::ServerApplyDesign` authority-side; no local publish path.
- Bots are not topped up during Hiding/Hunting.
- `AVoxelPropCharacter::bIsBotPawn` is set between spawn and `FinishSpawning` by `AVoxelPropGameMode::SpawnDefaultPawnAtTransform`; pawn `BeginPlay` runs inside spawn.
- Any "human on this screen" logic must check `bIsBotPawn`; `IsLocallyControlled()` is true for bot pawns on listen servers.

### Collision / Damage / Weapons

- Damage routes by target type via `FVoxelWorldDamage::ApplySphere`.
- Level props may use plugin damage/override behavior for content/disk-backed assets.
- Player bodies are transient dense sculpts; plugin sparse damage overrides are lost on next edit.
- Shots fire along camera ray.
- Crosshair only in first person; third-person screen center is not shot line.
- First-person muzzle/viewmodel moves to eye with `SetFirstPersonAim`.
- RMB aim and toggled first-person mode publish a persistent aim-presentation state. Observers see a
  separate owner-hidden voxel weapon aligned to the replicated view direction; the shooter keeps the
  existing owner-only camera viewmodel.
- Remote aim direction is unreliable at 20 Hz while active, interpolated by observers. Activation,
  deactivation, weapon/skin changes, and 0/16/32 uu forward-offset changes are reliable.
- The 20 Hz direction packet doubles as an authority heartbeat; a world gun expires after one second
  without it. Opening Cube Builder also sends an unconditional reliable deactivation immediately,
  preventing a stale observer weapon from surviving as a detached coloured slab.
- In FPS/RMB aim, the pawn continuously adopts controller yaw and suspends orient-to-movement, so
  observers see the character body and weapon face the same direction as the player's view.
- World-weapon placement begins at the occupied model centre, resolves overlap to the foremost
  occupied voxel along the aim axis, then adds 16 uu of separate forward clearance by default
  (configurable as 0/16/32). The support scan is cached and invalidated by body edits/damage
  or a material aim-direction change; it is not a per-frame 32^3 scan.
- Weapon visuals render at the authored 1x voxel scale. Their component origin is the occupied rear
  face and the occupied +X length defines the barrel tip; authoritative projectiles, remote muzzle
  flashes and beams use that same endpoint instead of capsule/eye-height approximations.
- Weapon events are server-authoritative.
- Presentation effects derive from replicated gameplay events, not extra cosmetic replication.

### Lobby / UI / Settings / Sessions / Cursor

- Lobby is playable home space using real `AVoxelPropCharacter`; customization edits match pawn class.
- Escape closes innermost UI or toggles cursor/input focus.
- Cursor states should remain `GameAndUI` where appropriate so WASD can continue while UI is visible.
- Use `VoxelCursorRequests` as the single cursor owner/arbitrator.
- Weapons are allowed in lobby for testing/feel.
- Lobby, Hunt and Climp share the complete bottom-left combat/build HUD: four primary weapons, three
  abilities, Q Paint, E Carry, material counts, and V/B/N build controls. Hunt hides only the four
  firearm tiles while the player is a Prop; both teams retain skills and Q/E. Authority assigns mode profiles of
  5,000/3,000/1,000 materials in Lobby, 1,000/500/300 in Hunt, and 300/200/100 in Climp.
- Hosting uses `UVoxelMapLibrary` map tiles when available; typed map field remains override/fallback.
- Host-selected match settings ride travel URL options.
- Main menu customization uses fixed-size child windows: `Characters`, `Weapons`, `Levels`, `Shop`.
- `SVoxelSettingsPanel` keeps a fixed 624-high body but uses a responsive 640 minimum / 760 maximum
  width in both the lobby and pause menu. Localized tabs and menu buttons use down-only scale boxes,
  so a long label grows its window where space permits and scales down only at the cap instead of
  clipping. The root menu likewise has a 344 minimum / 520 maximum responsive width.
- The language selector uses the light `SecondaryButton` surface inside its own one-pixel rainbow
  frame, plus a custom cream alternating-row, cyan-hover/selection table style and a capped scroll
  list. Its popup also has a coral outer frame and does not inherit Unreal's near-black editor combo.
  Rows are at least 34 pixels high with five-pixel vertical content padding and a 500-pixel list cap.
- Language is a saved machine preference (`UVoxelUserSettings::CultureName`) and is applied during
  `UVoxelGameInstance::Init` plus immediately when selected. The Settings Game tab exposes 23 staged
  cultures: English, French, German, European/Latin-American Spanish, Brazilian/European Portuguese,
  Italian, Dutch, Polish, Russian, Ukrainian, Turkish, Arabic, Indonesian, Vietnamese, Thai, Japanese,
  Korean, simplified/traditional Chinese, Czech and Swedish. Hebrew, Persian and Hindi were removed
  because they are absent from Steam's game-language list; old saved selections migrate to English. Every
  catalog currently covers all 805 gathered entries / 2,842 source words and compiles to its own
  `.locres`. The catalogs are machine-translation bootstraps and still require native-speaker
  editorial QA before release.
- The Cubiciousflage localization target is registered both through `[Internationalization]` and the game
  module's `GatherAdditionalLocResPathsCallback`. Culture changes are asynchronous inside Unreal;
  `UVoxelUserSettings::ApplyCulture` waits for the locres refresh before the current menu continues.
  Without both pieces the saved culture changed, but project menu `LOCTEXT`s remained English while
  only Engine-owned text such as key names appeared translated.
- Project dogica fonts use an explicitly owned Slate composite. Latin-script catalog output is
  intentionally ASCII-folded (`Magaza`, `Francais`, `Configuracion`) so it retains the dogica pixel
  face. Cyrillic, Arabic/Persian, Hebrew, Devanagari and Thai map to packaged Roboto/Noto faces, while
  CJK and any remaining glyphs use Droid Sans Fallback. The source `UFont` and every copied default
  `UFontFace` are held by strong references for the lifetime of the UI style. A standalone composite
  is not a UObject and does not otherwise report those face handles to GC; culture-change cache flushes
  therefore cannot leave Slate with dangling face data when a texture-heavy page such as Shop opens.
- `Scripts/Localization/generate_machine_translations.py` reproduces the Apache-2.0 OPUS-MT bootstrap,
  preserves formatting placeholders/newlines, applies the ASCII policy and deterministic core-menu
  glossary, and labels PO metadata as requiring native-speaker QA. Korean uses the dedicated
  English-to-Korean OPUS model because the multilingual model has no Korean target token.
- Library/shop grids use `SWrapBox`, not `SUniformGridPanel`.
- Target layout: six items per row for library/shop; four per row for Start Game level pickers.
- Characters/weapons/levels split base/shop entries from custom entries.
- Base levels: selectable/copyable, not directly editable, no Activate button in Levels library.
- Custom entries: edit/delete controls; destructive actions require confirmation.
- Character selection save slot: `__VoxelCharacterSelection`.
- The first two unified Character catalogue entries are permanent free base skins. They remain at the
  front of the Shop as direct Equip choices so the player's initial Builder character is selected
  explicitly; legacy Male/Female bodies remain migration data and are not selectable UI.
- Cube Builder model area shows last library-selected/equipped character thumbnail/name.
- Cube Builder model buttons are exactly `Reset`, `Cube`, and `Dot`. Cube/Dot are temporary editing
  templates and never replace the active character selection. Reset restores the last Library/Shop

### Initial character replication

- Simulated proxies never execute the local character fallback chain. In particular they must not
  reach `FillFloorLayer`: that one-cell layer was the large red square briefly shown for joining
  players and could persist when an earlier `OnRep_ReplicatedDesign` was overwritten in BeginPlay.
- `SetVoxelBody` creates a fresh transient asset. After binding it, non-authoring proxies explicitly
  rehydrate any authoritative design that arrived before BeginPlay; otherwise their body stays hidden
  until the first valid design arrives. `OnRep_ReplicatedDesign` unhides only after a successful apply.
- Listen-host characters publish their loaded design immediately. Remote server pawns and simulated
  client proxies remain hidden instead of inventing another player's body from local save/fallback data.

### Character motion presentation

- While first-person/RMB aim is active, the owning client sends its camera direction through the
  existing 20 Hz aim heartbeat. The server applies the flattened yaw to the authoritative character,
  so a stationary remote client turns on the listen host and on every other observer as well.
- Voxel breathing and gait are component-relative cosmetic offsets evaluated independently on each
  machine. Idle uses three low-amplitude, out-of-phase waves; walking adds alternating lateral and
  vertical steps, and sprinting increases their extent. No animation state or transform is replicated,
  and the capsule, dense voxel data, damage coordinates, and editor grid remain authoritative and still.
- Motion blends to zero while the Cube Builder is open. The voxel renderer combines cells into chunk
  meshes, so the bottom three cell layers cannot have independent transforms without splitting the
  body into separately rebuilt assets and breaking the single damage/edit coordinate space.

### Gameplay HUD and character presentation

- Persistent match information uses the same cream, coral, cyan and rainbow-framed visual language as
  the main menu. Phase/clock/readiness occupy the upper-left card; server-live and player/team rows are
  stacked directly below it. The centred round-result card uses the same rainbow frame, a pulsing
  local victory/defeat chip, result-specific summary, victory/defeat audio, and winner cube confetti.
- World nameplates use the project `dogicabold` font in white over a slightly larger black backing copy,
  creating a readable pixel-font outline without introducing a UMG widget or per-player screen overlay.
- The Cube Builder intercepts the configured Builder key during Slate preview routing, before focused
  buttons, lists or text fields can consume Tab for navigation. It still closes through the character's
  state machine, so a match prop below the safe voxel minimum is not accidentally released or locked.
- The two permanent free base Character skins have fixed voice identities: Rookie Builder is Male and
  Moss Scout is Female. The assignment is applied on equip and again on spawn to migrate old preferences.
  character from `__VoxelSelectedCharacter`, with the legacy custom selection and equipped base skin
  used only as migration/fresh-profile fallbacks.
- `Cube` is a neutral solid 10x10x10 volume: exactly 1,000 occupied voxels, all mapped to the
  project white palette slot and forced to the Matte surface.
- The Cube Builder panel's Library button lives ON the "Current character" tile (`OnLibraryClicked`,
  `SVoxelRuntimeEditorPanel.cpp`), not as its own top-bar button - and is `.IsFocusable(false)`, since a
  focused `SButton` was one of the things that made Tab-as-navigation (see `Do Not Break`) misbehave.
- The Cube Builder size slider sits directly below Templates, maps `0..1` to `0.5x..1.0x`, and is
  reactively collapsed for match props. Dragging updates the replicated body immediately and commits
  the local preference once at drag end rather than writing the save slot every slider tick.
- Opening the Cube Builder (`AVoxelPropCharacter::SetEditModeActive(true)`) keeps the numeric cube
  counter visible in both lobby and match, but hides the crosshair, match HUD (including messages),
  chat messages/input, party roster, and the weapon/prop-ability strip (`UVoxelWeaponComponent::
  SetWeaponHudVisible`), and both hides AND collapses the main menu (`AVoxelLobbyPlayerController::
  SetMenuFullyHiddenIn` + `CollapseMenuIn`) - collapsing, not just hiding, matters: hiding alone leaves
  whatever menu page was last open in place, so the next time the menu is shown it can reappear on a
  stale page (this was the actual cause of a "Tab reopens the Library instead of the builder" bug).
- Server-forced model changes close the Builder through the complete local teardown path. They do not
  publish the outgoing edit, and client design submits acknowledge the latest forced-design serial;
  stale pre-match/pre-conversion bodies are rejected instead of resurrecting ghost geometry.
- A keybind hint strip (`SVoxelKeybindHintBar.h/.cpp`) shows along the bottom-right of the screen while
  the Cube Builder or the level editor is open - transparent, no panel background, reads live rebound
  keys via `UVoxelUserSettings::GetControlKey` where the binding is rebindable.
- Start Game base/custom level pickers use level thumbnail pipeline and active border/tint.
- Selecting base map clears custom selection; selecting custom level clears typed/base-map selection.
- `SVoxelSettingsPanel` is shared by lobby and pause menu.
- Game prefs: first-person start, builder-on-lobby-spawn.
- Controls: mouse sensitivity, vertical-look inversion, runtime keybinds.
- Graphics: `UGameUserSettings`.
- Audio: `UVoxelUserSettings` + `UVoxelAudioSubsystem`; master/music/effects/voice/interface buses; subsystem multiplies master volume in playback path.
- Cute UI pack cues provide Cube Builder place/erase/paint/pick/open/close/save feedback plus primary
  click/back/invalid/panel/toggle transitions; hover keeps the quieter existing VAudio cue.
- Text chat is controller-owned through `AVoxelChatPlayerController`; Enter focus/submit, Escape closes, focus returns to game viewport.
- Voice push-to-talk: V calls `APlayerController::StartTalking` / `StopTalking`.
- Voice config: `[Voice] bEnabled=true`, `[OnlineSubsystem] bHasVoiceEnabled=true`, `GameSession.bRequiresPushToTalk=true`.
- If backend voice fails, wire same V path into Cross-Platform Voice Chat Pro rather than changing gameplay/UI call sites.

### Cosmetics

- Weapon skins and hidden legacy Male/Female rows are recolor + surface. Every live CubeKin Character skin names code-authored voxel geometry; it remains sparse `FVoxelRuntimeDesignData`, never a static/skeletal mesh.
- `BuildCharacterGeometry` selects immutable character geometry; `ApplySkinToDesign` remains the separate palette-only stage and never changes occupancy.
- Every geometry-bearing paid skin requires bounds, occupied-count, deterministic-build and competitive-exposure tests. Do not ship a smaller paid hit profile.
- Skin surfaces are on-disk `UMaterialInstanceConstant` assets selected by pointer in `FVoxelCubeMaterial::ConfigureAsset` before chunk build.
- `FVoxelCubeMaterial::Apply` must read material from the asset; it runs per frame from character and must not hardcode/pick a replacement.
- Purchased material packs cannot be directly assigned to voxel bodies; they do not sample palette texture through vertex-color red and render flat.
- Reusable pack data: textures/techniques.
- Emissive and albedo are separate. Do not put glow pixels in both; subtract glow from albedo.
- Source pack BaseColor is not project albedo; `DetailTintScale` compensates.
- `ActiveMaterialOverride` and `OwnedMaterials` store raw `uint8` indices. Reordering `EVoxelSkinMaterial` corrupts saves; bump `MaterialSetVersion`.
- Footstep and aura are separate slots worn together.
- 43 dash auras sample a skeletal-mesh data interface from owning actor.
- `AVoxelPropCharacter` has hidden `USkeletalMeshComponent` with `SK_Mannequin` only as aura shape source: never drawn, no AnimBlueprint, no pose tick.

### Presentation / Rendering

- Gameplay requests sound/VFX by stable cue/effect names only.
- Sound centralized in `UVoxelAudioSubsystem`; VFX centralized in `UVoxelVfxSubsystem`.
- Music uses one persistent component and a 2 s crossfade: `new_lobby` in lobby/pre-match,
  `new_PropHideCountDown` during Hiding, the original gameplay bed during Hunting/default states,
  `new_huntercatching` for one non-retriggering random 10-20 s section after valid prop damage, and
  a random `new_win` / `new_win2` track for the local winner. Losers return to gameplay music.
- Music assets are preloaded asynchronously and answer to the Music bus. A missing replacement is
  resolved before the current bed fades, so a renamed asset logs once without causing silence.
- Human voice assets come from ACV pack and are indexed separately from general cue table.
- Voice is an independent saved Cube Builder setting. Character presets and CubeKin skins never change it.
- Shipping visuals must not use debug draw; `DrawDebugLine/Box` compiles out in Shipping.
- Prefer project-owned cube-based effects/materials for Shipping compatibility.
- Player hit feedback: project-owned `M_PlayerHitRainbowOverlay`, square face-edge/corner flash, fast attack + longer fade via `HitGlowAlpha`.
- Player interaction feedback: `M_PlayerInteractionRainbowOverlay`, transparent body flash, held 0.5 s + faded 0.5 s via `InteractionGlowAlpha`.
- Prop Barrier uses cube-burst VFX plus a spherical mesh with the tested Voyager material
  `/Game/Cube/Mods/Voyager/Demo/Materials/M_Barrier_Inst` for the full duration.
- Barrier presentation is driven by replicated `PropBarrierActiveUntil` through its RepNotify. Do not
  move the visible shield back to an unreliable cosmetic multicast: losing that packet produced an
  active barrier with no rainbow overlay or opening burst.
- Regenerate rainbow assets with `VoxelBuildPlayerHitMaterials`; runtime loads from `/Game/Cube/Materials`.
- Camera post-process is set in C++ on `AVoxelPropCharacter::BeginPlay` and `AVoxelLevelEditorPawn::BeginPlay`.
- PP values:
  - `ColorSaturation=(1.06,1.06,1.06,1.0)`
  - `ColorContrast=(1.02,1.02,1.02,1.0)`
  - `BloomIntensity=0.35`
  - `BloomThreshold=1.0`
  - vignette `0.10`
  - motion blur `0.0`
  - ambient occlusion `0.0`
- Rendering config:
  - `r.AntiAliasingMethod=1` (FXAA; avoids temporal trails on procedural voxel meshes)
  - `r.ReflectionMethod=2`
  - `r.SSR.Temporal=0`
- Development command `VoxelTogglePostProcess` toggles camera PP overrides.

## Controls

Current input architecture: raw `BindKey` / `BindAxisKey`; runtime keybind settings rebind live. Enhanced Input migration is larger cleanup.

### Shared / Match Pawn

| Input | Behavior |
| --- | --- |
| `WASD` | Move |
| Mouse | Look; edit cursor in builder |
| `Space` | Jump |
| `Tab` | Toggle Cube Builder/build mode |
| `Enter` | Focus/submit chat |
| hold `V` | Push-to-talk |
| Builder right mouse | Orbit sculpt camera |
| Builder left mouse | Apply tool; some tools support drag/two-click preview |
| Mouse wheel | Zoom; play/build distances remembered separately |
| `T` | Toggle first/third person |
| `Escape` | Close UI or toggle cursor/menu focus |
| Level editor `Q` / `E` | Rotate current selection around shared pivot |

Cube Builder tools: place, erase, paint, box, 3D Bresenham line, BFS fill, extrude/face pull, sphere, wall, circle, mirror X/Y/Z, HSV wheel, RGB sliders, 256 swatches.

- Male/Female presets only reset geometry; voice remains independently selected and persistent.
- Other reset templates leave voice unchanged.

### Level Editor Pawn

Separate binding set on `AVoxelLevelEditorPawn`.

| Input | Behavior |
| --- | --- |
| `WASD` | Fly |
| `Space` / `Left Ctrl` | Fly up/down |
| Right mouse held | Look only |
| Left click | Grab gizmo arm, pick, or place armed object/marker |
| `Shift` + click | Toggle placement in selection |
| `Shift` or `Alt` into arm drag | Duplicate selection, then drag copies |
| `Q` / `E` | Rotate rigid selection around pivot fixed when selection changed/drag ended |
| `R` | Clear armed tool/selection; second press with nothing armed deletes cursor target |
| `Z` / `Y` | Undo/redo; plain keys because Ctrl flies down |

Level editor tools: catalog placement, spawn marker placement, select/delete, axis-gizmo move, keybound rotate, workbench object editing via Cube Builder, save/load via `UVoxelLevelSubsystem`.

Undo/redo:

- Implemented in `UVoxelLevelEditorComponent::UndoStack` / `RedoStack`.
- Capped at 50 whole-level snapshots restored through `AVoxelLevelDirector::BuildFrom`.
- Covers place, delete, move, group-rotate, duplicate, marker add/remove.
- Does not cover authoring/catalog ops: `CommitObject`, `SaveObjectToCatalog`, etc.

## Setup / Build / Packaging

### Maps / GameMode

`DefaultEngine.ini`:

- `GameDefaultMap=/Game/Voxel_Lobby.Voxel_Lobby`
- `EditorStartupMap=/Game/Voxel_Lobby.Voxel_Lobby`
- `GlobalDefaultGameMode=/Script/CubeCubeCube.VoxelPropGameMode`

Maps may override GameMode in World Settings. Main pawn: `AVoxelPropCharacter`.

### Palette / Materials

Required for visible voxel colors:

1. Project-owned `M_VoxelCube` and `MI_VoxelCube`.
2. Regenerate with editor command `VoxelBuildCubeMaterial` when needed.
3. Valid palette texture/source data.
4. Palette donor assigned or auto-resolved only when runtime sculpt/component needs it.

Symptoms:

- Default material or paint no-op: inspect palette source, material instance availability, palette texture readability.
- Skin surfaces missing: run `VoxelBuildSkinMaterials`.
- Skin instance cache stores failures; clear/restart after regenerating if rebuild appears ineffective.

### Cook Scope

- `MapsToCook` in `DefaultGame.ini` explicitly lists playable maps.
- Empty `MapsToCook` cooks every map under `/Game`; avoid.
- Add new playable maps to `MapsToCook` or packaged travel crashes.
- `Content/City`, `RealCitySF`, `Elite_FractalAlien`: source art for editor-only bulk converter; in `DirectoriesToNeverCook`.
- `/Game/City/Assets` is default input for `VoxelConvertFolder` / `VoxelConvertPlan`; keep if reconversion is needed.
- `/Game/VoxelDemo` is runtime content despite name; maps reference `VA_Building_07_Part_01`, `VA_SideWalk_01`, `VA_Rock_04`, `VA_Tree_03`.
- Before excluding a folder, grep `.umap` name tables for package paths.
- Cook scope does not affect C++ build time.
- Generated skin instances are loaded by name; `/Game/Cube/Materials` must stay in `DirectoriesToAlwaysCook`.
- Code-selected music lives in `/Game/Cube/Audio/Music` plus `SpaceMusicPack/Cues`; keep both in
  `DirectoriesToAlwaysCook`. Do not always-cook the entire marketplace audio source tree.
- Required loose files use `DirectoriesToAlwaysStageAsNonUFS`; paths are relative to `Content/`.
- `Saved/` is per-installation; editor-generated files there do not ship.

### Commands

From `D:\Program Files\UE_5.8`:

```bat
Build.bat Cubiciousflage Win64 Development -Project=D:\UE_Games\CubeCubeCube\Cubiciousflage.uproject
```

```bat
RunUAT.bat BuildCookRun -project=D:\UE_Games\CubeCubeCube\Cubiciousflage.uproject -noP4 -platform=Win64 -clientconfig=Shipping -build -cook -stage -pak -archive -archivedirectory=D:\UE_Games\Cubiciousflage_Shipping -nocompileeditor
```

- Keep archives separate: `Cubiciousflage_Build` = Development/debuggable; `Cubiciousflage_Shipping` = Steam.
- Shipping disables console; `VoxelSessionStatus`, `VoxelHost`, `VoxelFind`, `VoxelAddress` unavailable.
- Steam accepts Shipping builds only.
- Shipping build must be launched through Steam client per plugin docs.
- SteamPipe uses depot id, not app id; Valve typically allocates depot as app id + 1.
- Wrong depot id can report success with `0 new chunks uploaded` and no visible build.

## Base Level Baking / Thumbnails

- Base dressing is stored in `UVoxelLevelAsset`. The current converted assets are authoritative. The
  retired hard-coded procedural layout commands were removed because their coordinates describe the old
  maps, but the eight old `Level_Map_*` assets remain intact as themed object donors.
- Travel maps and voxel payloads are paired explicitly in `UVoxelMapLibrary`; do not derive one name
  from the other. This is required for pairs such as `Map_8_Sea` and `newLevel_Map_8_SeaPort`.

| Travel map | Voxel level asset |
| --- | --- |
| `/Game/Cube/Map/Map_1_City` | `/Game/Cube/Levels/newLevel_Map_1_City` |
| `/Game/Cube/Map/Map_2_Cemetery` | `/Game/Cube/Levels/newLevel_Map_2_Cemetery` |
| `/Game/Cube/Map/Map_3_Junkyard` | `/Game/Cube/Levels/newLevel_Map_3_Junkyard` |
| `/Game/Cube/Map/Map_4_AlienWorld` | `/Game/Cube/Levels/newLevel_Map_4_AlienWorld` |
| `/Game/Cube/Map/Map_5_Factory` | `/Game/Cube/Levels/newLevel_Map_5_Factory` |
| `/Game/Cube/Map/Map_6_Temple` | `/Game/Cube/Levels/newLevel_Map_6_Temple` |
| `/Game/Cube/Map/Map_7_Mansion` | `/Game/Cube/Levels/newLevel_Map_7_Mansion` |
| `/Game/Cube/Map/Map_8_Sea` | `/Game/Cube/Levels/newLevel_Map_8_SeaPort` |

- Each gameplay map contains one `AVoxelLevelDirector` whose `BakedLevel` points at the corresponding
  asset. `FVoxelMapEntry::LevelAsset` carries the same explicit pairing for host-menu preview and the
  Levels library's Copy action.
- `/Game/Cube/Levels` remains in `DirectoriesToAlwaysCook`, while all travel maps are listed individually
  in `MapsToCook`.
- `VoxelLevelDresser.cpp` provides the reproducible editor command `VoxelDressShippedLevels`. Run it with
  the corresponding travel map loaded so its authored volumes are available. With no `Apply` argument it
  dry-runs; `VoxelDressShippedLevels Apply <map>` creates a per-asset `.before-density-cut-backup` and saves.
  The tool imports curated definitions from the old donor assets, samples actual floor/ground voxels, rejects
  collisions and a 600 cm spawn radius, and assigns deterministic ids so repeat application adds nothing.
- A volume named `cuthere` clips existing baked placements at occupied-voxel precision; definitions and
  placements wholly outside are removed without re-running mesh or material conversion. Factory and
  Junkyard explicitly ignore this pass. Volumes named `spamhere` or `canvoxelspam` reserve 100 of the map's
  themed placements for their interior. All eight occupied-box corners must remain inside, and clearance is
  relaxed from 20 cm to a controlled one-voxel overlap only inside those volumes. Port maps the source-map
  volumes through its existing 0.7 conversion scale.
- Current themed additions: City 200, Cemetery 200, Junkyard 90, Alien World 200, Factory 180, Temple 200,
  Mansion 200, Sea Port 200 (1,470 total). Junkyard remains intentionally lighter because it is load-sensitive.
- Copying a base level creates and selects a custom `FVoxelLevelData` slot from the map entry's explicit
  asset, rebuilds the preview from that slot, then opens the level editor. It does not copy a live director.
- Level screenshots:
  - Captured/custom: `Saved/LevelThumbnails/<SlotName>.png`.
  - Shipped base map: `Content/BaseLevelThumbnails/<MapShortName>.png`.
  - `ResolveLevelThumbnailFile` checks `Saved` before `Content`.
  - Base screenshot key uses the travel map's short package name, such as `Map_2_Cemetery.png`; strip
    `UEDPIE_<n>_`, package paths, `.umap`, and object suffixes.
- `Map_1_City.png` remains valid. Any thumbnail named for a retired map is legacy and does not represent
  its replacement; the other current maps need fresh shipped captures.
- Levels window loads saved screenshots first, then code-rasterized voxel thumbnails from `FVoxelObjectThumbnail`; keep legacy fallback for older `UEDPIE_*_<MapName>.png` captures if needed.
### Voxel_Lobby's base village

- The travel lobby remains `/Game/Voxel_Lobby`; `DefaultGameMap`, `EditorStartupMap`, and session return
  travel continue to use it.
- `AVoxelLobbyGameMode::InitGame` spawns a fresh `AVoxelLevelDirector` with no fixed `BakedLevel`. A player's
  selected custom level still has priority. With no selected level, `ReloadFromSelection` loads
  `/Game/Cube/Levels/newLevel_Map_0_BaseVillage` as the shipped home background before considering the
  emergency code preset.
- Base Village is a lobby payload, not a hostable match map, so it is intentionally absent from the eight
  gameplay entries returned by `UVoxelMapLibrary::GetBuiltInBaseMaps()`.
- `UVoxelLevelSubsystem::EnsurePresetsSeeded` used to auto-select the first seeded preset as
  `SelectedLevelSlot` whenever the save-slot index was empty (fresh install, or every custom level deleted).
  Removed: `SelectedLevelSlot` is also what the fallback above reads, so an eagerly-selected preset meant
  `LoadSelectedLevel` always won and the shipped island was never reached. A player who wants a seeded
  preset as their level still picks it from the Levels window once, like any custom level.
- `FVoxelLevelPresets` is down to a single preset, `"Empty"` (`VoxelLevelPresets.cpp`): a plain grass
  square in the middle (`GrassSlot = 44`, the palette spectrum's `"grass"` index - presets have no named
  palette of their own, `CellSlots` are raw indices into the shared spectrum texture) with spawn markers
  only, no crates/walls/pillars. Previously two presets ("Courtyard", "Open Field") built from
  hand-painted primitive shapes in unchosen colours (small integer slot literals reading as whatever hue
  sits at that spectrum index - the flat red squares seen before this pass). Renaming or removing a preset
  changes its seeded slot name (`MakeSlotNameFromDisplayName`); a machine that already seeded the old ones
  needs both old slots deleted (emptying the index) before the next launch reseeds the current set.

## Multiplayer / Sessions

- Backend configured in `DefaultEngine.ini`, not code.
- `DefaultPlatformService=SteamCore`; `Null` remains fallback when Steam client absent.
- `UVoxelSessionSubsystem` does not name backend; it checks `IsSteamBackend()` and branches on lobby-vs-LAN settings.
- Configures `OnlineSubsystemSteamCore`, `OnlineSubsystemNull`, `SteamCoreSocketsNetDriver` -> `IpNetDriver` fallback.
- Steam uses lobbies, not internet sessions:
  - Host: `bUseLobbiesIfAvailable`.
  - Search: `SEARCH_LOBBIES`.
  - These must match, like `bIsLANMatch`.
- `SEARCH_PRESENCE` removed in UE 5.8; use `SEARCH_LOBBIES`.
- While Steam backend active, `bUseLAN` ignored; `ShouldUseLAN()` forces off and UI greys/relabels LAN checkbox.
- App id: `5041030`.
- `SteamAppId` and `SteamDevAppId` must match. Mismatch initializes against one app and matchmakes against another.
- Lobbies are keyed by app id; builds on different ids cannot see each other.
- Plugin writes `steam_appid.txt` next to exe from config. Useful in dev; do not upload a folder already run locally to Valve.
- `VoxelSessionStatus` showing `OSS=NULL` means Steam did not initialize.
- Steam hides lobbies across `BuildUniqueId` mismatch; both machines need same build.
- Quick Match asks for confirmation, then searches for 60 s, retrying every 2 s; its button becomes
  `Cancel Search` while active. It joins the best open result or creates an advertised 8-player session.
- Test matrix:
  - Real LAN/direct connect for two-machine tests.
  - Separate launched clients/machines for Null LAN discovery.
  - Single-process Multi-PIE may need LAN off because Null sessions are visible in-process.
  - Direct connect remains fallback when LAN discovery blocked.
- PIE can mask save-file and level-transfer problems because clients share local files.
- Custom levels are not transferred host -> client yet.
- Multi-PIE warning: generated runtime voxel chunks must never serialize as network references across PIE worlds. `DriverPIEInstanceID != ObjectPIEInstanceID` on `RuntimeSharedCluster_*` means transient component leaked into replication/based movement.
- If players jitter/teleport on jump or damage differs host/client, inspect movement bases and replicated component refs before CharacterMovement tuning.
- Steam invites/join via presence are enabled on sessions, but running-game join-request callback handler is not implemented.
- No Steam session has been verified end-to-end yet.

## Diagnostics

| Command | Scope |
| --- | --- |
| `VoxelLevelStress` | Custom level placement/build stress. |
| `VoxelLevelBuildTest` | Custom level build diagnostic. |
| `VoxelObjectList` | Catalog counts/script diagnostics. |
| `VoxelPaletteCheck` | Storable slots/named colors; requires real RHI for GPU/platform texture checks. |
| `VoxelBuildCubeMaterial` | Editor: generate/update cube material assets. |
| `VoxelBuildPlayerHitMaterials` | Editor: generate/update player hit/interaction rainbow materials. |
| `VoxelBuildSkinMaterials` | Editor: generate `M_VoxelCubeSkin` + 58 skin surface instances. |
| `VoxelSkinStatus` | Master/material/effect/equipped/driver resolution. |
| `VoxelSkinVerify [filter]` | Read baked parameter values and bound texture names from disk assets. |
| `VoxelForceMaterial <name>` | Bind surface directly to local body, bypassing store/ownership/equipped skin. |
| `VoxelSkinList [catalogue]` | Catalog rarity/price/ownership/surface/gifts. |
| `VoxelGrantDlc <character\|weapon\|deluxe\|all\|none\|clear\|id>` | Fake DLC ownership through real code path. |
| `VoxelGrantGold <amount>` | Credit spendable gold only; never rank. |
| `VoxelEquipSkin <SkinId>` | Equip without buying. |
| `VoxelTogglePostProcess` | Toggle camera post-process overrides in PIE/development. |
| `VoxelSessionStatus` | Active OSS, LAN flag, session, advertised map/name, open slots, listen address. |
| `VoxelAddress` | Host connect address. |

Caveats:

- `VoxelPaletteCheck` can correctly fail under `-nullrhi`.
- Build/headless diagnostics do not prove visual quality; inspect objects, thumbnails, materials, palette translation, VFX in editor/game.
- Live Coding can leave stale assumptions; for editor-module changes, confirm real relink/restart when behavior contradicts source.
- `BUILD SUCCESSFUL` does not prove files were cooked/staged; inspect staged output.

## Do Not Break

| Area | Constraint |
| --- | --- |
| Runtime voxel edits | Invalidate template cache before refresh. |
| Player damage | Do not rely on plugin sparse overrides for transient dense sculpts. |
| Custom levels | Do not make one giant sculpt. |
| Placements | Do not copy object assets per placement by default. |
| Object authoring | Edit workbench copy; commit later. |
| Palette | Slot index is meaningful only with matching effective palette. |
| Cooked textures | Push runtime palette writes through `UpdateTextureRegions` and set `NeverStream`. |
| Slate | Visible overlays are hit-testable unless configured otherwise. |
| Client code | `GameMode` exists only on server; prefer replicated `GameState`. |
| Bots | Test `bIsBotPawn` for human-only local branches. |
| Replicated UI | Use bound/reactive visibility for delayed replicated state. |
| Character scale | Scale `VoxelBody` plus the fitted capsule; do not change sculpt voxel size or actor scale. Match props resolve to `1.0` regardless of the replicated preference. |
| Saves | Maintain index saves; save slots cannot be enumerated. |
| Input | Raw key binding is current architecture. |
| Multiplayer | Custom levels do not transfer host -> client yet. |
| Travel | Host-selected match settings use travel URL options. |
| Cosmetics | Derive presentation from gameplay events; avoid redundant cosmetic replication. |
| Editor/runtime | Runtime must package without `VoxelEditorEditor` or editor-only APIs. |
| Materials | Required Shipping materials should be project-owned with correct usage flags. |
| Cook scope | Empty `MapsToCook` cooks every `/Game` map; missing playable map crashes on travel. |
| Loose files | `DirectoriesToAlwaysStageAsNonUFS` paths are relative to `Content/`. |
| `Saved/` | Per-installation; not packaged content. |
| Skin materials | `FVoxelCubeMaterial::Apply` must read asset's own instance; never overwrite skins per frame. |
| Skin materials | `/Game/Cube/Materials` must stay cooked. |
| Skin materials | Instance cache stores failures; clear after regenerating. |
| Skin material generation | `LoadObject` before creating package over existing asset; `CreatePackage` on unloaded on-disk package can crash on save. |
| Skin shader | MSVC string literal cap: 16380 bytes; `TEXT()` is wide. Shader chunks stay around 6500 chars; do not hand-merge. |
| Cosmetic saves | Material/effect slots are raw `uint8`; reordering enums needs version bump. |
| Purchased packs | Demo folders explicitly never-cooked; only referenced folders cook. |
| Editor FPS | 20 FPS editor cap lives in `CubeCubeCubeEditorModule` `OnPostEngineInit`, not `[ConsoleVariables]`; do not cap Shipping. |
| `SButton` padding | `ContentPadding` ADDS to the style's `NormalPadding`/`PressedPadding` (see `SButton::GetCombinedPadding`); it does not replace it. `SecondaryButton`'s style padding is 14 px/side, so tightening `ContentPadding` alone barely shrinks a button. Use `.NormalPaddingOverride(...)` / `.PressedPaddingOverride(...)` to actually replace it - and check this on any small icon-only button, since an unreplaced 14 px pad can squeeze content smaller than the box into near-zero visible space (looked exactly like a missing icon, was actually a layout squeeze). |
| Slate content icons | `ESlateBrushDrawType::RoundedBox` (`MakeContentImageBrush` in `VoxelUIStyle.cpp`) is right for tinted button-sized swatches but did not render the Cube Builder's uploaded `/Game/Cube/UI/editorui` icons at 18 px at all. `MakeContentIconBrush` (plain `ESlateBrushDrawType::Image`, no forced tint) is the one confirmed-working path for a small flat icon - `SVoxelCustomCursor`'s cursor image uses the same plain-Image approach and renders correctly. |
| Slate Tab key | `bTabNavigation` is set `false` project-wide (`FCubeCubeCubeModule::StartupModule`, `CubeCubeCube.cpp`, via `OnPostEngineInit`) because Slate's default Tab-as-focus-navigation was eating the Builder key binding (`EVoxelControlBinding::Builder`, default `Tab`) before `AVoxelPropCharacter::ToggleEditMode` ever saw it. Any future UI that wants Tab-to-next-field navigation will not get it for free. |
| Slate self-positioning widgets | Do not drive a "follow the OS cursor" widget with `FSlateApplication::GetCursorPos()` (desktop pixel space) minus `FGeometry::GetAbsolutePosition()` via `SetRenderTransform` every Tick - two different coordinate spaces, drifts worse at non-1:1 DPI/window scale and re-triggers Slate invalidation every frame. Use `FGeometry::AbsoluteToLocal` into an `SConstraintCanvas` slot's `SetOffset` instead (see `SVoxelCustomCursor.cpp`). |

## Known Gaps

### Map_Climp had 17 half-baked clusters, and it only showed up at cook (2026-08-20)

The Shipping cook failed with 68 `LogCook: Error: Content is missing from cook`. A partial bake of
Map_Climp had written 41 `SM_VA_Level_Map_Climp_*` proxies under `Content/VoxelEditor/BakedMesh` but
only 24 of the paired `MI_VA_Level_Map_Climp_*` instances under `Content/VoxelEditor/Materials`. Both
the baked mesh AND the `VA_Level_Map_Climp_*` voxel asset hard-reference that instance, so removing
the meshes alone only halved the errors.

Repaired by duplicating a surviving sibling into the 17 missing names, which is exactly what the bake
would have written: diffing two of the instances byte for byte shows the ONLY difference is the
package name and its GUID - same parent `M_VoxelBase`, same two palette textures, no per-instance
parameters. Script kept at `Scripts/Content/fix_climp_mats.py`.

**Nothing warned about this before the cook.** The editor loads a missing material as null and draws
the fallback checker, which on one cluster of a kilometre-tall tower nobody noticed. If Map_Climp is
rebaked, check that the two folders come out with equal counts before packaging again.


### Climp is feature-complete and unplayed (2026-08-20)

The ascent mode is done to the edge of what can be verified without a person climbing it. Ten generated
floors, mined ore and handed pickups, distance streaming for both the level and the pickups, per-run
persistence of what has been spent, floor-tinted lighthouses, and eleven Steam achievements. The full
reasoning is in `Docs/HANDOFF-ClimpTower.md`; the contract is `Docs/DESIGN-Climp.md`.

What is left is what no assertion can answer:

- **Nobody has climbed it.** Route metrics say it is spannable - worst step 1,756 uu, worst rise
  1,103 uu - but a 1,103 uu rise means roughly 850 uu of placed cubes, and whether that is satisfying
  is a question about playing, not measuring.
- **The summit lighthouse's roof has never been reached on foot.** `ClimpSummit` teleports onto it and
  the win fires correctly from there; whether a 3x building can be climbed at all is unknown.
- **A large metal deposit is now wider than a route island** - 836 uu against 650 - so one can sit
  across the step it stands on. Deliberate, for legibility, and the first thing to suspect if the
  route feels blocked. `VoxelClimpPickupDesigns::GetDepositScale` is the dial.
- **The eleven achievement ids must exist in Steamworks** before they can unlock. The build asserts
  they exist locally; nothing here can check the other half, and an unregistered id fails silently.
- **`Cubiciousflage.Voxel.BaseMapRegistry` still fails**, and predates all Climp work: `newLevel_Map_1_City`
  came back from a resave with 5,697 placements instead of 5,698. Decide between re-baking the asset
  and updating the expectation - do not silently do the latter.
- Tropical and Fire Magma read as seaside and scorched metal until those themed sets are authored.

Two things worth knowing before touching it:

- **Do not re-derive the tower.** It is deterministic from the seed only for identical INPUTS, and
  `BuildAndApply` passes resolvers a bare `Build()` does not. Read `AVoxelClimpGameState`. This has
  been the root cause of two separate "everything is in the wrong place" bugs.
- **An actor's existence is not the same statement as a pickup's existence** now that they stream.
  Anything meaning "this is gone for good" is recorded in the plan or the director, not in the actor.

### Settings readability, Turkish terminology and Shop resource-lifetime crash (2026-08-11)

- The English Start Game hint is now `Host, join, or quick-match.`; removing the trailing explanation
  lets the responsive root card return toward its 344-pixel minimum. The row chevron uses the plain
  heading face rather than the outlined title face, so it no longer draws a heavy black enclosure.
- The language field and its popup now have explicit outer frames matching the Cube Editor's framed
  combo language. Its rows now have breathing room, and the Settings tabs use the same rainbow frame,
  cream surface and cyan active treatment as the main-menu buttons.
- Settings text is explicitly near-black, tabs use clean two-pixel frames and both Game preferences
  are separate cream/rainbow-framed rows. This avoids relying on inherited foreground colors.
- The repeated Turkish-to-Shop `UObjectArray Index >= 0` assertion was not caused by translated text.
  Shop rarity cards lazily generated transient gradient textures, while their brush cache held only
  raw UObject pointers. Culture-change GC collected those textures. `FVoxelUIStyle` now owns the
  brushes in its style set and holds every generated texture with `TStrongObjectPtr` until Shutdown.
- Turkish is regenerated with a dedicated English-to-Turkish model and an explicit ASCII gameplay
  glossary. Menu, Shop and match terminology now uses contextual terms such as `AYARLAR`, `MAGAZA`,
  `NESNE`, `AVCI`, `Kullaniliyor`, `Sahip Olunan` and `MAC TAMAMLANDI`, rather than literal output.
  The other catalogs remain complete translation bootstraps and still require native-speaker release QA.
- Development Editor and standalone Game targets compile successfully. All 11 `Cubiciousflage` automation
  tests pass; `Cubiciousflage.UI.LocalizationRuntime` creates all five Shop rarity gradients, forces Unreal
  GC, verifies their texture UObjects and every project font-face, then cycles all 22 non-English
  supported cultures. The pre-existing third-party `FRuntimeVoxelSaveData::Size` diagnostic remains
  unrelated.

### Shield, fall recovery, result UI and localization validation (2026-08-11)

- UE 5.8 Development Editor and standalone Game targets compiled after the replicated barrier,
  immediate fall recovery, fixed settings layout, result card and language-preference changes.
- `Automation RunTests Cubiciousflage` completed all 11 discovered tests successfully. The localization
  runtime test now switches through all 22 non-English Steam-supported cultures and verifies a project-owned Shop
  translation from each compiled `.locres`.
- A real `-game` load entered `/Game/Voxel_Lobby`, brought the world up for play and built Base Village
  (886 placements / 283,677 voxels). The existing third-party `FRuntimeVoxelSaveData::Size`
  initialization diagnostic and unavailable Steam SDR warnings remain unrelated startup noise.
- The source localization gather completed cleanly (805 PO entries / 2,842 English words), producing
  complete PO/archive/locres files for the original 26-culture bootstrap under `Content/Localization/Cubiciousflage`; only
  the 23 Steam-supported targets are now selectable, regenerated and staged. The
  word-count report records 2,842 translated words for every culture. These are complete coverage
  bootstraps, not release-approved prose: native-speaker QA remains required for meaning, tone,
  gender/plural choices and game-specific terminology.

### Competitive minimum, skill VFX, menus and recovery validation (2026-08-11)

- Props, hunters and custom character saves now share a 1,000-occupied-voxel minimum. The Cube template
  is a solid 10x10x10 volume for exactly 1,000 cells. Closing Dot or any other undersized home-editor
  design resolves to this Cube instead of restoring a legacy humanoid.
- Barrier protection remains server-authoritative. A reliable multicast arms a local presentation
  lifetime and a replicated timer reconstructs it for late joins; the visible shield is an Engine
  sphere centred on the active occupied-voxel bounds and uniformly scaled to enclose their diagonal.
  It therefore stays spherical and follows edited/equipped body sizes without intersecting wide or
  tall builds. It uses `/Game/Cube/Mods/Voyager/Demo/Materials/M_Barrier_Inst`. Dash sends a reliable start/end cube
  burst. Weapon firing and aim presentation are first-person-only; third-person mouse presses no
  longer start firing or alter weapon visibility.
- Cube Builder Single brush amounts are `1x`, `2x`, `4x`, `8x`, mapped to 1, 4, 8 and 16 cells in
  the hit-face plane. The hover outline uses the same cell footprint, so 2x/4x/8x no longer preview
  as one cube. This is edit-event work only; it adds no Tick cost.
- Fall recovery retains immediate overlap handling and adds a 10 Hz authority bounds sweep for missed
  high-speed overlaps. Cost is O(recovery volumes x pawns) ten times per second; normal maps use one
  volume. Recovery prefers exact authored role markers and the volume no longer participates as ground.
- The compact root menu has one Play button. Its dedicated Play page contains Quick Match first,
  followed by Host, Join, Leaderboard, Achievements and the bottom rank bar. Host opens a separate
  page containing only server/rule/LAN controls, Play, and base/custom level selection; Back returns
  to the Play page. Confirmation text uses increased line height.
- Host Game uses a 1180x720 body with a 510-pixel rules column and a right-aligned 640-pixel map
  column, so the extra width belongs to the rule controls instead of becoming blank map space. Base/custom tile captions use the
  smaller heading face on near-black strips; Custom Levels shows two tile rows in a vertical scroll
  area. Rule value fields are widened, and Host/Join text fields plus discovered-game rows use the
  bordered white entry treatment shared by the main theme.
- Host defaults to `New Lobby`, with an optional masked password. Search results mark protected rooms;
  Join has a masked password field and checks the verifier before travel, while match `PreLogin`
  enforces the same verifier server-side. This is lobby access control, not a replacement for Steam
  authentication. The default prop hiding/head-start phase is 100 seconds.
- The expanded root menu shows `BASE GAME` plus owned DLCs on the left. Owning both Character and
  Weapon Packs is presented as `DELUXE`, including shop skins, special effects and materials.
- Join is Steam-session-only in the UI: it exposes discovered games, Refresh and Join, with no IP
  field, direct-connect button or LAN selector. Achievement completion/progress still comes from
  Steam, but achievement icons are local generated voxel thumbnails and remain visible without a
  running Steam client. Shop catalogue action buttons override Slate style padding and use the small
  heading face so `Steam Store` fits inside narrow cards.
- Cube Counter no longer repeats progress/rainbow line ornaments. Rank remains the progress-line
  presentation and the old texture-baked circular cap is replaced by a soft-square bar. Cube Builder
  and Settings sliders share neutral circular thumbs with explicit borders instead of blue squares.
- UE 5.8 Development Editor and standalone Game targets compile. All 11 `Cubiciousflage` automation tests
  pass. Real `-game` loads brought `/Game/Voxel_Lobby` and `/Game/Cube/Map/Map_1_City` up for play and
  built Base Village (886 placements / 283,677 voxels) and Map 1 City (5,737 placements / 1,193,194
  voxels). The existing third-party `FRuntimeVoxelSaveData::Size` diagnostic remains unrelated.

- The follow-up menu/UI, first-person firing, scaled brush-preview and persistent shield changes were
  rebuilt for UE 5.8 as both Development Editor and standalone Game. All 11 `Cubiciousflage` automation
  tests passed, and a fresh headless `/Game/Voxel_Lobby` runtime load brought the world up, built Base
  Village (886 placements / 283,677 voxels), spawned the character and preloaded all 11 VFX systems.

### Settings, loading, Level Editor and VFX-card visuals (2026-08-11)

- Selected window-mode, quality-preset and advanced-quality labels bind an explicit near-black
  foreground. They no longer inherit the pale secondary-button foreground over the cyan selected fill.
- Loading tips now rotate only through the thirteen current Legend/Ultimate identity skins; the old
  Recruit, Scout, Dot and crate preview entries are no longer selected as loading-screen characters.
- The Level Editor's nine key hints live in a fixed 270-pixel controls rail on the right. They no longer
  wrap through the centre of the world view. The compact Cube Builder hint strip remains unchanged.
- Lobby, Prop and Hunter placement markers are role-shaped emissive voxel glyphs on hollow pads: a cyan
  doorway, green prop cube and red +X hunter arrow. One instanced component owns each marker's cubes,
  rebuilds only when the role/visibility changes, has no collision, and is hidden during play. The
  project-owned `/Game/Cube/Materials/M_VoxelSpawnMarker` carries Instanced Static Mesh usage; regenerate
  it together with the player feedback materials via `VoxelBuildPlayerHitMaterials`.
- Footstep and Aura shop cards use cached voxel-rendered category glyphs rather than unavailable cooked
  Niagara editor thumbnails. Footsteps render offset voxel soles; auras render a halo and central spark.
  Effect id and rarity deterministically vary their palette. These are honest category illustrations;
  equipping the item remains the accurate animated Niagara preview.
- UE 5.8 Development Editor and standalone Game targets compile, all 11 `Cubiciousflage` automation tests
  pass, and a fresh `/Game/Voxel_Lobby` runtime load completed after building Base Village (886
  placements / 283,677 voxels). The one-shot material-builder startup process saved all assets before
  hitting an unrelated InteractiveToolsFramework shutdown exception; clean subsequent editor and game
  processes load `M_VoxelSpawnMarker` without warnings.

### Material cards, responsive results and Level Editor follow-up (2026-08-12)

- Shop material cards now have imagery. Texture-backed surfaces display their authored base-colour
  texture, retained by a strong cached brush for GC and cooked-build safety. Procedural, normal-only and
  glitter surfaces use a deterministic shaded voxel-cube swatch rather than showing a misleading purple
  normal map or an empty card. Ownership remains a compact overlay badge instead of replacing the art.
- The match-complete card has tighter vertical rhythm, a down-only scaling victory heading, and a centred
  auto-wrapped summary with an explicit line height. Long localized victory text can consume a second line
  without clipping the 590-pixel card.
- Level Editor removed the duplicate middle-of-panel shortcut paragraph and the `Esc to close` title hint.
  Action, marker, placement and object-row labels use the dark theme foreground; object and level-name
  fields use the bordered light entry style. Lobby, Prop and Hunter world labels are raised, enlarged from
  32 to 52 world units, keep their role colours, and sort in front of the emissive voxel glyphs.
- `Close editor` now hides the panel and markers while keeping the deliberately spawned editor fly pawn.
  The lobby controller toggles that component with the configured menu/Esc key. Because this path exists
  only while `AVoxelLevelEditorPawn` is possessed, a normal lobby or match pawn cannot accidentally open
  creator tools.
- UE 5.8 Development Editor and standalone Game targets compile. All 11 `Cubiciousflage` automation tests pass.
  A fresh headless `/Game/Voxel_Lobby` load brought the world up and built Base Village (886 placements /
  283,677 voxels); there were no new Slate or material-preview load errors. Existing plugin struct,
  generated-palette, Steam-network and missing optional save-slot diagnostics remain unrelated.

### Shop balance header and localized Level Editor layout (2026-08-12)

- The character catalogue's Flat Silhouette control and alternate grey thumbnail path were removed.
  Character cards always show their actual skin palette.
- The Golden Cube balance moved out of the shop body and into the Shop title bar beside the close button.
  It has no background badge or container border; the gold heading face itself has a one-pixel black outline
  so it remains readable over the title gradient without the bracket-like box treatment.
- Materials and Effects (including Footsteps and Auras) now use the same rarity-darkened outer card colour
  as character skins. Preview wells are near-black, authored material textures are deliberately subdued,
  and generated effect/voxel art is slightly dimmed so the border and rarity chips carry the colour.
- Host-map cards now put the map name on an explicit near-black strip with near-white text instead of
  inheriting the old grey/beige tile surface.
- The Level Editor panel widened from 300 to 370 pixels and is left-centre anchored; its controls rail is
  bottom-right anchored. Object rows have larger vertical/content padding and live inside a padded light
  list well. Headings wrap, status copy has its own padded surface and line height, and the three spawn-role
  actions are full-width rows (`Lobby spawn`, `Prop spawn`, `Hunter spawn`) so localized labels are not
  squeezed into three narrow columns.
- Spawn-role `UTextRenderComponent` labels are raised above their voxel glyphs and billboard in pitch as
  well as yaw. They now use the cooked offline `/Engine/EngineFonts/RobotoDistanceField` font rather than a
  runtime/composite Slate font that TextRender cannot reliably rasterize. A second 58-unit black text copy
  sits behind each 52-unit role-coloured `LOBBY`, `PROP` or `HUNTER` label as a world-space outline. Both
  copies share marker visibility and camera-facing rotation; ticking still occurs only while markers show.
- UE 5.8 Development Editor and standalone Game targets compile, all 11 `Cubiciousflage` automation tests pass,
  and a 20-second headless `/Game/Cube/Map/Map_0_Custom` load built all 16 spawn marker actors without font,
  TextRender, UI or marker errors.

### Match nameplates, Builder input and initial progression (2026-08-12)

- Player names use a transparent world-space `UWidgetComponent` with the same Gome Pixel Slate font as
  the menu. This replaces the broken `UTextRenderComponent` path, which cannot rasterize the project's
  runtime/composite font asset. Labels are white, camera-facing, visible for every player during
  `WaitingForPlayers` and `ReadyUp`, hidden for props once play begins, permanently visible for hunters,
  and temporarily revealed for five seconds when a prop takes hunter damage.
- Match controllers bind the saved Builder key directly and consume it above pawn input. The Cube Builder
  can therefore open or close with Tab even after a Slate button, list, or text field changed keyboard
  focus; the panel's preview-key path still handles the key while one of its descendants owns focus.
- Quick Match now waits 60 seconds before its first search deadline/fallback.
- Fresh profiles receive 100,000 Golden Cubes. Existing prototype saves receive the 99,900 difference
  from the old 100-cube grant once through `StartingGoldGrantVersion`, preserving purchases and rewards.
- Nameplate work remains O(players) per frame: one visibility/deadline check and, while visible, one
  camera-facing rotation per pawn. It performs no voxel traversal and adds no replicated payload beyond
  the existing reveal deadline float.
- UE 5.8 Development Editor and standalone Game targets compile. A headless Development runtime load
  entered `/Game/Cube/Map/Map_1_City` with `VoxelPropGameMode`, ran 240 frames and exited cleanly without
  nameplate/widget/font assertions. All 11 `Automation RunTests Cubiciousflage` tests passed; the known
  third-party `FRuntimeVoxelSaveData::Size` initialization diagnostic remains unrelated.

### Lobby, hiding and hunter-release placement (2026-08-12)

- Initial connection and round reset place every player on `Lobby` spawn points. Entering `Hiding` moves
  props to `Prop` points and the drafted hunter to a `Hunter` holding point. When the hiding timer expires,
  entering `Hunting` releases hunters by teleporting them back to `Lobby` points; props remain in place.
- Role moves log their destination and report failed `TeleportTo` calls instead of playing a false
  teleport presentation. Map 1's registry expectation is synchronized from 5,737 to the newly authored
  5,740-placement payload.
- Every successful role move also replaces that player's server-owned fall-recovery anchor. A hunter
  released to `Lobby` therefore returns to the latest Lobby point after falling into water, rather than
  being sent back to the obsolete Hunter holding point. Conversion clears a prop's old anchor; late-join
  fallback routing remains phase-aware (`Hunter` during Hiding, `Lobby` during Hunting).
- Loading cards now resolve exact unified `Skin.Character.*` Shop entries and use the Shop's own
  skin-specific geometry, paint application and thumbnail cache key. Legacy Male/Female silhouettes are
  no longer built for loading screens.

### Full-screen lobby entry gate (2026-08-12)

- The first lobby entered during an application session opens a dedicated full-screen gate before the
  ordinary lobby menu or chat is created. It presents `/Game/Cube/UI/library_logo2`, the same 23-language
  list used by Settings, and a localized `START GAME` action. Returning from a match does not replay it.
- While the gate is present, both controller and pawn input components are removed from the input stack
  and cinematic movement/look locks remain active. Builder, combat, chat, menu, movement and camera keys
  therefore cannot leak through the UI. Start restores the components, blends to the pawn camera over
  0.75 seconds, and initializes the normal collapsed lobby menu.
- A transient, client-only camera circles the local character at an exact 2,000-unit distance with a
  gently elevated orbit at 7 degrees per second. Its only recurring cost is one camera transform update
  per frame while the gate is visible; it is destroyed after the return blend and replicates nothing.
- The weapon and prop-skill strip is collapsed while the full-screen gate is visible and restored when
  Start is accepted. The hidden state also overrides any builder-close visibility update during gate
  setup, preventing the strip from briefly or permanently reappearing before entry is accepted.

### Character-size validation (2026-08-10)

- UE 5.8 Development Editor build passed after compiling the replicated character and Slate changes.
- A real `-game` load entered `/Game/Voxel_Lobby`, built Base Village (886 placements / 283,677
  voxels), ticked, and shut down cleanly.
- A second real `-game` load entered the match map `Map_1_City`, exercised the match-role spawn path,
  built 5,737 placements / 1,193,194 voxels, and shut down cleanly.
- `Automation RunTests Cubiciousflage` passed all five discovered tests. The run also corrected stale Map 1
  placement and Map 1/2 display-name expectations to the current rebuilt assets.
- Scaling is event-driven (slider, replicated preference, team, or match phase change); it adds no
  per-frame work and no voxel/design payload growth. Network impact is one replicated float per pawn
  when the preference changes.

### Dynamic-music validation (2026-08-10)

- UE 5.8 Development Editor build passed with music-state, phase, damage, HUD, and timer changes.
- Audio-enabled `/Game/Voxel_Lobby` runtime load created the WASAPI mixer, played the new lobby bed,
  completed cue/music preload, and reported no missing music path.
- `Cubiciousflage.Audio.MusicCatalog` loads all six non-silent states as unique `USoundBase` assets;
  the complete five-test `Automation RunTests Cubiciousflage` suite passed.
- Music transitions are event/timer driven. There is no new Tick work or network traffic; each client
  derives phase, damage, and victory music from state/events already replicated by gameplay.

- The generic converted-world green/beige cast was confirmed as a palette-identity mismatch: conversion
  quantized against the donor's authored table, then runtime replaced it with the generated ramp (251 of
  256 slots differed). Full-level and bulk conversion now apply and verify the runtime ramp before
  quantizing, fail closed on disagreement, and log per-object Oklab evidence. Christmas Town and Cemetery
  were reconverted and runtime-loaded on 2026-08-10; see `Docs/WORKFLOW-LevelVoxelConversion.md`.

### Gameplay / Bots / Combat

- No respawn on conversion; player keeps position and changes side.
- Bots never hunt; one human + bots always makes the human hunter.
- Bots do not use prop skills (barrier, dash, stationary lock, reshape) and never open builder.
- Bots ready 3-12 s after ready-up starts; with bots present, solo host round usually starts at 30 s soft timer.
- `EliminateBot` builds/reasoned through but needs runtime exercise by hunter shooting bot to zero.
- Several maps have few 1000-8000 voxel prop candidates near prop spawns; catalog top-up masks this.
- Level props do not feed integrity/scoring; only player bodies track destroyed voxels.
- No scoring/round history.
- Physics Launcher damage below one voxel default health; relies on impact shatter.
- `AVoxelFallRecoveryVolume` resolves immediately on authoritative BeginOverlap. A delayed timer tied
  to EndOverlap cannot catch a falling pawn: the pawn crosses the volume before the timer expires.
  During an active round, a human prop is converted to hunter (the prop death penalty) before the game
  mode selects a grounded role-appropriate spawn; bots are relocated without conversion.
- The same recovery actor now recognizes `AVoxelLobbyGameMode`: overlap or the 10 Hz missed-overlap
  bounds sweep stops movement and teleports the existing lobby pawn to a Lobby spawn. It must not
  unpossess or destroy the pawn, because that discards the live voxel design and locally owned
  weapon/skill HUDs. The shipped
  `/Game/Voxel_Lobby` map was audited and contains one `VoxelFallRecoveryVolume_0` instance.
- No damage numbers, kill feed, sound occlusion.
- One footstep surface; voxel geometry lacks physical material mapping.

### Custom Levels / Editing

- No host-to-client custom level transfer. Existing foundations: `ContentHash`, ascending `CellIndices`.
- Spawn convention mismatch: engine spawn treats point as pawn center; phase movement treats marker as feet + half height.
- Custom-level editor enforces one marker per role; hand-built maps can still place multiple `AVoxelMatchSpawnPoint`s per role.
- Old pre-height-fix markers may need replacing.
- Ghost preview transparency depends on voxel material opacity parameter.
- Hover/selection outlines use debug draw; not Shipping-ready.
- Gizmo moves/rotates multi-object selection as rigid group; no per-axis scale.
- Gizmo picking is analytical and ignores occlusion.
- Selection by nearest placement origin can fail in dense clusters.
- Converted objects default destructible; importer cannot infer structure.
- `.vox` import reads only first model.
- Cube Builder sculpt strokes have no undo. Natural undo unit: `{cell, previous slot, previous occupancy}` at `PlaceVoxel`/`EraseVoxel`/`PaintVoxel`.
- Only fixed home-character save exposed; named-slot machinery not exposed as multi-outfit UI.

### Cosmetics / Store / Steam / Presentation

- FX overlay materials (`Sheen`, `Pulse`, `Rgb`, `Prismatic`) are named by `GetFxOverlayPath` but absent; overlays do not apply.
- Characters window lacks Footstep/Aura sub-buttons; shop and Cube Builder have them.
- Steam Inventory Service for Gold packs is unbuilt. Character Pack `5102500` and Weapon Pack
  `5102510` are allocated and wired; their Steam store pages, pricing, review/release, and live-client
  entitlement checks remain. Deluxe is a Complete-the-Set bundle containing the base game and both
  DLC packages, not DLC app `5102520`; its store copy must mention special effects and materials.
- Material roughness/metallic/parallax/fresnel values need lit-scene visual pass.
- Grid materials share one roughness/metallic pair; some read too similarly.
- Cube burst/debris, laser beam, and gizmo may still need project-owned Shipping-safe material usage if any path uses Engine materials without `MATUSAGE_InstancedStaticMeshes`.
- Only two ACV voice archetypes exposed.
- Steam sessions, the Gold leaderboard and all 50 defined achievements are wired. Steamworks
  partner-site achievement/stat definitions and a live Steam launch remain required for end-to-end
  confirmation; DLC-only cosmetics now open their Steam store product through the overlay and refresh
  the two entitlement bits when the overlay closes. Friends and Rich Presence still have no
  game-facing feature.
- Palette donor is still auto-picked rather than explicitly assigned; color identity depends on asset registry order.
- Six of eight base maps lack shipped thumbnails.
- Shipped thumbnails are full-res screenshots (~1.4 MB each) drawn into ~100 px tiles.

## AI Session Checklist

- Read relevant source before editing; comments encode design decisions.
- Preserve runtime/editor split and UE lifecycle/reflection conventions.
- Preserve palette single-source path and material assignment order.
- For voxel edits, reason about dense asset state + plugin template cache.
- For multiplayer, define server authority, replicated state, late-join sync, travel URL effects.
- For custom levels, preserve object-definition sharing; per-placement unique assets are explicit perf tradeoff.
- For UI, account for Slate hit testing, `VoxelCursorRequests`, delayed replicated state, gamepad/mouse/keyboard focus.
- For Shipping visuals, avoid debug draw and editor-only material fixes.
- Validate meaningful changes with build, runtime check, and `Automation RunTests Cubiciousflage` when feasible.

## Optional Unreal MCP

Epic experimental Unreal MCP plugin (`ModelContextProtocol`) may expose `BlueprintTools`, `AssetTools`, `ObjectTools`, `SceneTools`, `MaterialTools`.

- Start: `ModelContextProtocol.StartServer`
- Default endpoint: `http://localhost:8000/mcp`
- `bAutoStartServer` off by default.
- Useful for GameMode overrides, Blueprint setup, material/palette assignment, level asset baking.
