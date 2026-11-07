# Voxel Mesh Ray Intersection via BVH

Also called: Voxel Mesh Raycasting (BVH Triangle Intersection), VoxelGeometryIntersect, VoxelMeshIntersect

## Overview

Voxel Mesh Ray Intersection via BVH is a raycasting method for voxel meshes that traverses a grid of oriented sub-cubes to find intersections with voxel geometry using Binary Volume Hierarchy (BVH) triangle intersection tests [S1, unofficial]. It handles multiple meshes by first checking bounding boxes and then delegating to the specific mesh's BVH [S1, unofficial]. This method builds on [Mesh BVH](1062-mesh-bvh.md) by providing the specific traversal logic and data structures required to query triangle meshes stored within a voxelized world structure.

## How it works

### Inputs and Outputs

**Inputs:**
*   `rayOrigin`: `vec3f` position of the ray start.
*   `rayDirection`: `vec3f` normalized direction vector.
*   `worldBoundsMin`: `vec3f` minimum bounds of the entire voxel world.
*   `meshData`: Buffer containing `VoxelMeshStruct` instances.
*   `voxel_bvh`: Buffer containing BVH nodes (`AABB` structures).
*   `voxel_indice`: Buffer containing triangle indices for all voxel nodes.
*   `indices`: Buffer containing global triangle indices.
*   `vertices`: Buffer containing vertex positions.
*   `normals`: Buffer containing vertex normals.
*   `uvs`: Buffer containing vertex UVs.
*   `MAX_DISTANCE`: `f32` maximum traversal distance.

**Outputs:**
*   `intersectResult`: Struct containing `found` (bool), `t` (f32), `hit` (bool), `error` (bool), and geometry data (normal, UV, triangle ID).

### Data Structures

**VoxelMeshStruct** (80 bytes, 4-byte aligned):
*   Offset 0-3: `state` (`vec4f`, `x` -> active flag).
*   Offset 4-7: `position` (`vec3f`).
*   Offset 8-11: `indexOffsets` (`vec4f`, `x` -> vertex offset, `y` -> indice offset, `z` -> BVH tree offset, `w` -> voxel indice offset).
*   Offset 12-15: `minBounds` (`vec3f`).
*   Offset 16-19: `maxBounds` (`vec3f`).

**AABB** (BVH Node):
*   `min`: `vec4f` (`xyz` -> position, `w` -> active flag).
*   `max`: `vec4f` (`xyz` -> position, `w` -> node type: `1` for inner, `2` for voxel geometry).

**StackNode**:
*   `nodeId`: `u32`.
*   `t`: `f32`.
*   Stack is a fixed-size array of 13 `StackNode` instances.

**Constants:**
*   `cubeSize`: 16.0.
*   `subCubeSize`: 8.0.
*   `halfSize`: `cubeSize / 2.0` (8.0).
*   `rayLength`: 16.0.
*   `STACK_SIZE`: 13.
*   `VOXEL_NODE_INDEX`: 4095u.

### Algorithm Steps

1.  **Inverse Ray Direction:** Calculate `invDir` components to handle near-zero values:
    $$ invDir.x = \frac{1.0}{\text{sign}(rayDirection.x) \cdot \max(|rayDirection.x|, 10^{-8})} $$
    Repeat for $y$ and $z$.

2.  **Initialization:** Set `t = INFINITY`, `totalDistance = 0.0`, `rayPosition = rayOrigin`.

3.  **Grid Traversal Loop:**
    *   Define oriented bounding cube corners relative to `rayPosition` using `rayDirection`, `rayRight`, and `rayUp` vectors with `halfSize` offset.
    *   Iterate through a $3 \times 3 \times 3$ grid of sub-cubes (indices $i, j, k$ from 0 to 2).
    *   Calculate sub-cube center:
        $$ subCubeCenter = cubeMinCorner + rayDirection \cdot (i \cdot subCubeSize) + rayRight \cdot (j \cdot subCubeSize) + rayUp \cdot (k \cdot subCubeSize) $$
    *   Calculate sub-cube min/max corners:
        $$ subCubeMinCorner = subCubeCenter - rayDirection \cdot \frac{subCubeSize}{2.0} - rayRight \cdot \frac{subCubeSize}{2.0} - rayUp \cdot \frac{subCubeSize}{2.0} $$
    *   Lookup world mesh at `subCubeMinCorner` using `getWorldMesh`. This checks bounds and returns an index if a mesh exists.
    *   Perform bounding box intersection test between ray and voxel mesh bounds using `BoundingBoxIntersect`, storing result in `chunkT`.
    *   If mesh found, `chunkT < t`, and `chunkT < MAX_DISTANCE`, call `VoxelMeshIntersect` with `rayOrigin`, `rayDirection`, `meshLookUp.index`, and `MAX_DISTANCE`.
    *   If `VoxelMeshIntersect` returns found and `result.t <= t`, update `finalResult` and `t`.

4.  **BVH Traversal (`VoxelMeshIntersect`):**
    *   Initialize `stackIndex = 0`, `currentNodeIndex = 0`, `currentNodeT = 0.0`.
    *   Calculate `inverseDir = 1.0 / rayDirection`.
    *   Get mesh position and offset from `mesh.indexOffsets.z`.
    *   Loop:
        *   Fetch current node from `voxel_bvh[currentNodeIndex + offset]`.
        *   Check if leaf: if `currentNode.max.w == 2.0`, it is a leaf.
        *   **Leaf Processing:** Call `VoxelGeometryIntersect(rayOrigin, rayDirection, currentNodeIndex - VOXEL_NODE_INDEX, voxelMeshIndex)`. If hit, set `intersectResult.found = true`, `intersectResult.t = currentNodeT`, store geometry result, and break loop.
        *   **Internal Node Processing:**
            *   Calculate child IDs: `leftChildId = 2u * currentNodeIndex + 1u`, `rightChildId = 2u * currentNodeIndex + 2u`.
            *   Fetch children from `voxel_bvh`.
            *   Test left child: If `leftChild.min.w >= 0`, compute `leftChildT = BoundingBoxIntersect(...)`. If `leftChildT > MAX_DISTANCE`, set `leftChildT = INFINITY`.
            *   Test right child: If `rightChild.min.w >= 0`, compute `rightChildT = BoundingBoxIntersect(...)`. If `rightChildT > MAX_DISTANCE`, set `rightChildT = INFINITY`.
            *   Only left hit: If `leftChildT < INFINITY && rightChildT == INFINITY`, set `currentNodeIndex = leftChildId`, `currentNodeT = leftChildT`.
            *   Only right hit: If `rightChildT < INFINITY && leftChildT == INFINITY`, set `currentNodeIndex = rightChildId`, `currentNodeT = rightChildT`.
            *   Both children hit: If `leftChildT < INFINITY && rightChildT < INFINITY`, push the farther child onto the stack and traverse the closer child. Check stack overflow (`stackIndex >= STACK_SIZE`); if overflow, set `intersectResult.error = true` and break.

5.  **Triangle Intersection (`VoxelGeometryIntersect`):**
    *   Loop `indiceIndex` from 0 to `length` with step 3.
    *   Get vertex indices `v1, v2, v3` from `indices` array offset by `vertexOffset`.
    *   Call `BVH_TriangleIntersect_1` with transformed vertex positions (`vertices[...] + position`).
    *   Update closest hit: If `triResult.t < t && triResult.t != INFINITY`, update `intersectResult.hit = true`, `t = triResult.t`, `finalTriResult = triResult`, `triId = indiceIndex/3u`, `finalIndex = indiceOffset + indiceIndex`.

6.  **Normal Interpolation:**
    *   If hit, retrieve vertex normals `n1, n2, n3` and UVs `uv1, uv2, uv3` for the hit triangle.
    *   Interpolate normal:
        $$ interpolatedNormal = \text{normalize}(n1 \cdot (1.0 - u - v) + n2 \cdot u + n3 \cdot v) $$
        where `u, v` are from `finalTriResult`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants
CUBE_SIZE = 16.0
SUB_CUBE_SIZE = 8.0
HALF_SIZE = CUBE_SIZE / 2.0
MAX_DISTANCE = 1000.0 # Example value, typically scene-dependent
STACK_SIZE = 13
VOXEL_NODE_INDEX = 4095
EPSILON = 1e-8
INFINITY = float('inf')

class StackNode:
    def __init__(self, node_id, t):
        self.node_id = node_id
        self.t = t

class IntersectResult:
    def __init__(self):
        self.found = False
        self.t = INFINITY
        self.hit = False
        self.error = False
        self.normal = None
        self.uv = None
        self.tri_id = 0

def sign(x):
    if x > 0: return 1.0
    elif x < 0: return -1.0
    else: return 1.0 # Handle 0

def get_world_mesh_index(position, world_bounds_min):
    # Implementation detail from notes: floor and abs on position relative to worldBoundsMin, scaled by 16
    # Note: Exact formula not fully specified in sources, assuming standard grid lookup
    # Placeholder for specific DVE implementation logic
    return 0 # Placeholder

def bounding_box_intersect(ray_origin, ray_direction, inv_dir, aabb_min, aabb_max):
    # Slab method with early exit
    t1 = (aabb_min.x - ray_origin.x) * inv_dir.x
    t2 = (aabb_max.x - ray_origin.x) * inv_dir.x
    t_min = min(t1, t2)
    t_max = max(t1, t2)
    
    t3 = (aabb_min.y - ray_origin.y) * inv_dir.y
    t4 = (aabb_max.y - ray_origin.y) * inv_dir.y
    t_min = max(t_min, min(t3, t4))
    t_max = min(t_max, max(t3, t4))
    
    if t_min > t_max:
        return INFINITY
        
    t5 = (aabb_min.z - ray_origin.z) * inv_dir.z
    t6 = (aabb_max.z - ray_origin.z) * inv_dir.z
    t_min = max(t_min, min(t5, t6))
    t_max = min(t_max, max(t5, t6))
    
    if t_min > t_max:
        return INFINITY
        
    return t_min

def bvh_triangle_intersect_1(ray_origin, ray_direction, v0, v1, v2):
    # Moller-Trumbore algorithm
    edge1 = v1 - v0
    edge2 = v2 - v0
    h = cross(ray_direction, edge2)
    a = dot(edge1, h)
    
    if a < EPSILON:
        return None # Parallel or pointing away
    
    f = 1.0 / a
    s = ray_origin - v0
    u = f * dot(s, h)
    
    if u < 0.0 or u > 1.0:
        return None
        
    q = cross(s, edge1)
    v = f * dot(ray_direction, q)
    
    if v < 0.0 or u + v > 1.0:
        return None
        
    t = f * dot(edge2, q)
    
    if t > EPSILON:
        return {'t': t, 'u': u, 'v': v}
    return None

def voxel_geometry_intersect(ray_origin, ray_direction, node_index, voxel_mesh_index, mesh_data, voxel_bvh, voxel_indice, indices, vertices, normals, uvs):
    # Get mesh info
    mesh = mesh_data[voxel_mesh_index]
    vertex_offset = mesh.index_offsets.x
    indice_offset = mesh.index_offsets.y
    
    # Calculate length
    length = indice_offset - mesh.index_offsets.y # Note: Source says indiceOffsets.y - indiceOffsets.x, but logic implies range
    # Correction based on source: "length <= 0 (calculated as indiceOffsets.y - indiceOffsets.x)"
    # Assuming indices are stored sequentially per node
    # This part of the source is ambiguous regarding exact buffer layout for length
    # We assume a standard structure where length is known or derived
    # [G] Assuming length is stored or derived from mesh data structure not fully detailed in notes
    # For implementation, this would require access to a length field or end index
    # Placeholder: assume length is provided or calculated from global indices
    length = 10 # Placeholder
    
    if length <= 0:
        return None
        
    tri_result = None
    t = INFINITY
    
    for i in range(0, length, 3):
        v1_idx = indices[indice_offset + i]
        v2_idx = indices[indice_offset + i + 1]
        v3_idx = indices[indice_offset + i + 2]
        
        v1 = vertices[v1_idx] + mesh.position
        v2 = vertices[v2_idx] + mesh.position
        v3 = vertices[v3_idx] + mesh.position
        
        res = bvh_triangle_intersect_1(ray_origin, ray_direction, v1, v2, v3)
        
        if res and res['t'] < t:
            t = res['t']
            tri_result = res
            final_tri_id = i // 3
            final_index = indice_offset + i
            
    if tri_result:
        # Interpolate normal
        v1_idx = indices[indice_offset + final_index]
        v2_idx = indices[indice_offset + final_index + 1]
        v3_idx = indices[indice_offset + final_index + 2]
        
        n1 = normals[v1_idx]
        n2 = normals[v2_idx]
        n3 = normals[v3_idx]
        
        u = tri_result['u']
        v = tri_result['v']
        
        interpolated_normal = normalize(n1 * (1.0 - u - v) + n2 * u + n3 * v)
        
        return {
            'found': True,
            't': t,
            'hit': True,
            'normal': interpolated_normal,
            'uv': None, # UV interpolation not detailed in notes
            'tri_id': final_tri_id
        }
    return None

def voxel_mesh_intersect(ray_origin, ray_direction, mesh_index, max_distance, mesh_data, voxel_bvh, voxel_indice, indices, vertices, normals, uvs):
    stack = [StackNode(0, 0.0) for _ in range(STACK_SIZE)]
    stack_index = 0
    current_node_index = 0
    current_node_t = 0.0
    
    inv_dir = [
        1.0 / (sign(ray_direction.x) * max(abs(ray_direction.x), EPSILON)),
        1.0 / (sign(ray_direction.y) * max(abs(ray_direction.y), EPSILON)),
        1.0 / (sign(ray_direction.z) * max(abs(ray_direction.z), EPSILON))
    ]
    
    mesh = mesh_data[mesh_index]
    offset = mesh.index_offsets.z
    
    while True:
        node = voxel_bvh[current_node_index + offset]
        
        if node.max.w == 2.0: # Leaf
            result = voxel_geometry_intersect(
                ray_origin, ray_direction, 
                current_node_index - VOXEL_NODE_INDEX, 
                mesh_index, mesh_data, voxel_bvh, voxel_indice, 
                indices, vertices, normals, uvs
            )
            if result:
                result['t'] = current_node_t
                result['found'] = True
                return result
            else:
                # No hit in this leaf, backtrack or continue?
                # Source implies break if hit, else continue traversal
                # If no hit, we need to continue to next node
                pass
        else:
            left_child_id = 2 * current_node_index + 1
            right_child_id = 2 * current_node_index + 2
            
            left_child = voxel_bvh[left_child_id + offset]
            right_child = voxel_bvh[right_child_id + offset]
            
            left_child_t = INFINITY
            right_child_t = INFINITY
            
            if left_child.min.w >= 0:
                left_child_t = bounding_box_intersect(
                    ray_origin, ray_direction, inv_dir, 
                    left_child.min.xyz, left_child.max.xyz
                )
                if left_child_t > max_distance:
                    left_child_t = INFINITY
                    
            if right_child.min.w >= 0:
                right_child_t = bounding_box_intersect(
                    ray_origin, ray_direction, inv_dir, 
                    right_child.min.xyz, right_child.max.xyz
                )
                if right_child_t > max_distance:
                    right_child_t = INFINITY
                    
            if left_child_t < INFINITY and right_child_t == INFINITY:
                current_node_index = left_child_id
                current_node_t = left_child_t
            elif right_child_t < INFINITY and left_child_t == INFINITY:
                current_node_index = right_child_id
                current_node_t = right_child_t
            elif left_child_t < INFINITY and right_child_t < INFINITY:
                # Push farther child
                if stack_index >= STACK_SIZE:
                    return {'error': True}
                
                if left_child_t < right_child_t:
                    stack[stack_index] = StackNode(right_child_id, right_child_t)
                    stack_index += 1
                    current_node_index = left_child_id
                    current_node_t = left_child_t
                else:
                    stack[stack_index] = StackNode(left_child_id, left_child_t)
                    stack_index += 1
                    current_node_index = right_child_id
                    current_node_t = right_child_t
            else:
                # No hit in either child, backtrack
                if stack_index > 0:
                    stack_index -= 1
                    current_node_index = stack[stack_index].node_id
                    current_node_t = stack[stack_index].t
                else:
                    break
                    
    return None

def voxel_mesh_ray_intersection(ray_origin, ray_direction, world_bounds_min, mesh_data, voxel_bvh, voxel_indice, indices, vertices, normals, uvs):
    inv_dir = [
        1.0 / (sign(ray_direction.x) * max(abs(ray_direction.x), EPSILON)),
        1.0 / (sign(ray_direction.y) * max(abs(ray_direction.y), EPSILON)),
        1.0 / (sign(ray_direction.z) * max(abs(ray_direction.z), EPSILON))
    ]
    
    t = INFINITY
    ray_position = ray_origin
    
    # Infinite loop for grid traversal
    while True:
        # Define oriented bounding cube corners
        # Note: rayRight and rayUp are not defined in sources, assuming standard basis or orthogonal basis
        # [G] Assuming rayRight and rayUp are orthogonal to rayDirection and each other
        # For simplicity, using standard basis if rayDirection is aligned, otherwise need cross products
        # This is a placeholder for the oriented cube logic
        cube_min_corner = ray_position - HALF_SIZE * ray_direction # Simplified
        
        for i in range(3):
            for j in range(3):
                for k in range(3):
                    sub_cube_center = cube_min_corner + ray_direction * (i * SUB_CUBE_SIZE) + ray_direction * (j * SUB_CUBE_SIZE) + ray_direction * (k * SUB_CUBE_SIZE) # Simplified
                    sub_cube_min_corner = sub_cube_center - ray_direction * (SUB_CUBE_SIZE / 2.0)
                    
                    mesh_index = get_world_mesh_index(sub_cube_min_corner, world_bounds_min)
                    
                    if mesh_index != -1:
                        mesh = mesh_data[mesh_index]
                        chunk_t = bounding_box_intersect(
                            ray_origin, ray_direction, inv_dir,
                            mesh.min_bounds, mesh.max_bounds
                        )
                        
                        if chunk_t < t and chunk_t < MAX_DISTANCE:
                            result = voxel_mesh_intersect(
                                ray_origin, ray_direction, mesh_index, MAX_DISTANCE,
                                mesh_data, voxel_bvh, voxel_indice, indices, vertices, normals, uvs
                            )
                            
                            if result and result.get('found') and result['t'] <= t:
                                t = result['t']
                                final_result = result
                                
    return final_result
```

## Subtopics

## Operating characteristics

*   **Time Complexity:** $O(N)$ for grid traversal where $N$ is the number of sub-cubes visited, plus $O(H)$ for BVH traversal where $H$ is the height of the BVH tree. Triangle intersection is $O(T)$ where $T$ is the number of triangles in the leaf node.
*   **Memory Complexity:** $O(M)$ for storing mesh data, BVH nodes, and indices. Stack size is fixed at 13 nodes.
*   **Performance:** Depends on the density of the voxel mesh and the efficiency of the BVH. Fixed stack size prevents recursion depth issues but may limit traversal depth.
*   **Strengths:** Handles multiple meshes efficiently, avoids recursion overhead, fixed memory usage for stack.
*   **Limitations:** Fixed stack size may lead to missed intersections if the BVH is very deep. Requires careful handling of oriented cubes.

## Background

This method is part of the Divine Voxel Engine (DVE) and Divine Craft project, developed by Divine Star Software [S1, unofficial]. It is designed for real-time voxel rendering and raycasting in large voxel worlds.

## Sources

[S1] Divine-Star-Software/DivineVoxelEngine, https://github.com/Divine-Star-Software/DivineVoxelEngine, fetched 2026-10-26 (unofficial)
