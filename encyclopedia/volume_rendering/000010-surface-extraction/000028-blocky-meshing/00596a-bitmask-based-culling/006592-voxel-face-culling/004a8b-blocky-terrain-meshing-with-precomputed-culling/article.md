# Blocky Terrain Meshing with Precomputed Culling

Also called: Minecraft-style meshing, VoxelBlockyLibrary meshing

## Overview
Blocky Terrain Meshing with Precomputed Culling is a surface extraction method for blocky terrains that optimizes mesh generation by using precomputed bitmasks to determine face occlusion, avoiding full Constructive Solid Geometry (CSG) computations at runtime [S1]. It differs from its parent [Face Culling](13637-face-culling.md) by extending simple neighbor checks to complex model-side occlusion via a baked culling matrix, and from siblings like [Bitmask-based Culling](16406-bitmask-based-culling.md) by relying on rasterization-based shape comparison during library baking rather than runtime geometric intersection tests [S1].

## How it works
**Inputs:** A voxel terrain grid containing blocky models (e.g., slabs, stairs, custom shapes) and a precomputed culling library.
**Outputs:** A triangle mesh representing the visible surfaces of the terrain.

**Data Structures:**
1.  **Culling Matrix:** A 2D array or hash map where keys are pairs of model side identifiers (including rotation/flipping states) and values are boolean flags or bitmasks indicating if one side occludes the other.
2.  **Model Library:** A collection of blocky models, where each model consists of a set of faces (sides). Each face is defined by its geometry (vertices/indices) and its shape signature.
3.  **Rotation/Flip Lookup:** A precomputed mapping for each model that defines its geometric transformation for all 24 possible orientations (6 faces $\times$ 4 rotations) [S1].

**Steps:**
1.  **Library Baking (Precomputation):**
    1.  Gather all unique face shapes from the entire model library.
    2.  For every pair of face shapes (including their rotated/flipped variants), determine if they are geometrically identical or if one occludes the other.
    3.  Use rasterization to compare shapes: project the 2D footprint of each face onto a grid and compare pixel masks. If two sides have the same rasterized shape, they are considered matching [S1].
    4.  Populate the **Culling Matrix** with the results. If side $A$ occludes side $B$, mark the entry $(A, B)$ as occluded. This matrix allows $O(1)$ lookup during meshing [S1].
    5.  Precompute the rotation and flip transformations for each model to enable simple lookups during mesh generation, eliminating the need for runtime matrix multiplication for orientation [S1].

2.  **Mesh Generation (Runtime):**
    1.  Iterate through each voxel in the terrain chunk.
    2.  For each occupied voxel, identify its model and the required orientation based on its position and any transformations.
    3.  For each face of the model:
        a.  Check if the adjacent voxel is empty. If not, the face is potentially occluded.
        b.  If the adjacent voxel contains a model, retrieve the side identifier of the adjacent model's face.
        c.  Look up the pair (current face, adjacent face) in the **Culling Matrix**.
        d.  If the matrix indicates occlusion, skip generating the triangle for this face. Otherwise, generate the geometry using the precomputed transformation [S1].
    3.  Output the collected triangles.

**Parameters:**
*   **Rasterization Resolution:** The grid resolution used during baking to compare face shapes. Higher resolution improves accuracy but increases baking time. Not given in the sources.
*   **Culling Matrix Size:** Depends on the number of unique face shapes and orientations. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Transparency:** Full culling can expose underlying geometry if transparent blocks are involved, as the culling matrix assumes opaque occlusion. The method requires that model sides always match in opaque contexts (e.g., no glass slabs) to avoid visual artifacts [S1].
*   **Detailed Faces:** Using very small triangles or highly detailed faces may cause the rasterization-based shape detection to fail, leading to incorrect culling decisions [S1].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Mesh generation is $O(N)$ where $N$ is the number of voxel faces, with $O(1)$ culling checks per face due to the precomputed matrix. Baking time is $O(M^2)$ where $M$ is the number of unique face shapes, due to pairwise rasterization comparisons [S1].
*   **Memory Complexity:** Requires storage for the Culling Matrix and the Model Library. The matrix size is proportional to the square of the number of unique face orientations. Not given in the sources.
*   **Strengths:** Extremely fast runtime meshing due to lookup-based culling; eliminates complex CSG operations [S1].
*   **Limitations:** Inflexible with transparency; baking time can be significant for large libraries; rasterization inaccuracies for complex shapes [S1].
*   **When to use:** Blocky terrains with opaque models where runtime performance is critical and model variety is limited [S1].
*   **When not to use:** Scenes with transparent blocks or highly detailed, non-rasterizable geometry [S1].

## Background
This method is implemented in the Voxel Tools engine for Godot, specifically in the `VoxelMesherBlocky` and `VoxelBlockyLibrary` classes [S1]. It is inspired by the meshing techniques used in Minecraft, which also avoid full CSG for performance reasons [S1]. The approach is experimental and subject to change [S1].

## Sources
[S1] Blocky terrains - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/blocky_terrain/, fetched 2026-10-27
