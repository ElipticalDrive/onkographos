# SDF to Mesh Extraction

Also called: known SDF extraction

## Overview
Extracts a triangle mesh from a signed distance field (SDF) grid at a specified isosurface value [S1, unofficial]. It builds on [SDF to PLY Conversion](2343-sdf-to-ply-conversion.md) by converting the volumetric representation into a polygonal surface. Unlike optimized extraction methods that may simplify topology, this approach performs a direct extraction, often serving as a baseline or intermediate step for further processing like mesh relaxation [S2, unofficial].

## How it works
**Inputs:**
- `grid`: A signed distance field grid (e.g., OpenVDB `FloatGrid`).
- `iso`: The isosurface value (default: $1/128$) [S1, unofficial].
- `adapt`: Adaptive mesh refinement flag (default: 0) [S1, unofficial].
- `flag_relax`: Mesh relaxation flag (default: true) [S1, unofficial].

**Outputs:**
- `vertices`: Vector of 3D points.
- `faces`: Vector of triangle indices.
- `quads`: Vector of quadrilateral indices (optional, often converted to triangles later) [S1, unofficial].

**Data Structures:**
- `std::vector<openvdb::Vec3s> v`: Stores vertex coordinates.
- `std::vector<openvdb::Vec3I> f`: Stores triangle face indices.
- `std::vector<openvdb::Vec4I> q`: Stores quadrilateral face indices [S1, unofficial].

**Steps:**
1. Initialize output vectors `v`, `f`, and `q`.
2. Call the volume-to-mesh conversion function with the grid, output vectors, isovalue, adapt flag, and relaxation flag.
3. Convert the resulting vectors to the desired matrix format (e.g., Eigen matrices) for further use [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_mesh(grid, iso=1/128, adapt=0, flag_relax=True):
    """
    Extracts a mesh from an OpenVDB SDF grid.
    
    Args:
        grid: OpenVDB FloatGrid containing the SDF.
        iso: Isosurface value.
        adapt: Adaptive refinement flag.
        flag_relax: Mesh relaxation flag.
        
    Returns:
        vertices: List of [x, y, z] coordinates.
        faces: List of [i, j, k] triangle indices.
        quads: List of [i, j, k, l] quadrilateral indices.
    """
    # Initialize output containers
    v = []  # List of Vec3s
    f = []  # List of Vec3I
    q = []  # List of Vec4I
    
    # Call the core extraction function
    # Note: This assumes access to openvdb::tools::volumeToMesh
    # Implementation detail: The function populates v, f, q in-place
    openvdb_tools_volumeToMesh(grid, v, f, q, iso, adapt, flag_relax)
    
    # Convert to standard formats if necessary
    vertices = [list(point) for point in v]
    faces = [list(face) for face in f]
    quads = [list(quad) for quad in q]
    
    return vertices, faces, quads
```

**Edge Cases and Pitfalls:**
- **Precision:** The isovalue `iso` is typically small ($1/128$). Ensure the SDF grid values are normalized or scaled appropriately relative to this threshold [S1, unofficial].
- **Adaptive Refinement:** If `adapt` is enabled, the mesh density varies based on curvature or gradient, which may require post-processing for uniformity [S1, unofficial].
- **Relaxation:** If `flag_relax` is true, the mesh vertices are smoothed. This may alter the exact geometric fidelity to the original SDF isosurface [S1, unofficial].

## Subtopics

### FlexiCubes Extraction
A differentiable approach that extracts meshes from SDFs using a cube-based decomposition, allowing for gradient-based optimization. It handles quadrilaterals and splits them based on gradient consistency [S2, unofficial]. Choose this when differentiability or high-quality quad-based meshes are required. Link: [FlexiCubes](https://github.com/nv-tlabs/FlexiCubes).

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of active voxels in the grid. The exact constant depends on the grid resolution and the complexity of the isosurface [S1, unofficial].
- **Memory Complexity:** Proportional to the number of vertices and faces generated. The output vectors `v`, `f`, and `q` store all geometry explicitly [S1, unofficial].
- **Strengths:** Simple, direct conversion from volume to mesh. Supports adaptive refinement and mesh relaxation.
- **Limitations:** Does not include advanced optimization or simplification. May produce non-manifold geometry if not handled carefully. The default isovalue ($1/128$) may not be suitable for all SDF scales [S1, unofficial].
- **When to use:** When a quick, unoptimized mesh extraction from an SDF is needed, or as a preprocessing step for further mesh processing.

## Background
The method is implemented in the `Mesh2Volume` repository, which provides tools for converting between meshes and volumes using OpenVDB [S1, unofficial]. The `volumeToMesh` function is part of the OpenVDB library's toolset. The FlexiCubes method represents a more recent development in differentiable surface extraction [S2, unofficial].

## Sources
[S1] zzilch/Mesh2Volume, https://github.com/zzilch/Mesh2Volume, fetched 2026-10-26 (unofficial)
[S2] GitHub - nv-tlabs/FlexiCubes, https://github.com/nv-tlabs/FlexiCubes, fetched 2026-10-27 (unofficial)
