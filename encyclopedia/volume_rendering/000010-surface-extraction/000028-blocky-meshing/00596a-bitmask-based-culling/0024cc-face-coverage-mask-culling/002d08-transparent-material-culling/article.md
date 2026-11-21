# Transparent Material Culling
Also called: FaceOcclusion Culling

## Overview
Transparent Material Culling separates geometric face coverage from material occlusion behavior to correctly handle transparent or cutout materials in blocky meshing [S2, unofficial]. Unlike standard boolean solid flags that cull all shared faces between adjacent voxels, this method allows visibility through specific material layers (e.g., seeing leaves behind glass) by using compatibility rules based on transparency indices [S1]. It solves the problem where a single "solid" flag is insufficient for partial shapes or materials with mixed visible/hidden areas [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Voxel block data containing material properties (specifically `transparency_index` and `culls_neighbors` flags) and neighbor voxel states.
*   **Output:** A set of mesh faces to render, where faces between voxels are culled or retained based on material compatibility rather than simple adjacency.

**Data Structures:**
The core representation for culling decisions is the `FaceOcclusion` struct [S2, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
struct FaceOcclusion {
    coverage: FaceMask,  # Geometric coverage mask
    mode: OcclusionMode, # Material occlusion behavior
}
```
*   `coverage`: Represents the geometric presence of the face [S2, unofficial].
*   `mode`: An enum or type expressing behavior such as `opaque`, `cutout`, `translucent`, or `never-occluding` [S2, unofficial].

**VoxelBlockyModel Properties:**
*   `transparency_index`: A property on the voxel type that tunes occlusion compatibility [S1].
*   `culls_neighbors`: A boolean flag (default `true`) that determines if neighboring voxel sides are rendered [S1].

**Algorithm Steps:**
1.  **Identify Adjacent Faces:** For each voxel face shared between two adjacent voxels (Voxel A and Voxel B), retrieve their respective `transparency_index` values and `culls_neighbors` settings [S1].
2.  **Check Neighbor Rendering Policy:** If `culls_neighbors` is `false` for either voxel, the face is generally retained to ensure density (e.g., for foliage), unless other culling rules apply [S1].
3.  **Evaluate Transparency Compatibility:**
    *   If `VoxelA.transparency_index == VoxelB.transparency_index`: The faces are considered compatible for culling. The shared face is culled, allowing the viewer to see through the combined volume as if it were a single material [S1].
    *   If `VoxelA.transparency_index != VoxelB.transparency_index`: The faces are considered incompatible for culling. The shared face is retained, allowing visibility of the distinct material behind the front one (e.g., leaves behind glass) [S1].
4.  **Combine with Geometry Coverage:** The final culling decision combines the geometric coverage (`FaceMask`) with the material occlusion mode (`OcclusionMode`) [S2, unofficial]. The decision logic follows:
    $$ \text{Cull} = \text{Coverage} \land \text{MaterialCompatibility} $$
    Where `MaterialCompatibility` is determined by the transparency index equality and the `culls_neighbors` flag [S1][S2, unofficial].
5.  **Handle Missing Neighbors:** If a neighbor chunk is not available, the engine must apply a policy (e.g., treat as air, delay meshing, or use an unknown state) before applying transparency rules [S2, unofficial].

**Parameters and Typical Values:**
*   **Materials:** A typical texture atlas setup requires 3 materials: opaque, alpha clip, and transparent [S1].
*   **Transparency Index:** Integer values. Example: Glass = 2, Leaves = 1 [S1].
*   **Culls Neighbors:** Default `true` [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on integer equality for `transparency_index`. Floating-point comparisons should be avoided [S1].
*   **Boundaries:** Missing neighbor data requires explicit policy handling to avoid incorrect culling decisions at chunk boundaries [S2, unofficial].
*   **Degenerate Input:** If `culls_neighbors` is disabled, the mesh may become significantly larger, impacting performance [S1].

## Subtopics

### Face Coverage Mask Culling
The parent method that provides the geometric `FaceMask` used in `FaceOcclusion`. It handles partial shapes where boolean flags are insufficient. [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md)

### Face-Culling Geometry Shader
A GPU-based approach that may complement CPU-side transparency culling by handling post-meshing visibility. [Face-Culling Geometry Shader](3928-face-culling-geometry-shader.md)

### Per-face Column-based Face Culling with Opacity Masks
A method that uses column-based opacity masks for more granular culling, potentially integrating with transparency modes. [Per-face Column-based Face Culling with Opacity Masks](4821-per-face-column-based-face-culling-with-opacity-masks.md)

### Boundary Geometry Classification
Classifies boundary geometry to determine culling, which may inform the `coverage` aspect of `FaceOcclusion`. [Boundary Geometry Classification](5435-boundary-geometry-classification.md)

## Operating characteristics
*   **Time Complexity:** The culling decision is $O(1)$ per face, involving simple integer comparisons and bitwise operations on masks [S2, unofficial].
*   **Memory Complexity:** Minimal overhead, requiring storage of `transparency_index` per voxel type and the `FaceOcclusion` struct for culling decisions [S1][S2, unofficial].
*   **Performance:** Precomputed shape masks and small integer masks keep the common path cheap [S2, unofficial]. However, sophisticated culling schemes can increase mesh generation time if not optimized [S2, unofficial].
*   **Strengths:** Correctly handles transparent materials (glass, leaves) without visual artifacts like "hollow" interiors when `culls_neighbors` is managed appropriately [S1].
*   **Limitations:** Does not cut out faces geometrically, only culls them from rendering; requires careful material setup (alpha clip vs. alpha blend) [S1].
*   **When to Use:** When rendering voxel scenes with transparent or cutout materials where visibility through specific layers is required [S1][S2, unofficial].
*   **When Not to Use:** When performance is critical and transparency is not needed, or when a simple boolean solid flag suffices [S1].

## Background
The concept of separating geometric coverage from material occlusion was introduced to address limitations in standard boolean solid flags for blocky meshing [S2, unofficial]. The `VoxelBlockyModel` implementation with `transparency_index` and `culls_neighbors` was documented in the Voxel Tools documentation [S1]. The `FaceOcclusion` struct and the explicit separation of coverage and mode were proposed in discussions on face culling for voxel cubes and partial shapes [S2, unofficial].

## Sources
[S1] Blocky terrains - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/blocky_terrain/, fetched 2026-10-27
[S2] Face Culling for Voxel Cubes and Partial Shapes, https://ogyrecdev.github.io/posts/face-culling/, fetched 2026-10-28 (unofficial)
