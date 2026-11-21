# Vertex Cluster Assignment

Also called: RegularGrid::queryCluster, points variant

## Overview
Vertex Cluster Assignment assigns a cluster ID to each vertex or point based on which voxel it occupies in a regular grid [S1, unofficial]. It serves as the core assignment step within [VoxelGrid::CreateFromPointCloudWithinBounds](8505-voxelgrid-createfrompointcloudwithinbounds.md) [S1, unofficial]. Unlike methods designed for triangle meshes, it operates directly on point cloud coordinates to discretize continuous positions into a sparse voxel structure [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A point cloud (vertices), an axis-aligned bounding box minimum (`aabbMin`), a voxel size (`cellSize`), and grid dimensions (`gridDims`) [S1, unofficial].
- **Output:** A cluster ID for each vertex, stored in a `clusterSSBO` [S1, unofficial].

**Data Structures:**
- `vertexSSBO`: Buffer containing input vertex positions [S1, unofficial].
- `gridSSBO`: Buffer containing grid metadata [S1, unofficial].
- `clusterSSBO`: Write-only buffer for output cluster IDs [S1, unofficial].

**Steps:**
1. Bind `vertexSSBO`, `gridSSBO`, and `clusterSSBO` (write) to the shader [S1, unofficial].
2. Execute `_assignVertexClusterShader` with uniforms: `aabbMin`, `cellSize`, `gridDims`, `numVertices` [S1, unofficial].
3. For each vertex index `i`:
   a. Read vertex position `P` from `vertexSSBO[i]` [S1, unofficial].
   b. Compute relative position: `rel = P - aabbMin` [S1, unofficial].
   c. Compute voxel index: `idx = floor(rel / cellSize)` [S1, unofficial].
   d. Clamp `idx` to `[0, gridDims - 1]` to handle boundary cases [G].
   e. Write `idx` (or a derived cluster ID) to `clusterSSBO[i]` [S1, unofficial].
4. Read cluster IDs from `clusterSSBO` and populate `clusterIdx` vector [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

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
        
        # Compute voxel index using floor for correct truncation
        idx_x = math.floor(rel_x / cell_size)
        idx_y = math.floor(rel_y / cell_size)
        idx_z = math.floor(rel_z / cell_size)
        
        # Clamp to grid bounds [G]
        idx_x = max(0, min(idx_x, grid_dims[0] - 1))
        idx_y = max(0, min(idx_y, grid_dims[1] - 1))
        idx_z = max(0, min(idx_z, grid_dims[2] - 1))
        
        # Linearize 3D index to 1D cluster ID (standard practice, not specified in sources)
        cluster_id = idx_x + idx_y * grid_dims[0] + idx_z * grid_dims[0] * grid_dims[1]
        cluster_ids.append(cluster_id)
        
    return cluster_ids
```

**Parameters:**
- `cell_size`: Voxel edge length [S1, unofficial].
- `aabb_min`: Lower bound of the voxel grid [S1, unofficial].
- `grid_dims`: Number of voxels in x, y, z [S1, unofficial].

**Edge Cases:**
- Points outside the AABB: Clamped to nearest valid voxel [G].
- Precision: Floating-point division and floor operations may cause jitter at boundaries [G].

## Subtopics

### [Voxel Fragmentation via Grid Clustering](10530-voxel-fragmentation-via-grid-clustering.md)
Assigns a specific voxel cluster ID to each triangle face by statistically sampling intersections with a regular voxel grid. It differs from deterministic ray-casting by using Monte Carlo sampling to handle ambiguity at voxel boundaries and shared faces. Choose it when converting triangle meshes to sparse voxel grids where surface fragments cross voxel boundaries.

## Operating characteristics
- Time and memory complexity: Not given in the sources.
- Performance and scaling: Not given in the sources.
- Strengths and limitations: Not given in the sources.

## Background
Origin and development history: Not given in the sources.

## Sources
[S1] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
