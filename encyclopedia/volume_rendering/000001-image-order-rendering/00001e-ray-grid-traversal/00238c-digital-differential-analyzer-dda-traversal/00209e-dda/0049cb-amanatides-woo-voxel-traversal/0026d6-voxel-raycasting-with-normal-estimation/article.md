# Voxel Raycasting with Normal Estimation

Also called: DDA-like voxel traversal, Digital Differential Analyzer voxel traversal, Step-wise ray-sphere intersection for voxels, Voxel Raycasting via DDA, VoxelTrace

## Overview
Voxel Raycasting with Normal Estimation is a method for tracing a ray through a regular 3D voxel grid to locate the first solid voxel and estimate the surface normal at the point of intersection [S1, S2, unofficial]. It solves the problem of efficient block placement and collision detection in voxel-based environments by visiting only intersected cells, offering better performance than general-purpose physics raycasts for voxel grids [S1, unofficial]. It differs from its siblings by explicitly estimating the entry normal based on the axis of traversal (the axis with the smallest delta value in the DDA step) rather than relying on geometric surface reconstruction or post-processing [S1, S2, unofficial].

## How it works
This method builds on [Amanatides-Woo Voxel Traversal](3132-amanatides-woo-voxel-traversal.md) by adding a normal estimation step derived from the traversal direction. The algorithm steps through the grid using a Digital Differential Analyzer (DDA) approach, advancing to the nearest voxel boundary on the axis with the smallest t-value [S1, S2, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   `ray.origin`: The starting position of the ray in world coordinates ($\mathbb{R}^3$).
    *   `ray.direction`: The normalized direction vector of the ray ($\mathbb{R}^3$).
    *   `maxDist`: The maximum length of the ray cast ($\mathbb{R}$).
    *   `solidAt(x, y, z)`: A function returning true if the voxel at integer coordinates $(x, y, z)$ is occupied/solid.
*   **Outputs**:
    *   `hit`: Boolean indicating if a solid voxel was found.
    *   `hitPos`: The integer voxel coordinates $(x, y, z)$ of the first solid voxel hit.
    *   `normal`: The estimated surface normal vector ($\mathbb{R}^3$) at the entry point.

### Data Structures
*   **Ray**: Defined by origin and direction.
*   **Voxel Coordinates**: Integer triplets $(x, y, z)$.
*   **Delta Values**: Floating-point values representing the distance to the next voxel boundary along each axis.
*   **Step Vector**: Integer triplet $(sx, sy, sz)$ indicating the direction of traversal along each axis.

### Parameters
*   `epsilon`: A small constant to handle near-zero ray direction components and avoid division by zero or NaN issues. Value: $0.0001$ [S1, unofficial].
*   `maxDist`: Maximum ray length. Value: $6.0$ [S2, unofficial].
*   `stepSize`: Increment for the DDA loop. Value: $0.05$ [S2, unofficial]. Note: [S1, unofficial] uses a continuous DDA with variable step sizes based on ray direction, while [S2, unofficial] uses a fixed discrete step size. The implementation below follows the [S2, unofficial] fixed-step approach as it is explicitly detailed with normal estimation logic, but acknowledges the [S1, unofficial] continuous DDA structure for boundary calculation.

### Algorithm Steps

1.  **Initialization**:
    *   Set `pos = ray.origin`.
    *   Set `prev = ray.origin`.
    *   Initialize `hit = false`.
    *   Initialize `normal = (0, 0, 0)`.

2.  **Degeneracy Check**:
    *   If the squared magnitude of `ray.direction` is less than `epsilon`, return `hit = false` [S1, unofficial].

3.  **Loop**:
    *   Iterate `t` from $0$ to `maxDist` with increment `stepSize` ($0.05$) [S2, unofficial].
    *   Calculate current position: `pos = ray.origin + ray.direction * t`.
    *   Determine voxel coordinates:
        *   $x = \lfloor pos.x \rfloor$
        *   $y = \lfloor pos.y \rfloor$
        *   $z = \lfloor pos.z \rfloor$
    *   Check if voxel is solid:
        *   If `solidAt(x, y, z)` is true:
            *   Set `hitPos = (x, y, z)`.
            *   Set `hit = true`.
            *   Break loop.

4.  **Normal Estimation**:
    *   If `hit` is true:
        *   Calculate displacement vector: `diff = pos - prev`.
        *   Compare absolute components of `diff`:
            *   If $|diff.x| > |diff.y|$ and $|diff.x| > |diff.z|$:
                *   `normal.x = -1.0` if `diff.x > 0` else $1.0$.
                *   `normal.y = 0.0`.
                *   `normal.z = 0.0`.
            *   Else if $|diff.y| > |diff.z|$:
                *   `normal.x = 0.0`.
                *   `normal.y = -1.0` if `diff.y > 0` else $1.0$.
                *   `normal.z = 0.0`.
            *   Else:
                *   `normal.x = 0.0`.
                *   `normal.y = 0.0`.
                *   `normal.z = -1.0` if `diff.z > 0` else $1.0$.
        *   Return `hit`, `hitPos`, `normal`.

5.  **No Hit**:
    *   If the loop completes without finding a solid voxel, return `hit = false`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_raycast_with_normal_estimation(ray_origin, ray_direction, max_dist=6.0, step_size=0.05, epsilon=0.0001):
    """
    Traces a ray through a voxel grid to find the first solid voxel and estimate its normal.
    
    Args:
        ray_origin (Vec3): Starting position of the ray.
        ray_direction (Vec3): Normalized direction vector of the ray.
        max_dist (float): Maximum length of the ray cast.
        step_size (float): Increment for the DDA loop.
        epsilon (float): Small constant to handle near-zero components.
        
    Returns:
        tuple: (hit: bool, hit_pos: Vec3, normal: Vec3)
    """
    # Check for degenerate ray
    if ray_direction.length_squared() < epsilon:
        return (False, Vec3(0, 0, 0), Vec3(0, 0, 0))
    
    pos = ray_origin
    prev = ray_origin
    hit = False
    hit_pos = Vec3(0, 0, 0)
    normal = Vec3(0, 0, 0)
    
    # Iterate along the ray
    t = 0.0
    while t <= max_dist:
        # Calculate current position
        pos = ray_origin + ray_direction * t
        
        # Determine voxel coordinates
        x = floor(pos.x)
        y = floor(pos.y)
        z = floor(pos.z)
        
        # Check if voxel is solid
        if solid_at(x, y, z):
            hit = True
            hit_pos = Vec3(x, y, z)
            break
        
        # Update previous position
        prev = pos
        
        # Increment t
        t += step_size
    
    if hit:
        # Estimate normal based on displacement
        diff = pos - prev
        
        abs_diff_x = abs(diff.x)
        abs_diff_y = abs(diff.y)
        abs_diff_z = abs(diff.z)
        
        if abs_diff_x > abs_diff_y and abs_diff_x > abs_diff_z:
            # X-axis is dominant
            normal.x = -1.0 if diff.x > 0 else 1.0
            normal.y = 0.0
            normal.z = 0.0
        elif abs_diff_y > abs_diff_z:
            # Y-axis is dominant
            normal.x = 0.0
            normal.y = -1.0 if diff.y > 0 else 1.0
            normal.z = 0.0
        else:
            # Z-axis is dominant
            normal.x = 0.0
            normal.y = 0.0
            normal.z = -1.0 if diff.z > 0 else 1.0
            
    return (hit, hit_pos, normal)
```

### Edge Cases and Pitfalls
*   **Large Step Size**: If the step size is large relative to voxel size, the normal estimation may be inaccurate. The ray might pass through a corner or edge, and the displacement vector `diff` may not align with the actual entry face normal [S2, unofficial].
*   **Axis-Aligned Rays**: If the ray direction is exactly aligned with one of the grid axes, the displacement vector will have two zero components. The normal estimation logic handles this by selecting the non-zero component, but precision issues may arise if the ray origin is exactly on a voxel boundary [S1, S2, unofficial].
*   **Degenerate Rays**: Rays with zero length or near-zero direction vectors should be handled explicitly to avoid division by zero or NaN results [S1, unofficial].

## Subtopics

### [DN_step_map](8596-dn-step-map.md)
Computes the index of the first non-empty voxel intersected by a ray in a regular 3D grid using the Amanatides & Woo traversal algorithm. It differs from sibling methods by returning the specific voxel index of the first hit (or a max uint value if empty) rather than just a boolean hit status, and by handling zero-direction components explicitly to prevent division by zero. Choose this when you need the exact voxel index for further processing rather than just a hit/miss status.

## Operating characteristics
*   **Time Complexity**: $O(max\_distance)$, where $max\_distance$ is the length of the ray cast [S1, unofficial]. The number of steps is proportional to the ray length divided by the step size.
*   **Memory Complexity**: $O(1)$ auxiliary space, as the algorithm only stores a few variables for the current and previous positions [S1, S2, unofficial].
*   **Performance**: More efficient than PhysX raycasts for voxel grids due to direct grid traversal without physics engine overhead [S1, unofficial].
*   **Strengths**: Simple to implement, efficient for sparse or dense voxel grids, provides immediate normal estimation.
*   **Limitations**: Accuracy of normal estimation depends on step size; large steps can lead to incorrect normals. Not suitable for sub-voxel precision without significant optimization.
*   **When to Use**: Block placement, collision detection, and simple raycasting in voxel-based games or simulations where high precision is not critical.
*   **When Not to Use**: When sub-voxel accuracy is required, or when the ray needs to traverse very long distances with high precision.

## Background
This method is commonly used in voxel-based games and simulations for tasks such as block placement, breaking, and collision detection. It is often implemented in custom voxel engines or as part of a larger rendering pipeline. The approach is inspired by the Amanatides-Woo voxel traversal algorithm, which is widely used for ray tracing in voxel grids [S1, S2, unofficial]. The specific normal estimation technique is a heuristic derived from the direction of traversal, providing a fast approximation of the surface normal without additional geometric calculations.

## Sources
[S1] VektorKnight/vektor-voxels, https://github.com/VektorKnight/vektor-voxels, fetched 2026-10-27 (unofficial)
[S2] whudsontosh/CPU-Rasterizer, https://github.com/whudsontosh/CPU-Rasterizer, fetched 2026-10-27 (unofficial)
