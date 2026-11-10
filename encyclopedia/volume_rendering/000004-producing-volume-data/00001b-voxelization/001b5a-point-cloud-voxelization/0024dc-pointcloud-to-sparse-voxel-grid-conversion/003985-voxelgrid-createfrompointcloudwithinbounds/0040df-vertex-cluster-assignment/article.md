# Vertex Cluster Assignment

Also called: RegularGrid::queryCluster, points variant

## Overview
Vertex Cluster Assignment maps each input point to a specific voxel cluster ID within a regular sparse voxel grid. It solves the problem of associating unstructured point cloud data with a structured grid topology for downstream processing. It differs from its parent [VoxelGrid::CreateFromPointCloudWithinBounds](8505-voxelgrid-createfrompointcloudwithinbounds.md) by focusing specifically on the GPU-accelerated assignment of cluster identifiers rather than the full grid construction and bounds management.

## How it works
**Inputs:**
- `vertexSSBO`: Buffer containing point positions (float3).
- `gridSSBO`: Buffer containing grid metadata (AABB min, cell size, dimensions).
- `clusterSSBO`: Write-only buffer for output cluster IDs (int32).
- Uniforms: `aabbMin` (float3), `cellSize` (float), `gridDims` (int3), `numVertices` (int32).

**Outputs:**
- `clusterIdx`: Vector of cluster IDs corresponding to each input vertex.

**Data Structures:**
- `clusterSSBO`: Array of `int32` values, size equal to `numVertices`. Each element stores the cluster ID assigned to the corresponding vertex.

**Steps:**
1. Bind `vertexSSBO`, `gridSSBO`, and `clusterSSBO` to the shader program.
2. Set uniforms: `aabbMin`, `cellSize`, `gridDims`, and `numVertices`.
3. Execute the `_assignVertexClusterShader` with `numVertices` invocations.
4. Read the `clusterSSBO` contents into the `clusterIdx` vector.

**Formula:**
For a vertex at position $P = (x, y, z)$, the cluster index $C$ is calculated as:
$$
C = \lfloor \frac{P - \text{aabbMin}}{\text{cellSize}} \rfloor
$$
Where $\lfloor \cdot \rfloor$ denotes the floor function applied component-wise. The resulting vector is converted to an integer index if not already in that format.

**Parameters:**
- `cellSize`: The side length of each voxel. Typical values depend on scene scale.
- `aabbMin`: The minimum corner of the voxel grid.
- `gridDims`: The number of voxels in x, y, and z directions.

**Edge Cases:**
- Points outside the grid bounds: The sources do not specify behavior for points outside the AABB. [G] Implementation should clamp or discard such points to avoid out-of-bounds memory access.
- Precision: Floating-point division and floor operations may introduce small errors near voxel boundaries.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def assign_vertex_cluster(vertices, aabb_min, cell_size, grid_dims):
    """
    Assigns a cluster ID to each vertex based on its position.
    
    Args:
        vertices: List of (x, y, z) tuples.
        aabb_min: (x, y, z) tuple representing the minimum bound.
        cell_size: Float representing the voxel size.
        grid_dims: (x, y, z) tuple representing the grid dimensions.
        
    Returns:
        List of cluster IDs (integers).
    """
    cluster_ids = []
    for v in vertices:
        # Calculate relative position
        rel_x = v[0] - aabb_min[0]
        rel_y = v[1] - aabb_min[1]
        rel_z = v[2] - aabb_min[2]
        
        # Calculate voxel indices
        idx_x = int(rel_x // cell_size)
        idx_y = int(rel_y // cell_size)
        idx_z = int(rel_z // cell_size)
        
        # Check bounds (Guard)
        if 0 <= idx_x < grid_dims[0] and 0 <= idx_y < grid_dims[1] and 0 <= idx_z < grid_dims[2]:
            # Calculate linear cluster ID (assuming row-major order)
            cluster_id = idx_x + idx_y * grid_dims[0] + idx_z * grid_dims[0] * grid_dims[1]
            cluster_ids.append(cluster_id)
        else:
            # Handle out-of-bounds points (Guard)
            cluster_ids.append(-1)
            
    return cluster_ids
```

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of vertices. Each vertex is processed independently.
- **Memory Complexity:** $O(N)$ for storing the cluster IDs.
- **Performance:** Highly parallelizable on GPU due to independent vertex processing.
- **Strengths:** Simple, fast, and efficient for large point clouds.
- **Limitations:** Does not handle points outside the grid bounds explicitly in the source description.

## Background
This method is part of the VoxelFragmentML project, which focuses on voxel-based machine learning. It is used to prepare point cloud data for neural network processing by converting it into a structured grid format.

## Sources
[S1] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
