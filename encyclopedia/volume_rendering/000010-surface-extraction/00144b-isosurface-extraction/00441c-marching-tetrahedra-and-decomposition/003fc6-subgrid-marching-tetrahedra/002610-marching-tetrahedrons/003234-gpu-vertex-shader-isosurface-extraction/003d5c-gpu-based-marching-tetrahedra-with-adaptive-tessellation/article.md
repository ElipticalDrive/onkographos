# GPU-based Marching Tetrahedra with Adaptive Tessellation

Also called: Hybrid Cubes-Tetrahedra Extraction, Hybrid Cubes-Tetrahedra Isosurface Extraction, Hybrid marching cubes/marching tetrahedra approach, Two-pass Marching Tetrahedra

## Overview
A real-time isosurface extraction method that leverages the strengths of both [Marching Cubes](6705-marching-cubes.md) and [Marching Tetrahedra](6704-marching-tetrahedra.md), adapted for the GPU geometry shader pipeline (specifically DirectX 10). It solves the problem of high computational expense and register pressure associated with explicit isosurface extraction by using massive parallelism and dynamic load balancing. Unlike its siblings, it separates domain voxelization/tetrahedralization from polygon generation into a two-pass process to maximize SIMD utilization and reduce register pressure compared to Marching Cubes [S1].

## How it works
**Inputs:** Scalar volume data $F(\mathbf{x})$ (where $F(\mathbf{x}) = f(\mathbf{x}) - c$ and $c$ is the isovalue, frequently 0), and optionally precomputed gradients $\nabla F(\mathbf{x})$.
**Outputs:** A stream-out buffer containing isosurface triangles.

**Data Structures:**
*   **Stream-out buffer (Pass 1):** Stores tetrahedra as point primitives. Each tetrahedron is represented by 3 vertices. The source specifies these consist of "just 3 4-component floating point values" [S1]. This likely implies a compact representation where the 4th vertex is implicit or the data is packed to minimize footprint, though the exact bit-packing is not explicitly detailed beyond "3 4-component floating point values" [S1].
*   **Stream-out buffer (Pass 2):** Stores output triangles.

**Parameters:**
*   **Isovalue ($c$):** The constant defining the surface $f(\mathbf{x}) = c$.
*   **Grid Resolution:** Base cube grid resolution (e.g., $64^3$) [S1].
*   **Tetrahedralization Count:** Each cube is tessellated into at most 6 tetrahedra [S1].

**Steps:**

1.  **Preprocessing (Offline):** Collect scalar data (e.g., from Visible Human Project) and compute gradients $\nabla F(\mathbf{x})$ for lighting/shading [S1].
2.  **Pass 1: Domain Voxelization and Tetrahedralization:**
    1.  Render a vertex buffer containing $3n$ vertices (where $n$ is grid size) [S1].
    2.  **Vertex Shader:** Use primitive ID to generate voxel corner locations [S1].
    3.  **Geometry Shader (Cube Tetrahedralization):**
        *   Compute voxel cube corner locations.
        *   Evaluate $F(\mathbf{x})$ at each corner.
        *   Compare signs to determine if the voxel contains the isosurface (i.e., if signs of corners differ) [S1].
        *   **Optimization:** Only generate tetrahedra for voxels containing the isosurface [S1].
        *   **Adaptive Refinement:** In the 6-tetrahedra tessellation, each tetrahedron shares a common longest edge along the cube diagonal. Perform one function evaluation at the edge midpoint. Subdivide tetrahedra if signs differ from the center point. Emit only the 12 resulting tetrahedra that contain the surface [S1].
        *   Output these tetrahedra as point primitives to a stream-out buffer [S1].
3.  **Pass 2: Marching Tetrahedra:**
    1.  Render the stream-out buffer from Pass 1 as point primitives using `DrawAuto` [S1].
    2.  **Geometry Shader (Marching Tetrahedra):**
        *   **Edge Identification:** Identify edges of the current tetrahedron where the signs of endpoints differ [S1].
        *   **Root Finding:** Fit a parabola on the intersecting edge and find the root to determine the vertex position. Two options are provided:
            *   (a) **Inverse Quadratic Interpolation:** Perform a third function evaluation $F(x_3)$ along the edge. Use the formula below [S1].
            *   (b) **Gradient Estimation:** Use precomputed gradients $\nabla F(\mathbf{x})$ to estimate the parabola, avoiding the third function evaluation [S1].
        *   **Output:** Output each isosurface triangle to a stream-out buffer [S1].

**Formulas:**

*   **Linear Secant Method (for initial approximation of $x_3$):**
    $$t = \frac{-F(x_1)}{F(x_2) - F(x_1)}$$
    where $x_1, x_2$ are edge endpoints [S1].

*   **Inverse Quadratic Interpolation (Vertex Position $v$):**
    $$v = \frac{F(x_2)F(x_3)}{(F(x_1) - F(x_2))(F(x_1) - F(x_3))}x_1 + \frac{F(x_1)F(x_3)}{(F(x_2) - F(x_1))(F(x_2) - F(x_3))}x_2 + \frac{F(x_1)F(x_2)}{(F(x_3) - F(x_1))(F(x_3) - F(x_2))}x_3$$
    where $x_1, x_2$ are edge endpoints and $x_3$ is a third evaluation point [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: Domain Voxelization and Tetrahedralization
# Input: Grid size n, Scalar field F, Isovalue c
# Output: Stream-out buffer of tetrahedra

def pass1_vertex_shader(primitive_id):
    # Generate voxel corner locations based on primitive ID
    # Returns 3 vertices representing the cube corners? 
    # Source says "3n vertices" for grid size n, likely one vertex per grid point or similar.
    # Assuming standard grid generation logic [G]
    return voxel_corners

def pass1_geometry_shader(voxel_corners):
    # Evaluate F at corners
    f_values = [F(c) for c in voxel_corners]
    
    # Check if voxel contains isosurface
    if signs_differ(f_values):
        # Tessellate cube into 6 tetrahedra
        tetrahedra = tessellate_cube(voxel_corners)
        
        # Adaptive Refinement
        refined_tetrahedra = []
        for tet in tetrahedra:
            # Longest edge is along cube diagonal
            midpoint = tet.longest_edge_midpoint()
            f_mid = F(midpoint)
            
            # Subdivide if signs differ from center point
            if signs_differ_from_center(tet, f_mid):
                # Subdivide into 2 tetrahedra
                sub_tets = subdivide_tetrahedron(tet, midpoint)
                refined_tetrahedra.extend(sub_tetrahedra_containing_surface(sub_tets, f_mid))
            else:
                refined_tetrahedra.append(tet)
        
        # Output tetrahedra as point primitives
        # Each tetrahedron consists of 3 4-component floating point values [S1]
        return refined_tetrahedra
    else:
        return []

# Pass 2: Marching Tetrahedra
# Input: Stream-out buffer of tetrahedra from Pass 1
# Output: Stream-out buffer of triangles

def pass2_geometry_shader(tetrahedron):
    # Identify edges intersecting surface
    intersecting_edges = []
    for edge in tetrahedron.edges:
        if signs_differ(edge.endpoints):
            intersecting_edges.append(edge)
    
    triangles = []
    for edge in intersecting_edges:
        # Root Finding
        x1, x2 = edge.endpoints
        f1, f2 = F(x1), F(x2)
        
        # Option 1: Inverse Quadratic Interpolation
        # Perform third function evaluation
        x3 = edge.midpoint() # Or use secant method for initial guess [S1]
        f3 = F(x3)
        
        # Calculate vertex position v
        v = inverse_quadratic_interpolation(x1, x2, x3, f1, f2, f3)
        
        # Store vertex for triangle construction
        # Note: MT emits at most 2 triangles per tetrahedron [S1]
        # Logic to connect vertices into triangles based on MT topology [G]
        triangles.append(vertex_to_triangle(v))
    
    return triangles

def inverse_quadratic_interpolation(x1, x2, x3, f1, f2, f3):
    # Formula from [S1]
    term1 = (f2 * f3) / ((f1 - f2) * (f1 - f3)) * x1
    term2 = (f1 * f3) / ((f2 - f1) * (f2 - f3)) * x2
    term3 = (f1 * f2) / ((f3 - f1) * (f3 - f2)) * x3
    return term1 + term2 + term3
```

## Subtopics

### GPU Vertex Shader Isosurface Extraction
The parent method, which computes intersections in the vertex shader by drawing a single quadrilateral for each tetrahedron. This method improves upon it by using a two-pass approach with geometry shaders for better load balancing and reduced register pressure [S1].

### Marching Tetrahedra
The core algorithm used in Pass 2. It uses a simpler cell primitive with only 16 possible sign combinations (vs 256 for MC), emits at most 2 triangles (vs up to 6 for MC), and has no ambiguous cases [S1].

### Marching Cubes
The sibling method. MC has larger lookup tables (4608 entries) causing register pressure and potential connectivity ambiguities [S1]. This method avoids MC's pitfalls by using tetrahedra for extraction while using cubes for adaptive tessellation.

## Operating characteristics
*   **Time Complexity:** Real-time interactive rates. The method processes the Virtual Human Project dataset (576MB) at extremely fast frame rates [S1].
*   **Memory Complexity:** Uses stream-out buffers. Pass 1 outputs tetrahedra (3 4-component floats per tetrahedron). Pass 2 outputs triangles. Memory savings are achieved by only generating tetrahedra for voxels containing the isosurface [S1].
*   **Strengths:**
    *   Maximizes SIMD utilization by separating extraction into two phases [S1].
    *   Reduces register pressure compared to Marching Cubes due to smaller lookup tables [S1].
    *   Adaptive refinement improves surface quality with minimal cost (one extra function evaluation per cube) [S1].
    *   Avoids cracks by subdividing along the center shared edge, ensuring consistency with neighboring cells [S1].
    *   Reuses function evaluations from the initial cube grid [S1].
*   **Limitations:**
    *   Requires DirectX 10 or later for geometry shader and stream-out support [S1].
    *   Tetrahedra are irregular shapes, lacking the intuitive grid correspondence of cubes [S1].
    *   Adaptive refinement adds complexity to the geometry shader [S1].
*   **When to use:** Medical visualization (CT/MRI), colonoscopy tumor analysis, and applications requiring real-time isosurface extraction with dynamic isovalue changes [S1].
*   **When not to use:** Applications without programmable geometry shaders or stream-out capabilities [S1].

## Background
Originated by Natalya Tatarchuk, Jeremy Shopf, and Christopher DeCoro from AMD and Princeton University [S1]. Published in 2008 [S1]. It was designed to take advantage of the massive parallelism and high memory bandwidth of consumer GPUs (specifically ATI Radeon HD 2000 and 3000 series) and Shader Model 4.0 capabilities [S1]. Prior work lacked programmable primitive processing and had to redundantly perform extraction regardless of dynamic isovalue changes [S1]. This method was one of the first to leverage the geometry shader's amplification feature and stream-out for efficient isosurface extraction [S1].

## Sources
[S1] Real-Time Isosurface Extraction Using the GPU Programmable Geometry Pipeline, https://doi.org/10.1145/1281500.1361219, fetched 2026-11-03
