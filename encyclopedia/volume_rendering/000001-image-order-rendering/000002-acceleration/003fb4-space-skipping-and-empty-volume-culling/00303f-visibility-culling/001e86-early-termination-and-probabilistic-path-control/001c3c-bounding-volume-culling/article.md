# Bounding Volume Culling

Also called: Distance-based Culling, Ray Termination by Distance

## Overview
Bounding Volume Culling is an optimization that limits raymarching to a specific bounding volume, dropping rays that exceed a certain distance from the eye or miss the volume entirely to speed up rendering [S1, unofficial][S2]. It differs from its siblings, such as [Empty-space skipping](1002-empty-space-skipping.md) which skips empty voxels, by discarding rays based on geometric bounds before or during traversal [S2]. Unlike [Early Stop](1790-early-stop.md) which terminates based on accumulated opacity, Bounding Volume Culling relies on spatial limits [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, bounding volume geometry (e.g., AABB, sphere, or spheroid), and maximum travel distance $t_{max}$ [S1, unofficial][S2].
- Output: Valid ray segment $[t_{enter}, t_{exit}]$ for traversal, or a flag indicating the ray misses the volume [S2].

**Data Structures:**
- **Bounding Volume:** Defined by its geometric parameters. For an Axis-Aligned Bounding Box (AABB), this is typically the minimum and maximum coordinates $(\mathbf{min}, \mathbf{max})$ [S4, unofficial]. For a spheroid/cloud, it may be a pointer to a voxel grid and geometric bounds [S2].
- **Heap (for Heap-based variant):** A priority queue storing distances to bounding volumes [S3].

**Steps:**
1. **Intersection Test:** Determine if the ray intersects the bounding volume.
   - For an AABB, compute the intersection interval $[t_{min}, t_{max}]$ using the slab method [S4, unofficial].
   - For a spheroid, compute the intersection with the enclosing sphere [S2].
2. **Culling:**
   - If the ray does not intersect the bounding volume, discard the ray immediately and assign a background color [S2][S4, unofficial].
   - If the ray intersects, calculate the valid segment $[t_{enter}, t_{exit}]$ within the volume [S2].
3. **Distance Limiting:**
   - Track the total distance traveled $t$ along the ray [S1, unofficial].
   - If $t$ exceeds a predefined limit $t_{max}$, terminate the raymarch process [S1, unofficial].
   - Alternatively, limit the segment to the intersection with the bounding volume boundaries [S2].
4. **Traversal:** Perform volume rendering only within the valid segment $[t_{enter}, t_{exit}]$ [S2].

**Formulas:**
- **AABB Intersection:** For each axis $i \in \{x, y, z\}$:
  $$ t_{i, min} = \frac{min_i - o_i}{d_i}, \quad t_{i, max} = \frac{max_i - o_i}{d_i} $$
  $$ t_{enter} = \max_i(t_{i, min}), \quad t_{exit} = \min_i(t_{i, max}) $$
  If $t_{enter} > t_{exit}$, the ray misses [S4, unofficial].
- **Distance Limit:** Terminate if $t > t_{max}$ [S1, unofficial].

**Parameters:**
- $t_{max}$: Maximum ray length. Typical values depend on scene scale; e.g., 64 steps in some implementations [S1, unofficial].
- Bounding Box: Defined by scene extent [S4, unofficial].

**Edge Cases and Pitfalls:**
- **Parallel Rays:** If $\mathbf{d}_i = 0$, handle division by zero in AABB intersection by using the inverse direction or special casing [S4, unofficial].
- **Degenerate Volumes:** Ensure bounding volumes have non-zero extent to avoid invalid intervals [S4, unofficial].
- **Precision:** Floating-point errors may cause rays to miss thin bounding volumes; consider epsilon adjustments [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bounding_volume_cull(ray_origin, ray_direction, aabb_min, aabb_max, max_dist):
    # Initialize intersection interval
    t_min = -infinity
    t_max = max_dist
    
    for i in range(3): # x, y, z
        if abs(ray_direction[i]) < epsilon:
            # Ray parallel to slab
            if ray_origin[i] < aabb_min[i] or ray_origin[i] > aabb_max[i]:
                return None # Miss
        else:
            inv_d = 1.0 / ray_direction[i]
            t1 = (aabb_min[i] - ray_origin[i]) * inv_d
            t2 = (aabb_max[i] - ray_origin[i]) * inv_d
            
            # Update interval
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
            
            # Early exit if interval is invalid
            if t_min > t_max:
                return None # Miss
    
    # Check if intersection is in front of camera and within max_dist
    if t_max < 0 or t_min > max_dist:
        return None # Miss
    
    # Return valid segment
    return (max(0.0, t_min), min(t_max, max_dist))
```

## Subtopics

### Heap-based Bounding Volume Culling for Sphere Tracing
An algorithm to find the closest object in a collection by maintaining a heap of bounding volume distances [S3]. It computes distances to bounding volumes first, then contents, culling based on the smallest bounding volume distance [S3]. Unlike standard culling which discards rays, this variant optimizes sphere tracing by deferring expensive distance computations [S3].
**Steps:**
1. Compute distances to each bounding volume in the union/collection [S3].
2. Create a heap of these distances [S3].
3. Initialize $d_{min}$ to infinity [S3].
4. Loop: Let $d_{content}$ be $\min(d_{min}, \text{distance to contents of top BV})$ [S3].
5. Remove top of heap [S3].
6. Let $d_{bv}$ be distance to new top BV [S3].
7. Stop if $d_{content} < d_{bv}$ or heap empty [S3].
8. Return $d_{content}$ [S3].
**Parameters:**
- $d_{bv}$: The distance to the bounding volume currently at the top of the heap [S3].

### Axis-Aligned Bounding Box (AABB) Culling
A spatial acceleration technique that uses the outer boundaries of the data volume to quickly reject rays that do not intersect the volume, avoiding unnecessary computation [S4, unofficial]. It differs from other variants by using the six planar faces of the rectangular grid volume as the bounding object [S4, unofficial].
**Steps:**
1. Define the bounding object as the six planar faces of the rectangular grid volume [S4, unofficial].
2. For each pixel, test the cast ray against the six bounding faces before initiating the full volume trace [S4, unofficial].
3. If the ray misses the bounding volume, immediately assign a background/empty color to the pixel and skip volume traversal [S4, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per ray for AABB intersection [S4, unofficial]. Heap-based variant is $O(N \log N)$ for setup and $O(\log N)$ per query [S3].
- **Memory Complexity:** $O(1)$ for AABB [S4, unofficial]. $O(N)$ for heap-based variant where $N$ is the number of bounding volumes [S3].
- **Performance:** Increases FPS by discarding rays that never hit the boundaries [S2]. GPU fragment shader implementation has very reduced linear cost [S2].
- **Strengths:** Simple to implement, effective for sparse volumes [S4, unofficial].
- **Limitations:** Less effective for dense volumes where most rays intersect the bounding box [S4, unofficial].

## Background
Originating from early ray tracing and volume visualization research, bounding volumes were introduced to reduce the computational cost of locating objects [S4, unofficial]. Smits' algorithm was adapted for cloud rendering to limit ray-marching to cloud boundaries [S2]. The technique is widely used in GPU fragment shaders for real-time volumetric cloud rendering [S2].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] Efficient Algorithms for Real-Time GPU Volumetric Cloud Rendering with Enhanced Geometry, https://doi.org/10.3390/sym10040125, fetched 2026-10-26
[S3] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S4] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
