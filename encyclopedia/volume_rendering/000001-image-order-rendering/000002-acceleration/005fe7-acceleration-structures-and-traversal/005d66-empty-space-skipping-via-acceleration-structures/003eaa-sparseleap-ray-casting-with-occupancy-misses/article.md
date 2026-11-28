# SparseLeap Ray-Casting with Occupancy Misses

Also called: Ray-guided volume rendering with empty space skipping [S1, unofficial]

## Overview
SparseLeap is a volume rendering technique that separates meta-data for empty space skipping from volume data, using a ray-guided framework where ray segments are processed in a list [S1, unofficial]. It extends standard ray-casting by adding an outer loop over ray segments and handling 'occupancy misses' to delay the determination of whether a brick is empty or non-empty until it is actually needed [S1, unofficial]. Unlike octree-based skipping which incurs high depth complexity due to fragmented space subdivision, SparseLeap limits depth complexity significantly by substituting homogeneous sub-trees in the occupancy histogram tree on-the-fly [S1, unofficial]. See [SparseLeap](5152-sparseleap.md), [Hierarchical Bounding Box Tree for Empty-Space Skipping](8182-hierarchical-bounding-box-tree-for-empty-space-skipping.md), [Space Skipping and Empty Volume Culling](10063-space-skipping-and-empty-volume-culling.md), [Empty-space skipping via Acceleration Structure](13062-empty-space-skipping-via-acceleration-structure.md), and [Progressive Ray Casting for Volumetric Models on Mobile Devices](16346-progressive-ray-casting-for-volumetric-models-on-mobile-devi.md) for related methods.

## How it works
SparseLeap operates within a ray-guided volume rendering framework that relies on lazy evaluation [S1, unofficial]. The method augments the standard front-to-back compositing loop with an outer loop that iterates over a list of ray segments [S1, unofficial]. These segments are derived from an occupancy histogram tree, which tracks the empty/non-empty status of volume bricks [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Ray segment list: A list of segments along each view ray, each tagged with an occupancy class (`empty`, `nonempty`, or `unknown`) [S1, unofficial].
    *   Volume data: Stored in bricks, managed by an out-of-core memory system [S1, unofficial].
    *   Occupancy Histogram Tree: A hierarchical structure representing the occupancy status of volume bricks [S1, unofficial].
*   **Outputs:**
    *   Rendered image pixel colors.
    *   Updated occupancy histogram tree (with homogeneous sub-trees substituted and unused nodes retired) [S1, unofficial].

### Data Structures
**Ray Segment**
Each entry in the ray segment list represents a contiguous portion of a ray [S1, unofficial].
*   `start`: Ray parameter $t_{start}$ (float).
*   `end`: Ray parameter $t_{end}$ (float).
*   `occupancy_class`: Enum or integer code. `0`: Empty, `1`: Nonempty, `2`: Unknown [S1, unofficial].
*   `brick_id`: Identifier for the underlying volume brick [S1, unofficial].

**Occupancy Histogram Tree Node**
*   `occupancy_class`: The dominant occupancy class of the sub-tree [S1, unofficial].
*   `is_homogeneous`: Boolean flag indicating if all children share the same occupancy class [S1, unofficial].
*   `children`: References to child nodes (if not a leaf) [S1, unofficial].
*   `usage_count`: Integer for LRU eviction tracking [S1, unofficial].

### Numbered Steps
1.  **Initialization**: For each pixel, cast a view ray [G]. Generate an initial ray segment list covering the ray's path through the volume [S1, unofficial]. Initially, segments may be marked as `unknown` if their occupancy is not yet determined [S1, unofficial].
2.  **Outer Loop (Segment Iteration)**: Iterate through the ray segment list linearly [S1, unofficial]. Process consecutive overlapping pairs of ray events to define segments [S1, unofficial].
3.  **Segment Classification**:
    *   If `occupancy_class` is `empty`: Skip the segment [S1, unofficial]. Advance the ray position to $t_{end}$ of the current segment [S1, unofficial]. Continue to the next segment.
    *   If `occupancy_class` is `nonempty`: Proceed to sampling [S1, unofficial].
    *   If `occupancy_class` is `unknown`: Generate an **occupancy miss** [S1, unofficial].
4.  **Occupancy Miss Resolution**:
    *   When an occupancy miss occurs, trigger culling to determine the actual occupancy class of the underlying volume brick [S1, unofficial].
    *   If the brick is determined to be `empty`: Update the segment's class to `empty` and skip [S1, unofficial].
    *   If the brick is determined to be `nonempty`: Update the segment's class to `nonempty` and proceed to sampling [S1, unofficial].
    *   **Delayed Update**: Bricks never intersected by view rays (occluded or outside frustum) do not have their occupancy class determined, saving computation and memory bandwidth [S1, unofficial].
5.  **Sampling**:
    *   If the segment is `nonempty`, perform standard ray-casting sampling within the segment bounds [S1, unofficial].
    *   **Data Cache Miss Handling**:
        *   If the volume data for the current brick is not in cache, generate a data cache miss to load it [S1, unofficial].
        *   If the class is `unknown`, do *not* generate a data cache miss immediately [S1, unofficial]. Resolve the occupancy miss first [S1, unofficial]. Only load data if the resolution confirms the brick is `nonempty` [S1, unofficial].
        *   If the class is `empty` or `unknown` (and resolved as empty), no data cache miss is generated [S1, unofficial].
6.  **Homogeneous Sub-tree Substitution**:
    *   After occupancy updates, check the occupancy histogram tree for sub-trees where all nodes have the same occupancy class [S1, unofficial].
    *   Substitute these homogeneous sub-trees with a single node of that class [S1, unofficial]. This is determined efficiently using occupancy histograms [S1, unofficial].
7.  **Memory Management (LRU)**:
    *   The ray-caster reports usage of volume bricks to the out-of-core memory management system [S1, unofficial].
    *   Unused nodes in the occupancy histogram tree are retired regularly using a Least-Recently Used (LRU) scheme based on this usage information [S1, unofficial]. This scales the tree size to the current working set rather than the whole volume [S1, unofficial].

### Formulas
The method modifies the standard front-to-back compositing loop [G]:
$$ C_{acc} = C_{acc} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
$$ \alpha_{acc} = \alpha_{acc} + \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
where $C_{acc}$ is the accumulated color, $\alpha_{acc}$ is the accumulated opacity, $C_{sample}$ is the sampled color, and $\alpha_{sample}$ is the sampled opacity [G]. The outer loop skips iterations or reduces $C_{sample}$ queries based on occupancy classes [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
EMPTY = 0
NONEMPTY = 1
UNKNOWN = 2
EPSILON = 1e-4

def sparse_leap_ray_cast(ray, segment_list, occupancy_tree, volume_cache):
    color = [0.0, 0.0, 0.0, 0.0]
    transmittance = 1.0
    
    for segment in segment_list:
        t_start = segment.start
        t_end = segment.end
        occ_class = segment.occupancy_class
        
        if occ_class == EMPTY:
            continue
        elif occ_class == UNKNOWN:
            actual_class = resolve_occupancy_miss(segment.brick_id, occupancy_tree)
            if actual_class == EMPTY:
                continue
            elif actual_class == NONEMPTY:
                occ_class = NONEMPTY
            else:
                continue
                
        if occ_class == NONEMPTY:
            if not volume_cache.contains(segment.brick_id):
                volume_cache.load(segment.brick_id)
                
            sample_color, sample_alpha = integrate_segment(ray, t_start, t_end, volume_cache, segment.brick_id)
            
            alpha_contribution = sample_alpha * transmittance
            color[0] += sample_color[0] * alpha_contribution
            color[1] += sample_color[1] * alpha_contribution
            color[2] += sample_color[2] * alpha_contribution
            color[3] += sample_alpha * alpha_contribution
            transmittance *= (1.0 - sample_alpha)
            
            if transmittance < EPSILON:
                break

    return color

def resolve_occupancy_miss(brick_id, occupancy_tree):
    if occupancy_tree.is_classified(brick_id):
        return occupancy_tree.get_class(brick_id)
    
    is_empty = compute_occupancy(brick_id)
    new_class = EMPTY if is_empty else NONEMPTY
    
    occupancy_tree.set_class(brick_id, new_class)
    occupancy_tree.substitute_homogeneous_subtrees(brick_id)
    
    return new_class

def integrate_segment(ray, t_start, t_end, volume_cache, brick_id):
    sample_color = [0.0, 0.0, 0.0]
    sample_alpha = 0.0
    step_size = 0.01
    t = t_start
    while t < t_end:
        voxel = volume_cache.sample(ray, t, brick_id)
        c, a = voxel.color, voxel.alpha
        sample_color[0] += c[0] * a * (1.0 - sample_alpha)
        sample_color[1] += c[1] * a * (1.0 - sample_alpha)
        sample_color[2] += c[2] * a * (1.0 - sample_alpha)
        sample_alpha += a * (1.0 - sample_alpha)
        t += step_size
    return sample_color, sample_alpha

def compute_occupancy(brick_id):
    return query_culling_meta_data(brick_id)
```

### Parameters
*   **Empty Space Skipping Granularity**: Can be as small as $4^3$ blocks for terabyte-sized volumes [S1, unofficial]. This allows finer granularity than the brick size used for memory management (e.g., $32^3$ blocks) [S1, unofficial].
*   **Block Sizes for Culling**: Evaluated sizes include $4^3$, $8^3$, $16^3$, and $32^3$ blocks [S1, unofficial].

### Edge Cases and Pitfalls
*   **Order-Dependent Rasterization**: The rasterization of occupancy geometry into ray segment lists must process fragments in the same order as they are submitted to OpenGL [S1, unofficial]. This is handled using the `ARB_fragment_shader_interlock` capability [S1, unofficial].
*   **Short Empty Segments**: Very short empty ray segments that cannot be merged into a longer empty segment because they precede a non-empty segment are detected and removed to reduce depth complexity [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: The method reduces the number of samples processed by skipping empty space [G]. The complexity depends on the depth complexity of the ray segment list [S1, unofficial].
*   **Memory Complexity**: The occupancy histogram tree scales with the current output size (working set) rather than the whole volume, thanks to LRU retirement of unused nodes [S1, unofficial].
*   **Measured Performance**: On the SEM Mouse Cortex dataset, SparseLeap had an average of 3.84 segments per ray (min 1, max 31), while octree traversal had an average of 13.44 intersected nodes (min 4, max 45) [S1, unofficial].
*   **Scaling**: The approach scales better to finely detailed, sparse data [S1, unofficial].
*   **Strengths**: Limits depth complexity significantly compared to octree-based skipping [S1, unofficial]. Scales better to finely detailed, sparse data [S1, unofficial]. Output-sensitive culling avoids processing occluded regions [S1, unofficial].
*   **Limitations**: Requires implementation of fragment shader interlock for correct rasterization order [S1, unofficial]. Culling computation is deferred, which may cause latency spikes if many misses occur simultaneously [G].
*   **When to Use**: Large-scale, sparse volume datasets where empty space skipping is critical for performance [S1, unofficial].
*   **When Not to Use**: Dense volumes where empty space skipping provides little benefit, or hardware lacking `ARB_fragment_shader_interlock` [S1, unofficial].

## Background
SparseLeap was introduced in the paper "SparseLeap: Efficient Empty Space Skipping for Large-Scale Volume Rendering" [S1, unofficial]. It is implemented using OpenGL with GLSL shaders and integrated with a large-scale out-of-core raycasting system [S1, unofficial]. Evaluation was performed on an NVIDIA GeForce Titan X (Pascal) GPU with 12 GB RAM [S1, unofficial].

## Sources
[S1] SparseLeap: Efficient Empty Space Skipping, https://vcg.seas.harvard.edu/publications/20180101-sparseleap-efficient-empty-space-skipping-for-large-scale-volume-rendering/paper, fetched 2026-10-24 (unofficial)
