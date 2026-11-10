# Min-Max Block Culling

Also called: Active block culling, Min-Max Bricks, Min-max blocks, Object-Order Empty Space Skipping (Min-Max Bricks)

## Overview
Min-Max Block Culling is an object-order empty space skipping technique that reduces the number of volume samples by culling transparent or irrelevant regions before raycasting begins [S1, S2, unofficial]. It differs from its siblings by performing culling during the rasterization setup phase using precomputed block statistics, rather than during ray traversal or via bounding geometry rasterization [S1, S2, unofficial]. Unlike [Early Shader Termination](969-early-shader-termination.md), which stops rays dynamically during compositing, this method prevents rays from entering empty blocks entirely [S1, S2, unofficial]. Unlike [Rasterizing Bounding Geometry for Empty Space Skipping](5141-rasterizing-bounding-geometry-for-empty-space-skipping.md), it relies on scalar block bounds rather than polygonal approximations [S1, S2, unofficial].

## How it works
**Inputs:**
*   Volume dataset discretized into blocks (bricks) of fixed size.
*   Transfer function or isovalue defining the rendering range.
*   Camera parameters for ray generation.

**Outputs:**
*   A set of active blocks whose front and back faces are rasterized to initiate rays.
*   Final composited image.

**Data Structures:**
*   **Block Min-Max Values:** For each block $B_i$, store two scalar values: $min_i$ and $max_i$. These represent the minimum and maximum scalar values contained within the block's voxel grid [S1, S2, unofficial].
    *   Layout: A 1D array or 3D texture where each entry corresponds to a block, containing two floats (or fixed-point values) [S2, unofficial].
    *   Size: Equal to the number of blocks in the volume hierarchy [S2, unofficial].

**Steps:**
1.  **Precompute Block Statistics:** Calculate $min_i$ and $max_i$ for every block in the volume [S1, S2, unofficial].
2.  **Cull Blocks:** For each block $B_i$, check if its value range $[min_i, max_i]$ intersects the rendering range defined by the transfer function or isovalue [S1, S2, unofficial].
    *   If the block is outside the range (e.g., $max_i < \text{isovalue}$ or $min_i > \text{isovalue}$ for isosurface rendering), mark it as culled [S1, S2, unofficial].
    *   If the block intersects the range, mark it as active [S1, S2, unofficial].
3.  **Rasterize Active Blocks:** For each active block, rasterize its front and back faces against the screen [S1, S2, unofficial].
    *   This generates a coverage mask indicating which screen pixels have rays entering the block [S1, S2, unofficial].
    *   Note: If the viewpoint is inside the volume, near clipping planes may be used to fill holes [S1, unofficial].
4.  **Ray Initialization:** For each pixel covered by an active block's front face, initialize a ray starting at the intersection point with the front face [S1, S2, unofficial].
5.  **Raycasting and Termination:** Cast rays through the active blocks [S1, S2, unofficial].
    *   Terminate the ray when full opacity is reached [S1, S2, unofficial].
    *   Terminate the ray when the back face of the block is reached [S1, S2, unofficial].
    *   If the ray exits the block without full opacity, it may continue to the next block or terminate depending on the specific implementation of the parent method [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputation phase
def compute_block_bounds(volume_data, block_size):
    """
    Computes min and max values for each block in the volume.
    volume_data: 3D array of scalar values.
    block_size: integer, dimensions of a block (e.g., 8x8x8).
    Returns: list of (min_val, max_val) tuples for each block.
    """
    blocks = []
    num_blocks_x = len(volume_data[0]) // block_size
    num_blocks_y = len(volume_data[0][0]) // block_size
    num_blocks_z = len(volume_data) // block_size
    
    for z in range(num_blocks_z):
        for y in range(num_blocks_y):
            for x in range(num_blocks_x):
                # Extract block data
                x_start = x * block_size
                y_start = y * block_size
                z_start = z * block_size
                
                block_values = []
                for dz in range(block_size):
                    for dy in range(block_size):
                        for dx in range(block_size):
                            block_values.append(volume_data[z_start+dz][y_start+dy][x_start+dx])
                
                min_val = min(block_values)
                max_val = max(block_values)
                blocks.append((min_val, max_val))
    return blocks

def cull_blocks(block_bounds, isovalue):
    """
    Culls blocks that do not contain the isovalue.
    block_bounds: list of (min_val, max_val) tuples.
    isovalue: scalar value to render.
    Returns: list of booleans, True if block is active.
    """
    active = []
    for min_val, max_val in block_bounds:
        # Block is active if isovalue is within [min, max]
        if min_val <= isovalue <= max_val:
            active.append(True)
        else:
            active.append(False)
    return active

def render_min_max_culling(volume_data, block_size, isovalue, camera):
    """
    Main rendering loop using Min-Max Block Culling.
    """
    # 1. Precompute bounds
    block_bounds = compute_block_bounds(volume_data, block_size)
    
    # 2. Cull blocks
    active_flags = cull_blocks(block_bounds, isovalue)
    
    # 3. Rasterize active blocks to get ray origins
    # This step is typically handled by the GPU rasterizer in the parent method.
    # Here we simulate the logic:
    ray_origins = []
    ray_directions = []
    
    # For each active block, determine screen coverage
    # In a real implementation, this involves projecting block faces to screen space
    # and using a depth buffer or coverage buffer.
    # We assume a function `get_screen_coverage` returns list of (screen_x, screen_y)
    # for each active block's front face.
    
    for i, is_active in enumerate(active_flags):
        if is_active:
            # Get screen pixels covered by this block's front face
            covered_pixels = get_screen_coverage_for_block(i, camera)
            for px, py in covered_pixels:
                # Compute ray origin at front face of block
                origin = compute_ray_origin_at_front_face(i, px, py, camera)
                direction = compute_ray_direction(px, py, camera)
                ray_origins.append(origin)
                ray_directions.append(direction)
    
    # 4. Raycast and terminate
    image = create_image(camera.resolution)
    
    for idx in range(len(ray_origins)):
        origin = ray_origins[idx]
        direction = ray_directions[idx]
        
        # Determine block boundaries for this ray
        # In object-order, we know which block we started in.
        # We cast until we hit the back face or full opacity.
        
        current_pos = origin
        accumulated_opacity = 0.0
        full_opacity_reached = False
        
        # Simplified ray march within the single block for demonstration
        # In practice, the ray may traverse multiple blocks if the parent method
        # chains them, but Min-Max culling specifically skips the *blocks* themselves.
        # The parent method [Object-Order Empty Space Skipping] handles the traversal
        # through the volume structure.
        
        step_size = get_step_size(camera)
        block_back_face_dist = get_distance_to_back_face(idx, origin, direction)
        
        while current_pos < block_back_face_dist and not full_opacity_reached:
            sample_val = sample_volume(current_pos, volume_data)
            color, alpha = transfer_function(sample_val, isovalue)
            
            # Front-to-back compositing
            accumulated_opacity += alpha * (1.0 - accumulated_opacity)
            image[idx] += color * alpha * (1.0 - accumulated_opacity)
            
            if accumulated_opacity >= 1.0:
                full_opacity_reached = True
            
            current_pos += step_size * direction
            
        # If full opacity not reached, the ray might continue to next block
        # or terminate depending on parent implementation.
        # For this specific method's scope, we stop at block boundary if not full opacity.
        if not full_opacity_reached:
            # The parent method would handle the next block.
            # Here we just finalize the current block's contribution.
            pass
            
    return image
```

**Parameters:**
*   **Block Size:** Not given in the sources. Typical values in volume rendering are powers of 2 (e.g., $8^3$, $16^3$) [G].
*   **Isovalue/Transfer Function:** Depends on the dataset [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Incomplete Skipping:** This method does not skip all empty space. Gaps between cullable bricks or empty regions within brick interiors may still be sampled [S1, S2, unofficial].
*   **Viewpoint Inside Volume:** If the camera is inside the volume, near clipping planes must be used to fill holes in the ray coverage [S1, unofficial].
*   **Precision:** Min-max values must be computed accurately to avoid false culling of blocks that contain the isovalue [S1, S2, unofficial].

## Subtopics

### Active Block Culling
Same method, emphasizing the selection of blocks for rasterization [S1, unofficial].

### Min-Max Bricks
Emphasizes the data structure (bricks) storing the bounds [S2, unofficial].

### Object-Order Empty Space Skipping (Min-Max Bricks)
The broader category, where this method is a specific implementation using min-max bounds [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Precomputation is $O(N)$ where $N$ is the number of voxels [S1, S2, unofficial]. Culling is $O(B)$ where $B$ is the number of blocks [S1, S2, unofficial]. Rasterization and raycasting depend on the number of active blocks and their screen coverage [S1, S2, unofficial].
*   **Memory Complexity:** Requires storage for min-max values for each block, $O(B)$ [S1, S2, unofficial].
*   **Strengths:** Reduces sample count by skipping entire blocks that are transparent [S1, S2, unofficial]. Simple to implement as it relies on standard rasterization [S1, S2, unofficial].
*   **Limitations:** Does not skip all empty space; internal block emptiness is not exploited [S1, S2, unofficial]. Performance gain depends on the sparsity of the volume [S1, S2, unofficial].
*   **When to Use:** When the volume has large transparent regions that can be grouped into blocks [S1, S2, unofficial].
*   **When Not to Use:** When the volume is dense or the blocks are small, the overhead of culling may not be justified [S1, S2, unofficial].

## Background
Originated in GPU-based volume raycasting research, specifically in the work of Markus Hadwiger and others at VRVIS Research Center, Vienna, Austria [S1, S2, unofficial]. The technique is described in the context of advanced illumination techniques for GPU-based volume raycasting [S1, S2, unofficial]. It builds on image-based ray setup and allows for viewpoint inside the volume [S1, unofficial].

## Sources
[S1] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
[S2] GPU-Based Volume Ray-Casting with Advanced Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2008/advanced_illumination.pdf, fetched 2026-10-29 (unofficial)
