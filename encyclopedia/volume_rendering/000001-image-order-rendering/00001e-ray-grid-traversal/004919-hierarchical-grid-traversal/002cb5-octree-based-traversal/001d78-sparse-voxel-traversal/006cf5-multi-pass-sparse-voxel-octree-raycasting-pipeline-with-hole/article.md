# Multi-pass Sparse Voxel Octree Raycasting Pipeline with Hole Filling

Also called: OpenCL SVO Raycasting Driver, Raycast Pipeline with Hole Detection

## Overview
This method is a multi-pass rendering pipeline that projects rays to screen space, performs coarse and fine raycasting on a Sparse Voxel Octree (SVO), detects unfilled pixels ("holes"), and iteratively fills them using a hole buffer and ID tracking system [S1, unofficial]. It distinguishes itself from standard single-pass SVO raycasting by explicitly separating the initial raycast from a subsequent hole-detection and filling phase, allowing for adaptive refinement of sparse regions without re-traversing the entire tree for every pixel [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** SVO data structure (compact array), camera parameters (origin, basis vectors), rendering resolution ($res_x, res_y$), octree depth.
*   **Outputs:** Final color buffer (screen buffer), depth/accumulation buffers.

**Data Structures**
The implementation uses global OpenCL buffers with specific layouts [S1, unofficial]:
*   `mem_octree`: Compact SVO data. Size: `octree_array_compact.size() * 4` bytes.
*   `mem_backbuffer`: Size: `size * 16 * 4` bytes.
*   `mem_screenbuffer`: Size: `size * 4 * 4` bytes.
*   `mem_stack`: Size: `32 * 4 * 1024 * 1024` bytes.
*   `mem_idbuffer`: Size: `(WINDOW_WIDTH_MAX * WINDOW_HEIGHT_MAX + (WINDOW_WIDTH_MAX / 16) * (WINDOW_HEIGHT_MAX / 16)) * 4` bytes.
*   `mem_x`, `mem_y`, `mem_z`: Size: `4 * numpixels` bytes each.
*   `mem_bvh_nodes`, `mem_bvh_childs`: Mentioned as commented out in the source, implying they are not used in this specific pipeline [S1, unofficial].

**Parameters**
*   `octree_dim`: Dimension of the octree space, computed as $1 \ll \text{OCTREE\_DEPTH}$ [S1, unofficial].
*   `res_x`, `res_y`: Current rendering resolution [S1, unofficial].
*   `v0`, `vx`, `vy`, `vz`: Ray origin and basis vectors (right, up, forward) transformed by the view matrix. Passed as `cl_float[4]` arrays to kernels [S1, unofficial].

**Screen-Space Ray Gradients**
To compute ray directions for screen-space traversal, the following gradients are calculated [S1, unofficial]:
$$
a_{leftTop} = \text{direction} \times 2.0f - \text{right} - \text{up}
$$
$$
a_{dx} = \frac{\text{right} \times 2.0f}{\text{float}(res\_x)}
$$
$$
a_{dy} = \frac{\text{up} \times 2.0f}{\text{float}(res\_y)}
$$
Where `direction` is the camera forward vector, `right` and `up` are camera basis vectors, and `res_x`/`res_y` are pixel dimensions [S1, unofficial].

**Pipeline Steps**
The rendering process follows a strict sequence of kernel launches [S1, unofficial]:

1.  **Clear Screen Buffer:** Initialize the screen buffer to zero (with alpha mask `0xffffff00`).
    $$ \text{ocl\_memset}(mem\_screenbuffer, 0, 0xffffff00, res\_x \times res\_y \times 4) $$
2.  **Compute Ray Origins/Directions:** Prepare ray data for the current frame.
3.  **Coarse Raycast:** Launch `raycast_proj` kernel twice in a loop ($i=0..1$) with buffer source offset $(i+1) \times res\_x \times res\_y$. This performs the initial coarse traversal.
4.  **Compute View Plane Gradients:** Calculate $a_{dx}$ and $a_{dy}$ as defined above.
5.  **Hole Detection:**
    *   Launch `raycast_counthole` at grid resolution $res\_x/16 \times res\_y/16$ to identify potential holes.
    *   Launch `raycast_sumids` at resolution $1 \times 1$ to compute `idbuf_size`, the total number of holes to fill.
6.  **Hole Filling (Conditional):** If `idbuf_size > 0`, launch `raycast_holes` with `idbuf_size` threads to process the identified holes.
7.  **Fine Raycast:** Launch `raycast_fine_2` at resolution $res\_x/8 \times res\_y/4$ with additional offsets `add_x` and `add_y` to refine the image.
8.  **Copy Buffers:** Transfer intermediate results between buffers.
9.  **Final Fill:** Launch `raycast_fillhole2` at full resolution $res\_x \times res\_y$ to composite the final image.

**Pseudocode**
The following Python-like pseudocode represents the control flow of the pipeline. It assumes standard OpenCL-like buffer management and kernel launch semantics [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_pass_svo_raycasting_pipeline(octree_data, camera, resolution):
    res_x, res_y = resolution
    octree_dim = 1 << OCTREE_DEPTH
    
    # 1. Clear screen buffer
    # ocl_memset(mem_screenbuffer, 0, 0xffffff00, res_x * res_y * 4)
    screenbuffer = [0] * (res_x * res_y * 4)
    
    # 2. Compute ray origins/directions
    # v0, vx, vy, vz are transformed by view matrix
    v0 = camera.origin
    vx = camera.right
    vy = camera.up
    vz = camera.forward
    
    # 3. Launch raycast_proj kernel twice
    for i in range(2):
        buffer_src_ofs = (i + 1) * res_x * res_y
        # launch raycast_proj with buffer_src_ofs
        pass 
        
    # 4. Compute view plane gradients
    a_leftTop = vz * 2.0 - vx - vy
    a_dx = vx * 2.0 / float(res_x)
    a_dy = vy * 2.0 / float(res_y)
    
    # 5. Count holes
    # Launch raycast_counthole at res_x/16 x res_y/16
    # Launch raycast_sumids at 1x1 to get idbuf_size
    idbuf_size = compute_hole_count(res_x, res_y)
    
    # 6. If idbuf_size > 0, launch raycast_holes
    if idbuf_size > 0:
        # launch raycast_holes at idbuf_size threads
        pass
        
    # 7. Launch raycast_fine_2 at res_x/8 x res_y/4 with offset add_x/add_y
    # launch raycast_fine_2 with res_x/8, res_y/4, add_x, add_y
    pass
    
    # 8. Copy buffers
    # Copy intermediate buffers to final storage
    pass
    
    # 9. Launch raycast_fillhole2 at res_x x res_y
    # launch raycast_fillhole2 with res_x, res_y
    pass
    
    return screenbuffer
```

**Edge Cases and Pitfalls**
*   **Camera Position Wrapping:** To prevent the camera from leaving the octree bounds, coordinates are wrapped using modulo-like arithmetic [S1, unofficial]:
    ```python
    while p.x < 0:
        p.x += octree_dim * 2
    while p.x >= octree_dim * 2:
        p.x -= octree_dim * 2
    ```
    This applies to x, y, and z coordinates [S1, unofficial].
*   **Precision:** The use of fixed-point or integer-based hole counting (`idbuf_size`) requires careful management of buffer sizes to avoid overflow, especially at high resolutions [S1, unofficial].
*   **Boundary Conditions:** The hole detection grid resolution ($res\_x/16 \times res\_y/16$) may miss small holes if the resolution is low or the hole density is high [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The pipeline involves multiple passes, including coarse raycasting, hole detection, and fine raycasting. The complexity depends on the number of holes and the depth of the octree. The hole detection step reduces the number of pixels requiring fine traversal, potentially improving performance in sparse scenes [S1, unofficial].
*   **Memory Complexity:** Requires significant global memory for buffers (`mem_octree`, `mem_stack`, `mem_idbuffer`, etc.). The stack size is fixed at 32MB, which may limit the depth of traversal for complex scenes [S1, unofficial].
*   **Performance:** Measured performance is not explicitly provided in the sources. However, the multi-pass approach allows for adaptive refinement, which can be more efficient than uniform fine raycasting for sparse data [S1, unofficial].
*   **Strengths:** Explicit hole filling ensures no pixels are left unfilled, improving visual quality in sparse regions. The separation of coarse and fine passes allows for optimized traversal strategies [S1, unofficial].
*   **Limitations:** The fixed stack size and buffer allocations may not scale well to very high resolutions or deep octrees. The reliance on OpenCL-specific buffers limits portability [S1, unofficial].
*   **When to Use:** Suitable for real-time rendering of sparse voxel data where visual completeness is critical, and hardware supports OpenCL [S1, unofficial].
*   **When Not to Use:** Not suitable for dense voxel data where single-pass raycasting is more efficient, or for systems without OpenCL support [S1, unofficial].

## Background
This method originates from the "Sparse-Voxel-Octree-Raycasting" project by sp4cerat, an unofficial implementation of SVO raycasting using OpenCL [S1, unofficial]. It was developed to provide a robust, multi-pass rendering pipeline that handles sparse data efficiently by explicitly addressing the issue of unfilled pixels (holes) through a dedicated detection and filling phase [S1, unofficial].

## Sources
[S1] sp4cerat/Sparse-Voxel-Octree-Raycasting, https://github.com/sp4cerat/Sparse-Voxel-Octree-Raycasting, fetched 2026-10-24 (unofficial)
