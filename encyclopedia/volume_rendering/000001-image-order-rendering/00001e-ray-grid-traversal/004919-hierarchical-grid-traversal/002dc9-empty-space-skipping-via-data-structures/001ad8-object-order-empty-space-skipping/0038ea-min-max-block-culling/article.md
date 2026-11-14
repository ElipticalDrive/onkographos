# Min-Max Block Culling

Also called: Active block culling, Min-Max Bricks, Min-max blocks, Object-Order Empty Space Skipping (Min-Max Bricks)

## Overview
Min-Max Block Culling is an object-order empty space skipping technique that reduces the number of volume samples by culling transparent or irrelevant regions before raycasting begins [S1, S2, unofficial]. It differs from its siblings by performing culling during the rasterization setup phase using precomputed block statistics, rather than during ray traversal or via bounding geometry rasterization [S1, S2, unofficial]. Unlike [Early Shader Termination](969-early-shader-termination.md), which stops rays dynamically during compositing, this method prevents rays from entering empty blocks entirely [S1, S2, unofficial]. Unlike [Rasterizing Bounding Geometry for Empty Space Skipping](5141-rasterizing-bounding-geometry-for-empty-space-skipping.md), it relies on scalar block bounds rather than polygonal approximations [S1, S2, unofficial].

## How it works
**Inputs:**
*   Volume dataset discretized into blocks (bricks) of fixed size [G].
*   Transfer function or isovalue defining the rendering range [S1, S2, unofficial].
*   Camera parameters for ray generation [G].

**Outputs:**
*   A set of active blocks whose front and back faces are rasterized to initiate rays [S1, S2, unofficial].
*   Final composited image [S1, S2, unofficial].

**Data Structures:**
*   **Block Min-Max Values:** For each block $B_i$, store two scalar values: $min_i$ and $max_i$ [S1, S2, unofficial]. These represent the minimum and maximum scalar values contained within the block's voxel grid [S1, S2, unofficial].
    *   Layout: A 1D array or 3D texture where each entry corresponds to a block, containing two floats (or fixed-point values) [G].
    *   Size: One entry per block [G].

**Steps:**
1.  **Precompute Block Statistics:** Calculate $min_i$ and $max_i$ for every block in the volume [S1, S2, unofficial].
2.  **Cull Blocks:** For each block $B_i$, check if its value range $[min_i, max_i]$ intersects the rendering range defined by the transfer function or isovalue [S1, S2, unofficial].
    *   If the block is outside the range (e.g., $max_i < \text{isovalue}$ or $min_i > \text{isovalue}$ for isosurface rendering), mark it as culled [S1, S2, unofficial].
    *   If the block intersects the range, mark it as active [S1, S2, unofficial].
3.  **Rasterize Active Blocks:** For each active block, rasterize its front and back faces against the screen [S1, S2, unofficial].
    *   This generates a coverage mask indicating which screen pixels have rays entering the block [G].
    *   Note: If the viewpoint is inside the volume, near clipping planes may be used to fill holes [S1, unofficial].
4.  **Ray Initialization:** For each pixel covered by an active block's front face, initialize a ray starting at the intersection point with the front face [S1, S2, unofficial].
5.  **Raycasting and Termination:** Cast rays through the active blocks [S1, S2, unofficial].
    *   Terminate the ray when full opacity is reached [S1, S2, unofficial].
    *   Terminate the ray when the back face of the block is reached [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_block_bounds(volume_data, block_size):
    """Compute min and max values for each block."""
    bounds = []
    nx, ny, nz = len(volume_data[0][0]), len(volume_data[0]), len(volume_data)
    for z in range(0, nz, block_size):
        for y in range(0, ny, block_size):
            for x in range(0, nx, block_size):
                vals = []
                for dz in range(block_size):
                    for dy in range(block_size):
                        for dx in range(block_size):
                            if z+dz < nz and y+dy < ny and x+dx < nx:
                                vals.append(volume_data[z+dz][y+dy][x+dx])
                bounds.append((min(vals), max(vals)))
    return bounds

def cull_blocks(block_bounds, isovalue):
    """Return True for blocks containing the isovalue."""
    return [min_v <= isovalue <= max_v for min_v, max_v in block_bounds]

def render_min_max_culling(volume_data, block_size, isovalue, res_x, res_y):
    """Main rendering loop using Min-Max Block Culling."""
    block_bounds = compute_block_bounds(volume_data, block_size)
    active_flags = cull_blocks(block_bounds, isovalue)
    
    image = [[0.0, 0.0, 0.0] for _ in range(res_y)]
    
    for i, is_active in enumerate(active_flags):
        if not is_active:
            continue
        # Rasterize active block faces to determine covered pixels
        for py in range(res_y):
            for px in range(res_x):
                # Compute ray origin at front face of block
                origin = [px / res_x, py / res_y, 0.0]
                direction = [0.0, 0.0, 1.0]
                
                current_pos = origin
                accumulated_opacity = 0.0
                full_opacity_reached = False
                
                # Ray march until back face (z=1.0) or full opacity
                step_size = 0.01
                while current_pos[2] < 1.0 and not full_opacity_reached:
                    # Sample volume
                    vx = int(current_pos[0] * len(volume_data[0][0]))
                    vy = int(current_pos[1] * len(volume_data[0]))
                    vz = int(current_pos[2] * len(volume_data))
                    sample_val = volume_data[vz][vy][vx]
                    
                    # Transfer function: alpha = 1 if sample matches isovalue, else 0
                    alpha = 1.0 if sample_val == isovalue else 0.0
                    
                    # Front-to-back compositing
                    accumulated_opacity += alpha * (1.0 - accumulated_opacity)
                    if accumulated_opacity >= 1.0:
                        full_opacity_reached = True
                    
                    current_pos[2] += step_size
                
                if full_opacity_reached:
                    image[py][px] = [1.0, 1.0, 1.0]
    return image
```

**Parameters:**
*   **Block Size:** Not given in the sources. Typical values in volume rendering are powers of 2 (e.g., $8^3$, $16^3$) [G].
*   **Isovalue/Transfer Function:** Depends on the dataset [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Incomplete Skipping:** This method does not skip all empty space. Gaps between cullable bricks or empty regions within brick interiors may still be sampled [S1, S2, unofficial].
*   **Viewpoint Inside Volume:** If the camera is inside the volume, near clipping planes must be used to fill holes in the ray coverage [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Requires storage for min-max values for each block, $O(B)$ [G].
*   **Strengths:** Reduces sample count by skipping entire blocks that are transparent [S1, S2, unofficial]. Simple to implement as it relies on standard rasterization [S1, S2, unofficial].
*   **Limitations:** Does not skip all empty space; internal block emptiness is not exploited [S1, S2, unofficial].
*   **When to Use:** When the volume has large transparent regions that can be grouped into blocks [S1, S2, unofficial].
*   **When Not to Use:** Not given in the sources.

## Background
Originated in GPU-based volume raycasting research, specifically in the work of Markus Hadwiger and others at VRVIS Research Center, Vienna, Austria [S1, S2, unofficial]. The technique is described in the context of advanced illumination techniques for GPU-based volume raycasting [S1, S2, unofficial]. It builds on image-based ray setup and allows for viewpoint inside the volume [S1, unofficial].

## Sources
[S1] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
[S2] GPU-Based Volume Ray-Casting with Advanced Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2008/advanced_illumination.pdf, fetched 2026-10-29 (unofficial)
