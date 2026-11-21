# Ray-AABB Intersection and Distance


## Overview
This method computes the parametric entry and exit distances of a ray intersecting an Axis-Aligned Bounding Box (AABB), or determines the minimum intersection distance. It serves as the fundamental primitive test for ray traversal in volume rendering and ray tracing. It is distinguished from [Ray-Cube Intersection](2466-ray-cube-intersection.md) by its generalization to arbitrary box dimensions and from [AABB Ray Intersection](11086-aabb-ray-intersection.md) by its explicit provision of both entry and exit intervals rather than a simple boolean.

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and an AABB defined by minimum corner $\mathbf{b}_{min}$ and maximum corner $\mathbf{b}_{max}$.
*   **Output:** An optional interval $I = [t_{entry}, t_{exit}]$ representing the parametric distances along the ray where it enters and exits the box. If no intersection exists, the result is null/empty. For distance queries, the scalar $t_{entry}$ is returned, or $\infty$ if no intersection occurs.

**Data Structures:**
*   **Ray:** Contains fields `orig` (vec3) and `dir` (vec3).
*   **AABB:** Contains fields `min` (vec3) and `max` (vec3).
*   **FInterval:** Contains fields `low` (float) and `high` (float), representing the parametric entry and exit distances along the ray [S2].

**Algorithm Steps:**
1.  Compute the inverse direction of the ray: $\mathbf{inv\_dir} = 1.0 / \mathbf{d}$.
2.  Compute temporary intersection candidates for the minimum and maximum bounds along each axis:
    $$ \mathbf{t}_{min\_tmp} = (\mathbf{b}_{min} - \mathbf{o}) \odot \mathbf{inv\_dir} $$
    $$ \mathbf{t}_{max\_tmp} = (\mathbf{b}_{max} - \mathbf{o}) \odot \mathbf{inv\_dir} $$
    where $\odot$ denotes component-wise multiplication.
3.  Order the candidates per axis to ensure $\mathbf{t}_{min}$ holds the smaller value and $\mathbf{t}_{max}$ holds the larger value for each component:
    $$ \mathbf{t}_{min} = \min(\mathbf{t}_{min\_tmp}, \mathbf{t}_{max\_tmp}) $$
    $$ \mathbf{t}_{max} = \max(\mathbf{t}_{min\_tmp}, \mathbf{t}_{max\_tmp}) $$
4.  Compute the global entry distance $t_0$ as the maximum of the per-axis minimums:
    $$ t_0 = \max(t_{min,x}, \max(t_{min,y}, t_{min,z})) $$
5.  Compute the global exit distance $t_1$ as the minimum of the per-axis maximums:
    $$ t_1 = \min(t_{max,x}, \min(t_{max,y}, t_{max,z})) $$
6.  Determine validity: If $t_0 \le t_1$, the ray intersects the box. The interval is $[t_0, t_1]$. Otherwise, there is no intersection.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def intersectsp_ray_aabb(ray_orig, ray_dir, box_min, box_max):
    # Compute inverse direction
    inv_dir = 1.0 / ray_dir
    
    # Compute t-min and t-max candidates per axis
    tmin_tmp = (box_min - ray_orig) * inv_dir
    tmax_tmp = (box_max - ray_orig) * inv_dir
    
    # Order t-min and t-max per axis
    tmin = min(tmin_tmp, tmax_tmp)
    tmax = max(tmin_tmp, tmax_tmp)
    
    # Compute global entry t0 and exit t1
    t0 = max(tmin.x, max(tmin.y, tmin.z))
    t1 = min(tmax.x, min(tmax.y, tmax.z))
    
    # Return interval if valid
    if t0 <= t1:
        return FInterval(low=t0, high=t1)
    else:
        return None

def intersects(ray_orig, ray_dir, ray_tmax, box_min, box_max):
    interval = intersectsp_ray_aabb(ray_orig, ray_dir, box_min, box_max)
    if interval is not None:
        # Check if intersection is within the ray's valid range [0, tmax]
        # Note: Sources specify I.low < r.tmax && I.high > 0 [S2]
        return interval.low < ray_tmax and interval.high > 0
    return False

def distmin(ray_orig, ray_dir, box_min, box_max):
    interval = intersectsp_ray_aabb(ray_orig, ray_dir, box_min, box_max)
    if interval is not None:
        return interval.low
    return float('inf')
```

**Parameters and Edge Cases:**
*   **Zero Direction:** Division by zero in `inv_dir` occurs if any component of $\mathbf{d}$ is zero. Standard implementations handle this by treating the corresponding $t_{min\_tmp}$ and $t_{max\_tmp}$ as $-\infty$ or $+\infty$ appropriately, or by using a large epsilon. Not given in the sources.
*   **Ray Origin Inside Box:** If the ray origin is inside the box, $t_0$ may be negative. The intersection is still valid geometrically, but for ray marching, one might clamp $t_0$ to 0.
*   **Degenerate Box:** If $\mathbf{b}_{min} = \mathbf{b}_{max}$, the box is a point. The logic holds, returning an intersection only if the ray passes exactly through that point.

**Subtopics:**
*   [Ray-Cube Intersection](2466-ray-cube-intersection.md): Specialized for unit cubes where bounds are implicit. Simpler arithmetic but less general.
*   [AABB Ray Intersection](11086-aabb-ray-intersection.md): Often refers to boolean-only variants or specific optimizations for BVH traversal.
*   [Ray-AABB Intersection and Grid Traversal Setup](14352-ray-aabb-intersection-and-grid-traversal-setup.md): Extends this intersection to initialize voxel traversal steps (DDA).

## Operating characteristics
*   **Time Complexity:** $O(1)$ with a small constant number of arithmetic operations (approx. 10-15 flops).
*   **Memory Complexity:** $O(1)$.
*   **Strengths:** Extremely fast, branch-minimal potential, robust for standard AABBs.
*   **Limitations:** Does not handle rotated boxes. Precision issues can arise with very large coordinates or very small boxes due to floating-point cancellation in $(\mathbf{b} - \mathbf{o})$.
*   **When to use:** As the primary primitive test in ray tracers, BVH traversal, and volume rendering ray marching.

## Background
The slab method for AABB intersection is a classic technique in computer graphics. It is widely used in ray tracing kernels and volume rendering engines. The specific implementation details provided here are drawn from modern implementations in graphics libraries and research codebases.

## Sources
[S1] rordenlab/MRIcroWeb, https://github.com/rordenlab/MRIcroWeb, fetched 2026-10-26 (unofficial)
[S2] Decoupling Data Layouts from Bounding Volume Hierarchies, https://arxiv.org/pdf/2511.15028, fetched 2026-10-29
