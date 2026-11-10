# Marching Cubes with Hermite Data and Full Case Table

Also called: Accurate Isosurface Interpolation with Hermite Data, Full Case Marching Cubes

## Overview
This method extends the standard Marching Cubes algorithm by incorporating Hermite interpolation for higher-order surface positioning and a full case table to resolve topological ambiguities inherent in the trilinear interpolant [S1, unofficial]. While standard Marching Cubes relies on linear interpolation and a reduced 15-case lookup table, this approach uses a 256-case (full) table and second-order Hermite data derived from neighboring voxel differences to produce a more accurate and topologically consistent 2-manifold mesh [S1, unofficial]. It is chosen when the linear approximation of the isosurface within a voxel leads to significant geometric error or topological errors due to saddle points [S1, unofficial].

## How it works
The algorithm processes a regularly sampled 3D volume grid to extract an isosurface. It resolves the ambiguity of the standard 15-case table by evaluating the average value of face corners for specific ambiguous configurations and uses Hermite interpolation to refine vertex positions on edges [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   **Volume Data:** A 3D grid of scalar values. Stored in binary format as either unsigned chars (1 byte) or floats (4 bytes) [S1, unofficial].
*   **Resolution:** A triplet of integers $(R_x, R_y, R_z)$ defining the grid dimensions. If unspecified, the grid is assumed cubic, and the first 4 bytes of the file contain the resolution [S1, unofficial].
*   **Voxel Scale:** A triplet of floats $(S_x, S_y, S_z)$ defining the physical size of a voxel. Default is $(1, 1, 1)$ [S1, unofficial].
*   **Isovalue:** A float specifying the threshold for extraction. Default is $0$ [S1, unofficial].
*   **Smoothing Iterations:** An integer specifying the number of 1-ring smoothing iterations applied to the input grid before extraction. Default is $0$ [S1, unofficial].

**Outputs:**
*   **Mesh:** A triangle mesh (PLY format) representing the isosurface. If the `--polygon` flag is not set, raw iso-polygons are triangulated using a minimal-area-triangulation [S1, unofficial].

### Data Structures
The implementation assumes a linearized storage for the voxel grid. For a grid of resolution $R_x \times R_y \times R_z$, the value at index $(x, y, z)$ is stored at linear index:
$$ I(x, y, z) = x + y \cdot R_x + z \cdot R_x \cdot R_y $$
Each voxel is a unit cube in local coordinates, mapped to physical space via the voxel scale [S1, unofficial].

### Steps

1.  **Preprocessing (Optional Smoothing):**
    If `sIters > 0`, apply 1-ring smoothing to the input volume data `sIters` times. This averages each voxel's value with its immediate neighbors [S1, unofficial].

2.  **Hermite Data Generation (Optional):**
    If the `--quadratic` flag is specified, generate Hermite data. This involves computing the difference between neighboring voxel values to form a second-order interpolant. This data is used to determine the precise position of zero-crossings on voxel edges, replacing standard linear interpolation [S1, unofficial].

3.  **Voxel Processing:**
    Iterate through each voxel in the grid. For each voxel, determine the sign of the scalar field at its 8 corners relative to the `isovalue`.

4.  **Case Lookup and Ambiguity Resolution:**
    *   If the `--full` flag is specified, use the full 256-case lookup table.
    *   For ambiguous cases where a face has a zero-crossing and the values on the antipodal corners of that face are the same, resolve the topology by using the **average value of the face corners** [S1, unofficial].
    *   If `--full` is not specified, use the standard 15-case table (standard Marching Cubes behavior) [S1, unofficial].

5.  **Vertex Position Calculation:**
    *   If `--quadratic` is specified, calculate the position of the zero-crossing on each intersected edge using the second-order Hermite interpolant derived from the Hermite data [S1, unofficial].
    *   Otherwise, calculate the position using standard linear interpolation between the corner values [S1, unofficial].

6.  **Polygon Generation:**
    Generate the raw iso-polygons for the voxel based on the resolved case and calculated vertex positions [S1, unofficial].

7.  **Triangulation:**
    *   If the `--polygon` flag is NOT specified, triangulate the raw iso-polygons.
    *   Use **minimal-area-triangulation** to produce the final triangle mesh [S1, unofficial].
    *   **Manifoldness Check:** During triangulation, check if two polygons share two vertices that are not on a polygon edge. If this condition arises, the minimal area triangulation might introduce the same edge for both polygons, creating non-manifold edges. By default, introduce an additional vertex at the **plane's barycenter** to split the polygons and ensure a manifold mesh. If the `--nonManifold` flag is enabled, skip this step, potentially resulting in non-manifold edges [S1, unofficial].

8.  **Output:**
    Write the resulting mesh to the specified output file in PLY format. Apply the `--flip` flag to invert triangle orientations if required [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_isosurface(volume, resolution, scale, isovalue, s_iters, use_full_table, use_hermite, output_polygons, allow_non_manifold):
    """
    Extracts isosurface from a 3D volume using Marching Cubes with Hermite Data and Full Case Table.
    
    Args:
        volume: 3D array of scalar values (linearized or 3D).
        resolution: Tuple (Rx, Ry, Rz).
        scale: Tuple (Sx, Sy, Sz).
        isovalue: Float threshold.
        s_iters: Integer smoothing iterations.
        use_full_table: Boolean, use 256-case table.
        use_hermite: Boolean, use Hermite interpolation for edge crossings.
        output_polygons: Boolean, output raw polygons instead of triangles.
        allow_non_manifold: Boolean, skip barycenter insertion for non-manifold cases.
        
    Returns:
        List of triangles (vertices and indices) or polygons.
    """
    
    # 1. Preprocessing
    current_volume = volume
    if s_iters > 0:
        for _ in range(s_iters):
            current_volume = apply_1_ring_smoothing(current_volume, resolution)
            
    # 2. Generate Hermite Data if needed
    hermite_data = None
    if use_hermite:
        hermite_data = generate_hermite_data(current_volume, resolution)
        
    triangles = []
    polygons = []
    
    # 3. Iterate over voxels
    for z in range(resolution[2] - 1):
        for y in range(resolution[1] - 1):
            for x in range(resolution[0] - 1):
                # Get corner values
                corners = get_voxel_corners(current_volume, x, y, z, resolution)
                
                # Determine case index (8 bits)
                case_index = 0
                for i in range(8):
                    if corners[i] < isovalue:
                        case_index |= (1 << i)
                        
                # 4. Resolve case
                # If full table is used, handle ambiguous cases
                if use_full_table:
                    case_index = resolve_ambiguous_case(case_index, corners, resolution, x, y, z)
                    
                # Get edge intersections
                edges = get_edge_intersections(corners, case_index, isovalue, hermite_data)
                
                # 5. Generate polygon
                polygon_vertices = []
                for edge_idx in edges:
                    # Calculate physical position
                    local_pos = edges[edge_idx]
                    phys_pos = local_pos * scale + voxel_origin(x, y, z, resolution, scale)
                    polygon_vertices.append(phys_pos)
                    
                if output_polygons:
                    polygons.append(polygon_vertices)
                else:
                    # 6. Triangulate
                    tri = minimal_area_triangulation(polygon_vertices, allow_non_manifold)
                    if tri is not None:
                        triangles.extend(tri)
                        
    return triangles if not output_polygons else polygons

def resolve_ambiguous_case(case_index, corners, resolution, x, y, z):
    """
    Resolves ambiguous cases in the full table by using the average value of face corners.
    Specifically handles cases where a face has a zero-crossing and antipodal corners are equal.
    """
    # This is a placeholder for the specific logic described in [S1, unofficial].
    # The core idea is: if a face has a zero crossing, and the values on the antipodal 
    # corners of that face are the same, use the average value of the face corners 
    # to determine the topology.
    
    # Check for ambiguous faces. A face is ambiguous if it has a zero crossing 
    # (one vertex < iso, one > iso) and the other two vertices (antipodal on the face) 
    # have the same sign/value relationship that causes ambiguity in the 15-case table.
    
    # For each of the 6 faces of the cube:
    for face in get_faces():
        face_corners = get_face_corner_indices(face)
        face_values = [corners[i] for i in face_corners]
        
        # Check if face has zero crossing
        has_zero_crossing = any(v < isovalue for v in face_values) and any(v > isovalue for v in face_values)
        
        if has_zero_crossing:
            # Check if antipodal corners are the same (ambiguous condition)
            # In a square face, antipodal corners are (0,2) and (1,3)
            c0, c1, c2, c3 = face_corners
            if (corners[c0] - isovalue) * (corners[c2] - isovalue) > 0 and \
               (corners[c1] - isovalue) * (corners[c3] - isovalue) > 0:
                # Ambiguous case detected.
                # Use average value of face corners to resolve.
                avg_val = sum(face_values) / 4.0
                # Adjust case index based on avg_val vs isovalue
                # This effectively splits the ambiguous case into two distinct cases
                # based on the local geometry.
                if avg_val > isovalue:
                    # Modify case_index to reflect the resolved topology
                    pass 
                else:
                    pass
                    
    return case_index

def generate_hermite_data(volume, resolution):
    """
    Generates Hermite data by considering the difference between neighboring values.
    """
    # Compute gradients or differences between neighboring voxels
    # This data is used for second-order interpolation
    hermite = {}
    for z in range(resolution[2]):
        for y in range(resolution[1]):
            for x in range(resolution[0]):
                # Example: difference in x-direction
                if x < resolution[0] - 1:
                    diff_x = volume[x+1, y, z] - volume[x, y, z]
                    hermite[(x, y, z, 'dx')] = diff_x
                # Similarly for dy, dz
    return hermite

def minimal_area_triangulation(vertices, allow_non_manifold):
    """
    Triangulates a polygon using minimal area triangulation.
    Handles non-manifold cases by inserting a barycenter if necessary.
    """
    if len(vertices) < 3:
        return []
        
    # Check for non-manifold condition: two polygons sharing two vertices 
    # not on a polygon edge. This is typically detected during global assembly.
    # For a single polygon, we just triangulate.
    
    # Simple fan triangulation or ear clipping for minimal area
    # Note: The source mentions "minimal-area-triangulation" specifically.
    
    # If non-manifold edges are possible (detected globally), insert barycenter
    if not allow_non_manifold and is_non_manifold_risk(vertices):
        barycenter = sum(vertices) / len(vertices)
        # Split polygon into triangles using barycenter
        triangles = []
        for i in range(len(vertices)):
            v1 = vertices[i]
            v2 = vertices[(i + 1) % len(vertices)]
            triangles.append([barycenter, v1, v2])
        return triangles
    else:
        # Standard triangulation
        triangles = []
        for i in range(1, len(vertices) - 1):
            triangles.append([vertices[0], vertices[i], vertices[i+1]])
        return triangles
```

### Parameters
*   `--in <input volume>`: Input file path. [S1, unofficial]
*   `--res <x> <y> <z>`: Grid resolution. [S1, unofficial]
*   `--dim <x> <y> <z>`: Voxel scale. Default: `(1, 1, 1)`. [S1, unofficial]
*   `--iso <value>`: Isovalue. Default: `0`. [S1, unofficial]
*   `--sIters <n>`: Smoothing iterations. Default: `0`. [S1, unofficial]
*   `--full`: Use full case table. [S1, unofficial]
*   `--quadratic`: Use Hermite interpolation. [S1, unofficial]
*   `--polygon`: Output raw polygons. [S1, unofficial]
*   `--nonManifold`: Disable barycenter insertion for non-manifold cases. [S1, unofficial]
*   `--flip`: Flip triangle orientation. [S1, unofficial]
*   `--float`: Read input as floats. [S1, unofficial]
*   `--flipBytes`: Flip float byte order. [S1, unofficial]

### Edge Cases and Pitfalls
*   **Non-manifold Edges:** Two polygons may share two vertices that are not on a polygon edge. The minimal area triangulation might introduce the same edge for both polygons, creating non-manifold edges. By default, the code introduces an additional vertex (the plane's barycenter) to ensure a manifold mesh. The `--nonManifold` flag disables this, potentially resulting in non-manifold edges [S1, unofficial].
*   **Ambiguous Cases:** The standard 15-case table fails when the trilinear interpolant has a saddle point. The full case table resolves this by using the average value of face corners for specific ambiguous configurations [S1, unofficial].
*   **Precision:** Hermite interpolation uses second-order data derived from neighboring value differences. This requires careful handling of numerical precision, especially for small gradients [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, assuming constant-time lookup and interpolation. Hermite data generation adds a linear pass. Smoothing adds $O(k \cdot N)$ where $k$ is the number of iterations.
*   **Memory Complexity:** $O(N)$ for the volume data and Hermite data. The output mesh size depends on the surface area.
*   **Performance:** The implementation includes parallelization improvements [S1, unofficial]. Hermite interpolation is more computationally expensive than linear interpolation due to the need to compute and store differences.
*   **Strengths:** Higher geometric accuracy due to second-order interpolation. Topological correctness for ambiguous cases using the full case table.
*   **Limitations:** More complex implementation. Hermite data generation requires additional memory and computation. Non-manifold handling requires global checks or careful local heuristics.

## Background
This method is based on the work presented in the 3DV 2015 paper "Accurate Isosurface Interpolation with Hermite Data" [S1, unofficial]. It is an implementation of Marching Cubes that supports Hermite Data and a full case table to resolve the ambiguous cases present in the original Lorensen and Cline implementation [S1, unofficial]. The code is available from the Department of Computer Science at Johns Hopkins University [S1, unofficial].

## Sources
[S1] Isosurface Extraction - Department of Computer Science, https://www.cs.jhu.edu/~misha/Code/IsoSurfaceExtraction/, fetched 2026-10-27 (unofficial)
