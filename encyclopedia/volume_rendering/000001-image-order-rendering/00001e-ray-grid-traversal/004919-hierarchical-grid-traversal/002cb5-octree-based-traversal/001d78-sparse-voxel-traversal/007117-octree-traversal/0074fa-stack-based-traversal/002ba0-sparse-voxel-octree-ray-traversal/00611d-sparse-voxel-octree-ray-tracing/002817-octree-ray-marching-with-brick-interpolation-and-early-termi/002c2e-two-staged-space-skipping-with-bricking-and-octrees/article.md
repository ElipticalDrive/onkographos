# Two-staged Space-Skipping with Bricking and Octrees

Also called: Bricking with octree refinement, Two-stage space-skipping

## Overview
This method balances GPU bottlenecks by dividing space-skipping into two stages: coarse division using bricking to address bus and texture cache limitations, and fine division using octrees within bricks to address the rasterization bottleneck [S1, unofficial]. It differs from Tong et al.'s bricking technique by storing unmodified scalar values in textures and using a fragment shader for post-interpolative RGB$\alpha$ lookup, avoiding time-consuming texture regeneration when the transfer function changes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar voxel volume dataset; a transfer function mapping scalars to optical properties (RGB$\alpha$); rendering parameters (brick size $b$, overlap $n$, interpolation kernel).
*   **Output:** A rendered image.

**Data Structures:**
1.  **Bricks:** 3D textures containing unmodified scalar voxel values.
    *   **Layout:** Cubic blocks of size $b \times b \times b$ voxels.
    *   **Overlap:** Adjacent bricks overlap by $n$ voxels in every direction to support interpolation.
    *   **Memory Overhead:** Approximately $(3 \cdot n / b) \cdot 100\%$ [S1, unofficial].
2.  **Octrees:** Hierarchical structures stored within each brick.
    *   **Purpose:** To skip empty regions at a finer granularity than the brick level.
    *   **Content:** Not explicitly defined in sources, but implied to mark empty/occupied sub-volumes within the brick.

**Parameters:**
*   **Brick Size ($b$):** Must be equal to or smaller than the available texture memory size to handle datasets exceeding texture memory [S1, unofficial].
*   **Overlap ($n$):** Depends on the interpolation kernel [S1, unofficial]:
    *   Nearest neighbor: $n = 0$.
    *   Trilinear interpolation: $n = 1$ voxel in every direction.
    *   Pre-integrated rendering or on-the-fly gradient calculation: $n$ increased by another voxel in every direction (i.e., $n=2$ for trilinear with gradients) [S1, unofficial].
*   **Texture Format:** 3D textures. Many graphics cards require 3D texture sizes to be powers of 2 in every direction. If volume dimensions do not divide evenly, add partially empty bricks or use smaller rest-bricks [S1, unofficial].

**Steps:**
1.  **Brick Generation:** Divide the volume dataset into chunks called bricks. Ensure adjacent bricks overlap by $n$ voxels. Store unmodified scalar voxel values in these bricks.
2.  **Octree Construction:** For each brick, construct an octree to represent the occupancy of its internal voxels.
3.  **Memory Management:** Load bricks sequentially from main memory into texture memory. If the total data exceeds available texture memory, swap bricks between main and texture memory as needed [S1, unofficial].
4.  **Ray Traversal (Stage 1 - Coarse):** Traverse the brick structure. Skip entire bricks that are empty or do not intersect the view frustum. This addresses bus and texture cache bottlenecks.
5.  **Ray Traversal (Stage 2 - Fine):** For each active brick, traverse the internal octree. Skip empty sub-volumes within the brick. This addresses the rasterization bottleneck by reducing the number of fragment shader invocations [S1, unofficial].
6.  **Early Ray Termination:** Apply early ray termination within each brick to alleviate fragment shader load, particularly beneficial for highly-filled datasets [S1, unofficial].
7.  **Shading:** For each sampled voxel, use a fragment shader program to lookup RGB$\alpha$ values post-interpolatively based on the current transfer function and the scalar value. This allows fast runtime changes to the transfer function without regenerating textures [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for rendering with two-staged space-skipping
# Assumes: volume_data (scalar grid), transfer_function (scalar -> RGBa), 
#          bricks (list of brick objects), octrees (list of octree objects), 
#          ray (origin, direction)

def render_two_staged_bricking_octree(volume_data, transfer_function, bricks, octrees, ray):
    # Stage 1: Coarse division using bricking
    # Identify which bricks intersect the ray
    active_bricks = []
    for i, brick in enumerate(bricks):
        if brick_intersects_ray(brick, ray):
            active_bricks.append(i)
    
    accumulated_color = (0.0, 0.0, 0.0, 0.0)
    
    # Stage 2: Fine division using octrees within active bricks
    for brick_idx in active_bricks:
        brick = bricks[brick_idx]
        octree = octrees[brick_idx]
        
        # Traverse octree to find occupied sub-volumes
        # This reduces the number of samples taken within the brick
        samples = traverse_octree_ray(octree, ray, brick.origin, brick.size)
        
        # Process samples within the brick
        for sample in samples:
            # Get scalar value from brick texture (with overlap handling)
            scalar_value = get_scalar_value_from_brick(brick, sample.position)
            
            # Apply transfer function in fragment shader (post-interpolative)
            color, opacity = transfer_function(scalar_value)
            
            # Composite
            accumulated_color = composite(accumulated_color, color, opacity)
            
            # Early ray termination
            if accumulated_color[3] >= 1.0:
                break
                
    return accumulated_color

def brick_intersects_ray(brick, ray):
    # Check if ray intersects the brick's bounding box
    # Returns True if intersection occurs
    pass

def traverse_octree_ray(octree, ray, origin, size):
    # Traverse octree along ray
    # Returns list of sample points within occupied nodes
    # Handles overlap by adjusting sample coordinates relative to brick origin
    pass

def get_scalar_value_from_brick(brick, position):
    # Lookup scalar value in brick's 3D texture
    # Apply interpolation (e.g., trilinear) if necessary
    # Account for overlap by mapping position to local brick coordinates
    pass

def composite(color, new_color, opacity):
    # Standard front-to-back compositing
    # C_out = C_in + C_new * alpha_new * (1 - alpha_in)
    # alpha_out = alpha_in + alpha_new * (1 - alpha_in)
    pass
```

**Edge Cases and Pitfalls:**
*   **Triangle Throughput Bottleneck:** Space-skipping techniques may increase the amount of triangles considerably. If the triangle count becomes too high, it becomes a limiting factor for frame rate, potentially overloading the triangle throughput bottleneck when the rasterization bottleneck is lifted [S1, unofficial].
*   **Texture Memory Limits:** If brick size is not chosen correctly relative to available texture memory, excessive swapping will occur, negating the benefits of bricking [S1, unofficial].
*   **Power of 2 Constraints:** Graphics hardware may require 3D texture sizes to be powers of 2. Non-divisible volume dimensions require padding or smaller rest-bricks, which can complicate memory management and indexing [S1, unofficial].
*   **Transfer Function Changes:** Unlike Tong et al.'s method, this approach does not require texture regeneration for transfer function changes, but relies on fragment shader performance for the lookup [S1, unofficial].

## Subtopics

### [Quaternary Tree Volume Traversal](13509-quaternary-tree-volume-traversal.md)
Quaternary Tree Volume Traversal is a two-stage rasterization process distributed between a central processor and a graphics processing unit. The CPU divides object space using a quaternary tree to find intersections, and the GPU performs ray casting and shading.

## Operating characteristics
*   **Time Complexity:** Not explicitly given. Depends on the sparsity of the volume and the efficiency of the octree traversal.
*   **Memory Complexity:** Memory overhead for bricks is approximately $(3 \cdot n / b) \cdot 100\%$ [S1, unofficial]. Additional memory for octree structures within bricks.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales with the size of the volume dataset by loading bricks sequentially. Performance is limited by bus bandwidth and texture memory size [S1, unofficial].
*   **Strengths:**
    *   Addresses bus and texture cache bottlenecks via bricking [S1, unofficial].
    *   Addresses rasterization bottleneck via octree-based space-skipping [S1, unofficial].
    *   Allows runtime transfer function changes without texture regeneration [S1, unofficial].
    *   Early ray termination reduces fragment shader load [S1, unofficial].
*   **Limitations:**
    *   Risk of overloading triangle throughput bottleneck if space-skipping increases triangle count too much [S1, unofficial].
    *   Requires careful balancing of brick size and octree granularity to avoid shifting bottlenecks [S1, unofficial].
    *   Overlap increases memory usage [S1, unofficial].
*   **When to Use:** Suitable for large scalar volume datasets that exceed texture memory, where transfer function flexibility is required, and where the volume has sparse regions that can be skipped efficiently [S1, unofficial].
*   **When Not to Use:** When the volume is dense and space-skipping provides little benefit, or when the hardware has limited triangle throughput [S1, unofficial].

## Background
This method was introduced in 2006 in Plzen, Czech Republic, by Danny Ruijters et al. [S1, unofficial]. It builds on earlier bricking techniques by Eckhardt [Eck98] and Wang et al. [WWE04], and addresses limitations in Tong et al.'s bricking technique [TWTT99] which required texture regeneration for transfer function changes. The approach is based on an analysis of bottlenecks in the graphics pipeline during texture-mapping volume rendering, specifically targeting bus, texture cache, and rasterization bottlenecks [S1, unofficial].

## Sources
[S1] Optimizing GPU Volume Rendering, Danny Ruijters, https://dannyruijters.nl/docs/OptimizingGPUVolumeRendering_final.pdf, fetched 2026-10-26 (unofficial)
