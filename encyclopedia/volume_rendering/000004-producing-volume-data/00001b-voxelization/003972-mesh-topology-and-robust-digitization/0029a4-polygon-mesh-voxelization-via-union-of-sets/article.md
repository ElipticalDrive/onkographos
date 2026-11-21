# Polygon Mesh Voxelization via Union of Sets

Also called: PyVista Mesh Voxelization, PyVista Mesh Voxelization & Watertight Handling, Union of Sets Voxelization, VTK-based Mesh Voxelization

## Overview
This method converts a continuous polygon mesh into a discrete `UniformGrid` voxel representation by computing the union of bounding volumes around vertices, edges, and faces [S2]. It solves the problem of robust surface digitization for volume rendering by enforcing separability guarantees and providing built-in watertightness validation [S1, unofficial]. It differs from topological digitization methods (like [Mesh Topology and Robust Digitization](8496-mesh-topology-and-robust-digitization.md)) by relying on geometric bounding volumes rather than explicit connectivity graphs, and from primitive-based union methods by treating the mesh as a unified solid object [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A polygon mesh (`pyvista.PolyData`). Point clouds require triangulation first, optionally via `delaunay_3d` for simple surfaces [S1, unofficial].
*   **Output:** A `pyvista.UniformGrid` object containing the voxelized representation [S1, unofficial].

**Data Structures:**
*   **UniformGrid:** Stores voxel bounds, dimensions, and point data [S1, unofficial].
*   **Implicit Distance:** A scalar field that verifies voxel consistency relative to the mesh surface [S1, unofficial].

**Parameters:**
*   `check_surface` (bool, default `True`): Skips watertight validation when `False`, preventing errors on imperfect meshes but risking noise if large holes exist [S1, unofficial].
*   $R_c$ (radius for bounding volumes): Equals $L/2$ for 6-separability and $L/(2\sqrt{3})$ for 26-separability, where $L$ is voxel width [S2].

**Steps:**
1.  Validate mesh watertightness if `check_surface=True`; otherwise, proceed with `pyvista.voxelize(input_mesh)` to generate the `UniformGrid` [S1, unofficial].
2.  Compute the union of bounding volumes: for each vertex, include voxels whose centers fall inside a bounding sphere of radius $R_c$ [S2]. For each edge, include voxels inside a bounding cylinder of radius $R_c$ and length equal to the edge length [S2]. For the mesh body, include voxels between two bounding planes $G$ and $H$, plus planes perpendicular to each edge, forming a closed region [S2].
3.  Optionally compute implicit distances via `compute_implicit_distance` to verify consistency or filter internal voxels for a hollow representation [S1, unofficial].
4.  Optionally replace cube glyphs with other primitives using `voxelized_object.glyph(geom=type_of_primitive())` [S1, unofficial].

**Formulas:**
$$R_c = \frac{L}{2} \quad \text{(6-separability)}$$
$$R_c = \frac{L}{2\sqrt{3}} \quad \text{(26-separability)}$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import pyvista as pv
import numpy as np

def voxelize_mesh_union_of_sets(mesh, check_surface=True, separability=6):
    """
    Voxelize a polygon mesh using the Union of Sets method.
    
    Args:
        mesh: pyvista.PolyData mesh object.
        check_surface: bool, default True. Validates watertightness.
        separability: int, 6 or 26. Determines bounding volume radius.
        
    Returns:
        pyvista.UniformGrid: The voxelized volume.
    """
    # Compute voxel width L from mesh bounds or desired resolution
    # [G] Standard initialization: bounds = mesh.bounds; L = (bounds[1]-bounds[0])/resolution
    
    # Compute bounding volume radius Rc based on separability
    if separability == 6:
        Rc = L / 2.0
    elif separability == 26:
        Rc = L / (2.0 * np.sqrt(3.0))
    else:
        raise ValueError("Separability must be 6 or 26.")
    
    # Step 1: Voxelization
    # pyvista.voxelize generates the UniformGrid and enforces check_surface validation
    voxel_grid = pv.voxelize(mesh, check_surface=check_surface)
    
    # Step 2: Union of Sets inclusion
    # [S2] The algorithm includes voxels whose centers fall inside:
    # 1. Bounding spheres at vertices (radius Rc)
    # 2. Bounding cylinders along edges (radius Rc, length = edge length)
    # 3. Bounding planes forming a closed slab around the mesh body
    # [G] Implementation iterates over candidate voxels and tests geometric inclusion.
    
    # Step 3: Optional verification
    # implicit_dist = voxel_grid.compute_implicit_distance()
    
    return voxel_grid
```

**Edge Cases and Pitfalls:**
*   **Non-Watertight Meshes:** PyVista expects watertight meshes by default; holes, overlapping edges, or incorrect geometry cause errors [S1, unofficial].
*   **Skipping Validation:** Setting `check_surface=False` bypasses errors on imperfect meshes but may yield noise or incorrect voxelization if large holes or geometric imperfections exist [S1, unofficial].
*   **Intersection Holes:** Using a distance parameter $t$ instead of $R_c$ for bounding volumes causes holes at intersections (e.g., 45-degree lines) because the intersection point may fall outside both bounding volumes [S2].
*   **Point Cloud Input:** Point clouds must be triangulated into a mesh before voxelization [S1, unofficial].

## Subtopics

### Relaxed Connectivity Voxelization
Focuses on connectivity constraints rather than strict watertightness. Choose when 26-connectivity is required without surface validation. [Relaxed Connectivity Voxelization](2481-relaxed-connectivity-voxelization.md)

### Labeled Volume Creation via MultiBlock Iteration
Handles multiple mesh blocks with distinct labels. Choose when processing multi-part assemblies. [Labeled Volume Creation via MultiBlock Iteration](7389-labeled-volume-creation-via-multiblock-iteration.md)

### Improved Tetrahedral Mesh Generation for Voxelization
Uses tetrahedralization for complex geometries. Choose when high-fidelity internal structure is needed. [Improved Tetrahedral Mesh Generation for Voxelization](7476-improved-tetrahedral-mesh-generation-for-voxelization.md)

## Operating characteristics
*   **Time Complexity:** Depends on the implementation; the authors use a linked list to store the volume model, with complexity influenced by the number of voxels and mesh complexity [S2].
*   **Memory Complexity:** Not explicitly defined in the sources [S2].
*   **Measured Performance:** On an SGI workstation (195MHz R10000, 640MB memory, 512x512x512 resolution), the pump model (3030 facets) took 67.8 seconds for 6-separability (21.8 voxels/ms) and 95.4 seconds for 26-separability (21.2 voxels/ms) [S2].
*   **Strengths:** Simple interface, built-in watertightness checks, Python ecosystem integration [S1, unofficial].
*   **Limitations:** Requires watertight meshes by default, may produce noise if validation is skipped, limited to mesh inputs [S1, unofficial].
*   **When to Use:** Quick, robust voxelization of watertight meshes in Python [S1, unofficial].
*   **When Not to Use:** Non-watertight meshes requiring high-fidelity internal structure, or performance-critical large meshes needing C++ optimization [S1, unofficial].

## Background
This method is based on the PyVista library, which wraps VTK's geometric processing capabilities for mesh-to-`UniformGrid` transformation [S1, unofficial]. The theoretical foundation uses bounding spheres, cylinders, and plane slabs to guarantee 6- or 26-separability for planar objects [S2].

## Sources
[S1] How to Voxelize Meshes and Point Clouds in Python - Medium, https://medium.com/data-science/how-to-voxelize-meshes-and-point-clouds-in-python-ca94d403f81d, fetched 2026-10-28 (unofficial)
[S2] An Accurate Method for Voxelizing Polygon Meshes, https://web.eecs.utk.edu/~huangj/papers/polygon.pdf, fetched 2026-10-29
