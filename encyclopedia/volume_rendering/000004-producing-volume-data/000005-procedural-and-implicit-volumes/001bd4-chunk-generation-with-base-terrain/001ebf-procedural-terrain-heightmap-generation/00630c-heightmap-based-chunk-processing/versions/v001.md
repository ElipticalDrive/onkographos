# Heightmap-based Chunk Processing


## Overview
Heightmap-based Chunk Processing generates terrain data by dividing the world into spatial chunks, computing a 2D heightmap for each chunk, and converting that heightmap into either volumetric data or mesh geometry. It solves the problem of efficiently managing large-scale procedural terrain by decoupling surface generation from volumetric or geometric representation, allowing for independent culling, LOD management, and memory optimization per chunk. It differs from direct 3D noise evaluation by using a 2D height field as an intermediate representation, which simplifies the conversion to voxels or meshes and enables efficient boundary handling.

## Shared principles
All child methods in this family share the following mechanism:
1.  **Chunk Decomposition**: The world is divided into fixed-size or adaptive spatial chunks. Each chunk is identified by integer coordinates $(x, z)$ and potentially a Level of Detail (LOD) index.
2.  **Heightmap Generation**: For each chunk, a 2D grid of height values is generated. This grid has dimensions $PCS \times PCS$ (Pixels/Vertices per Chunk Size). The generation typically involves sampling a noise function (e.g., FBM) at specific world coordinates scaled by the chunk's LOD level.
3.  **Coordinate Mapping**: A consistent mapping exists between chunk-local indices $(i, j)$ and world coordinates $(x, z)$. For a chunk at $(cx, cz)$ with scale $S$, the world coordinate for sample $(i, j)$ is often $(cx \cdot S + i, cz \cdot S + j)$ or similar, depending on the specific implementation's origin and stride.
4.  **Boundary Handling**: To ensure seamless transitions between chunks, methods often require sampling data that extends beyond the immediate chunk boundaries (e.g., using an $(N+2) \times (N+2)$ grid for an $N \times N$ mesh) or sharing border values between adjacent chunks.
5.  **State Management**: Chunks maintain state flags (e.g., `DataFlagsTerrainGenDirty`) and counters (e.g., `generation`) to manage asynchronous generation tasks, detect stale requests, and avoid redundant processing when the camera moves rapidly.

## Subtopics

### [Chunk-based Terrain Generation and Culling](15193-chunk-based-terrain-generation-and-culling.md)
Generates terrain geometry using a hierarchical octree structure. It decouples heightmap generation from mesh creation, using culling based on camera distance and LOD thresholds. It adapts resolution dynamically based on viewer proximity.

### [Chunk Filling with Height Map](15196-chunk-filling-with-height-map.md)
Populates a 3D voxel chunk with solid material based on a 2D height map. It treats the height map as a surface, filling voxels below it. It differs by focusing strictly on vertical extrusion from a base plane for volumetric ray marching.

### [Chunk-based Terrain Mesh Generation](18521-chunk-based-terrain-mesh-generation.md)
Creates a 3D terrain mesh from a procedural noise function by dividing the area into spatial chunks. It explicitly manages mesh topology and normal calculation across chunk boundaries using border samples to ensure seamless normals.

## Background
This family of methods is commonly used in real-time game engines and procedural terrain systems to handle large-scale environments. The approach allows for CPU-side generation of terrain data, freeing GPU resources for rendering. It is particularly effective when combined with asynchronous task queues and spatial culling techniques. The specific implementations cited here originate from various game engine developments and procedural generation libraries, often adapting standard noise-based terrain techniques to a chunked architecture for performance.

## Sources
