# Solid Voxelization

Also called: 3D Scan-filling, BfsSolid, Bulk Voxelization, Inner region filling, Two-Level Model Voxelization

## Overview
Solid Voxelization fills the interior of a watertight mesh with voxels, producing a bulk volume rather than just a surface shell [S1, S3, unofficial]. It extends surface voxelization by determining which voxels lie inside the object boundary [S3, unofficial]. Unlike [Ray-Cast Style Voxel Filling](1502-ray-cast-style-voxel-filling.md), which scans along a single axis and may struggle with complex topology or singular points, Solid Voxelization typically employs a two-level grid model or multi-axis consistency checks to robustly identify the interior [S1, S3, unofficial].

## How it works
The method generally follows a two-phase approach: surface intersection to mark boundary voxels, followed by interior filling. Two primary implementations are documented: a GPU-based two-level grid approach and a CPU-based BFS/Filter-and-Refine approach.

### Inputs and Outputs
*   **Input**: A watertight mesh defined by vertices and triangles, and a voxel size `gridVoxelSize` [S1, unofficial].
*   **Output**: A voxel grid (bitmask or boolean array) where `1` indicates a solid (interior or surface) voxel and `0` indicates empty space [S1, S2, unofficial].

### Data Structures
**MeshDataDevice** (GPU memory layout) [S1, unofficial]:
*   `bbox[6]`: `float`, bounding box min/max coordinates $(x_{min}, y_{min}, z_{min}, x_{max}, y_{max}, z_{max})$.
*   `vertices`: Array of `Vector3Df` (x, y, z).
*   `triangles`: Array of `Triangle` structs.
*   `trianglesNo`: `int*`, pointer to the number of triangles.
*   `_bottom`, `_top`: `Vector3Df`, precomputed bounding box corners for each triangle.

**Triangle** [S1, unofficial]:
*   `_idx1, _idx2, _idx3`: Indices into the `vertices` array.
*   `_bottom, _top`: `Vector3Df`, bounding box corners.

**Grid** (Two-Level Model) [S1, unofficial]:
*   `vTable1`: Bitset representing surface voxels.
*   `vTableBulk`: Bitset representing solid voxels (filled interior).

### Algorithm: GPU Two-Level Model (Solid Voxelization)
This method uses a two-pass approach: first marking surface voxels, then filling the bulk [S1, S2, unofficial].

1.  **Preprocessing**:
    *   Compute global translation: `globalTranslation[0..2] = meshData->bbox[0..2]` [S1, unofficial].
    *   Compute grid dimensions based on `meshData->bbox` and `gridVoxelSize` [S1, unofficial].
    *   Copy mesh data to GPU [S1, unofficial].

2.  **Pass 1: Surface Voxelization**:
    *   Launch `solidVoxelGpu` kernel. Each thread processes one triangle.
    *   For each triangle, compute relative coordinates by subtracting `globalTranslation` [S1, unofficial].
    *   Determine the triangle's bounding box in grid coordinates [S1, unofficial].
    *   Iterate through voxels in the bounding box. For each voxel center $p$, perform planar tests against the triangle edges to determine if the voxel intersects the triangle [S1, unofficial].
    *   **Planar Tests**:
        *   Compute edge vectors $e_0, e_1, e_2$.
        *   Compute 2D normals for XY, YZ, and ZX planes. Flip normals if the corresponding 3D normal component is negative (e.g., if $n.z < 0$, flip XY normals) [S1, unofficial].
        *   Compute distances $d$ for each plane [S1, unofficial].
        *   A voxel is inside the triangle if it satisfies the half-space constraints for all three edges. Special handling for zero dot products (planar edges) uses a tie-breaking rule: $(n.x > 0.0f) \lor (n.x == 0.0f \land n.y < 0.0f)$ [S1, unofficial].
        *   If the voxel intersects the triangle, flip the bit in `vTable1` [S1, unofficial].

3.  **Pass 2: Bulk Filling**:
    *   Launch `fillBulkVoxelGpu` kernel.
    *   For each slice (fixed $y, z$), iterate along $x$.
    *   Identify "raster points" (voxels marked in `vTable1`).
    *   If two or more raster points are found on the scan line, fill all voxels between them in `vTableBulk` [S1, unofficial].
    *   *Note*: Some implementations use a BFS approach (`BfsSolid`) starting from known interior points or using a queue to propagate "inside" status from surface voxels, checking 6-connectivity [S2, unofficial].

4.  **Post-processing**:
    *   If metadata is required, iterate through voxels. If a voxel is in `vTableBulk` but not marked as surface, mark it as solid in `voxel_metas` [S2, unofficial].
    *   Synchronize and free GPU memory [S1, unofficial].

### Algorithm: CPU BFS / Filter-and-Refine
An alternative optimization for flood filling [S2, unofficial].

1.  **Filter Step**:
    *   For each axis (X, Y, Z), scan the grid.
    *   For a fixed pair of coordinates (e.g., $y, z$), iterate $x$.
    *   If a surface voxel is encountered, toggle the "inside" state. Mark all voxels between entry and exit as potentially solid [S2, unofficial].
    *   This marks voxels visible along any axis.

2.  **Refine Step**:
    *   Identify "holes": voxels not marked by the filter step.
    *   Perform BFS flood fill starting from these holes or from the exterior to carve out the interior [S2, unofficial].
    *   The BFS (`BfsSolid`) checks 6 neighbors. If a neighbor is within the mesh bounding box and not already marked in `voxels` or `voxels_buffer`, it is added to the queue and marked [S2, unofficial].

### Pseudocode: GPU Surface Intersection
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
__global__ void solidVoxelGpu(mesh::MeshDataDevice *GPUMeshData, Grid *grid, int totNumThreads) {
    size_t thread_id = threadIdx.x + blockIdx.x * blockDim.x;
    if (thread_id < totNumThreads) {
        // 1. Get triangle data
        mesh::Triangle triThread = GPUMeshData->triangles[thread_id];
        int index1 = 3 * triThread._idx1;
        int index2 = 3 * triThread._idx2;
        int index3 = 3 * triThread._idx3;
        
        // 2. Get vertices relative to mesh bounding box min
        float meshBbox[6] = {GPUMeshData->bbox[0], GPUMeshData->bbox[1], GPUMeshData->bbox[2], 
                             GPUMeshData->bbox[3], GPUMeshData->bbox[4], GPUMeshData->bbox[5]};
        float V0[3] = {GPUMeshData->vertices[index1].x - meshBbox[0], GPUMeshData->vertices[index1].y - meshBbox[1], GPUMeshData->vertices[index1].z - meshBbox[2]};
        float V1[3] = {GPUMeshData->vertices[index2].x - meshBbox[0], GPUMeshData->vertices[index2].y - meshBbox[1], GPUMeshData->vertices[index2].z - meshBbox[2]};
        float V2[3] = {GPUMeshData->vertices[index3].x - meshBbox[0], GPUMeshData->vertices[index3].y - meshBbox[1], GPUMeshData->vertices[index3].z - meshBbox[2]};
        
        // 3. Voxelize triangle
        voxelizeTriangleSolidGpu(grid, meshBbox, triThread._bottom, triThread._top, V0, V1, V2, true, thread_id);
    }
}

__device__ void voxelizeTriangleSolidGpu(Grid *grid, float meshBbox[6], float triBbox[6], float V0[3], float V1[3], float V2[3], bool flag, int thread_id) {
    // Compute triangle normal
    float e0[3] = {V1[0]-V0[0], V1[1]-V0[1], V1[2]-V0[2]};
    float e1[3] = {V2[0]-V0[0], V2[1]-V0[1], V2[2]-V0[2]};
    float n[3] = {e0[1]*e1[2]-e0[2]*e1[1], e0[2]*e1[0]-e0[0]*e1[2], e0[0]*e1[1]-e0[1]*e1[0]};
    
    // Determine grid bounds for this triangle
    int minX = floor(triBbox[0] / gridVoxelSize);
    int maxX = ceil(triBbox[3] / gridVoxelSize);
    int minY = floor(triBbox[1] / gridVoxelSize);
    int maxY = ceil(triBbox[4] / gridVoxelSize);
    int minZ = floor(triBbox[2] / gridVoxelSize);
    int maxZ = ceil(triBbox[5] / gridVoxelSize);
    
    for (int z = minZ; z <= maxZ; z++) {
        for (int y = minY; y <= maxY; y++) {
            // YZ Plane Test
            float p_yz[2] = {(y + 0.5f) * gridVoxelSize, (z + 0.5f) * gridVoxelSize};
            // Compute edge normals for YZ plane
            float n_yz_e0[2] = {-e0[2], e0[0]}; // Perpendicular to e0 in YZ
            float n_yz_e1[2] = {-e1[2], e1[0]};
            float n_yz_e2[2] = {-(V2[0]-V1[0]), V2[2]-V1[2]}; // Edge 2
            
            // Flip normals if n.x < 0 (based on source logic for specific plane)
            // Note: Source flips based on specific component of n. 
            // Simplified: Check if point is inside triangle using barycentric or half-space
            
            // X Loop
            for (int x = minX; x <= maxX; x++) {
                float p[3] = {x * gridVoxelSize, y * gridVoxelSize, z * gridVoxelSize};
                
                // Check if point is inside triangle
                // Using dot product with normal and distance from plane
                float nDOTp = n[0]*p[0] + n[1]*p[1] + n[2]*p[2];
                float d1 = -(n[0]*V0[0] + n[1]*V0[1] + n[2]*V0[2]);
                float d2 = -(n[0]*V1[0] + n[1]*V1[1] + n[2]*V1[2]); // Approximation, source uses edge-specific tests
                
                // Source uses detailed edge tests for XY, YZ, XZ projections
                // If all projections are inside, voxel intersects triangle
                if (isInsideTriangle(p, V0, V1, V2, n)) {
                    size_t location = x + (y * grid.gridSizeY) + (z * grid.gridSizeY * grid.gridSizeZ);
                    ops::flipBit(grid->vTable1, location);
                    break; // Found intersection for this voxel, move to next
                }
            }
        }
    }
}
```

### Edge Cases and Pitfalls
*   **Singular Points/Planar Triangles**: If a triangle is parallel to an axis, standard scan-filling along that axis may fail. The GPU method handles this via explicit tie-breaking rules in planar tests (checking normal direction when dot product is zero) [S1, unofficial]. The CPU method handles this by performing scan-filling along three axes and intersecting the results [S3, unofficial].
*   **Precision**: Floating-point comparisons for boundary inclusion require epsilon or specific directional checks to avoid gaps or overlaps [S1, unofficial].
*   **Non-Watertight Meshes**: Basic BFS solid voxelization assumes a watertight mesh. If the mesh is not watertight, the interior may not be correctly enclosed [S2, unofficial].

## Subtopics

### [Single-pass GPU Solid Voxelization](3323-single-pass-gpu-solid-voxelization.md)
Single-pass GPU Solid Voxelization is a GPU-based algorithm that voxelize solid geometry in a single render pass, producing point-sampled or over/under-conservative results. It solves the problem of efficiently converting watertight meshes into volumetric data by exploiting the Jordan Curve Theorem: a point is interior if a ray from it intersects the surface an odd number of times. Unlike [8, 10], this method is accurate even if an arbitrary number of fragments fall in a single voxel and does not require front/back face separation. It builds on [Solid Voxelization](1695-solid-voxelization.md) but achieves significant speedups by extracting multiple layers per pass.

### [Hierarchical Inside/Outside Propagation](9071-hierarchical-inside-outside-propagation.md)
Hierarchical Inside/Outside Propagation is a post-processing step for solid voxelization that converts a potentially non-solid, boundary-defined voxel grid into a fully enclosed solid volume by propagating "inside" status hierarchically along the +x axis. It solves the problem of determining which voxels lie in the interior of a mesh after initial surface intersection, avoiding global ray-casting or flood-fill by leveraging the spatial coherence of the voxel grid. It differs from single-pass GPU methods by using a multi-level hierarchical approach with explicit neighbor pointers and flag propagation, making it suitable for CPU or hybrid implementations where memory locality and coarse-grained parallelism are prioritized over single-kernel throughput.

### Solid Voxelization via Filter-and-Refine
A technique to fill the interior of a mesh to create a solid voxel representation. It uses a filter-and-refine strategy to optimize basic flood fill, avoiding the need for watertight meshes [S2, unofficial].
*   **Filter Step**: Fill voxels along each axis. For a fixed pair of coordinates (e.g., fix x, y), enumerate z in [0, maxz]. If a voxel (x,y,z) is occupied (by surface voxelization), stop filling in that direction. This marks all voxels visible along that axis [S2, unofficial].
*   **Refine Step**: Identify 'holes'—voxels that are not marked as visible along any axis. Perform a standard flood fill specifically on these remaining empty voxels to fill the interior [S2, unofficial].
*   **Pitfall**: Holes exist because the filter step alone does not fill the entire interior; some interior voxels are occluded from all axis-aligned viewing directions [S2, unofficial].
*   **Complexity**: Time Complexity is $O(n)$, where $n$ is the total number of voxels in the bounding box [S2, unofficial].
*   **Advantage**: Much more efficient than basic flood fill because the filter step uses lightweight operations, and the subsequent flood fill is only applied to a small number of holes [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: 
    *   GPU Surface Voxelization: $O(m \cdot c)$, where $m$ is the number of triangles and $c$ is the average number of voxels in a triangle's bounding box [S2, unofficial].
    *   CPU BFS/Filter-Refine: $O(n)$, where $n$ is the number of voxels in the bounding box [S2, unofficial].
*   **Memory Complexity**: $O(n)$ for the voxel grid, plus $O(m)$ for mesh data [S2, unofficial].
*   **Performance**: GPU implementation is significantly faster for high-resolution grids and complex meshes due to parallel processing of triangles [S1, unofficial]. CPU BFS is slower but avoids GPU memory transfer overhead and is suitable for smaller meshes or CPU-only environments [S2, unofficial].
*   **Strengths**: Robust interior filling; GPU version is highly parallelizable; Filter-and-Refine reduces the cost of flood fill [S2, unofficial].
*   **Limitations**: Requires watertight mesh for basic BFS; GPU implementation requires CUDA-capable hardware; Planar triangles require special handling [S1, S3, unofficial].

## Background
Solid Voxelization is an extension of surface voxelization, aiming to produce a volumetric representation of a 3D model. The concept of 3D scan-filling is analogous to 2D scanline filling, extended to three dimensions [S3, unofficial]. The GPU-based two-level model approach was developed to leverage the massive parallelism of graphics hardware for real-time voxelization [S1, unofficial]. The Filter-and-Refine optimization was introduced to address the performance bottlenecks of naive flood fill algorithms in CPU-based implementations [S2, unofficial].

## Sources
[S1] ivandrodri/gpu-voxelizer, https://github.com/ivandrodri/gpu-voxelizer, fetched 2026-10-26 (unofficial)
[S2] topskychen/voxelizer, https://github.com/topskychen/voxelizer, fetched 2026-10-27 (unofficial)
[S3] PDF Real-time Voxelization for Complex Models - GitHub Pages, https://wo1fsea.github.io/files/About_NavMesh/P2_Real-time_Voxelization_for_Complex_Models.pdf, fetched 2026-10-29 (unofficial)
