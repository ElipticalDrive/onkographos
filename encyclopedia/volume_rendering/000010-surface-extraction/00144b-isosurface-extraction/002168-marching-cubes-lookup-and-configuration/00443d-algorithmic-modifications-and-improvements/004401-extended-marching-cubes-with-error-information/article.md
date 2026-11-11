# Extended Marching Cubes with Error Information

Also called: Extended Marching Cubes Algorithm with Error Information, Marching Cubes with Uncertainty

## Overview
This method extends the standard Marching Cubes algorithm to incorporate local error information into the iso-surface representation, enabling the visualization of data reliability alongside the geometry [S1]. It solves the problem that standard Marching Cubes assigns constant properties (like color) to vertices, ignoring the uncertainty inherent in multi-resolution or processed data [S1]. By interpolating error metrics alongside geometric coordinates, it allows users to identify which parts of the iso-surface are authentic and which are unreliable [S1]. It differs from siblings like [Snap Marching Cubes](2181-snap-marching-cubes.md) and [Bitwise Marching Cubes](7374-bitwise-marching-cubes.md) by explicitly carrying and interpolating a scalar error field rather than modifying lookup tables or snapping values.

## How it works
The algorithm operates on a structured grid where the smallest processing unit is a cube. It iterates through every cube in the dataset to construct a 3D iso-surface representation with associated error metrics [S1].

### Inputs and Outputs
*   **Input:** A scalar field defined on a regular grid, where each vertex has an associated attribute value and an error value. The error values are typically derived from a multi-resolution decomposition, such as the Haar wavelet transformation [S1].
*   **Output:** A triangle mesh where each vertex carries geometric coordinates, a normal vector, and an interpolated error value [S1].

### Data Structures
The algorithm uses two primary structures to store cube and triangle information [S1]:

1.  **CUBE**: Stores information for a single voxel.
    *   `p[8]`: Array of 8 `XYZ` coordinates representing the vertices of the cube (vertex 0 to vertex 7).
    *   `val[8]`: Array of 8 double-precision attribute values corresponding to the 8 vertices.
    *   `errorInfo[8]`: Array of 8 double-precision error values corresponding to the 8 vertices. These values represent the local measurement of error, often calculated using a Standard Deviation Model (SDM) based on grid points [S1].

2.  **TRIANGLE**: Stores information for a generated triangular facet.
    *   `p[3]`: Array of 3 `XYZ` coordinates representing the vertices of the triangle. These vertices lie on the edges of the cube where the iso-surface intersects.
    *   `errorInfo[3]`: Array of 3 double-precision error values, one for each vertex of the triangle.
    *   `n`: A `XYZ` vector representing the normal of the triangular facet [S1].

### Algorithm Steps
1.  **Iterate Cubes:** Traverse the entire dataset cube by cube.
2.  **Load Cube Data:** For the current cube, populate the `CUBE` structure with the 8 vertex coordinates (`p`), scalar values (`val`), and error values (`errorInfo`) from the input grid.
3.  **Determine Configuration:** Apply the standard Marching Cubes lookup logic (as defined in [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md)) using the `val` array and a specified `isolevel` to determine which edges are intersected by the iso-surface.
4.  **Generate Triangles:** For the identified configuration, generate up to five triangular facets that approximate the iso-surface within the cube [S1].
5.  **Interpolate Geometry and Error:** For each vertex of each generated triangle:
    *   Identify the two cube vertices defining the edge on which the triangle vertex lies. Let these be Vertex 1 and Vertex 2.
    *   Let $val_{p1}$ and $val_{p2}$ be the scalar values at Vertex 1 and Vertex 2.
    *   Let $e_1$ and $e_2$ be the error values at Vertex 1 and Vertex 2.
    *   Calculate the interpolation factor $\mu$ using the scalar values:
        $$ \mu = \frac{isolevel - val_{p1}}{val_{p2} - val_{p1}} $$
    *   Calculate the interpolated error $e$ using linear interpolation:
        $$ e = e_1 + \mu \cdot (e_2 - e_1) $$
    *   Calculate the interpolated geometric position $P$ similarly:
        $$ P = P_1 + \mu \cdot (P_2 - P_1) $$
    *   Assign $P$ to the triangle vertex's `p` field and $e$ to the triangle vertex's `errorInfo` field.
6.  **Compute Normal:** Calculate the normal vector $n$ for the triangle based on its three vertices.
7.  **Store Triangle:** Populate the `TRIANGLE` structure with the three vertices (including their interpolated errors) and the normal.
8.  **Repeat:** Continue until all cubes are processed.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extended_marching_cubes(grid_coords, grid_vals, grid_errors, isolevel):
    """
    grid_coords: List of XYZ coordinates for all grid vertices.
    grid_vals: List of scalar values for all grid vertices.
    grid_errors: List of error values for all grid vertices.
    isolevel: The scalar threshold for iso-surface extraction.
    Returns: List of TRIANGLE structures.
    """
    triangles = []
    # Assume grid is a regular 3D lattice. 
    # Iterate over all cubes defined by indices (i, j, k)
    for i in range(num_x - 1):
        for j in range(num_y - 1):
            for k in range(num_z - 1):
                # 1. Identify the 8 vertices of the current cube
                # Vertex indices depend on the grid indexing scheme.
                # Let's assume a standard ordering:
                # v0=(i,j,k), v1=(i+1,j,k), v2=(i+1,j+1,k), v3=(i,j+1,k)
                # v4=(i,j,k+1), v5=(i+1,j,k+1), v6=(i+1,j+1,k+1), v7=(i,j+1,k+1)
                vertex_indices = [
                    idx(i, j, k), idx(i+1, j, k), idx(i+1, j+1, k), idx(i, j+1, k),
                    idx(i, j, k+1), idx(i+1, j, k+1), idx(i+1, j+1, k+1), idx(i, j+1, k+1)
                ]
                
                # 2. Populate CUBE structure
                cube_p = [grid_coords[v] for v in vertex_indices]
                cube_val = [grid_vals[v] for v in vertex_indices]
                cube_err = [grid_errors[v] for v in vertex_indices]
                
                # 3. Determine configuration using standard MC lookup
                # This returns a list of edges intersected by the isolevel.
                # Edges are typically indexed 0-11.
                intersected_edges = get_marching_cubes_configuration(cube_val, isolevel)
                
                # 4. Generate triangles for this configuration
                # The standard MC algorithm defines which triangles to form based on the configuration index.
                # We must map the intersected edges to triangle vertices.
                # For each triangle in the configuration:
                for triangle_edges in get_triangles_for_configuration(intersected_edges):
                    triangle_vertices = []
                    for edge_idx in triangle_edges:
                        # Identify the two cube vertices forming this edge
                        v1_idx, v2_idx = get_edge_vertices(edge_idx)
                        
                        val_p1 = cube_val[v1_idx]
                        val_p2 = cube_val[v2_idx]
                        e1 = cube_err[v1_idx]
                        e2 = cube_err[v2_idx]
                        p1 = cube_p[v1_idx]
                        p2 = cube_p[v2_idx]
                        
                        # 5. Interpolate Error and Position
                        # Handle edge cases to avoid division by zero
                        if abs(isolevel - val_p1) < 0.00001:
                            mu = 0.0
                        elif abs(isolevel - val_p2) < 0.00001:
                            mu = 1.0
                        elif abs(val_p1 - val_p2) < 0.00001:
                            mu = 0.0
                        else:
                            mu = (isolevel - val_p1) / (val_p2 - val_p1)
                        
                        interpolated_error = e1 + mu * (e2 - e1)
                        interpolated_pos = p1 + mu * (p2 - p1)
                        
                        triangle_vertices.append({
                            'p': interpolated_pos,
                            'errorInfo': interpolated_error
                        })
                    
                    # 6. Compute Normal
                    # Calculate normal from the three interpolated positions
                    v0 = triangle_vertices[0]['p']
                    v1 = triangle_vertices[1]['p']
                    v2 = triangle_vertices[2]['p']
                    normal = compute_normal(v0, v1, v2)
                    
                    # 7. Store Triangle
                    tri = {
                        'p': [tv['p'] for tv in triangle_vertices],
                        'errorInfo': [tv['errorInfo'] for tv in triangle_vertices],
                        'n': normal
                    }
                    triangles.append(tri)
                    
    return triangles

def get_triangle_vertex_error(isolevel, e1, e2, val_p1, val_p2):
    """
    Calculates the interpolated error for a triangle vertex.
    """
    if abs(isolevel - val_p1) < 0.00001:
        return e1
    if abs(isolevel - val_p2) < 0.00001:
        return e2
    if abs(val_p1 - val_p2) < 0.00001:
        return e1
    
    mu = (isolevel - val_p1) / (val_p2 - val_p1)
    e = e1 + mu * (e2 - e1)
    return e
```

### Parameters
*   **isolevel**: The scalar value defining the iso-surface. Typical values depend on the data range.
*   **Threshold for stability**: The sources use a fixed threshold of `0.00001` to check for near-zero denominators or when the isolevel coincides with a vertex value [S1].

### Edge Cases and Pitfalls
*   **Division by Zero/Instability**: If the `isolevel` is very close to `val_p1` or `val_p2`, or if `val_p1` equals `val_p2`, the interpolation factor $\mu$ calculation becomes unstable or undefined. The algorithm explicitly checks for these conditions (using a tolerance of `0.00001`) and returns the error of the nearest vertex or the first vertex respectively [S1].
*   **Error Source**: The error values must be pre-computed. The sources mention using the Standard Deviation Model (SDM) derived from wavelet transformations (e.g., Haar wavelet) to quantify errors in multi-resolution data [S1]. The algorithm itself does not compute the error; it only interpolates it.

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of voxels/cubes in the grid, $O(N)$, where $N$ is the number of cubes. Each cube is processed independently with a constant number of operations (lookup, interpolation, triangle generation).
*   **Memory Complexity**: Linear with respect to the number of output triangles. The memory footprint is slightly higher than standard Marching Cubes due to the storage of the `errorInfo` array for each vertex in the output triangles.
*   **Strengths**: Provides explicit visual feedback on data reliability. It allows users to distinguish between high-confidence and low-confidence regions of the iso-surface, which is critical for medical imaging (MR data) and other scientific domains where data quality varies [S1].
*   **Limitations**: The accuracy of the error visualization depends entirely on the quality of the pre-computed error metrics. If the error model (e.g., SDM) is inaccurate, the visualization will be misleading. The method assumes a regular grid structure [S1].
*   **When to use**: When visualizing multi-resolution data (like MR images) where uncertainty is a key factor in interpretation.
*   **When not to use**: When the data is exact and error information is not available or relevant, or when performance is critical and the overhead of error interpolation is unjustified.

## Background
The method was introduced in the context of visualizing multi-resolution volumetric data, specifically Magnetic Resonance (MR) data [S1]. The authors utilized Haar wavelet transformation to create a multi-resolution hierarchy of the data, which naturally provides error estimates (approximation errors) at different resolutions [S1]. The standard Marching Cubes algorithm was extended to incorporate these error values, allowing for uncertainty visualization. The work builds upon the standard Marching Cubes algorithm [11] and previous attempts at uncertainty visualization [14], providing specific implementation details for the interpolation of error information [S1].

## Sources
[S1] Visualizing uncertainty in multi-resolution volumetric data using marching cubes, https://doi.org/10.1145/2254556.2254649, fetched 2026-11-05
