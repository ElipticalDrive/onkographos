# SparseLeap Ray-Casting with Occupancy Misses

Also called: Ray-guided volume rendering with empty space skipping

## Overview
SparseLeap is a ray-guided volume rendering technique that separates empty-space skipping metadata from volume data, using a ray segment list to process rays in discrete intervals [S1, unofficial]. It distinguishes itself from sibling methods like [Empty Space Skipping with Octree](1610-empty-space-skipping-with-octree.md) by delaying the determination of a brick's occupancy status until it is actually intersected by a view ray, thereby reducing depth complexity and avoiding unnecessary culling computations for occluded or out-of-frustum regions [S1, unofficial].

## How it works
SparseLeap extends standard ray-casting by introducing an outer loop over a pre-computed list of ray segments. It utilizes an "occupancy histogram tree" to manage metadata and supports an "unknown" occupancy class to enable lazy evaluation of empty space [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Volume data (stored in bricks).
    *   Occupancy histogram tree (metadata structure).
    *   Ray segment list (generated via rasterization of occupancy geometry).
    *   Ray origin and direction.
*   **Outputs:**
    *   Rendered pixel color and opacity.

### Data Structures

**Ray Segment List**
A linear list of ray events defining intervals along a ray. Each event marks a transition between segments. The list allows linear iteration through consecutive overlapping pairs to determine segment boundaries.

**Occupancy Histogram Tree**
A hierarchical structure storing occupancy metadata for volume bricks.
*   **Nodes:** Represent spatial regions (bricks or sub-bricks).
*   **Fields:**
    *   `occupancy_class`: Enum {EMPTY, NONEMPTY, UNKNOWN}.
    *   `histogram`: Counts of voxel types within the node's region (used to detect homogeneity).
*   **Homogeneity:** A sub-tree is homogeneous if all leaf nodes share the same `occupancy_class`. Such sub-trees are substituted by a single representative node.

**Occupancy Class**
*   `EMPTY`: The region contains no visible data.
*   `NONEMPTY`: The region contains visible data.
*   `UNKNOWN`: The occupancy status has not yet been determined.

### Algorithm Steps

1.  **Ray Segment Iteration:**
    The ray-casting loop is augmented by an outer loop that iterates through the ray segment list. The loop processes consecutive pairs of ray events to define the start and end of each segment $[t_{start}, t_{end}]$.

2.  **Segment Classification:**
    For each segment, retrieve the `occupancy_class` from the occupancy histogram tree.
    *   If `occupancy_class` is `EMPTY`:
        *   Skip the segment.
        *   Advance the ray position to $t_{end}$.
    *   If `occupancy_class` is `NONEMPTY`:
        *   Proceed to sampling.
    *   If `occupancy_class` is `UNKNOWN`:
        *   Generate an **occupancy miss**.
        *   Resolve the miss (see Step 3).

3.  **Occupancy Miss Resolution:**
    When an occupancy miss occurs:
    *   Perform culling to determine the actual occupancy class of the underlying brick/region.
    *   Update the `occupancy_class` in the occupancy histogram tree to `EMPTY` or `NONEMPTY`.
    *   If the updated sub-tree becomes homogeneous, substitute it with a single node of that class.
    *   Re-evaluate the segment classification using the updated class.

4.  **Sampling:**
    If the segment is classified as `NONEMPTY` (either initially or after resolution):
    *   Perform standard ray-casting sampling within the segment bounds.
    *   Check for data cache residency.

5.  **Data Cache Management:**
    *   If the volume data for the current brick is not in memory, generate a **data cache miss**.
    *   Load the brick into memory.
    *   Note: A data cache miss is *not* generated for `UNKNOWN` segments until after the occupancy miss is resolved and the class is determined to be `NONEMPTY`. This prevents loading data that may turn out to be empty or unnecessary [S1, unofficial].

6.  **Memory Management (LRU):**
    *   The ray-caster reports usage of volume regions to the out-of-core memory management system.
    *   Unused nodes in the occupancy histogram tree are retired using a Least-Recently Used (LRU) scheme based on this usage report. This scales the metadata size to the current working set rather than the entire volume [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_leap_ray_cast(ray_origin, ray_direction, ray_segment_list, occupancy_tree, volume_cache):
    """
    Renders a single ray using SparseLeap.
    
    Args:
        ray_origin: vec3
        ray_direction: vec3
        ray_segment_list: List of ray events (t_start, t_end, segment_id)
        occupancy_tree: OccupancyHistogramTree
        volume_cache: Out-of-core cache for volume bricks
        
    Returns:
        color: vec4
    """
    color = vec4(0.0, 0.0, 0.0, 0.0)
    t = ray_origin
    alpha = 0.0
    
    # Iterate through ray segments
    for i in range(0, len(ray_segment_list) - 1):
        event_start = ray_segment_list[i]
        event_end = ray_segment_list[i + 1]
        
        t_start = event_start.t
        t_end = event_end.t
        
        # Determine occupancy class
        class_id = occupancy_tree.get_occupancy_class(event_start.segment_id)
        
        if class_id == OCCUPANCY_EMPTY:
            # Skip empty segments
            t = ray_origin + ray_direction * t_end
            continue
            
        elif class_id == OCCUPANCY_UNKNOWN:
            # Resolve occupancy miss
            actual_class = culling_resolve(event_start.segment_id, occupancy_tree)
            if actual_class == OCCUPANCY_EMPTY:
                t = ray_origin + ray_direction * t_end
                continue
            elif actual_class == OCCUPANCY_NONEMPTY:
                class_id = OCCUPANCY_NONEMPTY
            else:
                # Should not happen if culling is correct
                t = ray_origin + ray_direction * t_end
                continue
        
        # If NONEMPTY, sample
        # Check data cache
        brick = volume_cache.get_brick(event_start.segment_id)
        if brick is None:
            # Data cache miss: load brick
            brick = volume_cache.load(event_start.segment_id)
            
        # Standard ray-casting sampling within [t_start, t_end]
        # Note: t_start and t_end are relative to ray origin
        sample_t = t_start
        step_size = 1.0 / volume_resolution # Or adaptive step
        while sample_t < t_end:
            sample_pos = ray_origin + ray_direction * sample_t
            density = brick.sample(sample_pos)
            color_sample = transfer_function(density)
            
            # Alpha blending
            alpha += (1.0 - alpha) * density
            color += color_sample * (1.0 - alpha)
            
            if alpha >= 1.0:
                break
            sample_t += step_size
            
        t = ray_origin + ray_direction * t_end
        
        if alpha >= 1.0:
            break
            
    return color
```

### Parameters
*   **Culling Granularity:** The size of blocks used for empty space skipping. Can be as small as $4^3$ blocks for terabyte-sized volumes, allowing finer granularity than the memory management brick size (e.g., $32^3$ blocks) [S1, unofficial].
*   **Evaluated Block Sizes:** $4^3, 8^3, 16^3, 32^3$ blocks [S1, unofficial].

### Edge Cases and Pitfalls
*   **Order-Dependent Rasterization:** The rasterization of occupancy geometry into ray segment lists must process fragments in the same order as submitted to OpenGL to ensure correct segment ordering. This is handled using the `ARB_fragment_shader_interlock` extension [S1, unofficial].
*   **Precision:** Not given in the sources.
*   **Degenerate Input:** Not given in the sources.

## Operating characteristics
*   **Time Complexity:** Depends on the number of ray segments. SparseLeap significantly limits depth complexity compared to octree-based skipping. For the SEM Mouse Cortex dataset, SparseLeap had an average of 3.84 segments per ray (min 1, max 31), while octree traversal had an average of 13.44 intersected nodes (min 4, max 45) [S1, unofficial].
*   **Memory Complexity:** Scales with the current output size (working set) rather than the whole volume, due to LRU retirement of unused nodes in the occupancy histogram tree [S1, unofficial].
*   **Measured Performance:** Tested on NVIDIA GeForce Titan X (Pascal) GPU with 12 GB RAM [S1, unofficial]. Performance scales better to finely detailed, sparse data compared to octree-based approaches. Octree skipping can be slower than no skipping for some resolution levels in sparse data [S1, unofficial].
*   **Strengths:** Reduced depth complexity; lazy evaluation of occupancy metadata; fine-grained empty space skipping independent of brick size.
*   **Limitations:** Requires fragment shader interlock support; relies on pre-computed ray segment lists.
*   **When to use:** Large-scale, sparse volume rendering where metadata overhead and depth complexity of octrees are prohibitive.
*   **When not to use:** Dense volumes where empty space skipping provides little benefit; hardware lacking `ARB_fragment_shader_interlock`.

## Background
SparseLeap was introduced in the paper "SparseLeap: Efficient Empty Space Skipping for Large-Scale Volume Rendering" [S1, unofficial]. It is implemented using OpenGL with GLSL shaders and integrated with a large-scale out-of-core raycasting system [S1, unofficial]. The method builds on the concept of ray-guided volume rendering and lazy evaluation [S1, unofficial].

## Sources
[S1] SparseLeap: Efficient Empty Space Skipping, https://vcg.seas.harvard.edu/publications/20180101-sparseleap-efficient-empty-space-skipping-for-large-scale-volume-rendering/paper, fetched 2026-10-24 (unofficial)
