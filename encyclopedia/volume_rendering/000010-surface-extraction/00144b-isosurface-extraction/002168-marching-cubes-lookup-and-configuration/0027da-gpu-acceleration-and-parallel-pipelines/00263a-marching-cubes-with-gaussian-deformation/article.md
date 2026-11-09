# Marching Cubes with Gaussian Deformation

Also called: Geometry Shader Marching Cubes, Marching Cubes Geometry Shader Variant, Marching Cubes with Brush Deformation

## Overview
This method extracts an isosurface from a scalar field using a Geometry Shader, applying a Gaussian-based deformation to the generated vertices along a user-provided force vector [S1, unofficial]. It differs from standard GPU Marching Cubes by integrating the deformation directly into the geometry shader's vertex emission phase, allowing for real-time, dynamic modification of the mesh topology and shape based on external forces [S1, unofficial]. Unlike [GPU Marching Cubes with Atomic Counter Emission](4082-gpu-marching-cubes-with-atomic-counter-emission.md) which focuses on buffer management, this variant prioritizes the mathematical deformation of the surface geometry itself [S1, unofficial].

## How it works
The method builds on [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md), utilizing the precomputed `edgeTable` and `triangleTable` structures [S4, S8, S12, unofficial]. It operates on a point cloud input where each point represents the center of a voxel in the scalar field grid.

### Inputs and Outputs
*   **Input**: A point primitive buffer where each point corresponds to a voxel center $(x, y, z)$ in world space.
*   **Uniforms**:
    *   `gridSize` (float): The edge length of a voxel. Typical value: $0.25$ [S1, unofficial].
    *   `i_force` (vec3): The deformation direction and magnitude vector $\mathbf{F}$ [S1, unofficial].
    *   `sigma` (float): The width of the Gaussian falloff. Typical value: $0.8$ [S1, unofficial].
    *   `lightPos` (vec3): Light position in world space [S2, unofficial].
*   **Output**: A `TriangleStream` of deformed vertices with positions, normals, and UVs [S2, unofficial].

### Data Structures
The lookup tables are stored in 2D textures for GPU access [S1, unofficial]:
1.  **`edgeTableTex`**: Stores the 256-entry `edgeTable`. Format: `GL_R16I` (signed 16-bit integer), dimensions $256 \times 1$. Accessed via `texelFetch`.
2.  **`triTableTex`**: Stores the $256 \times 16$ `triangleTable`. Format: `GL_R16I`, dimensions $256 \times 16$. Accessed via `texelFetch`.
3.  **`cube_diff_indices`**: A static array of size 24 (12 edges $\times$ 2 vertices) mapping edge index $i$ to vertex indices $v_1, v_2$.
    *   Layout: `[0,1, 1,2, 2,3, 3,0, 4,5, 5,6, 6,7, 7,4, 0,4, 1,5, 2,6, 3,7]` [S1, unofficial].

### Steps
1.  **Vertex Shader**:
    *   Transform the input voxel center position $\mathbf{p}_{center}$ to world space using `unity_ObjectToWorld` [S2, unofficial].
    *   Pass normal and texcoords unchanged [S2, unofficial].

2.  **Geometry Shader (Per Voxel)**:
    *   **Scalar Field Evaluation**: For the voxel center $\mathbf{p}_{center}$, compute the scalar field value $f(\mathbf{p})$ at the 8 corners. The corners are defined relative to the center and `gridSize`.
        *   Formula: $f(\mathbf{p}) = \|\mathbf{p} - \mathbf{c}\| - (r + s\sqrt{3})$ [S1, unofficial].
        *   Where $\mathbf{c} = (0, 0, -20)$ is the field center, $r = 4.0$ is the radius, and $s$ is `gridSize` [S1, unofficial].
    *   **Case Index Calculation**: Construct the 8-bit case index $C$. If $f(\mathbf{p}_i) < 0.0$, set bit $i$ in $C$ [S1, unofficial].
    *   **Edge Intersection Check**: Fetch `edgeTable[C]` from `edgeTableTex`. If the result is 0, discard the primitive (no intersection) [S1, unofficial].
    *   **Vertex Interpolation**: Iterate over the 12 edges. For each edge $i$ where the $i$-th bit of the edge table result is set:
        *   Retrieve the two corner vertices $\mathbf{p}_1, \mathbf{p}_2$ using `cube_diff_indices` [S1, unofficial].
        *   Compute the interpolation factor $\alpha = \frac{|f(\mathbf{p}_1)|}{|f(\mathbf{p}_1)| + |f(\mathbf{p}_2)|}$ [S1, unofficial].
        *   Compute the undeformed intersection point: $\mathbf{v} = (1 - \alpha)\mathbf{p}_1 + \alpha\mathbf{p}_2$ [S1, unofficial].
        *   Store $\mathbf{v}$ in a temporary array `undeformed_vertices[edge_index]`.
    *   **Triangle Emission with Deformation**: Iterate through `triangleTable[C]` in steps of 3. For each triangle defined by edge indices $e_1, e_2, e_3$:
        *   Retrieve the undeformed positions $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$ from `undeformed_vertices` [S1, unofficial].
        *   For each vertex $\mathbf{v}_k$:
            *   Apply Gaussian deformation: $\mathbf{p}' = \mathbf{v}_k + \mathbf{F} \cdot 5 \cdot \mathcal{N}(\sigma, d_{\perp}^2)$ [S1, unofficial].
            *   Gaussian function: $\mathcal{N}(\sigma, x) = \frac{e^{-x / (2\sigma^2)}}{\sigma\sqrt{2\pi}}$ [S1, unofficial].
            *   Perpendicular distance squared: $d_{\perp}^2 = \|\mathbf{v}_k - \mathbf{c}\|^2 - \frac{((\mathbf{v}_k - \mathbf{c}) \cdot \mathbf{F})^2}{\|\mathbf{F}\|^2}$ [S1, unofficial].
            *   **Guard**: If $\|\mathbf{F}\| \approx 0$, skip deformation (return $\mathbf{v}_k$) to avoid NaN [S1, unofficial].
            *   **Guard**: If $((\mathbf{v}_k - \mathbf{c}) \cdot \mathbf{F}) \le 0$, skip deformation to prevent backward pushing [S1, unofficial].
            *   Set the output position to $\mathbf{p}'$.
        *   Compute the normal for the triangle using the cross product of the deformed vertices: $\mathbf{n} = \text{normalize}((\mathbf{p}_2' - \mathbf{p}_1') \times (\mathbf{p}_3' - \mathbf{p}_1'))$ [S1, unofficial].
        *   Transform the deformed position $\mathbf{p}'$ to clip space. **Note**: The source code uses `mul(UNITY_MATRIX_MVP, unity_WorldToObject)` which is mathematically incorrect for standard MVP transformation (should be $VP \cdot World \cdot V$) [S2, unofficial]. This likely produces incorrect clip-space positions [S2, unofficial].
        *   Emit the vertex with position, normal, and UVs to the `TriangleStream` [S2, unofficial].
        *   After emitting 3 vertices, call `RestartStrip()` [S2, unofficial].

3.  **Fragment Shader**:
    *   Compute lighting using normalized light vector $\mathbf{L}$ derived from `lightPos` and fragment position [S2, unofficial].
    *   Calculate diffuse lighting: $diffuse = \max(\mathbf{N} \cdot \mathbf{L}, 0)$ [S2, unofficial].
    *   Sample sprite texture and multiply by diffuse light [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Geometry Shader Logic (Pseudocode)

def geometry_shader(point_input):
    # point_input contains voxel center in world space
    voxel_center = point_input.pos
    
    # 1. Evaluate Scalar Field at 8 corners
    cube_values = []
    for i in range(8):
        # Determine corner offset based on bit i
        offset = get_corner_offset(i, gridSize)
        corner_pos = voxel_center + offset
        # f(p) = ||p - c|| - (r + s*sqrt(3))
        dist = length(corner_pos - center_c)
        f_val = dist - (radius_r + gridSize * sqrt(3))
        cube_values.append(f_val)
        
    # 2. Compute Case Index C
    case_index = 0
    for i in range(8):
        if cube_values[i] < 0.0:
            case_index |= (1 << i)
            
    # 3. Lookup Edge Table
    edge_mask = texelFetch(edgeTableTex, ivec2(case_index, 0), 0).r
    
    if edge_mask == 0:
        return # Discard primitive
        
    # 4. Intersect Edges
    undeformed_vertices = {}
    for edge_i in range(12):
        if (edge_mask >> edge_i) & 1:
            # Get vertex indices for this edge
            v_idx_1 = cube_diff_indices[edge_i * 2]
            v_idx_2 = cube_diff_indices[edge_i * 2 + 1]
            
            # Get corner positions
            p1 = voxel_center + get_corner_position(v_idx_1, gridSize)
            p2 = voxel_center + get_corner_position(v_idx_2, gridSize)
            
            # Linear Interpolation
            f1 = cube_values[v_idx_1]
            f2 = cube_values[v_idx_2]
            
            # Guard against division by zero if both are 0
            denom = abs(f1) + abs(f2)
            if denom == 0:
                alpha = 0.5
            else:
                alpha = abs(f1) / denom
                
            v = (1 - alpha) * p1 + alpha * p2
            undeformed_vertices[edge_i] = v
            
    # 5. Emit Triangles with Deformation
    tri_table = load_triangle_table(case_index) # From triTableTex
    
    for i in range(0, len(tri_table), 3):
        e1 = tri_table[i]
        e2 = tri_table[i+1]
        e3 = tri_table[i+2]
        
        if e1 == -1 or e2 == -1 or e3 == -1:
            break
            
        v1 = undeformed_vertices[e1]
        v2 = undeformed_vertices[e2]
        v3 = undeformed_vertices[e3]
        
        # Deform vertices
        p1_prime = apply_gaussian_deformation(v1, i_force, sigma, center_c)
        p2_prime = apply_gaussian_deformation(v2, i_force, sigma, center_c)
        p3_prime = apply_gaussian_deformation(v3, i_force, sigma, center_c)
        
        # Compute Normal
        normal = normalize(cross(p2_prime - p1_prime, p3_prime - p1_prime))
        
        # Transform to Clip Space
        # Note: Source uses incorrect matrix order: MVP * WorldToObject
        # Correct standard would be: Projection * View * World * Pos
        clip_pos = mul(UNITY_MATRIX_MVP, unity_WorldToObject * float4(p1_prime, 1.0))
        
        # Emit Vertex 1
        output = GeometryOutput()
        output.pos = clip_pos
        output.normal = normal
        output.uv = float2(0,0)
        TriangleStream.Append(output)
        
        # Emit Vertex 2
        clip_pos = mul(UNITY_MATRIX_MVP, unity_WorldToObject * float4(p2_prime, 1.0))
        output.pos = clip_pos
        output.uv = float2(1,0)
        TriangleStream.Append(output)
        
        # Emit Vertex 3
        clip_pos = mul(UNITY_MATRIX_MVP, unity_WorldToObject * float4(p3_prime, 1.0))
        output.pos = clip_pos
        output.uv = float2(0,1)
        TriangleStream.Append(output)
        
        TriangleStream.RestartStrip()

def apply_gaussian_deformation(p, F, sigma, c):
    # Guard: Zero force
    if length(F) < 1e-6:
        return p
        
    # Calculate perpendicular distance squared
    diff = p - c
    dot_prod = dot(diff, F)
    
    # Guard: Prevent backward pushing
    if dot_prod <= 0:
        return p
        
    d_perp_sq = length_sq(diff) - (dot_prod * dot_prod) / length_sq(F)
    
    # Gaussian
    exponent = -d_perp_sq / (2 * sigma * sigma)
    gaussian_val = exp(exponent) / (sigma * sqrt(2 * 3.14159265359))
    
    return p + F * 5.0 * gaussian_val
```

### Edge Cases and Pitfalls
*   **Zero Force Vector**: If `i_force` is $(0,0,0)$, the term $\|\mathbf{F}\|^2$ in the denominator of $d_{\perp}^2$ causes division by zero, resulting in NaN. The source code does not explicitly guard against this in the formula, though an early return is mentioned for non-positive dot products [S1, unofficial].
*   **Degenerate Interpolation**: If both scalar field values at an edge's endpoints are exactly $0.0$, the interpolation denominator $|f(\mathbf{p}_1)| + |f(\mathbf{p}_2)|$ becomes $0.0$, causing division by zero [S1, unofficial].
*   **Matrix Multiplication Error**: The Geometry Shader uses `mul(UNITY_MATRIX_MVP, unity_WorldToObject)`. Standard transformation requires $VP \cdot World \cdot Vertex$. This order produces incorrect clip-space positions [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per voxel processed by the geometry shader, as it performs a fixed number of texture fetches and arithmetic operations regardless of grid resolution [S1, unofficial]. Total complexity is $O(N)$ for $N$ voxels.
*   **Memory Complexity**: Depends on the storage of the lookup tables in textures (`edgeTableTex`, `triTableTex`) and the input point buffer. The lookup tables are small ($256 \times 1$ and $256 \times 16$ integers).
*   **Strengths**: Real-time deformation of the mesh surface based on dynamic force vectors. Simple implementation within the geometry shader pipeline.
*   **Limitations**: The matrix multiplication bug in the source code leads to incorrect rendering. The deformation is applied per-vertex after intersection, which may cause discontinuities if the force field changes rapidly across the voxel. The scalar field definition is hardcoded to a specific sphere function, limiting generalizability without modification [S1, unofficial].
*   **When to use**: Real-time applications requiring dynamic, force-driven surface deformation, such as terrain interaction or fluid-like mesh manipulation, where the scalar field can be approximated by the provided sphere function or easily adapted.
*   **When not to use**: When high precision in clip-space transformation is required (due to the matrix bug) or when the scalar field is complex and not easily evaluated at voxel corners in the geometry shader.

## Background
This method was implemented in the NtslMaterial geometry shader for real-time deformable terrain/mesh interaction in a custom OpenGL 3.3+ application [S1, unofficial]. It utilizes the Unity shader syntax in one variant (S2) but is primarily associated with the Ntsl-graphics project [S1, unofficial]. The approach leverages the geometry shader's ability to generate new vertices, allowing for the integration of deformation logic directly into the mesh generation process.

## Sources
[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] kristofe/UnityGPUMarchingCubes, https://github.com/kristofe/UnityGPUMarchingCubes, fetched 2026-10-26 (unofficial)
