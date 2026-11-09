# Sparse Voxel Octree Ray Tracing

Also called: Octree Ray Casting, Octree Raymarching, SVO Ray Marching, SVO traversal, SVO::getHit, Sparse Voxel Octree (SVO) Ray Traversal, Sparse Voxel Octree Ray Marching

## Overview
Sparse Voxel Octree (SVO) Ray Tracing is a ray traversal algorithm that finds the first intersection with a solid voxel in a 3D sparse voxel octree [S1, S2, S6, unofficial]. It solves the problem of efficiently rendering large, complex voxel scenes by skipping empty space using axis-aligned slab intersection logic, avoiding the need to sample every voxel [S1, S6, unofficial]. Unlike standard rasterization which requires mesh generation and draw calls per chunk, SVO ray tracing renders the entire scene with a single draw call (e.g., a fullscreen triangle) and no vertex buffers [S6, unofficial]. It differs from its parent, [Sparse Voxel Octree Ray Traversal](2451-sparse-voxel-octree-ray-traversal.md), by explicitly handling backtracking via a stack or recursive logic to manage hierarchical node descent, rather than relying solely on parallel array-based stack simulation [S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin ($O$), ray direction ($D$), SVO root node index, world bounds (origin, size), maximum traversal depth/iterations, and a children buffer for internal nodes [S2, S6, unofficial].
*   **Outputs:** Hit position, hit normal, hit depth ($t$), and status (hit, miss, out of bounds) [S1, S2, unofficial].

**Data Structures:**
*   **SVO Node:** Stored in a flat array or buffer. A 32-bit integer where the MSB (bit 31) indicates a solid leaf. If MSB is 0, the lower 24 bits store an index to the children array [S6, unofficial].
*   **Stack:** Used for backtracking when exiting child nodes. A fixed-size array (e.g., 64 elements) storing `ParentElement` structs [S2, unofficial].
    *   `ParentElement`: `float3 pos`, `float scale`, `float exit`, `uint index`, `uint childIndex` [S2, unofficial].
*   **RayHit Struct:** `status` (u32: 0=no hit, 1=hit octree, 2=hit voxel, 3=out of bounds), `iterations` (u32), `position` (vec3<f32>), `normal` (vec3<f32>) [S1, unofficial].

**Parameters:**
*   `mRootRadius`: Default root radius, typically 256.0f [S2, unofficial].
*   `MAX_STEPS`: Maximum march steps, e.g., 200 [S6, unofficial].
*   `EPS`: Epsilon for ray origin nudging, e.g., 0.0004 [S6, unofficial].
*   `MAXIMUM_TREE_DEPTH`: Max depth, e.g., 30 [S3, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Define world bounds ($world_{min}, world_{max}$) [S6, unofficial].
    *   Perform a ray-AABB slab test against world bounds. If no intersection ($t_{near} > t_{far}$ or $t_{far} < 0$), return sky color [S6, unofficial].
    *   Initialize traversal stack. Set root position to $(0,0,0)$ and root scale to $mRootRadius$ [S2, unofficial].
    *   If the ray origin is inside the root cube, locate the current node using `svo_locate_node` [S3, unofficial].

2.  **Root Intersection:**
    *   Compute intersection of the ray with the root box. Get entry ($t_{entry}$) and exit ($t_{exit}$) distances [S2, unofficial].
    *   If $t_{entry} \le 0$ and $t_{exit} \le 0$, the ray misses the root; return false/miss [S2, unofficial].
    *   Clamp $t_{entry}$ to 0.0 if negative to handle rays starting inside the volume.

3.  **Traversal Loop:**
    *   While steps < limit:
        *   **Child Selection:** Determine the child index intersected by the ray at the current entry point. This is done by comparing the ray position relative to the center of the current node's bounding box [S2, unofficial].
        *   **Node Check:** Check if the selected child exists and is valid using the node's descriptor mask (e.g., `getValidMask`) [S2, unofficial].
        *   **Leaf Check:** If the child is a leaf (`getLeafMask > 0`) and valid, and the ray intersects it ($t_{entry} \ge 0$), a hit is found.
            *   Compute hit position: $P = O + D \cdot t_{entry}$ [S2, unofficial].
            *   Compute normal based on the face of the bounding box hit [S7, unofficial].
            *   Set `didNotHit = false` and break [S2, unofficial].
        *   **Internal Node:** If the child is an internal node:
            *   Push current state (position, scale, exit distance, node index, next sibling index) onto the stack [S2, unofficial].
            *   Descend: Update current position to the child's box, update scale, and update the node index to the child's pointer [S2, unofficial].
            *   Recalculate ray intersection with the new child box to get new $t_{entry}$ and $t_{exit}$ [S2, unofficial].
        *   **Backtracking:** If the ray misses the current child or no valid child is found:
            *   Pop from stack to restore previous state [S2, unofficial].
            *   Continue traversal from the parent node, checking the next sibling child [S2, unofficial].

4.  **Termination:**
    *   If a hit is found, return hit data [S1, unofficial].
    *   If max iterations/steps are reached, return miss or last known position [S1, S6, unofficial].
    *   If out of world bounds, return out-of-bounds status [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_svo_ray(ray_origin, ray_dir, svo_nodes, children_buffer, root_index, world_min, world_max, max_steps):
    # 1. World Bounds Check
    t_near, t_far = ray_aabb(ray_origin, ray_dir, world_min, world_max)
    if t_near > t_far or t_far < 0:
        return {"status": 0, "position": None}

    # 2. Initialization
    stack = []
    current_node_idx = root_index
    current_pos = vec3(0.0, 0.0, 0.0)
    current_scale = 256.0 # mRootRadius
    t_entry = 0.0
    steps = 0
    
    # 3. Root Intersection
    root_box_min = current_pos - current_scale
    root_box_max = current_pos + current_scale
    t_entry, t_exit = ray_aabb(ray_origin, ray_dir, root_box_min, root_box_max)
    
    if t_entry <= 0.0 and t_exit <= 0.0:
        return {"status": 0, "position": None}
    if t_entry < 0.0:
        t_entry = 0.0
        
    # Nudge origin to avoid precision issues
    ray_origin = ray_origin + ray_dir * 0.0004

    while steps < max_steps:
        steps += 1
        
        # 4. Determine Child Index
        rel_pos = (ray_origin + ray_dir * t_entry - current_pos) / current_scale
        child_idx = 0
        if rel_pos.x > 0.0: child_idx |= 1
        if rel_pos.y > 0.0: child_idx |= 2
        if rel_pos.z > 0.0: child_idx |= 4
        
        # 5. Check Child Validity
        node_val = svo_nodes[current_node_idx]
        is_leaf = (node_val & 0x80000000) != 0
        
        if is_leaf:
            hit_pos = ray_origin + ray_dir * t_entry
            normal = compute_normal(hit_pos, current_pos, current_scale, child_idx)
            return {"status": 1, "position": hit_pos, "normal": normal, "depth": t_entry}
        
        # Internal node: Check if child exists
        child_node_idx = get_child_node_index(svo_nodes, children_buffer, current_node_idx, child_idx)
        
        if child_node_idx is None:
            # Child does not exist (empty space)
            # Backtrack to try next sibling
            if len(stack) == 0:
                return {"status": 0, "position": None}
            prev_state = stack.pop()
            current_node_idx = prev_state["node_idx"]
            current_pos = prev_state["pos"]
            current_scale = prev_state["scale"]
            t_entry = prev_state["t_entry"]
            child_idx = prev_state["next_child_idx"]
            continue
            
        # 6. Descend into Child
        child_scale = current_scale / 2.0
        child_offset = vec3(
            (child_idx & 1) * child_scale,
            (child_idx & 2) * child_scale,
            (child_idx & 4) * child_scale
        )
        child_pos = current_pos + child_offset - child_scale
        
        child_box_min = child_pos - child_scale
        child_box_max = child_pos + child_scale
        t_child_entry, t_child_exit = ray_aabb(ray_origin, ray_dir, child_box_min, child_box_max)
        
        if t_child_entry < 0:
            t_child_entry = 0.0
            
        if t_child_entry > t_child_exit:
            # Ray misses child box
            if len(stack) == 0:
                return {"status": 0, "position": None}
            prev_state = stack.pop()
            current_node_idx = prev_state["node_idx"]
            current_pos = prev_state["pos"]
            current_scale = prev_state["scale"]
            t_entry = prev_state["t_entry"]
            child_idx = prev_state["next_child_idx"]
            continue
            
        # Push current state to stack for potential backtracking
        stack.append({
            "node_idx": current_node_idx,
            "pos": current_pos,
            "scale": current_scale,
            "t_entry": t_entry,
            "next_child_idx": child_idx + 1
        })
        
        # Update current state
        current_node_idx = child_node_idx
        current_pos = child_pos
        current_scale = child_scale
        t_entry = t_child_entry

    return {"status": 0, "position": None}

def ray_aabb(ro, rd, box_min, box_max):
    inv_rd = 1.0 / rd
    t0 = (box_min - ro) * inv_rd
    t1 = (box_max - ro) * inv_rd
    t_near = max(min(t0.x, t1.x), min(t0.y, t1.y), min(t0.z, t1.z))
    t_far = min(max(t0.x, t1.x), max(t0.y, t1.y), max(t0.z, t1.z))
    return t_near, t_far

def compute_normal(hit_pos, node_pos, node_scale, child_idx):
    half_size = node_scale / 2.0
    diff = hit_pos - node_pos
    # Determine which face was hit by comparing normalized distances to half-size
    nx = abs(diff.x / half_size)
    ny = abs(diff.y / half_size)
    nz = abs(diff.z / half_size)
    max_comp = max(nx, max(ny, nz))
    normal = vec3(0.0, 0.0, 0.0)
    if max_comp == nx: normal.x = 1.0 if diff.x > 0.0 else -1.0
    elif max_comp == ny: normal.y = 1.0 if diff.y > 0.0 else -1.0
    else: normal.z = 1.0 if diff.z > 0.0 else -1.0
    return normal

def get_child_node_index(svo_nodes, children_buffer, parent_idx, child_idx):
    # Extract child pointer from parent node descriptor or children buffer
    # Assuming flat array layout where children are stored sequentially
    child_ptr = children_buffer[parent_idx * 8 + child_idx]
    return child_ptr if child_ptr != 0xFFFFFFFF else None
```

**Edge Cases and Pitfalls:**
*   **Precision:** Ray origin must be nudged forward by an epsilon (e.g., 0.0004) to avoid self-intersection artifacts when the ray starts inside a voxel [S6, S5, unofficial].
*   **NaN/Inf:** Handle NaN/Inf in direction calculations by coalescing to positive infinity [S3, unofficial].
*   **Zero Direction:** Clamp ray direction components close to zero (e.g., $< 1e^{-4}$) to avoid division by zero [S5, unofficial].
*   **Stack Overflow:** Use a fixed-size stack with a maximum depth limit (e.g., 16 or 64) to prevent infinite loops or memory errors [S2, unofficial]. If steps exceed a threshold (e.g., 500), terminate [S2, unofficial].
*   **Empty Space:** Efficiently skip empty octants by checking validity masks before descending [S2, unofficial].

## Subtopics

### [SVO Ray Marching with Stack-Based Navigation](2454-svo-ray-marching-with-stack-based-navigation.md)
Uses an explicit stack with fixed-point-like scale representation for robust navigation and early termination based on pixel error.

### [Octree Voxel Raycasting for Hover (Closest Active Voxel)](3219-octree-voxel-raycasting-for-hover-closest-active-voxel.md)
Specialized for interaction, finding the closest *active* leaf node and tracking empty space for fallback.

### [Octree Ray Marching with Brick Interpolation and Early Termination](4147-octree-ray-marching-with-brick-interpolation-and-early-termi.md)
Uses precomputed "bricks" to skip empty/uniform regions, reducing memory accesses compared to standard node-by-node traversal.

## Operating characteristics
*   **Time Complexity:** $O(\log N)$ on average for sparse scenes, where $N$ is the number of voxels, as it skips empty space. Worst case is $O(D)$ where $D$ is the depth of the tree [S1, S3, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the SVO structure, where $N$ is the number of nodes. The stack size is fixed [S2, unofficial].
*   **Performance:** Real-time capable on GPU due to single draw call and no mesh generation [S6, unofficial].
*   **Strengths:** No mesh generation overhead, single draw call, efficient handling of large sparse volumes [S6, unofficial].
*   **Limitations:** High memory usage for dense scenes, potential for precision issues near boundaries [S6, S5, unofficial].

## Background
Originating from research into efficient voxel rendering, SVO ray tracing has been implemented in various real-time engines and demos [S1, S2, S6, unofficial]. It builds on the concept of sparse voxel octrees introduced by Laine and Karras [S3, unofficial]. The method is widely used in voxel-based games and visualization tools where dynamic geometry and large volumes are common [S6, unofficial].

## Sources
[S1] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
[S2] Onlinerocker/SVO, https://github.com/Onlinerocker/SVO, fetched 2026-10-24 (unofficial)
[S3] realazthat/landscapes, https://github.com/realazthat/landscapes, fetched 2026-10-26 (unofficial)
[S4] krre/origin, https://github.com/krre/origin, fetched 2026-10-26 (unofficial)
[S5] cianjinks/Sparse-Voxel-Octree, https://github.com/cianjinks/Sparse-Voxel-Octree, fetched 2026-10-26 (unofficial)
[S6] scarlyle0/voxel-compare, https://github.com/scarlyle0/voxel-compare, fetched 2026-10-27 (unofficial)
[S7] GitHub - saurlax/model_voxelizer: A 3D model voxelization tool that ..., https://github.com/saurlax/model_voxelizer, fetched 2026-10-30 (unofficial)
