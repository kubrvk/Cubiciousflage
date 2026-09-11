# Third-Party Plugin Patches

## NC-001 — Per-voxel emissive payload

Plugin: `Plugins/VoxelEditor`

Files:

- `Source/VoxelEditor/Public/VoxelMeshAsset.h`
- `Source/VoxelEditor/Public/Runtime/VoxelSparseData.h`
- `Source/VoxelEditor/Private/Runtime/VoxelRuntimeTemplateCache.cpp`
- `Source/VoxelEditor/Private/Runtime/VoxelRuntimeMeshBuilder.cpp`

Reason: the plugin's voxel vertex payload carries palette, health, and transient stress but has no
authored emissive channel. Cubiciousflage map conversion needs emissive windows, lamps, and holiday lights.

Implementation: a sparse `VoxelEmissive` map is copied into shared runtime templates and encoded in
vertex-color blue at rest. Published collapse-stress overrides retain priority while the debug view is
active. Assets without emissive data are byte-for-byte equivalent at runtime apart from the added empty
serialized property.

## NC-002 — Cooked pristine voxel proxies for base levels

Plugin: `Plugins/VoxelEditor`

Files:

- `Source/VoxelEditor/Private/VoxelComponent.cpp`
- `Source/VoxelEditor/Private/Runtime/VoxelRuntimeTemplateCache.cpp`

Reason: authored base levels should not regenerate identical pristine render and collision meshes on every
load, but their actors must remain voxel-backed and fully damageable.

Implementation: a compatible cooked `BakedStaticMeshProxy` can serve as the initial shared runtime visual
cluster while the normal sparse template and editable chunk meshes remain authoritative. The first edit
replaces that proxy through the existing damaged-cluster path. Pristine edit-chunk surface meshes stay lazy
and are generated from sparse voxel state only for a cluster that is actually damaged. Definitions larger
than one runtime visual cluster fall back to procedural shared-cluster generation. The editor preview honors an explicitly
cleared bake state so the base-level builder can extract fresh procedural sections instead of reopening a
stale deterministic proxy.

## NC-003 — Initialize reflected runtime save size

Plugin: `Plugins/VoxelEditor`

File: `Source/VoxelEditor/Public/VoxelRuntimeBPFunctionLibrary.h`

Reason: UE 5.8 validates reflected struct defaults during startup. `FRuntimeVoxelSaveData::Size` had no
initializer, causing a `LogClass` error before automation and packaged runtime initialization.

Implementation: initialize `Size` to `FIntVector::ZeroValue`; serialized saves remain compatible and new
instances now have a deterministic empty-grid default.

## NC-004 — Initialize runtime voxel mesh UV streaming metadata

Plugin: `Plugins/VoxelEditor`

File: `Source/VoxelEditor/Private/Runtime/VoxelRuntimeStaticMeshBuilder.cpp`

Reason: render-asset streaming queries UV-channel metadata when transient runtime voxel meshes register.
Their null material slot did not initialize that metadata, causing a handled ensure in
`UStaticMesh::GetUVChannelData` during every packaged-game session.

Implementation: initialize the runtime material slot with a deterministic UV density before
`BuildFromMeshDescriptions`. Geometry, voxel UVs, material overrides, and serialization are unchanged.

## NC-005 — Voice capture device selection

Plugin: `Plugins/SteamCorePro`

Files:

- `Source/OnlineSubsystemSteamCore/Public/Voice/VoiceInputDeviceSteamCore.h` (new)
- `Source/OnlineSubsystemSteamCore/Private/Voice/VoiceInputDeviceSteamCore.cpp` (new)
- `Source/OnlineSubsystemSteamCore/Public/Voice/VoiceEngineSteamCore.h`
- `Source/OnlineSubsystemSteamCore/Public/Voice/VoiceInterfaceSteamCore.h`
- `Source/OnlineSubsystemSteamCore/Private/Voice/VoiceEngineSteamCore.cpp`

Reason: the plugin already supports choosing a microphone, but only once - `FVoiceEngineSteamCore::RegisterLocalTalker`
reads `[OnlineSubsystemSteamCore] VoiceInput` from `GameUserSettings.ini` at the moment it creates the
capture object and never looks again. A settings combo box built on that alone can only offer a choice
that takes effect on the next launch. `IVoiceCapture::ChangeDevice` exists for exactly this and nothing
in the plugin calls it.

Implementation: a public `ChangeVoiceInputDevice` on `FVoiceEngineSteamCore` forwarding to
`GetVoiceCapture()->ChangeDevice`, a forwarder of the same name on `FOnlineVoiceSteamCore` (which owns the
engine pointer), and `FSteamCoreVoiceInputDevice::Apply` as the entry point game code actually calls.

The shim exists because the two voice classes are unreachable from outside the plugin: they are guarded by
`WITH_STEAMCORE`, which only propagates from `SteamLibrary` - a private dependency - and their headers use
`LogSteamCoreVerbose`, defined in the module's PRIVATE logging header. `FSteamCoreVoiceInputDevice` takes an
`IOnlineSubsystem*`, checks its name against `STEAMCORE_SUBSYSTEM`, and returns false rather than acting on
anything else, so `Cubiciousflage` compiles and behaves the same with Steam absent.

Additive only: nothing existing changed behaviour, so the plugin runs identically for anyone who never calls
`Apply`. Reapply on plugin update by re-adding the two declarations, the two definitions, and the two new
files. Consumer: `UVoxelAudioDeviceSubsystem::ApplyInputDevice`, which writes the config key regardless of
what `Apply` returns - a false return means voice capture has not started yet, and `RegisterLocalTalker`
picks the choice up from the key when it does.
