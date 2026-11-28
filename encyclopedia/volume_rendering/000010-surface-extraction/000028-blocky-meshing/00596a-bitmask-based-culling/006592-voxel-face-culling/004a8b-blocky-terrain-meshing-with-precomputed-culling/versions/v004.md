# Blocky Terrain Meshing with Precomputed Culling

Also called: Minecraft-style meshing, VoxelBlockyLibrary meshing

## Overview
Blocky Terrain Meshing with Precomputed Culling is a surface extraction method for blocky terrains that optimizes mesh generation by using precomputed bitmasks to determine face occlusion, avoiding full Constructive Solid Geometry (CSG) computations at runtime [S1]. It differs from its parent [Voxel Face Culling](8747-voxel-face-culling.md) by extending simple neighbor checks to complex model-side occlusion via a baked culling matrix, and from sibling methods by relying on rasterization-based shape comparison during library baking rather than runtime geometric intersection tests [S1].

## How it works
**Inputs:** A voxel terrain grid containing blocky models and a precomputed culling library [G].
**Outputs:** A triangle mesh representing the visible surfaces of the terrain [G].

**Data Structures:**
1.  **Culling Matrix:** A precomputed matrix of bitmasks where entries indicate if one model side covers another, enabling fast occlusion queries [S1].
2.  **Model Library:** A collection of blocky models, where each model consists of a set of faces (sides) [S1].
3.  **Rotation/Flip Lookup:** A precomputed mapping per model defining its geometric transformations, enabling orientation handling via simple lookups instead of runtime matrix multiplication [S1].

**Steps:**
1.  **Library Baking (Precomputation):** Gather all unique face shapes from the entire model library. Use rasterization to compare shapes by projecting each face onto a 2D grid and comparing pixel masks. If two sides produce the same rasterized shape, they are considered matching. Compare all shapes against each other to generate the culling matrix [S1].
2.  **Precompute Transformations:** Calculate and store rotation and flip transformations for each model to eliminate runtime orientation calculations, reducing meshing work to simple lookups [S1].
3.  **Runtime Mesh Generation:** Iterate through each voxel in a terrain chunk. For each occupied voxel, retrieve its model and apply the precomputed orientation lookup. For each face of the model, check the adjacent voxel. If the adjacent voxel contains a model, retrieve its side identifier. Look up the pair (current face, adjacent face) in the culling matrix. If the matrix indicates occlusion, skip the face; otherwise, generate the triangle geometry using the precomputed transformation [G].
4.  **Output:** Collect and output the generated triangles to form the final mesh [G].

**Parameters:**
*   **Rasterization Resolution:** The grid resolution used during baking to compare face shapes. Not given in the sources.
*   **Culling Matrix Size:** Depends on the number of unique face shapes and orientations. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Transparency:** Full culling can expose underlying geometry if transparent blocks are involved, as the culling matrix assumes opaque occlusion. The method requires that model sides always match in opaque contexts (e.g., no glass slabs or stairs, water flow matching sides) to avoid visual artifacts [S1].
*   **Detailed Faces:** Using very small triangles or highly detailed faces may cause the rasterization-based shape detection to fail, leading to incorrect culling decisions [S1].

## Subtopics

### [Cube Mesh](13803-cube-mesh.md)
[13803-cube-mesh.md]: A voxel-based terrain rendering acceleration structure that replaces traditional polygonal meshes with axis-aligned cubes enclosing height field data. It uses adaptive quadtree subdivision to generate a tight bounding volume hierarchy for efficient ray casting with early termination. Choose it when convexity limitations of envelope methods are problematic and view-frustum culling/LOD selection performance is critical. It differs from siblings by precomputing transformed unit cubes to minimize CPU computation.

## Operating characteristics
*   **Time Complexity:** Mesh generation is fast due to $O(1)$ culling lookups, avoiding full CSG computations per side combination [S1]. Baking time depends on the number of unique face shapes and the cost of pairwise rasterization comparisons [S1].
*   **Memory Complexity:** Requires storage for the culling matrix and model library. Not given in the sources.
*   **Strengths:** Eliminates complex CSG operations at runtime; relies on fast lookups for face occlusion [S1].
*   **Limitations:** Inflexible with transparency, as full culling exposes underlying geometry; rasterization-based shape detection may fail on highly detailed faces [S1].
*   **When to use:** Blocky terrains with opaque models where runtime performance is critical and model sides can be made to match consistently [S1].
*   **When not to use:** Scenes requiring transparent blocks or highly detailed geometry that breaks rasterization matching [S1].

## Background
This method is implemented in the Voxel Tools engine for Godot, specifically in the `VoxelMesherBlocky` and `VoxelBlockyLibrary` classes [S1]. It is inspired by the meshing techniques used in Minecraft, which also avoid full CSG for performance reasons [S1]. The approach is experimental and subject to change [S1].

## Sources
[S1] Blocky terrains - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/blocky_terrain/, fetched 2026-10-27
