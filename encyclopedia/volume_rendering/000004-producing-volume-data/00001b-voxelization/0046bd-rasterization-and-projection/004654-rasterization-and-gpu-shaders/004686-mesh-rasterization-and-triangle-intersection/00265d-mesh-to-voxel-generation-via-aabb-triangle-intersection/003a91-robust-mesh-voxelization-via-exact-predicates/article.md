# Robust Mesh Voxelization via Exact Predicates

Also called: Mesh-Native Voxelization with Exact Predicates, add-voxelization, cmg::voxelize, cmg::voxelize::voxelize, mesh-native voxelizer, voxelize(PLC, opts)

## Overview
A mesh-native voxelizer that produces an occupancy grid or a signed-distance field (SDF) over a Piecewise Linear Complex (PLC)'s bounding box, using exact-predicate classification to avoid epsilon-ray speckle [S1, unofficial]. It sets itself apart from its siblings by relying on Shewchuk's exact geometric predicates for robust inside/outside determination, ensuring watertight meshes classify correctly in single precision without the artifacts common in epsilon-based ray casting [S2]. Unlike B-Rep CAD kernels such as OCCT which classify against exact analytic surfaces, this method classifies against the input tessellation, meaning its fidelity is bounded by the input faceting accuracy [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A Piecewise Linear Complex (PLC), defined as a triangle mesh [S2].
*   **Output:** A `VoxelGrid` data structure containing origin, spacing, dimensions, and data fields [S2]. The data is exposed as a NumPy array of shape $(n_x, n_y, n_z)$ with `uint8` for occupancy or `float32` for SDF, along with origin and spacing metadata [S1, unofficial].

**Data Structures:**
*   **CarveGrid:** A boundary-triangle spatial index used for spatial queries [S2].
*   **VoxelGrid:** Contains `origin` (3D vector), `spacing` (scalar), `dims` (3D integer vector), and `data` (array) [S2].

**Parameters:**
*   `resolution`: Controls grid density. Default value is 64 [S1, unofficial].
*   `mode`: Selects output type. Values are `"occupancy"` or `"sdf"` [S1, unofficial].
*   `pad`: Padding for the grid (Python API) [S1, unofficial].

**Algorithm Steps:**
1.  **Spatial Indexing:** Bucket triangles by their xy-projection into the `CarveGrid` to optimize queries, achieving approximately $O(\text{cells} + \text{triangles})$ complexity [S1, unofficial].
2.  **Grid Iteration:** Iterate over each xy-column $(i, j)$ within the bounding box.
3.  **Classification:** Cast a vertical scanline per xy-column through the grid [S1, unofficial]. Count the number of crossings with the fan-triangulated boundary triangles that lie above the voxel center [S1, unofficial]. Use `cmg::robust::orient3d` (Shewchuk's exact predicate) to determine plane-side sign changes and matching edge orientations for each intersection [S1, unofficial]. If `orient3d` returns exactly 0 (grazing), apply a deterministic $(x, y)$ nudge to the scanline to ensure robustness [S1, unofficial]. The cell is **inside** if the number of crossings is odd [S1, unofficial].
4.  **SDF Calculation (if mode is "sdf"):** Compute the exact point-to-triangle distance using Ericson's algorithm [S1, unofficial]. Sign the distance: negative if the voxel center is inside (odd parity), positive if outside (even parity) [S1, unofficial].
5.  **Output Generation:** Populate the `VoxelGrid` data array with the computed values [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize(plc, resolution=64, mode="occupancy", pad=1):
    # 1. Setup grid parameters
    bbox_min = plc.bounding_box_min()
    bbox_max = plc.bounding_box_max()
    spacing = (bbox_max - bbox_min) / resolution
    dims = [int(resolution + 2 * pad) for _ in range(3)]
    origin = bbox_min - spacing * pad
    
    # 2. Build spatial index (CarveGrid)
    carve_grid = CarveGrid()
    for tri in plc.triangles:
        carve_grid.insert(tri)
        
    # 3. Initialize output grid
    grid = [[[[0.0 for _ in range(dims[2])] for _ in range(dims[1])] for _ in range(dims[0])]
    
    # 4. Iterate over xy-columns
    for j in range(dims[1]):
        for i in range(dims[0]):
            # Voxel center in world space (using first k for ray origin)
            center = [origin[k] + (k + pad) * spacing for k in range(3)]
            
            # Get candidate triangles for this xy-column
            candidates = carve_grid.query_xy(center[0], center[1])
            
            # Count crossings using exact predicates
            crossings = 0
            for tri in candidates:
                # Determine if vertical ray intersects triangle above center
                # Use orient3d for robustness [G]
                is_crossing, z_int = classify_vertical_ray(center, tri)
                if is_crossing and z_int > center[2]:
                    crossings += 1
                    
            # Determine occupancy
            is_inside = (crossings % 2 == 1)
            
            # Fill k-dimension
            for k in range(dims[2]):
                if mode == "occupancy":
                    grid[i][j][k] = 1 if is_inside else 0
                else:
                    # Compute SDF
                    dist = point_triangle_distance(center, plc.triangles)
                    grid[i][j][k] = -dist if is_inside else dist
                    
    return VoxelGrid(origin=origin, spacing=spacing, dims=dims, data=grid)

def classify_vertical_ray(center, tri):
    # Cast vertical ray from center upwards
    # Use orient3d to check intersection robustly [G]
    # Apply deterministic (x,y) nudge if orient3d is 0
    # Return (True/False, intersection_z)
    # Standard exact ray-triangle intersection logic [G]
    pass

def point_triangle_distance(point, triangles):
    # Ericson's exact point-to-triangle distance algorithm [G]
    min_dist = float('inf')
    for tri in triangles:
        d = distance_point_triangle(point, tri)
        if d < min_dist:
            min_dist = d
    return min_dist

def distance_point_triangle(point, tri):
    # Standard point-to-triangle distance computation [G]
    pass

class CarveGrid:
    def __init__(self):
        self.buckets = {}
    def insert(self, tri):
        # Bucket triangles by xy-projection bounding box
        pass
    def query_xy(self, x, y):
        # Return triangles in overlapping xy-buckets
        pass

class VoxelGrid:
    def __init__(self, origin, spacing, dims, data):
        self.origin = origin
        self.spacing = spacing
        self.dims = dims
        self.data = data
```

**Edge Cases and Pitfalls:**
*   **Grazing Intersections:** `orient3d` returning exactly 0 must be handled by a deterministic $(x, y)$ nudge to the scanline [S1, unofficial].
*   **Curved Inputs:** The method only sees the triangle mesh; curved CAD inputs (STEP/IGES) are not handled with exact surface precision, so fidelity is bounded by the input faceting [S1, unofficial][S2].
*   **Non-Watertight Input:** Healing non-watertight input is a non-goal; the method assumes a watertight PLC for robust parity classification [S1, unofficial].

## Subtopics

### [Bolt](15747-bolt.md)
An optimization for solid mesh voxelization that skips expensive BVH distance queries for exterior voxels by leveraging parity check results. It sets itself apart from standard solid voxelization by avoiding the computation of the exact Signed Distance Field (SDF) for points that are guaranteed to be discarded by the interiority filter, provided no narrow band is required. Choose this when performance is critical for solid meshes and you can accept placeholder infinite distances for exterior points.

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
