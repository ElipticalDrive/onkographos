# Voxel Block Shape & Culling Configuration

Also called: Block trait interface, Shape enum system

## Overview
This method defines the geometric topology and visibility rules for individual voxel types within a blocky meshing pipeline. It solves the problem of rendering non-cuboid blocks (e.g., slopes, fluids, chains) and managing transparency correctly by assigning each block type a specific shape geometry and a culling mode. It sets itself apart from sibling methods like [Boolean Voxel Meshing](971-boolean-voxel-meshing.md) by supporting complex multi-part geometries and distinct material layers (opaque, fluid, transparent) rather than a single binary transparency flag.

## How it works
**Inputs:** Block type identifier (e.g., `MyBlock` enum), face orientation (`AlignedFace` or `Face`), and neighbor block types for culling decisions.
**Outputs:** A list of mesh instances (vertices, UVs, normals, tints) and a culling decision for each face.

**Data Structures:**
1.  **Block Trait Interface:** A trait or interface implemented by each block type. It returns:
    *   `shape(block, face)`: The geometric primitive (e.g., `WholeBlock`, `Cross`, `Slope`).
    *   `cull_mode(block)`: The visibility rule (e.g., `Opaque`, `TransparentMerged`, `TransparentUnmerged`, `Empty`).
    *   `light(block, face)`: Per-face lighting value (float or u8).
2.  **Vertex Structure:**
    ```python
    struct Vertex {
        pos: Vec4<f32>   # Position
        uv: Vec2<f32>    # Texture coordinates
        normal: Vec3<f32> # Surface normal
        ao: f32          # Ambient occlusion (0.0 = occluded, 1.0 = lit)
        smooth_light: f32 # Smooth lighting value
        block: MyBlock   # Block type ID for render pass filtering
        atlas_offset: f32 # Pre-computed atlas tile offset
        two_sided: bool  # Flag for two-sided rendering
    }
    ```
3.  **Vertex Shader Output (`VsOut`):**
    ```python
    struct VsOut {
        uv: Vec2<f32>
        normal: Vec3<f32>
        atlas_u_offset: f32
        ao: f32
        smooth_light: f32
    }
    ```

**Steps:**
1.  **Classify Block:** For each block in the chunk, determine its `cull_mode` and `shape`.
2.  **Evaluate Culling:**
    *   If `cull_mode` is `Empty`, discard the block.
    *   If `cull_mode` is `Opaque`, render all visible faces.
    *   If `cull_mode` is `TransparentMerged(T)`, cull shared faces between two transparent blocks if their `TransparentGroup` values are equal. Note: This does not replace `PartialEq` for greedy meshing [S1, unofficial].
    *   If `cull_mode` is `TransparentUnmerged`, render all visible faces without merging transparent neighbors.
3.  **Generate Geometry:**
    *   For `WholeBlock`: Generate 6 faces.
    *   For `Slope` (Ramp Shape): Generate a slope quad (2 triangles) and two side triangles (left/right) [S2, unofficial].
    *   For `Cross` (e.g., chains): Generate specific cross-shaped geometry.
4.  **Assign Tints and Lighting:**
    *   Assign lighting values based on `light(block, face)` [S1, unofficial].
    *   Assign tints based on face orientation for slopes: `tints[0]` for Top/Front, `tints[1]` for Left, `tints[2]` for Right [S2, unofficial].
    *   If a specific tint is `Vector3.One`, fallback to another face's tint [S2, unofficial].
5.  **Frustum Culling (Optional):** Use view frustum planes to cull entire chunks or blocks. Extract planes from the view-projection matrix:
    *   Near: `Col3`
    *   Far: `Col4 - Col3`
    *   Top: `Col4 - Col2`
    *   Bottom: `Col4 + Col2`
    *   Left: `Col4 + Col1`
    *   Right: `Col4 - Col1`
    *   Check if block bounding box intersects any plane [S2, unofficial].

**Pitfalls:**
*   Setting `FLUID_ENABLED` to true without implementing fluid shape causes a debug assertion failure rather than silently drawing fluids as cubes [S1, unofficial].
*   Ensure `TransparentMerged` logic does not interfere with greedy meshing equality checks [S1, unofficial].

## Subtopics

### [Three-Valued Opacity Classification for Meshing](4414-three-valued-opacity-classification-for-meshing.md)
A classification system for block IDs into three mutually exclusive layers ('opaque', 'water', 'transparentSolid') to determine rendering material and buffer assignment. It solves the insufficiency of a single boolean 'transparent' flag by distinguishing between fluids (water) requiring bespoke shaders and see-through solids (glass/leaves) requiring alpha blending. Unlike a boolean transparent flag which collapses water and glass/leaves into one category, this three-valued model allows water to use a bespoke shader (ripple, refraction) while glass/leaves use the ordinary block atlas with alpha blending.

### Ramp Shape Geometry
A shape representing a sloped block face, composed of a slope quad (2 triangles) and two side triangles (left and right) [S2, unofficial]. Instance creation iterates visible faces; if face is Left, Right, Top, or Front, set `drawSlope = true` and assign tints. If face is Back or Bottom, add instance using `cubeBackFaceShapeID` or `cubeBottomFaceShapeID` [S2, unofficial]. Slope vertices (6 vertices): (0,1,0) with UV[0], (0,0,1) with UV[1], (1,1,0) with UV[2], (1,1,0) with UV[3], (0,0,1) with UV[4], (1,0,1) with UV[5]. Left triangle vertices (3 vertices): (0,0,1) with UV[0], (0,1,0) with UV[1], (0,0,0) with UV[2]. Right triangle vertices (3 vertices): (1,0,0) with UV[0], (1,1,0) with UV[1], (1,0,1) with UV[2] [S2, unofficial].

### Fluid Surface Stitching
A mesh generation technique where fluid columns share a surface height field with neighbors, stitching surfaces when fluid IDs match [S1, unofficial]. Two adjacent fluid blocks join their surfaces only when their `FluidInfo::id` (u8) values match [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** O(N) where N is the number of blocks, as each block is processed independently for shape and culling.
*   **Memory Complexity:** O(N) for storing block traits and generated mesh data.
*   **Strengths:** Supports complex block geometries (slopes, chains) and fine-grained transparency control.
*   **Limitations:** Requires per-block trait implementation; fluid stitching adds complexity to mesh generation.
*   **When to use:** When rendering games or simulations with non-cuboid blocks and varied transparency requirements.
*   **When not to use:** For simple voxel worlds with only opaque and fully transparent blocks.

## Background
Originated in voxel game engines (e.g., Minecraft clones) to support diverse block types beyond simple cubes. Developed by community contributors like finnbear and rarefushion for custom voxel renderers [S1, unofficial][S2, unofficial].

## Sources
[S1] finnbear/voxmesh, https://github.com/finnbear/voxmesh, fetched 2026-10-26 (unofficial)
[S2] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
