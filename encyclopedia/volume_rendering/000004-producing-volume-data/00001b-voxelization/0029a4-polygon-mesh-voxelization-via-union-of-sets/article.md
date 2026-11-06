# Polygon Mesh Voxelization via Union of Sets

Also called: PyVista Mesh Voxelization, PyVista Mesh Voxelization & Watertight Handling, Union of Sets Voxelization, VTK-based Mesh Voxelization

## Overview
This method transforms a polygon mesh into a UniformGrid voxel representation using the VTK/PyVista interface, relying on built-in checks for mesh watertightness to ensure topological correctness [S1, unofficial]. It differs from rasterization-based methods by operating directly on the mesh geometry in world space rather than projecting primitives to a screen or grid, and it differs from point-cloud voxelization by requiring a closed surface definition [S1, unofficial]. The primary trade-off is strict dependency on mesh validity; non-watertight meshes cause errors unless validation is explicitly bypassed, which risks introducing noise [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A polygon mesh (vertices and faces). If the input is a point cloud, it must first be triangulated into a mesh (e.g., via Delaunay 3D triangulation) [S1, unofficial].
*   **Output:** A `UniformGrid` object containing the voxelized representation, including voxel centers and bounds [S1, unofficial].

**Data Structures:**
*   **UniformGrid:** A regular grid structure where each cell corresponds to a voxel. The grid is defined by an origin, spacing (voxel size), and dimensions (number of cells per axis) [S1, unofficial].
*   **Mesh:** A collection of vertices and polygons (typically triangles) defining the surface [S1, unofficial].

**Parameters:**
*   `check_surface`: Boolean flag. Default is `True`. If `True`, the method validates that the input mesh is watertight. If `False`, validation is skipped [S1, unofficial].

**Steps:**
1.  **Validation (Optional):** If `check_surface` is `True`, verify that the mesh is watertight (no holes, no overlapping edges, correct normals). If the mesh is not watertight, the method raises an error [S1, unofficial].
2.  **Voxelization:** Call the voxelization function (e.g., `pyvista.voxelize(input_mesh)`) to generate the `UniformGrid`. This function maps the continuous mesh geometry to the discrete grid indices based on the grid resolution and bounding box [S1, unofficial].
3.  **Post-processing (Optional):**
    *   Extract voxel centers or bounds from the `UniformGrid` [S1, unofficial].
    *   Compute implicit distances using `compute_implicit_distance` to verify consistency or filter internal voxels for a hollow representation [S1, unofficial].
    *   Replace voxel cubes with other geometric primitives using `glyph` if needed [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import pyvista as pv

def voxelize_mesh(input_mesh, check_surface=True):
    """
    Voxelize a polygon mesh into a UniformGrid.
    
    Args:
        input_mesh: A PyVista PolyData mesh object.
        check_surface: Boolean flag to skip watertight validation. Default is True.
        
    Returns:
        A PyVista UniformGrid object representing the voxelized mesh.
    """
    # Step 1: Validation
    if check_surface:
        # PyVista/VTK checks for watertightness internally.
        # If the mesh has holes, overlapping edges, or incorrect geometry,
        # this call will raise an error.
        pass # The error is raised by the next step if validation fails.

    # Step 2: Voxelization
    # The voxelize function transforms the mesh into a UniformGrid.
    # It uses the mesh's bounds and a default or specified resolution.
    voxel_grid = pv.voxelize(input_mesh, check_surface=check_surface)
    
    return voxel_grid

def process_voxel_grid(voxel_grid, compute_distances=False):
    """
    Post-process the voxel grid.
    
    Args:
        voxel_grid: A PyVista UniformGrid object.
        compute_distances: Boolean flag to compute implicit distances.
        
    Returns:
        Processed voxel grid or additional data.
    """
    # Step 3: Post-processing
    if compute_distances:
        # Compute implicit distances to verify consistency or filter internal voxels.
        # This helps in creating a hollow representation or verifying surface proximity.
        distances = voxel_grid.compute_implicit_distance()
        return distances
    else:
        # Extract voxel centers or bounds if needed.
        centers = voxel_grid.cell_centers()
        return centers
```

**Edge Cases and Pitfalls:**
*   **Non-Watertight Meshes:** If the mesh has holes, overlapping edges, or incorrect geometry, the voxelization will fail if `check_surface=True` [S1, unofficial].
*   **Bypassing Validation:** Setting `check_surface=False` allows voxelization of imperfect meshes but may result in noise or incorrect voxelization if there are large holes or geometric imperfections [S1, unofficial].
*   **Point Cloud Input:** PyVista's `voxelize` function expects a mesh. Point clouds must be triangulated first (e.g., using `delaunay_3d`) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The complexity depends on the underlying VTK implementation and the mesh complexity.
*   **Memory Complexity:** Not given in the sources. The memory usage is proportional to the number of voxels in the `UniformGrid`.
*   **Performance:** Specific performance metrics are not provided in the sources for the PyVista/VTK implementation. However, general voxelization performance can vary based on resolution and mesh complexity [S1, unofficial].
*   **Strengths:** Straightforward interface, built-in watertightness checks, integration with PyVista's visualization tools [S1, unofficial].
*   **Limitations:** Strict dependency on mesh validity, potential noise if validation is bypassed, requires mesh input (not direct point cloud) [S1, unofficial].
*   **When to Use:** When working with valid, watertight polygon meshes and needing a quick voxelization within the PyVista/VTK ecosystem [S1, unofficial].
*   **When Not to Use:** When dealing with non-watertight meshes that cannot be easily fixed, or when high-performance voxelization of large point clouds is required without prior triangulation [S1, unofficial].

## Background
This method is based on the VTK (Visualization Toolkit) interface, which is wrapped by PyVista. VTK is a widely used library for 3D computer graphics, image processing, and visualization [S1, unofficial]. The specific implementation in PyVista provides a high-level Python interface for mesh voxelization, leveraging VTK's robust geometric processing capabilities [S1, unofficial].

## Sources
[S1] How to Voxelize Meshes and Point Clouds in Python - Medium, https://medium.com/data-science/how-to-voxelize-meshes-and-point-clouds-in-python-ca94d403f81d, fetched 2026-10-28 (unofficial)
