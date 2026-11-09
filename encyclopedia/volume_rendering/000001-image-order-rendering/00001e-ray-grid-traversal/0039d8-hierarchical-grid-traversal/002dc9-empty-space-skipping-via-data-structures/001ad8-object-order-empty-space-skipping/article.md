# Object-Order Empty Space Skipping

Also called: Block min-max culling, Front-to-back ray-casting with bounding box culling, GPU-based Ray-Casting with Object-Order Empty Space Skipping

## Overview
Object-Order Empty Space Skipping is a volume rendering optimization technique that reduces the number of volume samples by culling transparent or irrelevant regions before raycasting begins [S1][S2, unofficial]. It differs from its siblings by performing culling during the rasterization setup phase using precomputed block statistics, rather than during ray traversal or via bounding geometry rasterization [S1][S2, unofficial]. Unlike [Early Shader Termination](969-early-shader-termination.md), which stops rays dynamically during compositing, this method prevents rays from entering empty blocks entirely [S1][S2, unofficial]. Unlike [Rasterizing Bounding Geometry for Empty Space Skipping](5141-rasterizing-bounding-geometry-for-empty-space-skipping.md), it relies on scalar block bounds rather than polygonal approximations [S1][S2, unofficial]. It builds on [Empty Space Skipping via Data Structures](5619-empty-space-skipping-via-data-structures.md) by using a secondary data structure (min-max blocks) to represent active regions, allowing rays to skip inactive volumes [S1].

## How it works
The method combines object-order skipping (via bounding geometry of active blocks) with image-order skipping (per-sample checks within active blocks) to minimize overhead [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Volume texture $V$ containing scalar values (e.g., density or iso values) [S1][S2, unofficial].
    *   Camera parameters: position $c$, screen coordinates $(x, y)$ [S1].
    *   Isovalue threshold $\tau$ (or transfer function parameters) defining "active" data [S1][S2, unofficial].
    *   Block size $B$ (e.g., $4^3$ or $8^3$ voxels) [S1].
*   **Outputs:**
    *   Final image with rays marching only through active regions [S1].

### Data Structures
1.  **Min-Max Block Structure:** A 3D grid storing metadata for each block of the volume.
    *   **Layout:** For a volume of size $N_x \times N_y \times N_z$ and block size $B$, the structure has dimensions $\lceil N_x/B \rceil \times \lceil N_y/B \rceil \times \lceil N_z/B \rceil$.
    *   **Fields:** Each element stores the minimum and maximum scalar values found within that block in the original volume texture [S1].
    *   **Bit Packing/Status:** A block is marked "active" if its maximum value exceeds the isovalue threshold $\tau$ [S1]. This status is updated whenever $\tau$ changes [S1].

2.  **Bounding Geometry (Implicit or Explicit):**
    *   The set of faces of active blocks that are adjacent to inactive blocks [S1].
    *   This geometry defines the entry and exit points for rays. It is not necessarily stored as a single mesh but is conceptually defined by the active/inactive block boundaries [S1].

### Numbered Steps

1.  **Initialization (Pre-computation):**
    *   Subdivide the volume into blocks of size $B^3$ [S1].
    *   Compute the min-max values for each block by scanning the volume texture [S1].
    *   Mark blocks as active if $\max(\text{block}) > \tau$ [S1].
    *   Identify the bounding faces of active blocks that border inactive blocks [S1].

2.  **Ray Direction Computation:**
    *   For each pixel $(x, y)$, compute the ray direction $\mathbf{d}(x, y)$ from the camera position $c$ and screen coordinates [S1]:
        $$ \mathbf{p}(t, x, y) = c + t \mathbf{d}(x, y) $$
    *   $\mathbf{d}(x, y)$ is normalized [S1].

3.  **Object-Order Skip Range Determination (Rasterization Pass):**
    *   Compute the initial search range $[t_{\text{start}}(x, y), t_{\text{exit}}(x, y)]$ for each ray [S1].
    *   **Front Faces:** Rasterize the front faces of the active block bounding geometry. Use a depth test (e.g., `GL_LESS`) to retain the closest intersection depth $t_{\text{start}}$ [S1].
    *   **Back Faces:** Rasterize the back faces of the active block bounding geometry. Use an inverted depth test (e.g., `GL_GREATER`) to retain the farthest intersection depth $t_{\text{exit}}$ [S1].
    *   *Note:* This step effectively skips large inactive regions outside the active block cluster [S1].

4.  **Image-Order Ray Marching (Fragment Shader Loop):**
    *   Initialize ray position $t = t_{\text{start}}$ [S1].
    *   Loop in front-to-back order while $t < t_{\text{exit}}$ [S1]:
        *   Map current ray position $\mathbf{p}(t)$ to block coordinates $(u_{\text{occ}}, v_{\text{occ}}, w_{\text{occ}})$ using division by block size $B$:
            $$ u_{\text{occ}} = \lfloor u_{\text{vol}} / B \rfloor $$
            (where $u_{\text{vol}}$ is the coordinate in the volume grid) [S1].
        *   Query the Min-Max Block Structure for the block at $(u_{\text{occ}}, v_{\text{occ}}, w_{\text{occ}})$ [S1].
        *   **Check Activity:** If the block is inactive (max value $\le \tau$):
            *   Skip the samples within this block.
            *   Advance $t$ to the exit of the current block [S1].
            *   *Pitfall:* Inactive blocks enclosed by active blocks are not excluded by the bounding geometry. They are skipped here on a per-sample basis [S1].
        *   If the block is active:
            *   Perform standard ray marching/sampling within the block [S1].
            *   Update accumulated color and opacity [S1].
            *   Advance $t$ by the step size [S1].
            *   *Optional:* Use adaptive sampling or iterative bisection for intersection refinement if rendering isosurfaces [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computation (CPU or Compute Shader)
def compute_min_max_blocks(volume, block_size):
    num_blocks_x = ceil(volume.width / block_size)
    num_blocks_y = ceil(volume.height / block_size)
    num_blocks_z = ceil(volume.depth / block_size)
    
    # Structure: 3D array of (min_val, max_val)
    block_stats = empty_3d_array(num_blocks_x, num_blocks_y, num_blocks_z)
    
    for z in range(num_blocks_z):
        for y in range(num_blocks_y):
            for x in range(num_blocks_x):
                min_v = infinity
                max_v = -infinity
                
                # Scan voxels in this block
                for vz in range(z * block_size, (z + 1) * block_size):
                    for vy in range(y * block_size, (y + 1) * block_size):
                        for vx in range(x * block_size, (x + 1) * block_size):
                            val = volume[vx, vy, vz]
                            if val < min_v: min_v = val
                            if val > max_v: max_v = val
                            
                block_stats[x, y, z] = (min_v, max_v)
    return block_stats

def get_active_block_faces(block_stats, block_size, volume_dims):
    # Returns list of bounding faces of active blocks adjacent to inactive blocks
    # This is used for the rasterization pass to get t_start and t_exit
    # Implementation detail: Rasterize front/back faces of active blocks
    # that have at least one inactive neighbor.
    pass

# Fragment Shader Execution
def fragment_shader(pixel_x, pixel_y, camera, block_stats, block_size, isovalue):
    # 1. Compute Ray
    ray_origin = camera.position
    ray_dir = compute_direction(camera, pixel_x, pixel_y)
    
    # 2. Object-Order Skip: Get t_start and t_exit from rasterized bounding geometry
    # This is typically done in a separate render pass or via a texture lookup 
    # if pre-computed for the current view.
    # For the sake of this algorithm description, we assume t_start and t_exit 
    # are available for the current pixel.
    t_start = get_t_start_from_rasterization(pixel_x, pixel_y)
    t_exit = get_t_exit_from_rasterization(pixel_x, pixel_y)
    
    # 3. Image-Order Marching
    t = t_start
    accumulated_color = (0, 0, 0, 0)
    
    while t < t_exit:
        # Map ray position to block coordinates
        voxel_pos = ray_origin + t * ray_dir
        block_x = floor(voxel_pos.x / block_size)
        block_y = floor(voxel_pos.y / block_size)
        block_z = floor(voxel_pos.z / block_size)
        
        # Check block activity
        min_val, max_val = block_stats[block_x, block_y, block_z]
        
        if max_val <= isovalue:
            # Block is inactive, skip it
            # Advance t to the end of this block
            # Calculate exit t for the current block
            block_exit_t = compute_block_exit_t(ray_origin, ray_dir, block_x, block_y, block_z, block_size)
            t = max(t, block_exit_t)
        else:
            # Block is active, sample it
            # Perform standard ray marching step
            sample_val = sample_volume(voxel_pos)
            color, opacity = apply_transfer_function(sample_val)
            
            # Front-to-back compositing
            accumulated_color = composite(accumulated_color, color, opacity)
            
            if accumulated_color.a >= 1.0:
                break # Early termination if fully opaque
                
            t += step_size # Adaptive step size if applicable
            
    return accumulated_color
```

### Parameters
*   **Block Size ($B$):** Regular subdivision of volume into small blocks. Typical values are $4^3$ or $8^3$ voxels per block [S1].
*   **Isovalue Threshold ($\tau$):** The threshold above which a block is considered active [S1].

### Edge Cases and Pitfalls
*   **Enclosed Inactive Blocks:** Inactive blocks enclosed by active blocks are not excluded from the search range by the bounding geometry (object-order skip). Samples in these blocks are skipped on a per-sample basis (image-order skip) early in the ray-casting loop [S1].
*   **Isovalue Changes:** The active status of blocks must be recomputed whenever the isovalue changes, as this affects the min-max culling [S1].

## Subtopics

### Overlapped Min-Max Block
A data structure for empty-space skipping in pre-integrated volume rendering [S3]. It modifies the region covered by each block so that each block covers some region of which thickness is at least the sampling distance [S3]. By overlapping the region of each block, a line segment formed by two sampling points can be tested for transparency using only one block, avoiding the overhead of determining which of multiple intersecting blocks are transparent [S3]. Unlike previous methods where a line segment may intersect at most three blocks and requires testing all intersecting blocks (creating overhead to determine cases like Fig. 1a vs 1b), this method tests only one block due to the overlapping region design [S3].

## Operating characteristics
*   **Time Complexity:** Reduces the number of volume samples significantly in sparse volumes. The overhead of computing min-max blocks is $O(N)$ where $N$ is the number of voxels, but this is done once or when the isovalue changes [S1]. The rendering cost is proportional to the number of active blocks and samples within them [S1].
*   **Memory Complexity:** Requires additional memory for the min-max block structure, which is $O(N/B^3)$, significantly smaller than the volume itself [S1].
*   **Strengths:** Reduces empty space skipping overhead by using a two-level approach. Most empty space is skipped with no cost using modified ray segments $[t_{\text{start}}, t_{\text{exit}}]$. Only for a small number of samples empty space has to be skipped on a sample-by-sample basis [S1].
*   **Limitations:** Inactive blocks enclosed by active blocks are not excluded by the bounding geometry, requiring per-sample checks [S1]. The bounding geometry must be updated or recomputed if the isovalue changes [S1].
*   **When to use:** Suitable for GPU-based ray-casting where the volume fits in memory or can be managed with bricks, and where the isovalue may change interactively [S1].

## Background
Originating from real-time ray-casting techniques for discrete isosurfaces, this method was introduced to overcome limitations of earlier multi-pass approaches [S1]. It allows for a single rendering pass with front-to-back looping in the fragment shader, employing both object-order and image-order empty space skipping [S1]. The technique is particularly effective for interactive applications where the isovalue needs to be adjusted in real-time [S1].

## Sources
[S1] Real-Time Ray-Casting and Advanced Shading of Discrete Isosurfaces, https://cgl.ethz.ch/people/archive/siggc/publications/eg05.pdf, fetched 2026-10-26
[S2] implement volume rendering with object order empty space skipping, https://www.stackoverflow.com/questions/12216008, fetched 2026-10-27 (unofficial)
[S3] Efficient Interactive Pre-integrated Volume Rendering, https://doi.org/10.1007/11428862_118, fetched 2026-10-26
