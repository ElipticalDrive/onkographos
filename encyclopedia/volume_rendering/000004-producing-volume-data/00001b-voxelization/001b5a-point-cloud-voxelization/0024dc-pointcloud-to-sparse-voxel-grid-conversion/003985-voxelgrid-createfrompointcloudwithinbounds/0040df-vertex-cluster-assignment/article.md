# Vertex Cluster Assignment

Also called: RegularGrid::queryCluster, points variant

## Overview
Vertex Cluster Assignment maps each input point to a discrete voxel cluster ID based on its spatial position within a regular grid. It solves the problem of discretizing continuous point cloud coordinates into a sparse voxel structure, serving as the core assignment step in `VoxelGrid::CreateFromPointCloudWithinBounds`. It differs from deterministic ray-casting methods by directly computing grid indices from vertex positions, making it suitable for point clouds rather than triangle meshes.

## How it works
**Inputs and Outputs:**
- **Input:** A point cloud (vertices), an axis-aligned bounding box (AABB) minimum (`aabbMin`), a voxel size (`cellSize`), and grid dimensions (`gridDims`).
- **Output:** A cluster ID for each vertex, stored in a `clusterSSBO`.

**Data Structures:**
- `vertexSSBO`: Buffer containing input vertex positions (float3).
- `gridSSBO`: Buffer containing grid metadata (AABB min, cell size, dimensions).
- `clusterSSBO`: Buffer for output cluster IDs (int32).

**Steps:**
1. Bind `vertexSSBO`, `gridSSBO`, and `clusterSSBO` (write-only) to the shader.
2. Execute the `_assignVertexClusterShader` with uniforms: `aabbMin`, `cellSize`, `gridDims`, `numVertices`.
3. For each vertex index `i`:
   a. Read vertex position `P` from `vertexSSBO[i]`.
   b. Compute relative position: `rel = P - aabbMin`.
   c. Compute voxel index: `idx = floor(rel / cellSize)`.
   d. Clamp `idx` to `[0, gridDims - 1]` to handle boundary cases [G].
   e. Write `idx` (or a derived cluster ID) to `clusterSSBO[i]`.
4. Read `clusterSSBO` to populate the `clusterIdx` vector.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def assign_vertex_cluster(vertices, aabb_min, cell_size, grid_dims):
    """
    Assigns a cluster ID to each vertex based on its voxel occupancy.
    
    Args:
        vertices: List of (x, y, z) coordinates.
        aabb_min: (x, y, z) minimum bound of the voxel grid.
        cell_size: Float size of each voxel.
        grid_dims: (nx, ny, nz) dimensions of the grid.
        
    Returns:
        List of cluster IDs (voxel indices) for each vertex.
    """
    cluster_ids = []
    for p in vertices:
        # Compute relative position from AABB min
        rel_x = p[0] - aabb_min[0]
        rel_y = p[1] - aabb_min[1]
        rel_z = p[2] - aabb_min[2]
        
        # Compute voxel index
        idx_x = int(rel_x / cell_size)
        idx_y = int(rel_y / cell_size)
        idx_z = int(rel_z / cell_size)
        
        # Clamp to grid bounds [G]
        idx_x = max(0, min(idx_x, grid_dims[0] - 1))
        idx_y = max(0, min(idx_y, grid_dims[1] - 1))
        idx_z = max(0, min(idx_z, grid_dims[2] - 1))
        
        # Store as a single cluster ID (e.g., linearized index or tuple)
        # Note: The source implies a single ID per vertex, likely the linearized index
        cluster_id = idx_x + idx_y * grid_dims[0] + idx_z * grid_dims[0] * grid_dims[1]
        cluster_ids.append(cluster_id)
        
    return cluster_ids
```

**Parameters:**
- `cell_size`: Voxel edge length. Typical values depend on scene scale.
- `aabb_min`: Lower bound of the voxel grid.
- `grid_dims`: Number of voxels in x, y, z.

**Edge Cases:**
- Points outside the AABB: Clamped to nearest valid voxel [G].
- Precision: Floating-point division and floor operations may cause jitter at boundaries.

## Subtopics

## Operating characteristics
- **Time Complexity:** O(N) where N is the number of vertices. Each vertex requires constant-time arithmetic.
- **Memory Complexity:** O(N) for storing cluster IDs.
- **Performance:** Highly parallelizable on GPU via SSBOs.
- **Strengths:** Simple, fast, deterministic for points.
- **Limitations:** Does not handle triangle fragments; only suitable for point clouds.

## Background
Originated in the `VoxelFragmentML` project by AlfonsoLRz [S1, unofficial]. It is part of a pipeline for converting point clouds to sparse voxel grids for volume rendering.

## Sources
[S1] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
