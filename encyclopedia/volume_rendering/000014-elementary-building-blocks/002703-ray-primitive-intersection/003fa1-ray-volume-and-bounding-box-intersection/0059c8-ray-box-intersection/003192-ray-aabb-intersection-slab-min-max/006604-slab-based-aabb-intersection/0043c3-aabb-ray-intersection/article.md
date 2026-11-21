# AABB Ray Intersection

Also called: slab method, Ray-AABB Intersection Kernel

## Overview
The AABB Ray Intersection method computes the entry ($t_{min}$) and exit ($t_{max}$) distances of a ray against an Axis-Aligned Bounding Box (AABB), returning a boolean hit status. It solves the fundamental problem of determining whether a ray intersects a volume primitive and, if so, the interval along the ray parameter $t$ where the intersection occurs. It is distinguished from its siblings by its reliance on the "slab" method (computing intersections with min/max planes per axis) and its common implementation as a batched kernel for GPU acceleration, whereas [Ray-Bounding Box Intersection and Sampling Preparation](2364-ray-bounding-box-intersection-and-sampling-preparation.md) focuses on sampling preparation and [Ray-Cube Intersection](2466-ray-cube-intersection.md) assumes unit cubes.

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   Ray origins: $\mathbf{o} \in \mathbb{R}^{N \times 3}$
    *   Ray directions: $\mathbf{d} \in \mathbb{R}^{N \times 3}$
    *   AABBs: $\mathbf{B} \in \mathbb{R}^{M \times 6}$, where each row is $[x_{min}, y_{min}, z_{min}, x_{max}, y_{max}, z_{max}]$.
*   **Outputs:**
    *   $t_{min} \in \mathbb{R}^{N \times M}$: Entry distances.
    *   $t_{max} \in \mathbb{R}^{N \times M}$: Exit distances.
    *   $hit \in \{0, 1\}^{N \times M}$: Boolean intersection mask.

**Data Structures:**
*   **VolumeRange (Optional):** A data structure containing fields $t_0$ (entry), $t_1$ (exit), and $intersected$ (boolean mask), all with shape $[batch\_size, *shape, 1]$ [S3, unofficial].

**Parameters:**
*   $\epsilon$: Small value to prevent division by zero (default $10^{-6}$) [S3, unofficial].
*   $min\_dist$: Minimum allowed value for $t_0$ (default $0.0$) [S3, unofficial].
*   $min\_t\_range$: Minimum step size for valid intersection (default $10^{-3}$) [S3, unofficial].
*   $near\_plane$: Clamping lower bound for $t_{min}$ (often $0.0$ during inference) [S2, unofficial].

**Steps:**
1.  **Compute Direction Fractions:** To avoid division by zero, compute the reciprocal of the ray direction components:
    $$ \mathbf{f} = \frac{1}{\mathbf{d} + \epsilon} $$
    where $\epsilon$ is a small constant (e.g., $10^{-6}$) [S2, unofficial].

2.  **Compute Slab Intersections:** For each axis $i \in \{x, y, z\}$, calculate the intersection distances with the min and max planes of the AABB. Let $\mathbf{B}_{min} = [x_{min}, y_{min}, z_{min}]$ and $\mathbf{B}_{max} = [x_{max}, y_{max}, z_{max}]$.
    $$ t_{i,1} = (\mathbf{B}_{min}[i] - \mathbf{o}[i]) \cdot \mathbf{f}[i] $$
    $$ t_{i,2} = (\mathbf{B}_{max}[i] - \mathbf{o}[i]) \cdot \mathbf{f}[i] $$
    This yields six values per ray-AABB pair: $t_{x,1}, t_{x,2}, t_{y,1}, t_{y,2}, t_{z,1}, t_{z,2}$ [S2, unofficial].

3.  **Determine Entry and Exit:**
    *   The entry distance $t_{min}$ is the maximum of the minimums along each axis:
        $$ t_{min} = \max_{i \in \{x,y,z\}} (\min(t_{i,1}, t_{i,2})) $$
    *   The exit distance $t_{max}$ is the minimum of the maximums along each axis:
        $$ t_{max} = \min_{i \in \{x,y,z\}} (\max(t_{i,1}, t_{i,2})) $$
    [S2, unofficial][S3, unofficial].

4.  **Apply Constraints and Clamping:**
    *   Clamp $t_{min}$ to be at least $min\_dist$ (typically $0.0$) to handle rays starting inside the box or behind the origin [S3, unofficial].
    *   Clamp $t_{min}$ to the $near\_plane$ [S2, unofficial].
    *   Ensure $t_{max} > t_{min}$ by at least a small margin (e.g., $10^{-6}$) to prevent degenerate intervals [S2, unofficial].
    *   If a lower bound $t_{0,lower}$ is provided (e.g., from a previous traversal step), update $t_{min} = \max(t_{min}, t_{0,lower})$ [S3, unofficial].

5.  **Determine Hit Status:**
    *   A ray intersects if $t_{min} + min\_t\_range < t_{max}$ [S3, unofficial].
    *   If not intersected, set $t_{min}$ and $t_{max}$ to a miss value (e.g., $-\infty$ or a specific flag) [S1, unofficial][S5, unofficial].

**Pseudocode (PyTorch-like):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_aabb_intersect(rays_o, rays_d, aabbs, epsilon=1e-6, min_dist=0.0, min_t_range=1e-3):
    """
    rays_o: [N, 3]
    rays_d: [N, 3]
    aabbs: [M, 6] -> [x_min, y_min, z_min, x_max, y_max, z_max]
    Returns: t_min [N, M], t_max [N, M], hit [N, M]
    """
    # 1. Compute direction fractions
    dir_frac = 1.0 / (rays_d[:, None, :] + epsilon) # [N, 1, 3]

    # 2. Extract AABB bounds
    aabb_min = aabbs[:, :3] # [M, 3]
    aabb_max = aabbs[:, 3:] # [M, 3]

    # 3. Compute t1 and t2 for all 6 planes
    # t1: intersection with min planes
    t1 = (aabb_min[None, :, :] - rays_o[:, None, :]) * dir_frac # [N, M, 3]
    # t2: intersection with max planes
    t2 = (aabb_max[None, :, :] - rays_o[:, None, :]) * dir_frac # [N, M, 3]

    # 4. Compute t_min and t_max
    # t_min: max of mins along axis dim (dim=-1)
    t_min = torch.max(torch.minimum(t1, t2), dim=-1).values # [N, M]
    # t_max: min of maxs along axis dim (dim=-1)
    t_max = torch.min(torch.maximum(t1, t2), dim=-1).values # [N, M]

    # 5. Apply constraints
    t_min = torch.clamp(t_min, min=min_dist)
    # Ensure t_max > t_min
    t_max = torch.maximum(t_max, t_min + 1e-6)

    # 6. Determine hit status
    hit = (t_min + min_t_range < t_max)

    # 7. Set miss values
    miss_value = -1e10 # Or appropriate miss flag
    t_min = torch.where(hit, t_min, miss_value)
    t_max = torch.where(hit, t_max, miss_value)

    return t_min, t_max, hit
```

## Subtopics

### Ray-AABB Intersection (PyTorch)
A pure PyTorch implementation of the slab method. It is functionally identical to the CUDA kernel but slower due to lack of specialized hardware instructions and memory coalescing. It follows the same steps: broadcasting ray origins/directions against AABB bounds, computing $t_1/t_2$, reducing via min/max, and clamping [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ for $N$ rays and $M$ AABBs, with a very small constant factor (6 subtractions, 6 multiplications, 6 comparisons per pair).
*   **Memory Complexity:** $O(N \cdot M)$ to store the output tensors $t_{min}, t_{max}, hit$.
*   **Performance:** Highly parallelizable on GPUs. Batched implementations (e.g., processing chunks of rays) are used to avoid out-of-memory errors when $N$ is large [S4, unofficial].
*   **Strengths:** Fast, branchless (in optimized implementations), handles arbitrary AABB sizes.
*   **Limitations:** Does not handle rotated boxes (use [Ray-OBB Intersection Test](6603-ray-obb-intersection-test.md) instead). Precision issues can arise if ray directions are near-zero without epsilon handling [S2, unofficial].

## Background
The slab method is a standard technique in ray tracing, originally described in [S1, unofficial] and [S5, unofficial] as part of modern neural rendering pipelines (NeRF, SDFStudio). It is a direct application of the separating axis theorem for axis-aligned boxes. The specific batching and clamping strategies described here are adapted from implementations in [S1, unofficial] (taylor-nerf), [S2, unofficial] (sdfstudio), [S3, unofficial] (gsgen), [S4, unofficial] (NSVF), and [S5, unofficial] (CNC).

## Sources
[S1] taylor2nerf/taylor-nerf, https://github.com/taylor2nerf/taylor-nerf, fetched 2026-10-26 (unofficial)
[S2] autonomousvision/sdfstudio, https://github.com/autonomousvision/sdfstudio, fetched 2026-10-26 (unofficial)
[S3] gsgen3d/gsgen, https://github.com/gsgen3d/gsgen, fetched 2026-10-27 (unofficial)
[S4] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
[S5] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
