# Detached Re-voxelize

Also called: detached re-voxelize P3, re-voxelize, round-to-cell re-voxelization

## Overview
Detached re-voxelize is a rendering technique for detached solids that re-voxelizes geometry into a grid, resolves occlusion via a distance buffer, and projects to trixels [S1, unofficial]. It differs from standard voxelization by using a round-to-cell mapping for grid placement and an exposed-face mask to gate emission, specifically addressing defects in prior iterations such as coverage holes and stale-mask artifacts [S1, unofficial].

## How it works
The method operates on a pool of detached solids, flagging them for re-voxelization and recomputing their spatial occupancy every frame to match the GPU scatter cells.

**Inputs and Outputs**
*   **Input**: Model-space geometry of detached solids; current frame transformation (rotation/translation); voxel grid resolution and bounding box.
*   **Output**: A voxel grid with density/color values and an exposed-face mask used during the voxel-to-trixel projection stage.

**Data Structures**
*   **Voxel Data Structure**: Contains `visibleFaceIds`. The `.w` lane of `visibleFaceIds` is repurposed as a free marker lane to flag solids requiring re-voxelization [S1, unofficial].
*   **Face Occupancy Mask**: A per-voxel boolean or bitmask indicating if a face is exposed to the camera.

**Steps**
1.  **Flagging**: Identify solids in the detached pool that require re-voxelization. Set the `visibleFaceIds_.w` lane to a non-zero marker value [S1, unofficial].
2.  **World Cell Mapping**: Map model-space voxel coordinates to world-space grid cells. Use `roundHalfUp` rounding to ensure byte-for-byte matching with the GPU scatter cells [S1, unofficial].
    *   Formula: $C_{world} = \text{roundHalfUp}(C_{model})$
3.  **Mask Recomputation**: Recompute the per-voxel exposed face mask every frame against the rotated destination cells. This is performed by `face_occupancy::recomputeFaceOccupancyOnCells` [S1, unofficial].
    *   This step replaces the stale model-space mask with a mask derived from the current rotated grid position.
4.  **Occlusion Resolution**: In the `c_voxel_to_trixel_stage_{1,2}` (GLSL+Metal), drop the model-space `faceIsExposed` gate. Instead, rely on the distance buffer to resolve occlusion [S1, unofficial].
5.  **Projection**: Project the voxel data to trixels (2D screen pixels) using the recomputed exposed mask and distance buffer for depth testing [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def detached_revoxelize_step(solids, frame_transform, voxel_grid):
    # Step 1: Flag solids
    for solid in solids:
        if solid.requires_revoxelize:
            solid.visibleFaceIds.w = 1  # Marker flag

    # Step 2 & 3: Recompute mask against rotated cells
    # Map model-space voxels to world cells using roundHalfUp
    for voxel in voxel_grid:
        world_cell = round_half_up(voxel.model_space_coords)
        
        # Recompute face occupancy based on rotated destination cells
        # This matches the GPU scatter cells byte-for-byte
        voxel.exposed_mask = face_occupancy.recompute_face_occupancy_on_cells(
            voxel, 
            world_cell, 
            frame_transform.rotation
        )

    return voxel_grid

def voxel_to_trixel_stage(voxel_grid, distance_buffer):
    # Step 4: Drop model-space faceIsExposed gate
    # Let distance buffer resolve occlusion
    for voxel in voxel_grid:
        if voxel.exposed_mask:  # Use recomputed mask, not stale model-space mask
            color = voxel.color
            depth = voxel.depth
            
            # Distance buffer handles occlusion
            if depth < distance_buffer.get(voxel.screen_coords):
                distance_buffer.set(voxel.screen_coords, depth)
                render_trixel(voxel.screen_coords, color)
```

**Parameters**
*   **Rounding Mode**: `roundHalfUp` is required for consistency with GPU scatter [S1, unofficial].
*   **Marker Lane**: `visibleFaceIds_.w` [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Sub-cell Gaps**: Round-to-cell re-voxelization of a rotated solid leaves sub-cell surface gaps, causing coverage holes [S1, unofficial].
*   **Stale Mask Artifacts**: Using a stale model-space exposed mask causes over-emission of interior faces. This accidentally fills coverage holes but results in wrong-face colors on multi-color or sparse content [S1, unofficial].
*   **Color vs. Coverage Tension**: Recomputing the mask fixes colors but re-exposes holes. Keeping the stale mask preserves coverage but causes color artifacts [S1, unofficial].

## Subtopics

### Conservative Coverage Dilation
A coverage fix that dilates the re-voxelize face footprint so adjacent surface faces overlap and fill gaps with the correct color, resolving the tension between correct colors and zero holes [S1, unofficial]. It is fully correct but requires a raster/shader change gated to re-voxelize canvases, needing new infrastructure and Metal parity [S1, unofficial].

## Operating characteristics
*   **Performance**: The method introduces a tension between performance and visual fidelity. Keeping the stale mask (over-emission) is cheaper but causes color artifacts. Recomputing the mask is more expensive and re-exposes holes [S1, unofficial].
*   **Accuracy**: On a uniform teal cube, the method shows 0.30% of frame with max_delta 231 when recomputing the mask. On an L-prism surface, it shows +1.9% black-pixel holes [S1, unofficial].
*   **Limitations**: The method struggles with multi-color content where wrong-face colors are visible. It is less effective for high-contrast neighbors [S1, unofficial].
*   **When to Use**: Suitable for dense, uniform-color solids where coverage is prioritized over per-face color accuracy. For multi-color content, conservative coverage dilation or supersampling may be required [S1, unofficial].

## Background
Detached re-voxelize was introduced in the Irreden Engine as part of the detached re-voxelize pipeline (P3), addressing defects observed in P1 screenshots: round-to-cell coverage holes and stale-mask flag trixels [S1, unofficial]. It evolved from P1 and P2 iterations, where P2 rendered demo solids cleanly but relied on side-effect over-emission for coverage [S1, unofficial].

## Sources
[S1] render: detached re-voxelize — round-to-cell occlusion + aliasing (P3) (#1557), https://github.com/jakildev/IrredenEngine/pull/1566, fetched 2026-10-27 (unofficial)
