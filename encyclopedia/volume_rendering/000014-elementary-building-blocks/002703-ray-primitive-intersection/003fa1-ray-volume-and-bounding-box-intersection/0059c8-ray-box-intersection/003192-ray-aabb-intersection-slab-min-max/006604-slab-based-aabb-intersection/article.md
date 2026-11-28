# Slab-based AABB Intersection
Also called: Slab method, Min-Max method, Ray-AABB slab test

## Overview
This family implements the standard algorithm for determining the intersection interval $[t_{min}, t_{max}]$ between a ray and an Axis-Aligned Bounding Box (AABB) by computing intersections with the box's six defining planes (slabs) [G]. It solves the fundamental problem of efficiently culling rays that miss a volume primitive or determining the precise entry and exit distances for ray marching [G]. It is distinguished from [Ray-Cube Intersection](2466-ray-cube-intersection.md) by returning scalar $t$-intervals rather than 3D coordinates and face indices, and from [AABB Intersection via Face Traversal](13498-aabb-intersection-via-face-traversal.md) by computing the full interval upfront rather than stepping through voxels [G].

## Shared principles
The core mechanism relies on the "slab" concept: an AABB is the intersection of three orthogonal slabs, where each slab is the region between two parallel planes perpendicular to an axis [G]. For a ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and an AABB defined by bounds $[x_{min}, x_{max}]$, $[y_{min}, y_{max}]$, $[z_{min}, z_{max}]$, the intersection with the slab along axis $i$ is found by solving for $t$ in:
$$ o_i + t \cdot d_i = b_i $$
where $b_i$ is either the min or max bound [G]. To avoid division by zero, the reciprocal direction $\mathbf{r}_i = 1/d_i$ is precomputed [G]. The intersection distances with the min and max planes for axis $i$ are:
$$ t_{i,1} = (b_{i,min} - o_i) \cdot r_i $$
$$ t_{i,2} = (b_{i,max} - o_i) \cdot r_i $$
The entry distance $t_{min}$ for the AABB is the maximum of the minimums of each slab pair:
$$ t_{min} = \max_{i \in \{x,y,z\}} (\min(t_{i,1}, t_{i,2})) $$
The exit distance $t_{max}$ is the minimum of the maximums of each slab pair:
$$ t_{max} = \min_{i \in \{x,y,z\}} (\max(t_{i,1}, t_{i,2})) $$
A ray intersects the AABB if $t_{min} \le t_{max}$ and the interval is valid (e.g., $t_{max} > t_{min} + \epsilon$) [G]. If $t_{min} > t_{max}$, the ray misses [G]. This method is $O(1)$ and branchless when implemented with vector operations, making it suitable for GPU acceleration [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def slab_aabb_intersect(o, d, bmin, bmax, epsilon=1e-6):
    inv_d = [1.0 / (d[i] + epsilon) for i in range(3)]
    t_min = -float('inf')
    t_max = float('inf')
    for i in range(3):
        t1 = (bmin[i] - o[i]) * inv_d[i]
        t2 = (bmax[i] - o[i]) * inv_d[i]
        if t1 > t2:
            t1, t2 = t2, t1
        t_min = max(t_min, t1)
        t_max = min(t_max, t2)
    if t_min > t_max:
        return False, -float('inf'), float('inf')
    return True, t_min, t_max
```

## Subtopics

### [Ray-Cube Intersection](2466-ray-cube-intersection.md)
Calculates precise entry/exit 3D positions and face indices for unit cubes. Chosen when hardware-accelerated voxel rendering requires the specific exit face for shading or traversal logic, rather than just the $t$-interval.

### [Ray-AABB Intersection](11086-ray-aabb-intersection.md)
Computes batched $t_{min}/t_{max}$ intervals for GPU kernels. Chosen for high-performance volume rendering where rays and AABBs are processed in parallel batches, handling division-by-zero via epsilon and clamping.

### [Ray-AABB Intersection and Grid Traversal Setup](14352-ray-aabb-intersection-and-grid-traversal-setup.md)
Prepares rays for multi-grid traversal by computing intervals, clamping to view frustum, and sorting. Chosen when a ray must traverse multiple distinct spatial grids or partitions in a specific order.

## Background
The slab method is a standard technique in ray tracing and volume rendering, originating from early ray-box intersection algorithms in computer graphics [G]. It is widely used in BVH traversal, voxel rendering, and collision detection due to its simplicity and efficiency [G].

## Sources
