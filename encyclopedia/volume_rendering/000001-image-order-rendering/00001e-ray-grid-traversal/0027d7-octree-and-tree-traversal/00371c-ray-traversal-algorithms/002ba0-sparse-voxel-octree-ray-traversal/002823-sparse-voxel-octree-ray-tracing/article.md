# Sparse Voxel Octree Ray Tracing

Also called: Octree Raymarching, SVO Ray Marching, SVO traversal, SVO::getHit, Sparse Voxel Octree (SVO) Ray Traversal, Sparse Voxel Octree Ray Marching

## Overview
Sparse Voxel Octree (SVO) Ray Tracing is a ray marching algorithm that traverses a 3D Sparse Voxel Octree to find the first occupied voxel (leaf node) intersected by a ray, enabling real-time voxel rendering without polygonal geometry [S1, unofficial][S2, unofficial][S6, unofficial]. It differs from standard ray-triangle rasterization by using a single draw call for a fullscreen triangle to fire one ray per pixel, descending the hierarchical octree structure to skip empty space efficiently via axis-aligned slab intersection logic, and shading the hit voxel using the face normal derived from the ray-AABB intersection [S1, unofficial][S6, unofficial]. It builds on [Sparse Voxel Octree Ray Traversal](2451-sparse-voxel-octree-ray-traversal.md) by adding the shading and hit-computation logic required for final image generation.

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $O$ and direction $D$ (per pixel), SVO data structure (nodes/leaves), camera parameters (FOV, view-projection matrix), world bounds.
*   **Output:** Hit position, hit normal, color, and status (hit, miss, out-of-bounds).

**Data Structures:**
*   **SVO Node:** Stored in a flat array. A 32-bit integer where MSB (bit 31) indicates a solid leaf. Lower 24 bits store RGB color (8 bits/channel). If MSB is 0, lower bits are an index to a children array [S6, unofficial].
*   **RayHit Struct:** `status` (u32: 0=no hit, 1=hit octree, 2=hit voxel, 3=out of bounds), `iterations` (u32), `position` (vec3<f32>), `normal` (vec3<f32>), `voxelData` (u32), `uv` (vec2<f32>) [S1, unofficial].
*   **Stack (Optional):** For backtracking in non-leaf traversal, a stack of `ParentElement` structs containing `pos` (float3), `scale` (float), `exit` (float), `index` (uint), `childIndex` (uint) [S2, unofficial]. Fixed size typically 16-64 elements [S2, unofficial].

**Parameters:**
*   `MAX_STEPS`: Maximum march steps per ray (e.g., 200) [S6, unofficial].
*   `EPS`: Epsilon for ray origin nudging (e.g., 0.0004) [S6, unofficial].
*   `mRootRadius`: Root radius of SVO (e.g., 256.0f) [S2, unofficial].
*   `MAXIMUM_TREE_DEPTH`: Max octree depth (e.g., 30) [S3, unofficial].

**Steps:**

1.  **Ray Generation:** For each pixel, transform normalized device coordinates (NDC) using the inverse view-projection matrix to get world-space ray origin and direction [S6, unofficial].
2.  **World Bounds Check:** Perform a ray-AABB slab test against the world bounds ($world_{min}, world_{max}$). If no intersection or $t_{far} < 0$, return sky color [S6, unofficial].
    $$ t_{near} = \max(\min(\frac{box_{min}-O}{D}), \min(\frac{box_{min}-O}{D}), \min(\frac{box_{min}-O}{D})) $$
    $$ t_{far} = \min(\max(\frac{box_{max}-O}{D}), \max(\frac{box_{max}-O}{D}), \max(\frac{box_{max}-O}{D})) $$
3.  **Root Intersection:** Check intersection with the root octree box. If the ray does not intersect the root, return miss [S2, unofficial].
4.  **Traversal Loop:** Iterate up to `MAX_STEPS`.
    *   **Node Check:** Determine if the current voxel/octant at the ray's current position is occupied using `GetOctant` or by checking the SVO node's valid mask [S1, unofficial][S2, unofficial].
    *   **Hit Detection:** If the node is a solid leaf (MSB set) or active, calculate the exact intersection depth with the voxel's AABB using `RayBoxIntersect` [S1, unofficial][S6, unofficial].
    *   **Backtracking (if using stack):** If the current node is empty or not a leaf, push the current state to the stack, move to the next child along the ray direction, and continue. If no children exist, pop from the stack to backtrack [S2, unofficial].
    *   **Step Forward:** If the current octant is empty, determine the next axis to step along by finding the minimum component of the translation vector to the next slab boundary. Update ray position and translation parameters [S1, unofficial].
    *   **Bounds Check:** If the new position is out of world bounds, set status to 3 (out of bounds) and return [S1, unofficial].
5.  **Termination:**
    *   If a hit is found, compute the hit position $P = O + D \cdot t_{min}$ and the normal [S7, unofficial].
    *   If the loop completes without a hit, calculate the intersection with the final voxel box or return a miss status [S1, unofficial].
6.  **Shading:** Shade the hit voxel using the computed normal. Common shading includes ambient and diffuse lighting [S6, unofficial][S5, unofficial].
    $$ Color = (Ambient + Diffuse) \cdot ObjectColor $$
    $$ Diffuse = \max(0, \hat{N} \cdot \hat{L}) \cdot LightColor $$
    [S5, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_STEPS = 200
EPS = 0.0004
SKY_COLOR = vec3(0.53, 0.81, 0.98)

def ray_aabb(ro, inv_rd, box_min, box_max):
    t0 = (box_min - ro) * inv_rd
    t1 = (box_max - ro) * inv_rd
    t_near = max(min(t0.x, t1.x), min(t0.y, t1.y), min(t0.z, t1.z))
    t_far = min(max(t0.x, t1.x), max(t0.y, t1.y), max(t0.z, t1.z))
    return t_near, t_far

def get_octant_normal(hit_pos, node_center, half_size):
    center_to_hit = hit_pos - node_center
    max_comp = max(abs(center_to_hit.x / half_size.x), 
                   abs(center_to_hit.y / half_size.y), 
                   abs(center_to_hit.z / half_size.z))
    normal = vec3(0.0)
    if abs(center_to_hit.x / half_size.x) == max_comp:
        normal.x = 1.0 if center_to_hit.x > 0 else -1.0
    elif abs(center_to_hit.y / half_size.y) == max_comp:
        normal.y = 1.0 if center_to_hit.y > 0 else -1.0
    else:
        normal.z = 1.0 if center_to_hit.z > 0 else -1.0
    return normal

def shade_hit(hit_pos, normal, light_pos, light_color, object_color):
    ambient = 0.1 * light_color
    diffuse = max(dot(normal, normalize(light_pos - hit_pos)), 0.0) * light_color
    return (ambient + diffuse) * object_color

def trace_svo_ray(ray_origin, ray_dir, svo_info, light_pos, light_color):
    # 1. Ray Generation & Inverse Direction
    inv_rd = 1.0 / ray_dir
    # Clamp near-zero components to avoid division by zero
    inv_rd = vec3(1e-4) if abs(inv_rd) < 1e-4 else inv_rd
    
    # 2. World Bounds Check
    t_near, t_far = ray_aabb(ray_origin, inv_rd, svo_info.world_min, svo_info.world_max)
    if t_near > t_far or t_far < 0:
        return SKY_COLOR
    
    t = max(t_near, 0.0) + EPS
    t_max = t_far
    
    # 3. Root Intersection
    # Assuming root box is defined by svo_info.origin and svo_info.size
    root_min = svo_info.origin
    root_max = svo_info.origin + svo_info.size
    root_t_near, root_t_far = ray_aabb(ray_origin, inv_rd, root_min, root_max)
    
    if root_t_near > root_t_far or root_t_far < 0:
        return SKY_COLOR
        
    # Adjust t to start inside or just after root entry
    t = max(root_t_near, 0.0) + EPS
    
    # 4. Traversal Loop
    for i in range(MAX_STEPS):
        # Determine current voxel position based on t
        # Note: In a real SVO, this involves traversing the tree structure
        # Here we simulate the "step" logic from S1/S2 for simplicity of the core concept
        # In practice, you would use the stack-based or slab-based traversal to find the current node
        
        # Check if current voxel is solid (Leaf check)
        # This requires accessing the SVO data structure at the current coordinate
        current_node = get_node_at_position(ray_origin + ray_dir * t, svo_info)
        
        if current_node.is_leaf and current_node.is_solid:
            # 5. Hit Detection
            # Calculate exact intersection with this voxel's AABB
            voxel_min = current_node.position
            voxel_max = current_node.position + current_node.size
            hit_t, _ = ray_aabb(ray_origin, inv_rd, voxel_min, voxel_max)
            
            if hit_t >= 0 and hit_t <= t_max:
                hit_pos = ray_origin + ray_dir * hit_t
                normal = get_octant_normal(hit_pos, current_node.position + current_node.size/2, current_node.size/2)
                return shade_hit(hit_pos, normal, light_pos, light_color, current_node.color)
        
        # 6. Step Forward (Simplified slab stepping for empty space)
        # In a full implementation, this is replaced by tree traversal logic
        # Find the next slab boundary
        t_next_x = (voxel_min.x - ray_origin.x) * inv_rd.x
        t_next_y = (voxel_min.y - ray_origin.y) * inv_rd.y
        t_next_z = (voxel_min.z - ray_origin.z) * inv_rd.z
        
        # Advance to the next voxel along the ray
        t += max(t_next_x, t_next_y, t_next_z) + EPS
        
        if t > t_max:
            break
            
    # 7. Miss
    return SKY_COLOR
```

## Subtopics

### [SVO Ray Marching with Stack-Based Navigation](2454-svo-ray-marching-with-stack-based-navigation.md)
Uses an explicit stack to manage traversal state, decoupling logic from parametric distance and allowing robust handling of empty space and early termination based on pixel error thresholds.

### [Octree Voxel Raycasting for Hover (Closest Active Voxel)](3219-octree-voxel-raycasting-for-hover-closest-active-voxel.md)
Specialized for interaction, finding the closest *active* leaf node while tracking empty space for fallback, rather than full volume integration.

### [Octree Ray Marching with Brick Interpolation and Early Termination](4147-octree-ray-marching-with-brick-interpolation-and-early-termi.md)
Combines coarse space-skipping via precomputed "bricks" with fine-grained interpolation, reducing memory accesses for uniform regions.

### [Sparse Voxel Octree Ray Casting with Distance Fields](13018-sparse-voxel-octree-ray-casting-with-distance-fields.md)
Uses distance fields on bricks of voxels at the bottom levels to render isosurfaces, scaling better for planetary sizes.

## Operating characteristics
*   **Time Complexity:** $O(\log N)$ for tree traversal to the first hit, where $N$ is the number of voxels, assuming a balanced octree. However, worst-case is $O(D)$ where $D$ is the depth of the tree [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the SVO nodes, where $N$ is the number of occupied voxels. The stack size is fixed and small (e.g., 16-64 elements) [S2, unofficial].
*   **Performance:** Real-time capable on modern GPUs. Renders the entire scene with a single draw call (fullscreen triangle) [S6, unofficial]. Performance scales with the number of steps and tree depth.
*   **Strengths:** No mesh generation required, efficient culling of empty space, supports dynamic updates [S6, unofficial].
*   **Limitations:** Precision issues with ray origin nudging and slab boundaries [S6, unofficial][S3, unofficial]. Limited by the maximum tree depth and stack size for backtracking [S2, unofficial].

## Background
Sparse Voxel Octree Ray Tracing originated as a method to render large voxel datasets efficiently without the overhead of polygonal conversion. It leverages the hierarchical nature of octrees to skip empty space, making it suitable for real-time applications like voxel-based games and planetary visualization [S1, unofficial][S6, unofficial]. The method is often implemented in GLSL or WGSL fragment/compute shaders [S1, unofficial][S6, unofficial].

## Sources
[S1] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
[S2] Onlinerocker/SVO, https://github.com/Onlinerocker/SVO, fetched 2026-10-24 (unofficial)
[S3] realazthat/landscapes, https://github.com/realazthat/landscapes, fetched 2026-10-26 (unofficial)
[S4] krre/origin, https://github.com/krre/origin, fetched 2026-10-26 (unofficial)
[S5] cianjinks/Sparse-Voxel-Octree, https://github.com/cianjinks/Sparse-Voxel-Octree, fetched 2026-10-26 (unofficial)
[S6] scarlyle0/voxel-compare, https://github.com/scarlyle0/voxel-compare, fetched 2026-10-27 (unofficial)
[S7] GitHub - saurlax/model_voxelizer: A 3D model voxelization tool that ..., https://github.com/saurlax/model_voxelizer, fetched 2026-10-30 (unofficial)
