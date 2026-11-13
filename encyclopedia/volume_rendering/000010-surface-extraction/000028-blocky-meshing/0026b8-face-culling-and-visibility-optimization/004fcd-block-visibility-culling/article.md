# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, Visibility Culling via Normal Dot Product, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and orientation, avoiding unnecessary geometry generation when the viewing direction changes minimally [S1, S3, unofficial]. It operates at the coarse block level by comparing camera coordinates and view vectors against block normals, distinguishing itself from sibling methods that operate at the fine-grained face level within a block [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera world position $(C_x, C_y, C_z)$, camera rotation angles $(\theta_x, \theta_y)$, block grid dimensions, block size $B$.
*   **Outputs:** A set of visible blocks or a boolean mask indicating which blocks need rendering.

**Data Structures:**
*   **Block Face Visibility:** An array or enum `BlockFace` with flags: `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` [S3, unofficial].
*   **Camera State:** Previous frame's integer block coordinates and greatest direction vector [S1, unofficial].

**Steps:**

1.  **Compute Camera Block Coordinates:**
    Calculate the integer block coordinates of the camera by flooring its world position:
    $$
    C_{blockX} = \lfloor C_x / B \rfloor, \quad C_{blockY} = \lfloor C_y / B \rfloor, \quad C_{blockZ} = \lfloor C_z / B \rfloor
    $$
    [S1, unofficial]

2.  **Compute View Vector:**
    Calculate the view vector components $(v_x, v_y, v_z)$ from the camera's rotation angles $(r_x, r_y)$:
    $$
    v_x = \cos(r_x) \sin(r_y), \quad v_y = \sin(r_x), \quad v_z = \cos(r_x) \cos(r_y)
    $$
    [S1, unofficial]

3.  **Change Detection (Optimization):**
    Compare the current camera block coordinates and greatest direction with the previous frame's values. If both match, skip the index update process [S1, unofficial].

4.  **Surface Orientation and Visibility Check:**
    For each block surface:
    a.  **Calculate Center:** Compute the center of the surface as the mean of its vertices [S3, unofficial].
    b.  **Calculate Average Normal:** Compute the average normal as the mean of its vertex normals and normalize it [S2, unofficial].
    c.  **Compute View Direction:** Compute the view direction from the center to the camera position and normalize it [S2, unofficial].
    d.  **Dot Product Test:** If the dot product of the average normal and the view direction is greater than 0, the surface is considered visible [S2, unofficial].

    Alternatively, for specific implementations using screen-space projection:
    a.  **Transform to Camera Space:**
        $$
        cs = \cos(\text{player.angle} \cdot \pi/180), \quad sn = \sin(\text{player.angle} \cdot \pi/180)
        $$
        $$
        \text{blockX} = \text{chunkX} + x \cdot B, \quad \text{blockY} = \text{chunkY} + y \cdot B
        $$
        $$
        \text{blockZBottom} = \text{chunkZ} + z \cdot B, \quad \text{blockZTop} = \text{blockZBottom} + B
        $$
        [S3, unofficial]
    b.  **Determine Orientation:**
        $$
        \text{surfaceOrientation} = \begin{cases} 
        1 & \text{if } \text{player.z} < \text{blockZBottom} \\
        2 & \text{if } \text{player.z} > \text{blockZTop} \\
        0 & \text{otherwise}
        \end{cases}
        $$
        [S3, unofficial]
    c.  **Culling Decision:**
        Check `block->visibleFaces[BlockFace::...]` before rendering. If the face is not visible, skip rendering [S3, unofficial].

**Parameters:**
*   **Surface Scale Multiplier:** 120 (used as `surfaceScale * 120` for tile calculation) [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point errors in dot product calculations may cause flickering on boundary cases where the dot product is near zero.
*   **Boundaries:** Out-of-bounds checks are required for camera positions outside the voxel grid.
*   **Degenerate Input:** If the camera is exactly aligned with a face normal, the dot product is exactly 1 or -1, requiring robust comparison logic.

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility at the face level within a block using geometric relationships between face normals and view direction. Chosen when fine-grained culling is needed within a single block, unlike the coarse block-level approach of this method.

## Operating characteristics
*   **Time Complexity:** $O(1)$ for change detection if coordinates match; $O(N)$ for $N$ blocks if full recalculation is required.
*   **Memory Complexity:** $O(1)$ additional memory for state comparison; $O(N)$ for storing visibility flags if precomputed.
*   **Performance:** Significantly reduces mesh updates when camera movement is minimal [S1, unofficial].
*   **Strengths:** Simple to implement, effective for static scenes or slow-moving cameras.
*   **Limitations:** Does not account for occlusion by other blocks, only visibility relative to camera orientation.
*   **When to Use:** Ideal for voxel engines where camera movement is discrete or slow, and mesh generation is expensive.
*   **When Not to Use:** In dynamic scenes with fast camera movement or complex occlusion scenarios where face-level culling is more appropriate.

## Background
Originating from early voxel rendering techniques, this method was popularized in projects like Exolands and Voxel1993 [S1, S3, unofficial]. It leverages the simplicity of block-based worlds to optimize rendering by avoiding unnecessary geometry updates.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] matthxie/NeRF-Renderer, https://github.com/matthxie/NeRF-Renderer, fetched 2026-10-27 (unofficial)
[S3] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
