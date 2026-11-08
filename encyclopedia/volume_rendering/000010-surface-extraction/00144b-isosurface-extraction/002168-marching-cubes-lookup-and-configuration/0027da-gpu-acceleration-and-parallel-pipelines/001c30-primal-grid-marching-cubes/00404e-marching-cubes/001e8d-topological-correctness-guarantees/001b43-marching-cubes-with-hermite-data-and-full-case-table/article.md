# Marching Cubes with Hermite Data and Full Case Table

Also called: Accurate Isosurface Interpolation with Hermite Data

## Overview
This method extends the standard Marching Cubes algorithm by incorporating Hermite interpolation for higher-order surface positioning and a full case table to resolve topological ambiguities inherent in the original Lorensen and Cline lookup table [S1, unofficial]. It solves the problem of geometric inaccuracy in linear interpolation and topological errors in ambiguous cell configurations [S1, unofficial]. Unlike the [Asymptotic Decider](790-asymptotic-decider.md) which analyzes the trilinear field interior, or [Topologically Correct Marching Cubes](1368-topologically-correct-marching-cubes.md) which inserts cell-center vertices, this method relies on a pre-defined full case table and second-order Hermite interpolants derived from local gradients [S1, unofficial].

## How it works
**Inputs:**
*   Regularly sampled voxel grid values $V$ of resolution $R_x \times R_y \times R_z$.
*   Voxel scale (dimensions) $D_x, D_y, D_z$.
*   Isovalue $I$.
*   Flags: `--full` (enable full case table), `--quadratic` (enable Hermite interpolation), `--polygon` (output polygons instead of triangles), `--nonManifold` (disable barycenter insertion for non-manifold safety).

**Outputs:**
*   A triangle mesh (PLY format) or raw iso-polygons representing the isosurface $V(x,y,z) = I$.

**Data Structures:**
*   **Voxel Grid:** Linearized array of size $R_x R_y R_z$. Values are stored as unsigned chars (1 byte) or floats (4 bytes). Index for $(x,y,z)$ is $x + y R_x + z R_x R_y$ [S1, unofficial].
*   **Hermite Data:** For each voxel, the gradient or difference between neighboring values is computed to form a second-order interpolant [S1, unofficial].
*   **Full Case Table:** A lookup table containing all possible cell configurations, including those ambiguous in the standard 15-case table. Ambiguity is resolved using the average value of face corners [S1, unofficial].

**Steps:**

1.  **Preprocessing (Optional):** Apply 1-ring smoothing iterations to the voxel grid if `--sIters > 0` [S1, unofficial].
2.  **Cell Iteration:** Iterate over every voxel cell in the grid. A cell is defined by 8 vertices.
3.  **Case Identification:** Determine the binary index of the cell based on the sign of $(V_i - I)$ for each vertex $i$.
4.  **Ambiguity Resolution (Full Case Table):**
    *   If `--full` is specified, use the full case table.
    *   For ambiguous cases (where a face has a zero-crossing and antipodal corners have the same value), resolve the topology by using the average value of the face corners [S1, unofficial].
5.  **Zero-Crossing Calculation:**
    *   If `--quadratic` is specified, compute Hermite data by considering the difference between neighboring values [S1, unofficial].
    *   Use a second-order interpolant to define the positions of zero-crossings on the cell edges.
    *   If `--quadratic` is not specified, use standard linear interpolation [S1, unofficial].
6.  **Polygon Generation:** Generate the raw iso-polygons for the cell based on the resolved case and intersection points.
7.  **Triangulation:**
    *   If `--polygon` is NOT specified, triangulate the raw polygons.
    *   Use minimal-area-triangulation to produce the triangle mesh [S1, unofficial].
8.  **Manifold Enforcement:**
    *   Check for non-manifold edges (where two polygons share two vertices not on a polygon edge).
    *   If `--nonManifold` is NOT specified and a non-manifold configuration is detected, introduce an additional vertex at the plane's barycenter to ensure a manifold mesh [S1, unofficial].
    *   If `--nonManifold` is specified, skip this step, potentially resulting in non-manifold edges [S1, unofficial].
9.  **Orientation:** If `--flip` is specified, flip the triangle orientation [S1, unofficial].
10. **Output:** Write the resulting mesh to the specified PLY file [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_isosurface(volume, resolution, dimensions, isovalue, flags):
    """
    Extract isosurface using Marching Cubes with Hermite Data and Full Case Table.
    
    Args:
        volume: 1D array of voxel values, linearized row-major.
        resolution: Tuple (Rx, Ry, Rz).
        dimensions: Tuple (Dx, Dy, Dz).
        isovalue: Float isovalue.
        flags: Dict with keys 'full', 'quadratic', 'polygon', 'nonManifold', 'flip', 'sIters'.
    """
    Rx, Ry, Rz = resolution
    Dx, Dy, Dz = dimensions
    
    # Optional smoothing
    if flags.get('sIters', 0) > 0:
        volume = apply_1ring_smoothing(volume, resolution, flags['sIters'])
        
    vertices = []
    triangles = []
    
    # Iterate over all cells
    for z in range(Rz - 1):
        for y in range(Ry - 1):
            for x in range(Rx - 1):
                # Get 8 vertices of the cell
                cell_vertices = get_cell_vertices(volume, x, y, z, resolution)
                
                # Determine case index
                case_index = compute_case_index(cell_vertices, isovalue)
                
                # Resolve ambiguity if full table is used
                if flags.get('full', False):
                    case_index = resolve_ambiguous_case_full_table(case_index, cell_vertices)
                
                # Compute intersection points
                if flags.get('quadratic', False):
                    # Hermite interpolation
                    intersections = compute_hermite_intersections(cell_vertices, isovalue, resolution)
                else:
                    # Linear interpolation
                    intersections = compute_linear_intersections(cell_vertices, isovalue)
                
                # Generate polygon
                polygon = generate_polygon(case_index, intersections)
                
                if flags.get('polygon', False):
                    vertices.extend(polygon)
                else:
                    # Triangulate
                    tri_indices = minimal_area_triangulation(polygon)
                    
                    # Check for non-manifold edges
                    if not flags.get('nonManifold', False):
                        if is_non_manifold_potential(polygon):
                            # Insert barycenter
                            barycenter = compute_barycenter(polygon)
                            vertices.append(barycenter)
                            tri_indices = triangulate_with_barycenter(polygon, barycenter, tri_indices)
                    
                    # Add triangles
                    for tri in tri_indices:
                        triangles.append(tri)
                        
    if flags.get('flip', False):
        flip_triangle_orientations(triangles)
        
    return vertices, triangles

def compute_hermite_intersections(cell_vertices, isovalue, resolution):
    """
    Compute zero-crossing positions using second-order Hermite interpolation.
    Hermite data is generated by considering the difference between neighboring values.
    """
    # Implementation detail: Compute gradients/differences from neighboring voxels
    # Use these to form a quadratic interpolant along each edge
    intersections = {}
    for edge in cell_edges:
        # Get values and differences for the edge
        v_start, v_end = get_edge_vertices(edge, cell_vertices)
        diff_start, diff_end = get_edge_differences(edge, cell_vertices, resolution)
        
        # Solve for zero crossing in quadratic interpolant
        # P(t) = v_start + t * (v_end - v_start) + ... (Hermite terms)
        # Find t in [0, 1] such that P(t) = isovalue
        t = solve_quadratic_zero_crossing(v_start, v_end, diff_start, diff_end, isovalue)
        if t is not None:
            intersections[edge] = lerp(cell_vertices[edge[0]], cell_vertices[edge[1]], t)
    return intersections

def resolve_ambiguous_case_full_table(case_index, cell_vertices):
    """
    Resolve ambiguous cases using the average value of face corners.
    """
    # Specific logic for ambiguous cases (e.g., Case 13, 14 in standard table)
    # Check if face has zero-crossing and antipodal corners have same value
    # Use average of face corners to determine topology
    return resolved_case_index
```

**Parameters:**
*   `--iso`: Isovalue. Default: 0 [S1, unofficial].
*   `--dim`: Voxel scale $(D_x, D_y, D_z)$. Default: $(1, 1, 1)$ [S1, unofficial].
*   `--res`: Resolution $(R_x, R_y, R_z)$. If unspecified, assumes cube grid and reads first 4 bytes as resolution [S1, unofficial].
*   `--sIters`: Smoothing iterations. Default: 0 [S1, unofficial].
*   `--full`: Enable full case table.
*   `--quadratic`: Enable Hermite interpolation.
*   `--polygon`: Output raw polygons.
*   `--nonManifold`: Disable barycenter insertion.
*   `--flip`: Flip triangle orientation.
*   `--float`: Read input as floats.
*   `--flipBytes`: Flip float byte order.

**Edge Cases and Pitfalls:**
*   **Non-manifold edges:** Two polygons may share two vertices that are not on a polygon edge. Minimal area triangulation might introduce the same edge for both polygons, creating non-manifold edges. By default, the code introduces an additional vertex (the plane's barycenter) to ensure a manifold mesh. The `--nonManifold` flag disables this, potentially resulting in non-manifold edges [S1, unofficial].
*   **Input Format:** Values are linearized in row-major order. Incorrect resolution specification leads to misaligned data access [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, assuming constant-time lookup and interpolation per cell. Hermite interpolation adds constant overhead per edge.
*   **Memory Complexity:** $O(N)$ for the voxel grid and output mesh.
*   **Accuracy:** Hermite interpolation provides second-order accuracy, improving geometric fidelity over linear interpolation [S1, unofficial].
*   **Topology:** The full case table resolves ambiguities present in the original Lorensen and Cline implementation, ensuring topological correctness for the defined cases [S1, unofficial].
*   **Manifoldness:** The default behavior ensures a manifold mesh by inserting barycenters when necessary. Disabling this with `--nonManifold` risks non-manifold outputs [S1, unofficial].
*   **Hardware:** The source mentions improved parallelization in later versions, suggesting suitability for GPU or multi-core CPU implementation [S1, unofficial].

## Background
This method is based on the work presented in the 3DV 2015 paper "Accurate Isosurface Interpolation with Hermite Data" [S1, unofficial]. It is an implementation of Marching Cubes that specifically addresses the limitations of linear interpolation and the original lookup table's ambiguity handling. The code is available from the Department of Computer Science at Johns Hopkins University [S1, unofficial].

## Sources
[S1] Isosurface Extraction - Department of Computer Science, https://www.cs.jhu.edu/~misha/Code/IsoSurfaceExtraction/, fetched 2026-10-27 (unofficial)
