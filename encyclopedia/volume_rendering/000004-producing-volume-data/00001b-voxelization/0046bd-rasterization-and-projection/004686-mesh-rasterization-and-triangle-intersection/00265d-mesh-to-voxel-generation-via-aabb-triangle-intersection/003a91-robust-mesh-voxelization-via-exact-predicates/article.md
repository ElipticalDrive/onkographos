# Robust Mesh Voxelization via Exact Predicates

Also called: Mesh-Native Voxelization with Exact Predicates, add-voxelization, cmg::voxelize, cmg::voxelize::voxelize, mesh-native voxelizer, voxelize(PLC, opts)

## Overview
A mesh-native voxelizer that produces an occupancy grid or a signed-distance field (SDF) over a Piecewise Linear Complex (PLC)'s bounding box, using exact-predicate classification to avoid epsilon-ray speckle [S1, unofficial][S2]. Unlike [Level 1 Triangle-Voxel Classification](4872-level-1-triangle-voxel-classification.md) which may rely on approximate ray casting, this method uses Shewchuk's exact predicates for geometric calculations to ensure robust classification of watertight meshes in single precision [S2]. It differs from [Tessellation-based Voxelization](5430-tessellation-based-voxelization.md) by operating directly on the input triangle mesh without prior subdivision, and from [Fake Voxelization via Triangle-Cube Intersection](6951-fake-voxelization-via-triangle-cube-intersection.md) by providing true inside/outside parity checks rather than mere intersection counts [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A Piecewise Linear Complex (PLC), defined as a triangle mesh [S2].
*   **Parameters:**
    *   `resolution`: Grid density (default 64) [S1, unofficial].
    *   `mode`: Output type, either "occupancy" or "sdf" [S1, unofficial].
    *   `pad`: Padding value (default 1) [S1, unofficial].
*   **Output:** A `VoxelGrid` data structure containing `origin`, `spacing`, `dims`, and `data` [S2]. The `data` is exposed as a NumPy array with shape `(nx, ny, nz)` and dtype `uint8` for occupancy or `float32` for SDF [S1, unofficial][S2].

**Data Structures:**
*   **CarveGrid:** A boundary-triangle spatial index used for spatial queries [S2].
*   **VoxelGrid:** Contains `origin` (3D vector), `spacing` (scalar), `dims` (3D integer vector), and `data` (1D array reshaped to 3D) [S2].

**Algorithm Steps:**
1.  **Initialization:** Compute the bounding box of the input PLC. Determine grid dimensions `(nx, ny, nz)` based on `resolution` and bounding box size. Initialize the `VoxelGrid` with `origin`, `spacing`, and `dims` [S1, unofficial][S2].
2.  **Spatial Indexing:** Build the `CarveGrid` spatial index from the input triangles to enable efficient bucketing by xy-projection [S2].
3.  **Column Processing:** Iterate over each xy-column `(x, y)` in the grid. Retrieve the list of triangles intersecting the column's xy-projection from `CarveGrid` [S1, unofficial][S2].
4.  **Exact-Predicate Classification:** For each column, cast a vertical scanline. Count crossings against the fan-triangulated boundary using `cmg::robust::orient3d` [S1, unofficial].
    *   The `orient3d` predicate determines the plane-side sign change and matching edge orientations [S1, unofficial].
    *   **Grazing Case Handling:** If `orient3d` returns exactly 0 (grazing case), apply a deterministic `(x, y)` nudge to the scanline to ensure robustness [S1, unofficial].
5.  **Occupancy Calculation:** A cell is inside if an odd number of crossings lie above its center [S1, unofficial].
6.  **SDF Calculation:** If `mode` is "sdf", compute the exact point-to-triangle distance for each cell center using Ericson's method [S1, unofficial]. Sign the distance by the parity (negative inside, positive outside) [S1, unofficial].
7.  **Output:** Return the populated `VoxelGrid` [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from typing import Tuple, List

# Assume these are available from the library context
# cmg_robust_orient3d(p1, p2, p3, p4) -> int (sign of determinant)
# carve_grid_build(triangles) -> SpatialIndex
# carve_grid_query(index, x, y, z_min, z_max) -> List[Triangle]
# point_triangle_distance(p, t) -> float
# fan_triangulate_boundary(triangles) -> List[Triangle]

def voxelize(plc: PLC, resolution: int = 64, mode: str = "occupancy", pad: int = 1) -> VoxelGrid:
    """
    Voxelize a PLC using exact predicates.
    """
    # 1. Compute bounding box
    bbox_min, bbox_max = plc.compute_bounding_box()
    
    # 2. Determine grid dimensions
    dims = compute_dims(bbox_min, bbox_max, resolution)
    origin = bbox_min
    spacing = (bbox_max - bbox_min) / dims
    
    # 3. Initialize output grid
    if mode == "occupancy":
        data = np.zeros(dims, dtype=np.uint8)
    else:
        data = np.zeros(dims, dtype=np.float32)
        
    # 4. Build spatial index
    triangles = plc.get_triangles()
    carves_index = carve_grid_build(triangles)
    
    # 5. Process each column
    for x in range(dims[0]):
        for y in range(dims[1]):
            # Get triangles in this column's xy-projection
            col_triangles = carve_grid_query(carves_index, x, y, 0, dims[2])
            
            # Create vertical scanline for this column
            # Center of the voxel in world coordinates
            cx = origin[0] + (x + 0.5) * spacing[0]
            cy = origin[1] + (y + 0.5) * spacing[1]
            
            # Determine crossings
            crossings = 0
            for tri in col_triangles:
                # Use exact predicate to determine if the scanline crosses the triangle
                # We check the intersection of the vertical line (cx, cy, z) with the triangle plane
                # and if the intersection point lies within the triangle.
                # cmg::robust::orient3d is used for robustness.
                
                # Simplified logic: orient3d checks the orientation of 4 points.
                # Here, we use it to determine if the ray crosses the triangle edge or plane.
                # The spec mentions "plane-side sign change plus matching edge orientations".
                
                # For implementation, we assume a helper that uses orient3d to count crossings
                # robustly.
                if is_crossing_robust(cx, cy, tri):
                    crossings += 1
            
            # 6. Determine occupancy
            is_inside = (crossings % 2) == 1
            
            # 7. Fill grid
            for z in range(dims[2]):
                if mode == "occupancy":
                    data[x, y, z] = 1 if is_inside else 0
                else:
                    # Compute SDF
                    voxel_center = np.array([
                        origin[0] + (x + 0.5) * spacing[0],
                        origin[1] + (y + 0.5) * spacing[1],
                        origin[2] + (z + 0.5) * spacing[2]
                    ])
                    dist = point_triangle_distance(voxel_center, tri) # Find min dist to any triangle
                    if is_inside:
                        data[x, y, z] = -dist
                    else:
                        data[x, y, z] = dist
                        
    return VoxelGrid(origin=origin, spacing=spacing, dims=dims, data=data)

def is_crossing_robust(cx: float, cy: float, tri: Triangle) -> bool:
    """
    Determines if the vertical ray at (cx, cy) crosses the triangle.
    Uses cmg::robust::orient3d.
    """
    # Extract triangle vertices
    p1, p2, p3 = tri.vertices
    
    # Check if the ray intersects the triangle plane
    # Compute the intersection point z
    # Plane equation: n . (p - p1) = 0
    # Ray: r(t) = (cx, cy, t)
    # n . (r(t) - p1) = 0 => n.z * (t - p1.z) + n.x * (cx - p1.x) + n.y * (cy - p1.y) = 0
    
    # Use orient3d to determine the side of the triangle vertices relative to the ray
    # This is a simplified representation of the exact predicate logic.
    # The actual implementation uses orient3d to check the orientation of the ray segment
    # against the triangle edges.
    
    # If orient3d returns 0, apply a deterministic nudge
    # For this pseudocode, we assume the library handles the nudge internally if needed.
    
    # Count crossings based on sign changes
    # This is a placeholder for the exact orient3d logic described in the spec.
    # The spec states: "plane-side sign change plus matching edge orientations"
    
    # We assume a helper function that performs this exact check
    return check_exact_predicate_crossing(cx, cy, p1, p2, p3)

def check_exact_predicate_crossing(cx: float, cy: float, p1: Vec3, p2: Vec3, p3: Vec3) -> bool:
    # Placeholder for the actual orient3d-based crossing check
    # The spec requires using cmg::robust::orient3d
    # This function would implement the logic to count crossings robustly
    return False # Not given in the sources.
```

**Edge Cases and Pitfalls:**
*   **Grazing Cases:** When `orient3d` returns exactly 0, a deterministic `(x, y)` nudge must be applied to the scanline to avoid ambiguity [S1, unofficial].
*   **Curved Inputs:** The method only sees the triangle mesh, so curved inputs (e.g., from STEP/IGES) are only as accurate as their faceting [S1, unofficial][S2].
*   **Non-Watertight Input:** The method does not handle non-watertight input; it assumes a watertight PLC [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Approximately $O(\text{cells} + \text{triangles})$ due to bucketing triangles by xy-projection [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of voxels in the grid, $O(nx \cdot ny \cdot nz)$ [S1, unofficial][S2].
*   **Performance:** Fast, parallel spatial-grid classifier [S2].
*   **Strengths:** Robust classification in single precision, no epsilon-ray speckle, mesh-native (no CAD dependency) [S1, unofficial][S2].
*   **Limitations:** Fidelity bounded by input tessellation, does not handle B-Rep/exact-surface voxelization, does not heal non-watertight input [S1, unofficial][S2].
*   **When to Use:** When robust voxelization of a triangle mesh is required, especially in single precision, and CAD kernel integration is not desired [S2].
*   **When Not to Use:** When exact curved-surface fidelity is required (use OCCT instead) [S1, unofficial][S2].

## Background
Proposed by leonardoaraujosantos in Pull Request #20 to the CyberdyneCorp/CyberMeshGenerator repository on July 4, 2026 [S2]. The method is part of the CyberMeshGenerator (cmg) library, a modern C++20 library for generating tetrahedral meshes [S2]. It was designed to provide a mesh-native voxelizer that reuses existing point-in-solid classifiers and spatial indices within the cmg ecosystem [S2]. The method explicitly positions itself against OCCT, noting that while OCCT provides exact curved-surface fidelity, cmg offers speed, smaller footprint, and better integration with meshing tools [S1, unofficial][S2].

## Sources
[S1] Implement voxelization (occupancy + signed-distance field), https://github.com/CyberdyneCorp/CyberMeshGenerator/pull/21, fetched 2026-10-27 (unofficial)
[S2] openspec: propose voxelization (occupancy + SDF) under cmg, https://github.com/CyberdyneCorp/CyberMeshGenerator/pull/20, fetched 2026-10-27
