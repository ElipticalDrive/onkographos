# Blocky Terrain Meshing with Precomputed Culling

Also called: Minecraft-style meshing, VoxelBlockyLibrary meshing

## Overview
Blocky terrain meshing with precomputed culling is a surface extraction method for voxel-based environments that optimizes mesh generation by avoiding per-chunk Constructive Solid Geometry (CSG) operations. Instead of computing visibility dynamically for every voxel face, it relies on a precomputed matrix of bitmasks to determine if one model side occludes another, enabling fast culling during the baking phase [S1]. This approach significantly reduces the computational load during runtime meshing by replacing complex geometric checks with simple table lookups [S1].

## How it works
The method operates in two distinct phases: a library baking phase (offline) and a runtime meshing phase. It assumes a library of discrete "blocky" models (e.g., cubes, stairs, slabs) where each model consists of a set of triangles forming its visible sides.

### Inputs and Outputs
*   **Input:** A library of blocky models, where each model is defined by its constituent triangles and its orientation.
*   **Output:** A "culling matrix" (precomputed bitmasks) and a runtime mesher that uses this matrix to generate triangle lists for chunks.

### Data Structures
1.  **Culling Matrix:** A precomputed data structure (typically a 2D array or hash map) where entries indicate if one model side covers or occludes another. The exact internal layout is not specified, but it functions as a lookup table for fast boolean checks [S1].
2.  **Model Definition:** Each model in the library is preprocessed with its rotation and flipping states precomputed to eliminate runtime transformation overhead [S1].

### Steps

#### Phase 1: Library Baking (Precomputation)
1.  **Precompute Transformations:** For each model in the library, precompute all necessary rotation and flipping states. This ensures that runtime operations boil down to simple lookups rather than geometric transformations [S1].
2.  **Rasterization-Based Shape Comparison:** To determine occlusion relationships, the engine uses rasterization to check if two sides of different models (or different orientations of the same model) have the same shape.
    *   Gather all shapes from the entire library.
    *   Compare all shapes against each other using rasterization to identify identical or occluding profiles [S1].
3.  **Generate Culling Matrix:** Based on the rasterization results, generate the culling matrix. This matrix encodes which faces cover others, allowing the mesher to quickly decide if a face is exposed or hidden [S1].

#### Phase 2: Runtime Meshing
1.  **Lookup Visibility:** For each voxel in the chunk, determine its model type and orientation.
2.  **Apply Culling Matrix:** Use the precomputed culling matrix to check if the faces of the current voxel are occluded by adjacent voxels or by the model's own geometry.
    *   If a face is covered according to the matrix, it is culled.
    *   If not, the face is added to the mesh [S1].
3.  **Generate Mesh:** Assemble the visible faces into a triangle list for the GPU.

### Formulas
No explicit mathematical formulas are provided in the sources. The core operation is a boolean lookup:
$$ \text{IsVisible}(Face_i) = \neg \text{CullingMatrix}[Face_i, \text{Occluder}_j] $$
Where $\text{CullingMatrix}$ returns true if $Face_i$ is occluded by $\text{Occluder}_j$ [S1].

### Parameters and Typical Values
*   **Rasterization Resolution:** Not specified. However, using very small triangles or highly detailed faces during the baking phase may cause the rasterization-based shape detection to fail or produce incorrect culling results [S1].

### Edge Cases and Pitfalls
1.  **Transparency:** Full culling is problematic for transparent materials because it exposes underlying geometry that should be visible through the transparent surface. The method does not support transparency well; the only avoidance is ensuring that model sides always match (e.g., no glass slabs or stairs in Minecraft) [S1].
2.  **Detailed Geometry:** If models rely on very small triangles or very detailed faces, the rasterization-based shape detection used during baking may not work perfectly, leading to incorrect culling decisions [S1].
3.  **Experimental Status:** The implementation described (specifically `VoxelMesherBlocky` and `VoxelBlockyLibrary`) is marked as experimental, with potential bugs, missing parts, and future changes [S1].

## Subtopics

### [Cube Mesh](13803-cube-mesh.md)
A data structure consisting of a set of cubes that fully enclose a height field terrain, generated via adaptive quadtree subdivision to solve the convexity problem of the Envelope method.

## Operating characteristics
*   **Time Complexity:** Runtime meshing is fast because it relies on precomputed lookups rather than dynamic CSG or per-voxel rasterization [S1].
*   **Memory Complexity:** Requires storing the culling matrix and precomputed model transformations. The size depends on the number of models and their sides.
*   **Strengths:** Eliminates the need for expensive CSG operations on every possible combination of sides during runtime [S1].
*   **Limitations:** Does not support transparency well; requires careful model design to ensure sides match; rasterization-based baking may fail with highly detailed geometry [S1].
*   **When to Use:** Suitable for opaque, blocky terrains where performance is critical and transparency is not required.
*   **When Not to Use:** Not suitable for scenes requiring transparent blocks or highly detailed, non-blocky geometry.

## Background
This method is implemented in the Voxel Tools engine for Godot, specifically within the `VoxelMesherBlocky` and `VoxelBlockyLibrary` classes. It is inspired by the meshing techniques used in Minecraft, where full CSG is avoided in favor of precomputed visibility checks [S1]. The approach is part of an experimental feature set aimed at automating model organization for voxel games [S1].

## Sources
[S1] Blocky terrains - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/blocky_terrain/, fetched 2026-10-27
