# Voxel Mesh Ray Intersection via BVH

Also called: Voxel Mesh Raycasting (BVH Triangle Intersection), VoxelGeometryIntersect, VoxelMeshIntersect

## Overview
A raycasting method for voxel meshes that traverses a grid of oriented sub-cubes to find intersections with voxel geometry using BVH triangle intersection tests [S1, unofficial]. It differs from its parent [Mesh BVH](1062-mesh-bvh.md) by adding a spatial grid traversal layer that maps ray positions to specific voxel meshes before delegating to the mesh's internal BVH [S1, unofficial]. It differs from siblings like [Position-Normal Tree](4222-position-normal-tree.md) by relying on explicit triangle intersection tests within leaf nodes rather than implicit distance fields [S1, unofficial].

## How it works
**Inputs:** Ray origin ($ro$), ray direction ($rd$), maximum distance ($MAX\_DISTANCE$), voxel mesh data structures (indices, vertices, BVH nodes), and world bounds.
**Outputs:** Intersection result containing hit distance ($t$), hit position, normal, and triangle ID.

**Data Structures:**
*   **VoxelMesh struct** (80 bytes, 4-byte aligned):
    *   `state` (vec4f, offset 0): `x` indicates active status.
    *   `position` (vec3f, offset 4): World position.
    *   `indexOffsets` (vec4f, offset 8): `x` = vertex offset, `y` = index offset, `z` = BVH tree offset, `w` = voxel index offset.
    *   `minBounds` (vec3f, offset 12): Axis-aligned bounding box minimum.
    *   `maxBounds` (vec3f, offset 15): Axis-aligned bounding box maximum.
*   **AABB struct**:
    *   `min` (vec4f): `xyz` = position, `w` = active flag.
    *   `max` (vec4f): `xyz` = position, `w` = node type (1 for inner, 2 for voxel geometry).
*   **StackNode struct**: `{ nodeId: u32, t: f32 }`.
*   **Stack**: Fixed-size array of 13 `StackNode` elements (`STACK_SIZE = 13`) [S1, unofficial].
*   **Constants**: `cubeSize = 16.0`, `subCubeSize = 8.0`, `halfSize = 8.0`, `rayLength = 16.0`, `VOXEL_NODE_INDEX = 4095u` [S1, unofficial].

**Algorithm Steps:**

1.  **Inverse Direction Calculation**: Compute inverse ray direction components to handle near-zero values and avoid division by zero [S1, unofficial]:
    $$ invDir.x = \frac{1.0}{\text{sign}(rd.x) \cdot \max(|rd.x|, 10^{-8})} $$
    Repeat for $y$ and $z$ [S1, unofficial].

2.  **Initialization**: Set $t = \infty$, $totalDistance = 0.0$, $rayPosition = ro$ [S1, unofficial].

3.  **Grid Traversal Loop**:
    *   Define an oriented bounding cube relative to $rayPosition$ using $rd$, a right vector, and an up vector with `halfSize` offset [S1, unofficial].
    *   Iterate through a $3 \times 3 \times 3$ grid of sub-cubes (indices $i, j, k \in [0, 2]$) [S1, unofficial].
    *   Calculate sub-cube center:
        $$ subCubeCenter = cubeMinCorner + rd \cdot (i \cdot subCubeSize) + right \cdot (j \cdot subCubeSize) + up \cdot (k \cdot subCubeSize) $$
    *   Calculate sub-cube min/max corners:
        $$ subCubeMin = subCubeCenter - rd \cdot \frac{subCubeSize}{2} - right \cdot \frac{subCubeSize}{2} - up \cdot \frac{subCubeSize}{2} $$
    *   Lookup world mesh at $subCubeMin$ using `getWorldMesh`. This function checks bounds and returns the mesh index if found [S1, unofficial].
    *   Perform bounding box intersection test between ray and voxel mesh bounds using `BoundingBoxIntersect`, storing result in $chunkT$ [S1, unofficial].
    *   If mesh found, $chunkT < t$, and $chunkT < MAX\_DISTANCE$, call `VoxelMeshIntersect` with $ro$, $rd$, mesh index, and $MAX\_DISTANCE$ [S1, unofficial].
    *   If `VoxelMeshIntersect` returns a hit and $result.t \le t$, update $finalResult$ and $t$ [S1, unofficial].

4.  **BVH Traversal (`VoxelMeshIntersect`)**:
    *   Initialize `stackIndex = 0`, `currentNodeIndex = 0`, `currentNodeT = 0.0` [S1, unofficial].
    *   Calculate `inverseDir = 1.0 / rd` [S1, unofficial].
    *   Get mesh position and offset from `mesh.indexOffsets.z` [S1, unofficial].
    *   **Loop**:
        *   Fetch current node from `voxel_bvh[currentNodeIndex + offset]` [S1, unofficial].
        *   **Leaf Check**: If `currentNode.max.w == 2.0`, it is a leaf [S1, unofficial].
            *   Call `VoxelGeometryIntersect(ro, rd, currentNodeIndex - VOXEL_NODE_INDEX, voxelMeshIndex)` [S1, unofficial].
            *   If hit, set `intersectResult.found = true`, `intersectResult.t = currentNodeT`, store geometry result, and break loop [S1, unofficial].
        *   **Internal Node**:
            *   Calculate child IDs: `leftChildId = 2 * currentNodeIndex + 1`, `rightChildId = 2 * currentNodeIndex + 2` [S1, unofficial].
            *   Fetch children from `voxel_bvh` [S1, unofficial].
            *   Test left child: If `leftChild.min.w >= 0`, compute `leftChildT = BoundingBoxIntersect(...)`. If `leftChildT > MAX_DISTANCE`, set `leftChildT = \infty` [S1, unofficial].
            *   Test right child: If `rightChild.min.w >= 0`, compute `rightChildT = BoundingBoxIntersect(...)`. If `rightChildT > MAX_DISTANCE`, set `rightChildT = \infty` [S1, unofficial].
            *   **Traversal Logic**:
                *   If only left hit (`leftChildT < \infty` and `rightChildT == \infty`): `currentNodeIndex = leftChildId`, `currentNodeT = leftChildT` [S1, unofficial].
                *   If only right hit (`rightChildT < \infty` and `leftChildT == \infty`): `currentNodeIndex = rightChildId`, `currentNodeT = rightChildT` [S1, unofficial].
                *   If both hit: Push farther child onto stack. Check stack overflow (`stackIndex >= STACK_SIZE`); if overflow, set `intersectResult.error = true` and break [S1, unofficial]. Traverse closer child [S1, unofficial].

5.  **Triangle Intersection (`VoxelGeometryIntersect`)**:
    *   If `length <= 0` (calculated as `indiceOffsets.y - indiceOffsets.x`), return immediately [S1, unofficial].
    *   Loop `indiceIndex` from 0 to `length` with step 3 [S1, unofficial].
    *   Get vertex indices `v1, v2, v3` from `indices` array offset by `vertexOffset` [S1, unofficial].
    *   Call `BVH_TriangleIntersect_1` with transformed vertex positions (`vertices[...] + position`) [S1, unofficial].
    *   If `triResult.t < t` and `triResult.t != \infty`:
        *   Update `intersectResult.hit = true`, `t = triResult.t`, `finalTriResult = triResult`, `triId = indiceIndex / 3`, `finalIndex = indiceOffset + indiceIndex` [S1, unofficial].

6.  **Normal Interpolation**:
    *   If hit, retrieve vertex normals `n1, n2, n3` and UVs `uv1, uv2, uv3` [S1, unofficial].
    *   Interpolate normal:
        $$ interpolatedNormal = \text{normalize}(n1 \cdot (1.0 - u - v) + n2 \cdot u + n3 \cdot v) $$
        where $u, v$ are from `finalTriResult` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def VoxelMeshRayIntersection(ro, rd, max_distance, voxel_meshes, world_bounds):
    # 1. Inverse Direction
    inv_rd = vec3(
        1.0 / (sign(rd.x) * max(abs(rd.x), 1e-8)),
        1.0 / (sign(rd.y) * max(abs(rd.y), 1e-8)),
        1.0 / (sign(rd.z) * max(abs(rd.z), 1e-8))
    )
    
    t = float('inf')
    ray_position = ro
    cube_size = 16.0
    sub_cube_size = 8.0
    half_size = cube_size / 2.0
    
    # Define oriented basis vectors (simplified for example, assumes rd is aligned or basis is precomputed)
    # In practice, rayRight and rayUp are orthogonal to rd
    ray_right = vec3(0, 1, 0) # Placeholder for actual orthogonal basis calculation
    ray_up = vec3(0, 0, 1)    # Placeholder for actual orthogonal basis calculation
    
    # 3. Grid Traversal
    # Determine cube min corner based on ray_position and orientation
    cube_min_corner = ray_position - ray_right * half_size - ray_up * half_size # Simplified
    
    for i in range(3):
        for j in range(3):
            for k in range(3):
                # Calculate sub-cube center
                sub_cube_center = (
                    cube_min_corner + 
                    rd * (i * sub_cube_size) + 
                    ray_right * (j * sub_cube_size) + 
                    ray_up * (k * sub_cube_size)
                )
                
                # Calculate sub-cube min corner
                sub_cube_min = (
                    sub_cube_center - 
                    rd * (sub_cube_size / 2.0) - 
                    ray_right * (sub_cube_size / 2.0) - 
                    ray_up * (sub_cube_size / 2.0)
                )
                
                # Lookup mesh
                mesh_index = get_world_mesh(sub_cube_min, voxel_meshes, world_bounds)
                
                if mesh_index != -1:
                    mesh = voxel_meshes[mesh_index]
                    chunk_t = bounding_box_intersect(ro, rd, mesh.min_bounds, mesh.max_bounds)
                    
                    if chunk_t < t and chunk_t < max_distance:
                        result = voxel_mesh_intersect(ro, rd, mesh_index, max_distance)
                        if result.found and result.t <= t:
                            t = result.t
                            final_result = result
                            
    return final_result

def voxel_mesh_intersect(ro, rd, mesh_index, max_distance):
    stack = [StackNode(0, 0.0)] * 13
    stack_index = 0
    current_node_index = 0
    current_node_t = 0.0
    inv_rd = vec3(1.0 / max(abs(rd.x), 1e-8), 1.0 / max(abs(rd.y), 1e-8), 1.0 / max(abs(rd.z), 1e-8))
    
    # Get mesh data
    mesh = voxel_meshes[mesh_index]
    bvh_offset = mesh.index_offsets.z
    voxel_bvh = get_bvh_data(mesh_index)
    
    while True:
        node = voxel_bvh[current_node_index + bvh_offset]
        
        # Leaf check
        if node.max.w == 2.0:
            geo_result = voxel_geometry_intersect(ro, rd, current_node_index - 4095, mesh_index)
            if geo_result.found:
                return geo_result
            break # No more hits in this branch
        
        # Internal node
        left_id = 2 * current_node_index + 1
        right_id = 2 * current_node_index + 2
        
        left_child = voxel_bvh[left_id + bvh_offset]
        right_child = voxel_bvh[right_id + bvh_offset]
        
        left_t = float('inf')
        right_t = float('inf')
        
        if left_child.min.w >= 0:
            left_t = bounding_box_intersect(ro, rd, left_child.min.xyz, left_child.max.xyz)
            if left_t > max_distance: left_t = float('inf')
            
        if right_child.min.w >= 0:
            right_t = bounding_box_intersect(ro, rd, right_child.min.xyz, right_child.max.xyz)
            if right_t > max_distance: right_t = float('inf')
            
        if left_t < float('inf') and right_t == float('inf'):
            current_node_index = left_id
            current_node_t = left_t
        elif right_t < float('inf') and left_t == float('inf'):
            current_node_index = right_id
            current_node_t = right_t
        elif left_t < float('inf') and right_t < float('inf'):
            # Push farther child
            if stack_index >= 13:
                return IntersectionResult(error=True)
            if left_t < right_t:
                stack[stack_index] = StackNode(right_id, right_t)
                current_node_index = left_id
                current_node_t = left_t
            else:
                stack[stack_index] = StackNode(left_id, left_t)
                current_node_index = right_id
                current_node_t = right_t
            stack_index += 1
        else:
            # Both missed, pop from stack if available
            if stack_index > 0:
                stack_index -= 1
                current_node_index = stack[stack_index].node_id
                current_node_t = stack[stack_index].t
            else:
                break
                
    return IntersectionResult(found=False)

def voxel_geometry_intersect(ro, rd, node_idx, mesh_index):
    mesh = voxel_meshes[mesh_index]
    length = mesh.index_offsets.y - mesh.index_offsets.x
    
    if length <= 0:
        return IntersectionResult(found=False)
        
    t = float('inf')
    final_tri_result = None
    tri_id = -1
    
    vertex_offset = mesh.index_offsets.x
    indice_offset = mesh.index_offsets.y
    
    for indice_index in range(0, length, 3):
        v1_idx = mesh.indices[indice_index + vertex_offset]
        v2_idx = mesh.indices[indice_index + 1 + vertex_offset]
        v3_idx = mesh.indices[indice_index + 2 + vertex_offset]
        
        v1 = mesh.vertices[v1_idx] + mesh.position
        v2 = mesh.vertices[v2_idx] + mesh.position
        v3 = mesh.vertices[v3_idx] + mesh.position
        
        tri_result = bvh_triangle_intersect_1(ro, rd, v1, v2, v3)
        
        if tri_result.t < t and tri_result.t != float('inf'):
            t = tri_result.t
            final_tri_result = tri_result
            tri_id = indice_index // 3
            
    if final_tri_result is not None:
        # Interpolate normal
        v1_idx = mesh.indices[tri_id * 3 + vertex_offset]
        v2_idx = mesh.indices[tri_id * 3 + 1 + vertex_offset]
        v3_idx = mesh.indices[tri_id * 3 + 2 + vertex_offset]
        
        n1 = mesh.normals[v1_idx]
        n2 = mesh.normals[v2_idx]
        n3 = mesh.normals[v3_idx]
        
        u = final_tri_result.u
        v = final_tri_result.v
        normal = normalize(n1 * (1.0 - u - v) + n2 * u + n3 * v)
        
        return IntersectionResult(
            found=True, 
            t=t, 
            normal=normal, 
            tri_id=tri_id
        )
    return IntersectionResult(found=False)

def bvh_triangle_intersect_1(ro, rd, v0, v1, v2):
    # Möller-Trumbore algorithm
    edge1 = v1 - v0
    edge2 = v2 - v0
    h = cross(rd, edge2)
    a = dot(edge1, h)
    
    if a < 1e-8:
        return IntersectionResult(t=float('inf'))
        
    f = 1.0 / a
    s = ro - v0
    u = f * dot(s, h)
    
    if u < 0.0 or u > 1.0:
        return IntersectionResult(t=float('inf'))
        
    q = cross(s, edge1)
    v = f * dot(rd, q)
    
    if v < 0.0 or u + v > 1.0:
        return IntersectionResult(t=float('inf'))
        
    t = f * dot(edge2, q)
    
    if t > 1e-8:
        return IntersectionResult(t=t, u=u, v=v)
    return IntersectionResult(t=float('inf'))

def bounding_box_intersect(ro, rd, min_b, max_b):
    t_min = (min_b - ro) * inv_rd
    t_max = (max_b - ro) * inv_rd
    t1 = min(t_min, t_max)
    t2 = max(t_min, t_max)
    t_enter = max(t1.x, t1.y, t1.z)
    t_exit = min(t2.x, t2.y, t2.z)
    
    if t_enter > t_exit:
        return float('inf')
    if t_exit < 0:
        return float('inf')
    return max(t_enter, 0.0)
```

## Subtopics

### [Voxel Ray Marching with BVH Acceleration](6903-voxel-ray-marching-with-bvh-acceleration.md)
A compute shader-based ray tracing technique for voxel data that uses a Bounding Volume Hierarchy (BVH) to accelerate ray-mesh intersection tests for lighting and ambient occlusion calculations. It differs from hardware-accelerated approaches like [Sparse Voxel Set (SVS)](9654-sparse-voxel-set-svs.md) or [GPU-based Ray Shooting for Volumetric Mapping](18280-gpu-based-ray-shooting-for-volumetric-mapping.md) by performing the traversal logic entirely within a custom compute shader rather than relying on dedicated DXR hardware pipelines. This method is specifically designed for deferred shading pipelines where position and normal buffers are available, enabling efficient shadow and occlusion queries against sparse voxel meshes.

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M \cdot \log L)$ where $N$ is the number of sub-cubes traversed (fixed $3 \times 3 \times 3 = 27$ per grid step), $M$ is the number of meshes per sub-cube, and $L$ is the BVH depth. The grid traversal is constant per ray segment, but BVH traversal depends on tree height.
*   **Memory Complexity**: $O(V + B + S)$ where $V$ is vertex/index buffer size, $B$ is BVH node size, and $S$ is stack size (fixed 13 nodes) [S1, unofficial].
*   **Strengths**: Avoids recursion depth limits by using a fixed-size stack (13 nodes) [S1, unofficial]. Efficient for sparse voxel meshes by skipping empty space via grid traversal.
*   **Limitations**: Fixed stack size (13) can lead to overflow errors if the BVH is deeper than supported, potentially missing closer intersections [S1, unofficial]. Requires explicit stack management.
*   **When to use**: Deferred shading pipelines with available position/normal buffers, sparse voxel meshes, GPU compute shaders [S1, unofficial].
*   **When not to use**: Dense voxel grids where simpler grid traversal suffices, or when hardware ray tracing (DXR) is available and preferred.

## Background
Originated in the Divine Voxel Engine (DVE) by Divine Star Software [S1, unofficial]. It is part of the DVE monorepo, which includes packages for voxel rendering and WASM compilation [S1, unofficial]. The method is used in DVE's PBR and Classic rendering modes [S1, unofficial].

## Sources
[S1] Divine-Star-Software/DivineVoxelEngine, https://github.com/Divine-Star-Software/DivineVoxelEngine, fetched 2026-10-26 (unofficial)
