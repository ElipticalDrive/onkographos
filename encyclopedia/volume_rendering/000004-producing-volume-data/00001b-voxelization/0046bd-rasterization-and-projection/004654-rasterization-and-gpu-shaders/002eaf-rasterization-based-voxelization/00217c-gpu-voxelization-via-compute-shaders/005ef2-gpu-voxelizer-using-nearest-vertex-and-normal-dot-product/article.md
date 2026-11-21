# GPU Voxelizer using Nearest Vertex and Normal Dot Product

Also called: Compute Shader Voxelizer, GPUVoxelizer

## Overview
This method is a real-time, brute-force voxelization technique that determines the interiority of a voxel relative to a mesh by identifying the nearest mesh vertex and analyzing the geometric relationship between the vertex normal and the vector to the voxel [S1, unofficial]. It differs from sibling methods like [GPU Mesh Voxelization (Schwarz & Seidel 2010)](1633-gpu-mesh-voxelization-schwarz-seidel-2010.md) which rely on ray-casting or projection, and [Multi-channel Surface Voxelization](1899-multi-channel-surface-voxelization.md) which uses multi-sampling, by avoiding rasterization entirely in favor of direct vertex proximity checks [S1, unofficial]. The technique is primarily distinguished by its reliance on a dot-product threshold for interiority, although the source implementation notes this specific logic is potentially buggy and often superseded by distance-only checks in practice [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A mesh defined by an array of vertices (position and normal) and a voxel grid defined by an array of entities (voxel positions and scales).
*   **Output:** A voxel grid where each voxel's scale (opacity/visibility) is updated based on its proximity to the mesh surface and interiority.

**Data Structures:**
The implementation uses two `std430` buffers in a compute shader:
1.  `entities` (Binding 0): An array of `particle` structs representing voxels.
    *   `position`: `vec4` (xyz: voxel center, w: unused/padding).
    *   `scale`: `vec4` (xyz: unused/padding, w: opacity/visibility factor).
2.  `meshdata` (Binding 1): An array of `meshData` structs representing the mesh.
    *   `position`: `vec4` (xyz: vertex position, w: unused/padding).
    *   `normal`: `vec4` (xyz: vertex normal, w: unused/padding).

**Parameters:**
*   `meshdataCount`: Uniform integer, the number of vertices in the mesh.
*   `voxelSize`: Uniform float, the edge detection threshold.

**Steps:**
1.  Initialize a compute shader with a local group size of `(1024, 1, 1)`. Each work item corresponds to one voxel, identified by the global invocation ID `gid`.
2.  Retrieve the voxel's position `p` and current scale from the `entities` buffer at index `gid`.
3.  Iterate through all `meshdataCount` vertices in the `meshdata` buffer to find the nearest vertex:
    a. Calculate the squared Euclidean distance $d$ between the voxel position $p$ and the current vertex position $v_i$:
       $$d = (p.x - v_i.x)^2 + (p.y - v_i.y)^2 + (p.z - v_i.z)^2$$
       (Implemented as `pow(d1, 2.0) + pow(d2, 2.0) + pow(d3, 2.0)` in the source).
    b. Track the minimum squared distance `nearDist` and the index `nearestIndex` of the closest vertex.
4.  Calculate the vector from the nearest vertex to the voxel:
    $$toCell = \text{normalize}(p - meshdata[nearestIndex].position)$$
5.  Calculate the dot product between `toCell` and the nearest vertex's normal $n_{nearest}$:
    $$dotProduct = \text{dot}(toCell, meshdata[nearestIndex].normal)$$
6.  Determine the "inside" factor `isInside` using a step function with a threshold of -0.9:
    $$isInside = 1 - \text{step}(-0.9, dotProduct)$$
    If $dotProduct \le -0.9$, the ray from the vertex to the voxel is opposed to the normal, indicating the voxel is inside the mesh.
7.  Determine the "edge" factor `fDist` based on the distance to the nearest vertex:
    $$fDist = 1 - \text{step}(voxelSize, \sqrt{nearDist})$$
    If $\sqrt{nearDist} < voxelSize$, the voxel is considered on the edge.
8.  Calculate the final voxel scale:
    $$voxelScale = \max(fDist, isInside)$$
9.  Write `voxelScale` to the `scale` component (w) of the voxel entity in the `entities` buffer. The position remains unchanged.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader Main Function (GLSL-like pseudocode translated to Python-like logic)
# Global ID gid identifies the voxel
# entities: array of {position: vec4, scale: vec4}
# meshdata: array of {position: vec4, normal: vec4}
# meshdataCount: int
# voxelSize: float

def process_voxel(gid):
    # 1. Retrieve voxel data
    voxel = entities[gid]
    p = voxel.position.xyz
    
    # 2. Find nearest vertex
    near_dist_sq = float('inf')
    nearest_index = -1
    
    for i in range(meshdataCount):
        v_pos = meshdata[i].position.xyz
        # Squared Euclidean distance
        dx = p.x - v_pos.x
        dy = p.y - v_pos.y
        dz = p.z - v_pos.z
        dist_sq = dx*dx + dy*dy + dz*dz
        
        if dist_sq < near_dist_sq:
            near_dist_sq = dist_sq
            nearest_index = i
            
    # 3. Calculate vectors
    nearest_v_pos = meshdata[nearest_index].position.xyz
    nearest_v_norm = meshdata[nearest_index].normal.xyz
    
    # Vector from vertex to voxel
    to_cell_x = p.x - nearest_v_pos.x
    to_cell_y = p.y - nearest_v_pos.y
    to_cell_z = p.z - nearest_v_pos.z
    
    # Normalize to_cell
    mag = sqrt(to_cell_x**2 + to_cell_y**2 + to_cell_z**2)
    if mag > 0:
        to_cell_x /= mag
        to_cell_y /= mag
        to_cell_z /= mag
    else:
        # Degenerate case: voxel center coincides with vertex
        to_cell_x, to_cell_y, to_cell_z = 0.0, 0.0, 0.0
        
    # 4. Dot product
    dot_product = to_cell_x * nearest_v_norm.x + \
                  to_cell_y * nearest_v_norm.y + \
                  to_cell_z * nearest_v_norm.z
                  
    # 5. Determine factors
    # isInside: 1 if dot_product <= -0.9, else 0
    if dot_product <= -0.9:
        is_inside = 1.0
    else:
        is_inside = 0.0
        
    # fDist: 1 if sqrt(near_dist_sq) < voxelSize, else 0
    dist = sqrt(near_dist_sq)
    if dist < voxelSize:
        f_dist = 1.0
    else:
        f_dist = 0.0
        
    # 6. Final scale
    voxel_scale = max(f_dist, is_inside)
    
    # 7. Write output
    entities[gid].scale.w = voxel_scale
```

**Edge Cases and Pitfalls:**
*   **Buggy Logic:** The source explicitly notes that the dot-product method for determining interiority is considered "bugged" and that the final effective logic often relies primarily on the distance check (`fDist`) rather than the combination with `isInside` [S1, unofficial]. Implementers should be aware that `isInside` may produce incorrect results for non-convex or complex meshes where the nearest vertex is not a reliable indicator of interiority.
*   **Precision:** The squared distance calculation uses floating-point arithmetic. For very large meshes or high-precision requirements, precision loss in `near_dist_sq` tracking may occur.
*   **Degenerate Input:** If a voxel center coincides exactly with a mesh vertex, the `toCell` vector has zero magnitude. The pseudocode handles this by setting the vector to zero, which results in a dot product of 0. Since $0 > -0.9$, `isInside` becomes 0. The voxel will only be marked if it is within `voxelSize` of the vertex (which it is, distance 0), so `fDist` will be 1.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot V)$, where $N$ is the number of voxels (work items) and $V$ is the number of mesh vertices (`meshdataCount`) [S1, unofficial]. This is a brute-force approach with no spatial acceleration structures (like BVH or grids) for vertex lookup.
*   **Memory Complexity:** $O(N + V)$ for storing the voxel entities and mesh data in GPU buffers [S1, unofficial].
*   **Performance:** Tested on NVIDIA GTX 1070 and 1060 GPUs [S1, unofficial]. Performance degrades linearly with both voxel count and vertex count. It is suitable for real-time applications only if the vertex count is low or the voxel grid is small, as it lacks optimization for sparse or large meshes.
*   **Strengths:** Simple implementation; no complex data structures required; fully GPU-accelerated via compute shaders.
*   **Limitations:** Poor scalability for high-poly meshes; inaccurate interiority detection for complex geometry due to reliance on nearest vertex; no handling of mesh edges/faces directly, only vertices.

## Background
This technique originates from an unofficial implementation by `alexr4` (Bonjour Lab) for the Processing.org environment, aiming to provide a real-time voxelization solution using GPU compute shaders [S1, unofficial]. It was developed as a practical experiment for real-time rendering and voxel display, tested on consumer NVIDIA hardware [S1, unofficial]. The method is part of a broader effort to explore GPU-based voxelization techniques outside of traditional rasterization pipelines.

## Sources
[S1] alexr4/Processing-GPUVoxelizer, https://github.com/alexr4/Processing-GPUVoxelizer, fetched 2026-10-26 (unofficial)
