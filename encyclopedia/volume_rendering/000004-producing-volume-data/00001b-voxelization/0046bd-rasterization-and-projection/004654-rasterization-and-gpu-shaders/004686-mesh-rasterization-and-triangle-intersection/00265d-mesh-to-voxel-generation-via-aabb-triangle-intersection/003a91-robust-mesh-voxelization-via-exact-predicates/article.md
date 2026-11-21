# Robust Mesh Voxelization via Exact Predicates

Also called: Mesh-Native Voxelization with Exact Predicates, add-voxelization, cmg::voxelize, cmg::voxelize::voxelize, mesh-native voxelizer, voxelize(PLC, opts)

## Overview
A mesh-native voxelizer that produces an occupancy grid or a signed-distance field (SDF) over a Piecewise Linear Complex (PLC)'s bounding box, using exact-predicate classification to avoid epsilon-ray speckle [S1, unofficial]. It sets itself apart from its siblings by relying on Shewchuk's exact geometric predicates for robust inside/outside determination, ensuring watertight meshes classify correctly in single precision without the artifacts common in epsilon-based ray casting [S2]. Unlike [Fake Voxelization via Triangle-Cube Intersection](6951-fake-voxelization-via-triangle-cube-intersection.md) which approximates intersections, this method provides exact topological classification for the input tessellation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A Piecewise Linear Complex (PLC), defined as a triangle mesh [S2].
*   **Output:** A `VoxelGrid` data structure containing origin, spacing, dimensions, and data fields. The data is exposed as a NumPy array of shape $(n_x, n_y, n_z)$ with `uint8` for occupancy or `float32` for SDF, along with origin and spacing metadata [S1, unofficial].

**Data Structures:**
*   **CarveGrid:** A boundary-triangle spatial index used for spatial queries [S2].
*   **VoxelGrid:** Contains `origin` (3D vector), `spacing` (scalar), `dims` (3D integer vector), and `data` (array) [S2].

**Parameters:**
*   `resolution`: Controls grid density. Default value is 64 [S1, unofficial].
*   `mode`: Selects output type. Values are `"occupancy"` or `"sdf"` [S1, unofficial].
*   `pad`: Padding for the grid (Python API) [S1, unofficial].

**Algorithm Steps:**
1.  **Spatial Indexing:** Bucket triangles by their xy-projection into the `CarveGrid` to optimize queries, achieving approximately $O(\text{cells} + \text{triangles})$ complexity [S1, unofficial].
2.  **Grid Iteration:** Iterate over each voxel cell $(i, j, k)$ within the PLC's bounding box.
3.  **Classification (Occupancy):**
    *   Cast a vertical scanline (along the z-axis) through the center of the voxel cell's xy-projection [S1, unofficial].
    *   Count the number of crossings with the fan-triangulated boundary triangles that lie above the voxel center [S1, unofficial].
    *   Use `cmg::robust::orient3d` (Shewchuk's exact predicate) to determine plane-side sign changes and matching edge orientations for each intersection [S1, unofficial].
    *   **Grazing Case:** If `orient3d` returns exactly 0 (grazing), apply a deterministic $(x, y)$ nudge to the scanline to ensure robustness [S1, unofficial].
    *   The cell is **inside** if the number of crossings is odd [S1, unofficial].
4.  **SDF Calculation (if mode is "sdf"):**
    *   Compute the exact point-to-triangle distance using Ericson's algorithm [S1, unofficial].
    *   Sign the distance: negative if the voxel center is inside (odd parity), positive if outside (even parity) [S1, unofficial].
5.  **Output Generation:** Populate the `VoxelGrid` data array with the computed values [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize(plc, resolution=64, mode="occupancy", pad=1):
    # 1. Setup grid parameters
    bbox = plc.bounding_box()
    dims = compute_dims(bbox, resolution, pad)
    spacing = compute_spacing(bbox, dims)
    origin = bbox.min - spacing * pad
    
    # 2. Build spatial index (CarveGrid)
    carve_grid = CarveGrid()
    for tri in plc.triangles:
        carve_grid.insert(tri)
        
    # 3. Initialize output grid
    if mode == "occupancy":
        grid = numpy.zeros(dims, dtype=numpy.uint8)
    else:
        grid = numpy.zeros(dims, dtype=numpy.float32)
        
    # 4. Iterate over voxels
    for k in range(dims.z):
        for j in range(dims.y):
            for i in range(dims.x):
                # Voxel center in world space
                center = origin + numpy.array([i, j, k]) * spacing
                
                # Get candidate triangles for this xy-column
                candidates = carve_grid.query_xy(center.x, center.y)
                
                # Count crossings using exact predicates
                crossings = 0
                for tri in candidates:
                    # Determine if the vertical ray from center intersects the triangle
                    # and if the intersection point is above the center
                    # Use orient3d for robustness
                    is_crossing = classify_vertical_ray(center, tri)
                    if is_crossing:
                        crossings += 1
                        
                # Determine occupancy
                is_inside = (crossings % 2 == 1)
                
                if mode == "occupancy":
                    grid[i, j, k] = 1 if is_inside else 0
                else:
                    # Compute SDF
                    dist = compute_point_triangle_distance(center, plc.triangles)
                    if is_inside:
                        grid[i, j, k] = -dist
                    else:
                        grid[i, j, k] = dist
                        
    return VoxelGrid(origin=origin, spacing=spacing, dims=dims, data=grid)

def classify_vertical_ray(center, tri):
    # Cast vertical ray from center upwards
    # Use orient3d to check intersection robustly
    # Apply deterministic nudge if orient3d is 0
    # Return True if intersection exists and z_intersect > center.z
    pass

def compute_point_triangle_distance(point, triangles):
    # Ericson's exact point-to-triangle distance algorithm
    min_dist = infinity
    for tri in triangles:
        d = distance_point_triangle(point, tri)
        if d < min_dist:
            min_dist = d
    return min_dist
```

**Edge Cases and Pitfalls:**
*   **Grazing Intersections:** `orient3d` returning exactly 0 must be handled by a deterministic $(x, y)$ nudge to the scanline [S1, unofficial].
*   **Curved Inputs:** The method only sees the triangle mesh; curved CAD inputs (STEP/IGES) are not handled with exact surface precision, so fidelity is bounded by the input faceting [S1, unofficial][S2].
*   **Non-Watertight Input:** Healing non-watertight input is a non-goal; the method assumes a watertight PLC for robust parity classification [S1, unofficial].

## Subtopics

### [Bolt](15747-bolt.md)
An optimization for solid mesh voxelization that skips expensive BVH distance queries for exterior voxels by leveraging parity check results. Choose this when performance is critical for solid meshes and you can accept placeholder infinite distances for exterior points.

## Operating characteristics
*   **Time Complexity:** Approximately $O(\text{cells} + \text{triangles})$ due to triangle bucketing by xy-projection [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of voxels in the grid plus the spatial index size [S2].
*   **Performance:** Validated with 138 tests including cube and sphere volume convergence, determinism, and watertight no-speckle checks in single precision [S1, unofficial].
*   **Strengths:** Robust exact-predicate classification avoids epsilon-ray speckle; mesh-native (no CAD kernel dependency); fast parallel spatial-grid classifier [S2].
*   **Limitations:** Fidelity is bounded by input tessellation; does not support B-Rep/exact-surface voxelization; SDF calculation is brute-force point-triangle distance in v1 [S1, unofficial].
*   **When to Use:** When robust voxelization of a triangle mesh is required without external CAD dependencies, and exact topological classification is critical.
*   **When Not to Use:** When exact curved-surface fidelity (STEP/IGES) is required (use OCCT instead) [S1, unofficial][S2].

## Background
Proposed by leonardoaraujosantos in Pull Request #20 to the CyberdyneCorp/CyberMeshGenerator repository on July 4, 2026 [S2]. It is part of the CyberMeshGenerator (cmg) library, a modern C++20 library for generating tetrahedral meshes of 3D polyhedral geometry [S2]. The method was designed to provide a mesh-native voxelizer that reuses existing point-in-solid classifiers and Shewchuk's exact predicates, distinguishing it from B-Rep based tools like OCCT which offer exact analytic surface classification but are heavier and CAD-focused [S2].

## Sources
[S1] Implement voxelization (occupancy + signed-distance field), https://github.com/CyberdyneCorp/CyberMeshGenerator/pull/21, fetched 2026-10-27 (unofficial)
[S2] openspec: propose voxelization (occupancy + SDF) under cmg, https://github.com/CyberdyneCorp/CyberMeshGenerator/pull/20, fetched 2026-10-27
