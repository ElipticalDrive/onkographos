# Crossed Vegetation Rendering

Also called: Crossed cutout planes, Crossed planes

## Overview
Crossed Vegetation Rendering renders vegetation (grass, ferns, bushes, flowers, vines) as two intersecting, double-sided alpha-cutout planes instead of solid cubes [S1, unofficial]. This approach avoids changing atlas dimensions or shader pipelines while preventing the visual artifacts caused by solid occluders [S1, unofficial]. It differs from [Block Mesh Alpha Rendering](3569-block-mesh-alpha-rendering.md) by using specific planar geometry for vegetation rather than volumetric alpha blending, and from [Quality Enhancement (Smoothing and Tessellation)](10395-quality-enhancement-smoothing-and-tessellation.md) by relying on discrete geometric primitives rather than mesh refinement [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid containing voxel types, including a set of vegetation types defined in `CROSS_PLANT_TYPES`.
*   **Output:** A mesh containing crossed planes for vegetation voxels and standard cube faces for other voxels.

**Data Structures:**
*   `CROSS_PLANT_TYPES`: A set of integer voxel IDs representing vegetation types (e.g., grass, ferns, bushes, flowers, vines) [S1, unofficial].
*   `VoxelType`: An enumeration of voxel types, including `LEAVES`, `GLASS`, and `W` (water) [S1, unofficial].

**Steps:**
1.  **Voxel Classification:** For each voxel in the chunk, check if it belongs to the vegetation set:
    ```python
    is_cross_plant = voxel in CROSS_PLANT_TYPES
    ```
2.  **Mesh Generation Routing:**
    *   If `is_cross_plant` is true, route the voxel to the cutout mesh path [S1, unofficial].
    *   Emit two intersecting, double-sided alpha-cutout planes for the voxel [S1, unofficial].
    *   Apply type-specific tint and selected height values to the vertices [S1, unofficial].
    *   Reuse the existing foliage atlas tile and cutout material [S1, unofficial].
    *   If `is_cross_plant` is false, generate the standard six faces of a cube [S1, unofficial].
3.  **Occlusion Logic Update:** Modify neighbor-occlusion checks to exclude crossed plants from occluding adjacent cube or water faces. This prevents visual holes where the crossed planes do not cover the removed face [S1, unofficial].

**Formulas:**
The occlusion check determines if a neighbor `nb` blocks the current face. The variable `nbIsCrossPlant` is defined as:
$$ nbIsCrossPlant = nb \in CROSS\_PLANT\_TYPES $$

The occlusion conditions for different render classes are:

1.  **Cutout Render Class:**
    $$ \text{occludes} = (nb \neq 0) \land (nb \neq \text{VoxelType.LEAVES}) \land (\neg nbIsCrossPlant) $$

2.  **Translucent Render Class:**
    $$ \text{occludes} = (nb = \text{VoxelType.GLASS}) \lor ((nb \neq 0) \land (nb \neq \text{VoxelType.LEAVES}) \land (\neg nbIsCrossPlant)) $$

3.  **Opaque Render Class:**
    $$ \text{occludes} = (nb \neq 0) \land (nb \neq \text{VoxelType.GLASS}) \land (nb \neq \text{VoxelType.LEAVES}) \land (\neg nbIsCrossPlant) $$

The water-face occlusion helper `occ(v)` is updated to:
$$ occ(v) = (v \neq 0) \land (v \neq \text{VoxelType.GLASS}) \land (v \neq \text{VoxelType.LEAVES}) \land (v \neq W) \land (\neg (v \in CROSS\_PLANT\_TYPES)) $$

**Parameters:**
*   `CROSS_PLANT_TYPES`: Set of voxel IDs for vegetation. Typical values include IDs for grass, ferns, bushes, flowers, and vines [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Visual Holes:** If crossed plants are treated as full occluders, they hide adjacent cube or water faces, creating localized visual holes because the crossed planes do not cover the removed face [S1, unofficial].
*   **Light Propagation:** Vegetation should be kept in the cutout/depth rendering path and made non-occluding for light propagation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The method reuses existing foliage atlas tiles and cutout materials, avoiding changes to atlas dimensions or shader pipelines [S1, unofficial].
*   **Strengths:** Prevents visual holes caused by occlusion; simple geometric representation for vegetation; no shader pipeline changes required [S1, unofficial].
*   **Limitations:** Does not support distinct per-plant resource-pack textures, multi-block flower models, or non-solid interaction/collision behavior in this implementation [S1, unofficial].
*   **When to use:** When rendering vegetation in a voxel engine that uses cutout rendering and wants to avoid the visual artifacts of solid occluders without modifying the shader pipeline [S1, unofficial].
*   **When not to use:** When distinct per-plant textures or complex collision behavior is required [S1, unofficial].

## Background
This method was introduced in the Physicscraft project to add a wider variety of voxel blocks, including wood, plants, flowers, terrain, Nether blocks, ores, and utilities [S1, unofficial]. The implementation was part of a pull request that added voxel type IDs 15–120 and modified the meshing logic in `src/rendering/VoxelWorld.ts` and `src/types/physics.ts` [S1, unofficial].

## Sources
[S1] Add voxel block types and crossed vegetation rendering, https://github.com/Debarghyalol/Physicscraft/pull/43, fetched 2026-10-26 (unofficial)
