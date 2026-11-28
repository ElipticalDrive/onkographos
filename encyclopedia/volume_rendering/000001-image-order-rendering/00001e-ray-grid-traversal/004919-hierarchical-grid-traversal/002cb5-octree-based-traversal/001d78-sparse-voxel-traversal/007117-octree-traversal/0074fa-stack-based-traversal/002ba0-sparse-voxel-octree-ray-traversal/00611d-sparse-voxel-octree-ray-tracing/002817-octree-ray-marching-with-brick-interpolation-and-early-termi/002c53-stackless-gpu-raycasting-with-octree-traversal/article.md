# Stackless GPU Raycasting with Octree Traversal

Also called: Adaptive out-of-core volume rendering, GPU ray-casting with octree, Out-of-Core GPU Volume Ray Casting, Stackless octree traversal on the GPU

## Overview
This method performs scalable, out-of-core volume rendering on the GPU by decomposing volumetric data into small cubical bricks organized in an octree structure [S1, S2]. It differs from its parent, [Octree Ray Marching with Brick Interpolation and Early Termination](4147-octree-ray-marching-with-brick-interpolation-and-early-termi.md), by explicitly managing an out-of-core working set and using a 3D texture spatial index to enable stackless traversal without maintaining a node stack in shader memory [S1, S2]. It differs from its sibling, [Two-staged Space-Skipping with Bricking and Octrees](5210-two-staged-space-skipping-with-bricking-and-octrees.md), by integrating the spatial index construction and neighbor pointer encoding directly into the fragment shader's traversal loop, allowing for on-the-fly bounding box computation [S1, S2]. The method solves the problem of rendering very large datasets that exceed GPU texture memory by asynchronously fetching a view-dependent working set of bricks [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Large volumetric dataset (16-bit scalars, optionally 32-bit gradients), camera parameters, transfer function.
*   **Output:** Rendered image (frame-buffer) and depth information (depth-buffer) [S1].

**Data Structures:**
1.  **Out-of-Core Octree:** Organizes data into small cubical bricks. Leaves store original data; inner nodes store filtered representations of children, value ranges, and optionally precomputed gradients [S1]. Neighboring samples are replicated inside bricks to support interpolation [S1].
2.  **Spatial Index (3D Texture):** A compact indexing structure encoding the current working set's octree hierarchy [S1].
    *   **Inner Nodes:** The RGB components store pointers to children. The W component indicates if the node is inner.
    *   **Leaf Nodes:** The RGB components store a data pointer (if non-empty) and 6 consecutive texels storing pointers to neighbors [S2].
    *   **Neighbor Pointers:** The A component encodes the octree level of the neighbor (same or coarser) [S2].
    *   **Termination:** The last leaf data texel always contains a NULL pointer to simplify handling of rays that do not exit the current box [S2].
3.  **Traversal State (Shader Registers):**
    *   `node_ptr`: `float3` coordinate in the spatial index texture.
    *   `octree_level`: Integer, current depth in the octree.
    *   `box_min`: `float3`, minimum corner of the current voxel brick in texture coordinates.
    *   `box_dim`: `float3`, dimensions of the current voxel brick (uniform).
    *   `color`: `float4`, accumulated color.
    *   `depth`: `float`, accumulated depth.

**Parameters:**
*   **Early Ray Termination Threshold:** 0.99 opacity [S2].
*   **Data Precision:** 16-bit scalars, 32-bit gradients (8 bits per direction, 8 bits norm) [S1].
*   **Gradient Filter:** 5x5x5 Sobel filtering [S1].

**Algorithm Steps:**

1.  **Working Set Generation:** An adaptive loader asynchronously fetches bricks from the out-of-core octree based on the view and transfer function, maintaining a working set on CPU/GPU memory [S1].
2.  **Spatial Index Construction:** At each frame, a compact indexing structure is created from the working set and encoded into a 3D texture acting as the spatial index [S1]. Neighbor pointers link leaf nodes via six faces to adjacent nodes or the smallest enclosing node [S1].
3.  **Initialization:**
    *   Set `fragment.color = float4(0,0,0,0)`.
    *   Set `fragment.depth = FAR`.
    *   Set `node_ptr = float3(0,0,0)` (root).
    *   Set `octree_level = 0`.
    *   Set `box_min = float3(0,0,0)`.
    *   Set `box_dim = float3(1,1,1)` [S2].
4.  **Main Traversal Loop:** While `node_ptr` is not null and `color.a < 1`:
    1.  **Find Leaf:** Determine the leaf node containing the current sampling point $P = \text{ray.start} + \text{ray.dir} \times t_{min}$ [S2].
    2.  **Down Traversal:** While the current node is inner (`is_inner(node.w)`):
        *   Subdivide box: `box_dim /= 2`.
        *   Compute center: `box_mid = box_min + box_dim`.
        *   Compute child offset vector: `s = step(P, box_mid)` (0 if $P < box\_mid$, 1 otherwise) [S2].
        *   Update box origin: `box_min += s * box_dim`.
        *   Compute child pointer offset: `child_offset = dot3(s, float3(1,2,4)) * texel_sz` [S2].
        *   Update `node_ptr`: `node_ptr = node.xyz + float4(child_offset, 0, 0, 0)`.
        *   Fetch node: `node = tex3d(spatial_index, node_ptr)`.
        *   Increment `octree_level` [S2].
    3.  **Clip Ray:** Clip the ray to the current box to find the exit face and $t_{max}$ [S2].
    4.  **Accumulate:** If the node is not empty (`!is_empty(node.w)`):
        *   Access data pointer: `data_ptr = tex3d(spatial_index, node.xyz)`.
        *   Accumulate color and opacity using adaptive sampling density matching brick resolution [S2].
        *   Update `fragment.color` and `fragment.depth` [S2].
    5.  **Neighbor Traversal:** If the ray exits the current block:
        *   Compute neighbor offset based on exit face index [S2].
        *   Fetch neighbor node: `neighbor = tex3d(spatial_index, node.xyz + neighbor_offset)`.
        *   Update `node_ptr = neighbor.xyz`.
        *   Update `octree_level = neighbor.w` [S2].
        *   If neighbor level is coarser, coarsen box:
            *   `box_dim = exp2(-octree_level)`.
            *   `box_min = trunc(box_min / box_dim) * box_dim` [S2].
        *   Update `t_min = box_t_max` [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
fragment_color = float4(0.0, 0.0, 0.0, 0.0)
fragment_depth = FAR
node_ptr = float3(0.0, 0.0, 0.0)
octree_level = 0
box_min = float3(0.0, 0.0, 0.0)
box_dim = float3(1.0, 1.0, 1.0)

# Main Loop
while not is_null(node_ptr) and fragment_color.a < 1.0:
    # Find leaf containing current sampling point
    P = ray.start + ray.dir * t_min
    node = tex3d(spatial_index, node_ptr)
    
    # Down Traversal
    while is_inner(node.w):
        box_dim = box_dim / 2.0
        box_mid = box_min + box_dim
        s = step(P, box_mid) # Component-wise step function
        box_min = box_min + s * box_dim
        child_offset = dot3(s, float3(1.0, 2.0, 4.0)) * texel_sz
        node_ptr = node.xyz + float4(child_offset, 0.0, 0.0, 0.0)
        node = tex3d(spatial_index, node_ptr)
        octree_level = octree_level + 1
        
        # Clip ray to box and find exit face
        box_t_max, exit_face_idx, exit_dir = box_clip(ray, t_min, t_max, box_min, box_dim)
        
        # If non-empty block, access data and accumulate
        if not is_empty(node.w):
            data_ptr = tex3d(spatial_index, node.xyz)
            fragment_color, fragment_depth = accumulate(fragment_color, ray, t_min, box_t_max, data_ptr, box_min, box_max)
            
            # If ray exits from current block, move to neighbor
            neighbor_offset = float3(1 + exit_face_idx, 0.0, 0.0) * texel_sz
            neighbor = tex3d(spatial_index, node.xyz + neighbor_offset)
            node_ptr = neighbor.xyz
            octree_level = neighbor.w
            box_dim = exp2(-octree_level)
            box_min = trunc(box_min / box_dim) * box_dim
            t_min = box_t_max
```

## Subtopics

### [GPU Octree Raycasting with Brick Storage](20179-gpu-octree-raycasting-with-brick-storage.md)
A variant that packs leaf node data into a 3D texture and encodes brick coordinates within the node's color data, allowing for efficient brick-level raymarching. Choose this when memory bandwidth for leaf data access is a bottleneck and brick storage optimization is required.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of non-empty bricks traversed by the ray. The stackless traversal avoids stack push/pop overhead, reducing shader complexity [S2].
*   **Memory Complexity:** Proportional to the size of the working set and the spatial index texture, which is significantly smaller than the full volume [S1].
*   **Performance:** Enables interactive real-time visualization of high-resolution anatomical data [S1]. Performance scales with the number of leaf nodes in the working set (typically a few thousands) [S2].
*   **Strengths:** Out-of-core capability allows rendering datasets larger than GPU memory; stackless traversal reduces shader register pressure; adaptive sampling and early termination optimize computation [S1, S2].
*   **Limitations:** Requires on-the-fly construction of the spatial index each frame, which may add CPU/GPU synchronization overhead; occlusion culling is limited to early ray termination, not data loading [S1].
*   **When to use:** Large volumetric datasets that do not fit in GPU memory; interactive applications requiring view-dependent LOD [S1].
*   **When not to use:** Small datasets that fit entirely in texture memory (simpler methods may be more efficient); scenes requiring complex lighting or global illumination beyond ray casting [S1].

## Background
The method was developed by Agus et al. and presented in 2008 as part of a system for advanced volumetric display of the human musculoskeletal system [S1]. It builds on the stackless ray traversal method for kd-trees by Havran [HBS98] and its GPU extension for surface rendering by Pfaff et al. [PGSS07] [S1]. The approach addresses the limitations of previous GPU ray casting methods that were restricted to moderate size volumes fitting entirely into texture memory [S1].

## Sources
[S1] Towards Advanced Volumetric Display of the Human Musculoskeletal System, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/021-025, fetched 2026-10-26
[S2] PDF A single-pass GPU ray casting framework for interactive out-of-core ..., https://www.crs4.it/vic/data/papers/cgi2008-movr.pdf, fetched 2026-10-29
