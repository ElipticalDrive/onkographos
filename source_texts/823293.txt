# Procedural Voxel World

![Unity](https://img.shields.io/badge/Unity-2022+-000000?logo=unity&logoColor=white)
![C Sharp](https://img.shields.io/badge/C%23-Programming-512BD4?logo=csharp&logoColor=white)
![Project](https://img.shields.io/badge/Project-Bachelor's%20Thesis-2563EB)

A Minecraft-inspired voxel sandbox developed in **Unity** and **C#** as my Bachelor's thesis in Computer Science at the University of Bucharest.

The project explores procedural generation, chunk-based world management and runtime mesh construction while providing a playable first-person environment.

## Features

- Procedural terrain generated with multi-octave Perlin noise
- Domain warping for more varied and natural terrain
- Chunk-based world generation using global coordinates
- Continuous terrain across chunk boundaries
- Runtime voxel mesh generation
- Hidden-face removal to avoid rendering internal geometry
- Configurable blocks, terrain layers and biomes
- Procedural tree placement based on terrain data
- First-person movement and camera controls
- Block placement and destruction
- Inventory and item system
- Data-driven configuration with Unity ScriptableObjects

## Generation pipeline

```text
Noise configuration
        |
        v
Perlin noise + domain warping
        |
        v
Terrain and biome rules
        |
        v
Global voxel data
        |
        v
Chunk generation
        |
        v
Visible-face mesh construction
        |
        v
Playable Unity world
```

Noise is sampled in global world coordinates, keeping the generated terrain consistent between neighboring chunks. Each chunk stores its voxel data independently and constructs a mesh containing only faces exposed to air.

## Main systems

| System | Responsibility |
| --- | --- |
| World generation | Coordinates chunks and global voxel data |
| Terrain generation | Produces terrain height and shape from noise |
| Domain warping | Distorts noise coordinates to reduce repetition |
| Chunk renderer | Builds optimized voxel meshes at runtime |
| Biome system | Selects terrain composition and block layers |
| Tree generation | Places vegetation using generated terrain data |
| Player controller | Handles movement, camera and input |
| Inventory | Manages items and inventory slots |

## Technical concepts

- Object-oriented programming
- Procedural generation
- Perlin noise and domain warping
- Voxel data structures
- Chunk coordinate systems
- Runtime mesh construction
- Basic mesh optimization
- Unity ScriptableObjects

## Repository structure

```text
Scripts/
|-- BlockLayers/       Terrain composition rules
|-- DomainWarping/     Noise distortion components
|-- Player/            Movement, camera and input
|-- Trees/             Procedural vegetation
|-- Inventory/         Items and inventory management
|-- Chunk.cs           Chunk lifecycle and voxel data
|-- ChunkRenderer.cs   Runtime mesh construction
|-- TerrainGenerator.cs
|-- BiomeGenerator.cs
`-- World.cs           World and chunk coordination
```

## Running the project

This repository contains the Unity assets and source code used for the thesis project rather than a packaged executable.

1. Create or open a compatible Unity 3D project.
2. Copy the repository contents into the project's `Assets` directory.
3. Open the included scene.
4. Configure the terrain parameters in the Unity Inspector if necessary.
5. Enter Play Mode.

## Academic context

Developed as a Bachelor's thesis to study the algorithms and software architecture behind a continuous procedurally generated voxel world.

## Author

**Silviu Andrei Popa**  
Computer Science graduate, University of Bucharest
