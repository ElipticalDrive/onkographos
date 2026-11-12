# Interior Voxelize

Also called: Naive Model Filling, fillNaive

## Overview
Interior Voxelize determines which voxels in a discrete 3D grid lie inside a target mesh by sampling points within the grid volume and testing for geometric containment [S1, unofficial][S2, unofficial][S3, unofficial]. It solves the problem of converting a continuous surface representation into a solid volumetric occupancy field. It differs from its siblings [Shell-based Voxelization](1748-shell-based-voxelization.md) and [Ray-Cast Style Voxel Filling](1502-ray-cast-style-voxel-filling.md) by relying on direct point-in-mesh queries (often via spatial acceleration structures) rather than ray intersection counts or boundary shell extraction [S1, unofficial][S2, unofficial][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A closed 3D mesh (triangle soup or BRep) and a target grid resolution $N$ (typically along the maximum bounding box dimension) [S3, unofficial].
*   **Output:** A 3D boolean grid (Bitmap) where `true` indicates the voxel center is inside the mesh, and `false` indicates it is outside [S3, unofficial].

**Data Structures:**
1.  **Spatial Index:** A bounding volume hierarchy (e.g., AABB tree) built over the input mesh triangles to accelerate point-in-mesh queries [S3, unofficial].
2.  **Grid Indexer:** A mapping structure (e.g., `ShiftGridIndexer3`) that converts integer grid coordinates $(i, j, k)$ to world-space coordinates $(x, y, z)$ based on the mesh's bounding box and cell size [S3, unofficial].
3.  **Bitmap:** A 3D array of booleans with dimensions $(N, N, N)$ [S3, unofficial].

**Parameters:**
*   `numcells` ($N$): The resolution of the voxel grid along the longest axis of the bounding box. Typical value: 32 [S3, unofficial].
*   `numVoxelizationSamples`: In some implementations, the number of random samples used instead of a regular grid. Calculated as $Volume(AABB) \times 10000$ [S1, unofficial].

**Steps:**
1.  **Build Spatial Index:** Construct a spatial acceleration structure (e.g., `DMeshAABBTree3`) from the input mesh with auto-build enabled [S3, unofficial].
2.  **Compute Bounds:** Determine the axis-aligned bounding box (AABB) of the mesh, denoted as $B_{min}$ and $B_{max}$ [S3, unofficial].
3.  **Calculate Cell Size:** Compute the voxel edge length $s$ using the maximum dimension of the bounding box:
    $$ s = \frac{\max(B_{max} - B_{min})}{N} $$
    [S3, unofficial]
4.  **Initialize Indexer:** Create a grid indexer mapping grid indices to world space using $B_{min}$ and $s$ [S3, unofficial].
5.  **Initialize Bitmap:** Create a 3D boolean array `bmp` of size $(N, N, N)$ [S3, unofficial].
6.  **Iterate and Test:** For each grid index $(i, j, k)$ in the bitmap:
    a. Map the index to a world-space point $P$ using the indexer: $P = \text{Indexer.FromGrid}(i, j, k)$ [S3, unofficial].
    b. Query the spatial index to determine if $P$ is inside the mesh: $inside = \text{Spatial.IsInside}(P)$ [S3, unofficial].
    c. Set `bmp[i, j, k] = inside` [S3, unofficial].

**Alternative Implementation (Random Sampling):**
If using random sampling instead of a regular grid [S1, unofficial]:
1.  Calculate total samples $S = Volume(AABB) \times 10000$ [S1, unofficial].
2.  Sample $S$ points uniformly from the mesh volume or bounding box [S1, unofficial].
3.  Map each point to a grid index.
4.  Mark the corresponding grid cell as occupied if the point is inside [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Watertight Meshes:** Point-in-mesh tests assume a closed surface. Open meshes may yield undefined interior/exterior classifications depending on the winding number convention [S3, unofficial].
*   **Precision:** Floating-point errors in `IsInside` queries near the surface may cause "leaks" or isolated false positives/negatives [S3, unofficial].
*   **Resolution:** Low resolution (e.g., $N=32$) results in blocky approximations of curved surfaces [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def interior_voxelize(mesh, numcells=32):
    # Step 1: Build Spatial Index
    # Assumes a library function that builds an AABB tree from mesh triangles
    spatial_index = build_aabb_tree(mesh)

    # Step 2: Compute Bounds
    bounds_min = mesh.get_min_bounds()
    bounds_max = mesh.get_max_bounds()

    # Step 3: Calculate Cell Size
    # MaxDim is the maximum of (max_x - min_x), (max_y - min_y), (max_z - min_z)
    max_dim = max(bounds_max.x - bounds_min.x,
                  bounds_max.y - bounds_min.y,
                  bounds_max.z - bounds_min.z)
    cell_size = max_dim / numcells

    # Step 4: Initialize Bitmap
    # Bitmap is a 3D array of booleans, initialized to False
    bitmap = create_3d_bitmap(numcells, numcells, numcells)

    # Step 5: Iterate and Test
    for i in range(numcells):
        for j in range(numcells):
            for k in range(numcells):
                # Map grid index to world coordinate
                # ShiftGridIndexer3 logic:
                # x = bounds_min.x + (i + 0.5) * cell_size
                # y = bounds_min.y + (j + 0.5) * cell_size
                # z = bounds_min.z + (k + 0.5) * cell_size
                world_x = bounds_min.x + (i + 0.5) * cell_size
                world_y = bounds_min.y + (j + 0.5) * cell_size
                world_z = bounds_min.z + (k + 0.5) * cell_size

                # Step 6: Query Containment
                is_inside = spatial_index.is_inside(world_x, world_y, world_z)
                bitmap[i, j, k] = is_inside

    return bitmap
```

## Subtopics

### [Stochastic Ray-Casting Voxelization](1067-stochastic-ray-casting-voxelization.md)
Uses random ray casting for interior determination; robust to non-manifold geometry.

### [Ray-Cast Style Voxel Filling](1502-ray-cast-style-voxel-filling.md)
Uses deterministic Z-axis rays; sensitive to mesh orientation.

### [Front/Back Face Flagging](1506-front-back-face-flagging.md)
Uses face orientation flags; tracks surface traversal.

### [Tight Voxelization (Point-in-Mesh Test)](1709-tight-voxelization-point-in-mesh-test.md)
Post-processing step verifying voxel centers; deterministic geometric check.

### [Mesh Winding Number Voxelization](2328-mesh-winding-number-voxelization.md)
Uses solid angle integration; handles non-manifold geometry and holes.

### [Polyhedron Moment Integration and Voxelization](3722-polyhedron-moment-integration-and-voxelization.md)
Analytic integration of density moments; supports polynomial densities.

### [Point-Containment Voxelization](3882-point-containment-voxelization.md)
Samples grid vertices for BRep containment; exact geometric queries.

### [CPU Solid Voxelization](6756-cpu-solid-voxelization.md)
Rasterization-based projection; efficient CPU implementation.

### [Sub-Voxel Volume Clipping](6989-sub-voxel-volume-clipping.md)
Smooths clipped voxel boundaries; linear falloff.

## Operating characteristics
*   **Time Complexity:** $O(N^3 \cdot \log T)$, where $N$ is the grid resolution and $T$ is the number of triangles, due to the spatial index query cost [S3, unofficial].
*   **Memory Complexity:** $O(N^3)$ for the bitmap storage [S3, unofficial].
*   **Performance:** Fast for moderate resolutions ($N \le 128$) on CPU with efficient spatial indexing [S3, unofficial]. Slower than rasterization-based methods for very high resolutions due to per-voxel queries [S3, unofficial].
*   **Strengths:** Simple to implement; robust to complex mesh topologies if the spatial index is accurate [S3, unofficial].
*   **Limitations:** Computationally expensive for high-resolution grids; sensitive to mesh watertightness [S3, unofficial].
*   **When to Use:** When a simple, deterministic interior volume is needed for moderate-resolution rendering or simulation [S3, unofficial].
*   **When Not to Use:** For real-time applications requiring high-resolution volumes; for non-watertight meshes without winding number support [S3, unofficial].

## Background
Originating from tutorials on 3D bitmaps and mesh winding numbers, this method is presented as a straightforward approach to voxelizing meshes using spatial acceleration structures like AABB trees [S3, unofficial]. It is commonly used in procedural generation and basic volumetric rendering pipelines where simplicity and correctness for watertight meshes are prioritized over performance [S3, unofficial].

## Sources
[S1] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
[S2] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S3] 3D Bitmaps, Minecraft Cubes, and Mesh Winding Numbers — gradientspace, https://www.gradientspace.com/tutorials/2017/12/14/3d-bitmaps-and-minecraft-meshes, fetched 2026-10-29 (unofficial)
