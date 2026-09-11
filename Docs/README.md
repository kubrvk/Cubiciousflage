# Voxel Asset Plugin Documentation

This repository contains the documentation for the Voxel Asset Plugin, a voxel asset creation and runtime destruction system for Unreal Engine.

The plugin provides:

- A dedicated Voxel Asset Editor
- Sparse runtime voxel data with shared visual/collision templates
- Optional Nanite runtime rendering for shared clusters and baked editor anchors
- Standard StaticMesh runtime rendering with distance-based streaming
- Runtime destruction, damage, and save/load APIs
- Structural physics with fragments, an anchor face, and an optional collapse solver
- Multi-asset structures, so a building made of one asset per floor collapses as one building
- A Voxel Merge tool that bakes voxel meshes from the level into one new asset, like Merge Actors
- Multiplayer replication for destruction and damage, with server-authoritative shatter debris
- A global Niagara VFX spawner driven by voxel edits
- Built-in performance tracking for sparse edits, streaming states, and runtime overrides
- Blueprint API for gameplay integration

The documentation covers editor tools, project settings, runtime usage, Blueprint functions, streaming behavior, physics and destruction, multi-asset structures, networking, VFX, performance guidelines, and generated content paths.

## Source Compatibility

- Current Engine Version: UE 5.8
- Current Plugin Version: VE 2.0.0
- Current Demo Project Version: 2.0.0.0
- Supported Platform in this project package: Win64

## Documentation Index

- [Getting Started](01-Getting-Started.md)
- [Project Settings](02-Project-Settings.md)
- [Voxel Asset Editor](03-Voxel-Asset-Editor.md)
- [Runtime Usage](04-Runtime-Usage.md)
- [Blueprint API](05-Blueprint-API.md)
- [Streaming Subsystem](06-Streaming-Subsystem.md)
- [Animation System](07-Animation.md)
- [Performance](08-Performance.md)
- [Generated Content Paths](09-Content-Paths.md)
- [Nanite Runtime Pipeline](10-Nanite-Runtime-Pipeline.md)
- [Physics and Destruction](11-Physics-and-Destruction.md)
- [Networking and Replication](12-Networking-Replication.md)
- [VFX and Particle System](13-VFX-Particle-System.md)
- [Voxel Structures (Multi-Asset Buildings)](14-Voxel-Structures.md)
- [Voxel Merge Tool](15-Voxel-Merge-Tool.md)

## Requirements

- Unreal Engine 5.8
- Win64 editor/runtime target
- DirectX 12 and SM6 when using the Nanite runtime backend
- Editor build with support for custom asset editors
- Blueprint or C++ project

## Notes

All runtime examples assume the sparse runtime pipeline introduced in VE 1.2.0. The demo project is currently configured to use Nanite runtime rendering by default. Physics, replication, and the global VFX spawner were added in VE 2.0.0.
