# Marching Cubes with Gaussian Deformation

Also called: Geometry Shader Marching Cubes, Marching Cubes Geometry Shader Variant, Marching Cubes with Brush Deformation

## Overview
This method extracts an isosurface from a scalar field using the Marching Cubes algorithm, implemented within a GPU Geometry Shader to allow for real-time, procedural deformation of the resulting mesh [S1, unofficial]. It differs from standard static Marching Cubes by applying a Gaussian-based displacement to vertices along a user-defined force vector during primitive generation, enabling dynamic terrain or object interaction [S1, unofficial]. Unlike [Dual Contouring and Surface Nets](6319-dual-contouring-and-surface-nets.md) which optimize for vertex count and normal consistency, this approach prioritizes shader-based interactivity and immediate visual feedback over mesh topology optimization [S1, unofficial].

## How it works
The algorithm operates on a grid of point primitives, where each point represents the center of a voxel in the scalar field. The core logic resides in the Geometry Shader, which processes each voxel independently to generate the corresponding mesh triangles.

### Inputs and Outputs
*   **Input**: A stream of point primitives. Each point contains the world-space position $\mathbf{p}$ of the voxel center.
*   **Uniforms**:
    *   `gridSize` (float): The edge length of a voxel. Typical value: $0.25$ [S1, unofficial].
    *   `i_force` (vec3): The direction and magnitude of the deformation force [S1, unofficial].
    *   `sigma` (float): The width parameter for the Gaussian falloff. Typical value: $0.8$ [S1, unofficial].
    *   `edgeTableTex`, `triTableTex` (isampler2D): Textures storing the standard Marching Cubes lookup tables [S1, unofficial].
*   **Output**: A `TriangleStream` of vertices with positions, normals, and UVs [S2, unofficial].

### Data Structures
The lookup tables are stored in 2D integer textures to allow parallel access by geometry shader invocations.
*   **Edge Table**: Stored in `edgeTableTex`. Accessed via `texelFetch`. Maps a 256-entry cube index to a bitmask indicating which of the 12 edges intersect the isosurface.
*   **Triangle Table**: Stored in `triTableTex`. Maps a 256-entry cube index to a list of triangle vertex indices (terminated by -1).
*   **Edge Vertex Mapping**: A static array `cube_diff_indices` of size 24 (12 edges $\times$ 2 vertices) defines the corner indices for each edge. The mapping is:
    ```text
    0,1, 1,2, 2,3, 3,0, 4,5, 5,6, 6,7, 7,4, 0,4, 1,5, 2,6, 3,7
    ```
    [S1, unofficial]

### Steps
1.  **Voxel Center Evaluation**: For each input point primitive at position $\mathbf{p}_{center}$, calculate the 8 corner positions of the voxel. The corners are offset by $\pm \frac{gridSize}{2}$ in x, y, and z.
2.  **Scalar Field Computation**: Evaluate the scalar field $f(\mathbf{p})$ at each of the 8 corners. The field is defined as a sphere function:
    $$ f(\mathbf{p}) = \|\mathbf{p} - \mathbf{c}\| - (r + s\sqrt{3}) $$
    where $\mathbf{c} = (0, 0, -20)$ is the sphere center, $r = 4.0$ is the radius, and $s$ is the `gridSize` [S1, unofficial].
3.  **Cube Index Generation**: Construct an 8-bit integer `cubeIndex`. If $f(\mathbf{p}_i) < 0.0$ for corner $i$, set the $i$-th bit.
4.  **Edge Intersection Check**: Look up `edgeTableTex` at `cubeIndex`. If the result is 0, discard the primitive (no surface intersection) [S1, unofficial].
5.  **Vertex Interpolation**: For each of the 12 edges $i$ where the $i$-th bit of the edge table result is set:
    *   Retrieve the two corner positions $\mathbf{p}_1, \mathbf{p}_2$ using `cube_diff_indices`.
    *   Compute the interpolation factor $\alpha$:
        $$ \alpha = \frac{|f(\mathbf{p}_1)|}{|f(\mathbf{p}_1)| + |f(\mathbf{p}_2)|} $$
    *   Compute the undeformed intersection point $\mathbf{v}$:
        $$ \mathbf{v} = (1 - \alpha)\mathbf{p}_1 + \alpha\mathbf{p}_2 $$
    [S1, unofficial]
6.  **Gaussian Deformation**: Apply the deformation to $\mathbf{v}$ to get $\mathbf{v}'$:
    $$ \mathbf{v}' = \mathbf{v} + \mathbf{F} \cdot 5 \cdot \mathcal{N}(\sigma, d_{\perp}^2) $$
    where $\mathbf{F}$ is the `i_force` vector, and $\mathcal{N}$ is the Gaussian function:
    $$ \mathcal{N}(\sigma, x) = \frac{e^{-x / (2\sigma^2)}}{\sigma\sqrt{2\pi}} $$
    The squared perpendicular distance $d_{\perp}^2$ is calculated as:
    $$ d_{\perp}^2 = \|\mathbf{v} - \mathbf{c}\|^2 - \frac{((\mathbf{v} - \mathbf{c}) \cdot \mathbf{F})^2}{\|\mathbf{F}\|^2} $$
    [S1, unofficial]
    *   **Guard**: If $\mathbf{F}$ is zero, skip deformation to avoid division by zero [S1, unofficial].
    *   **Guard**: If the dot product of the displacement and force is non-positive, skip deformation to prevent backward pushing [S1, unofficial].
7.  **Triangle Generation**: Look up `triTableTex` at `cubeIndex`. Iterate through the table in steps of 3. For each valid triangle index triplet:
    *   Retrieve the three undeformed vertex positions from the interpolated vertices computed in Step 5.
    *   Apply the deformation (Step 6) to each vertex.
    *   Emit the three deformed vertices to the `TriangleStream`.
    [S1, unofficial]
8.  **Transform and Output**: In the Geometry Shader, transform the emitted vertex positions to clip space. Note: The source implementation uses `mul(UNITY_MATRIX_MVP, unity_WorldToObject)` which is mathematically incorrect for standard MVP transformation (should be $VP \cdot World \cdot Vertex$) [S2, unofficial]. Pass normals and UVs to the fragment shader [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
GRID_SIZE = 0.25
CENTER = (0, 0, -20)
RADIUS = 4.0
SIGMA = 0.8
FORCE = (0, 0, 0) # Uniform input

# Lookup Tables (Simplified representation)
# edge_table is a 256-entry bitmask array
# tri_table is a 256-entry array of integer lists (terminated by -1)
# cube_diff_indices is a 24-entry array mapping edge index i to (v1, v2)

def scalar_field(p):
    """
    Computes the scalar field value at point p.
    f(p) = ||p - c|| - (r + s*sqrt(3))
    """
    dist = length(p - CENTER)
    return dist - (RADIUS + GRID_SIZE * sqrt(3))

def gaussian_deformation(p, force, sigma):
    """
    Applies Gaussian deformation to point p along force vector.
    """
    if length(force) == 0:
        return p
    
    # Calculate perpendicular distance squared
    # d_perp^2 = ||p - c||^2 - ((p - c) . F)^2 / ||F||^2
    diff = p - CENTER
    force_norm_sq = dot(force, force)
    proj_sq = (dot(diff, force) ** 2) / force_norm_sq
    d_perp_sq = dot(diff, diff) - proj_sq
    
    # Gaussian value
    # N(sigma, x) = exp(-x / (2*sigma^2)) / (sigma * sqrt(2*pi))
    gaussian_val = exp(-d_perp_sq / (2 * sigma ** 2)) / (sigma * sqrt(2 * 3.141592653589793))
    
    displacement = force * 5.0 * gaussian_val
    
    # Guard: prevent backward pushing
    if dot(displacement, force) <= 0:
        return p
        
    return p + displacement

def interpolate_vertex(p1, p2, f1, f2):
    """
    Linearly interpolates the zero-crossing point on the edge.
    v = (1 - alpha)*p1 + alpha*p2
    alpha = |f(p1)| / (|f(p1)| + |f(p2)|)
    """
    abs_f1 = abs(f1)
    abs_f2 = abs(f2)
    
    # Guard: Division by zero if both are 0
    if abs_f1 + abs_f2 == 0:
        return (p1 + p2) / 2.0
        
    alpha = abs_f1 / (abs_f1 + abs_f2)
    return (1 - alpha) * p1 + alpha * p2

def process_voxel(voxel_center):
    """
    Geometry Shader entry point for a single voxel.
    """
    half_size = GRID_SIZE / 2.0
    
    # Generate 8 corners
    corners = []
    for x in [-1, 1]:
        for y in [-1, 1]:
            for z in [-1, 1]:
                p = voxel_center + (x * half_size, y * half_size, z * half_size)
                corners.append(p)
                
    # Evaluate scalar field
    f_values = [scalar_field(p) for p in corners]
    
    # Compute cube index
    cube_index = 0
    for i, f in enumerate(f_values):
        if f < 0.0:
            cube_index |= (1 << i)
            
    # Check edge table
    edge_mask = edge_table[cube_index]
    if edge_mask == 0:
        return # No intersection
        
    # Intersect edges
    edge_vertices = []
    for edge_idx in range(12):
        if (edge_mask >> edge_idx) & 1:
            v1_idx, v2_idx = cube_diff_indices[edge_idx * 2], cube_diff_indices[edge_idx * 2 + 1]
            p1, p2 = corners[v1_idx], corners[v2_idx]
            f1, f2 = f_values[v1_idx], f_values[v2_idx]
            
            v = interpolate_vertex(p1, p2, f1, f2)
            edge_vertices.append(v)
            
    # Generate triangles
    tri_indices = tri_table[cube_index]
    i = 0
    while i < len(tri_indices) and tri_indices[i] != -1:
        v1 = edge_vertices[tri_indices[i]]
        v2 = edge_vertices[tri_indices[i+1]]
        v3 = edge_vertices[tri_indices[i+2]]
        
        # Apply deformation
        v1_deformed = gaussian_deformation(v1, FORCE, SIGMA)
        v2_deformed = gaussian_deformation(v2, FORCE, SIGMA)
        v3_deformed = gaussian_deformation(v3, FORCE, SIGMA)
        
        emit_triangle(v1_deformed, v2_deformed, v3_deformed)
        
        i += 3
```

### Edge Cases and Pitfalls
*   **Zero Force Vector**: If `i_force` is $(0,0,0)$, the term $\|\mathbf{F}\|^2$ in the denominator of $d_{\perp}^2$ becomes zero, causing NaN. The implementation must check for this [S1, unofficial].
*   **Zero Scalar Values**: If $f(\mathbf{p}_1) = f(\mathbf{p}_2) = 0$, the interpolation denominator is zero. This results in NaN positions. A guard is required [S1, unofficial].
*   **Matrix Multiplication Error**: The Unity variant uses `mul(UNITY_MATRIX_MVP, unity_WorldToObject)` for clip-space transformation. This is incorrect; the standard order is `UNITY_MATRIX_VP * unity_ObjectToWorld * vertex`. This leads to incorrect rendering [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per voxel processed by the geometry shader, as it performs a fixed number of texture fetches and arithmetic operations. Total complexity is $O(N)$ for $N$ voxels [S1, unofficial].
*   **Memory Complexity**: Low. The algorithm relies on texture-based lookup tables and does not require storing intermediate mesh data structures in CPU memory. The scalar field is evaluated on-the-fly [S1, unofficial].
*   **Performance**: Real-time performance is achieved on modern GPUs due to the parallel nature of the geometry shader. The use of texture fetches for lookup tables avoids large constant buffers [S1, unofficial].
*   **Strengths**: Real-time deformability; simple implementation within the graphics pipeline; no need for pre-computed mesh data [S1, unofficial].
*   **Limitations**: The Gaussian deformation is applied per-vertex, which may cause discontinuities if the force field changes rapidly across the voxel grid. The incorrect matrix multiplication in the Unity variant is a significant pitfall [S2, unofficial].

## Background
The method originates from the NtslMaterial geometry shader implementation in the `ntsl-graphics` project, designed for real-time deformable terrain/mesh interaction in custom OpenGL 3.3+ applications [S1, unofficial]. It extends the classic Marching Cubes algorithm [30-isosurface-extraction.md] by integrating procedural deformation directly into the primitive generation stage. A variant also exists in the Unity ecosystem, though it contains implementation errors regarding matrix transformations [S2, unofficial].

## Sources
[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] kristofe/UnityGPUMarchingCubes, https://github.com/kristofe/UnityGPUMarchingCubes, fetched 2026-10-26 (unofficial)
