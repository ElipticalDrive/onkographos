# Marching Cubes with Gaussian Deformation

Also called: Geometry Shader Marching Cubes, Marching Cubes Geometry Shader Variant, Marching Cubes with Brush Deformation

## Overview
This method extracts an isosurface from a scalar field using the Marching Cubes algorithm, but performs the extraction and subsequent geometric deformation entirely within a GPU geometry shader [S1, S2, unofficial]. It differs from standard CPU-based or vertex-shader-only implementations by generating the final triangle mesh primitives directly from point primitives representing voxel grid centers, applying a Gaussian-based deformation to the surface vertices in real-time [S1, unofficial]. This allows for dynamic, force-driven deformation of the mesh during rendering without requiring CPU-side mesh regeneration [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A point cloud representing the voxel grid centers. Each point corresponds to a voxel in the 3D grid [S1, unofficial].
*   Scalar field parameters: center $\mathbf{c}$, radius $r$, grid size $s$ (voxel edge length) [S1, unofficial].
*   Deformation parameters: force vector $\mathbf{F}$, Gaussian width $\sigma$ [S1, unofficial].
*   Rendering matrices: Model-View-Projection matrix ($M_{VP}$), World-to-Object matrix ($W_{obj}$) [S2, unofficial].
*   Lookup tables: Edge table and Triangle table stored in 2D integer textures [S1, unofficial].

**Outputs:**
*   A `TriangleStream` of 3D triangles in clip space [S2, unofficial].
*   Per-vertex attributes: position, normal, and texture coordinates passed to the fragment shader [S2, unofficial].

### Data Structures
*   **Lookup Tables:** The standard Marching Cubes edge and triangle tables are stored in `isampler2D` textures (`edgeTableTex`, `triTableTex`) [S1, unofficial]. They are accessed via `texelFetch(tex, ivec2(index, 0), 0).r` [S1, unofficial].
*   **Edge Vertex Mapping:** A static array `cube_diff_indices` of size 24 maps the 12 cube edges to their two endpoint vertex indices [S1, unofficial]. The layout is:
    ```python
    cube_diff_indices = [
        0, 1, 1, 2, 2, 3, 3, 0,  # Bottom face edges
        4, 5, 5, 6, 6, 7, 7, 4,  # Top face edges
        0, 4, 1, 5, 2, 6, 3, 7   # Vertical edges
    ]
    ```
*   **Shader Structures:**
    *   Vertex Shader Output / Geometry Shader Input: `float4 pos`, `float3 normal`, `float2 tex0` [S2, unofficial].
    *   Geometry Shader Output: `float4 pos`, `float3 normal`, `float2 uv` [S2, unofficial].

### Steps

1.  **Voxel Evaluation (Vertex Shader):**
    For each input point primitive (representing a voxel center $\mathbf{p}_{center}$), the vertex shader transforms the position to world space [S2, unofficial]. It does not compute the cube index here; this is deferred to the geometry shader to allow access to all 8 corners of the voxel cell [S1, unofficial].

2.  **Cube Index Calculation (Geometry Shader):**
    The geometry shader receives one point primitive per voxel. It computes the coordinates of the 8 corners of the cube surrounding the input point. For a voxel at grid position $(i, j, k)$ with grid size $s$, the corners are at $\mathbf{p}_{center} + \frac{s}{2}(\pm 1, \pm 1, \pm 1)$ [S1, unofficial].
    Evaluate the scalar field $f(\mathbf{p})$ at each corner:
    $$ f(\mathbf{p}) = \|\mathbf{p} - \mathbf{c}\| - (r + s\sqrt{3}) $$
    where $\mathbf{c} = (0, 0, -20)$, $r = 4.0$, and $s$ is the grid size [S1, unofficial].
    Construct the 8-bit cube index: if $f(\mathbf{p}_{corner}) < 0.0$, set the corresponding bit [S1, unofficial].

3.  **Topology Lookup:**
    Use the cube index to fetch the edge mask from `edgeTableTex` and the triangle configuration from `triTableTex` [S1, unofficial]. If the edge mask is 0, discard the primitive [S1, unofficial].

4.  **Vertex Interpolation:**
    Iterate over the 12 edges of the cube. For each edge $i$ where the $i$-th bit of the edge mask is set:
    *   Retrieve the two endpoint vertex indices $v_1, v_2$ from `cube_diff_indices` [S1, unofficial].
    *   Get the world positions $\mathbf{p}_1, \mathbf{p}_2$ of these corners.
    *   Evaluate $f(\mathbf{p}_1)$ and $f(\mathbf{p}_2)$.
    *   Compute the interpolation factor $\alpha$:
        $$ \alpha = \frac{|f(\mathbf{p}_1)|}{|f(\mathbf{p}_1)| + |f(\mathbf{p}_2)|} $$
    *   Compute the undeformed intersection point $\mathbf{v}$:
        $$ \mathbf{v} = (1 - \alpha)\mathbf{p}_1 + \alpha\mathbf{p}_2 $$
    *   Store $\mathbf{v}$ for the triangle generation step [S1, unofficial].

5.  **Gaussian Deformation:**
    For each generated vertex $\mathbf{v}$, apply the Gaussian deformation to obtain $\mathbf{p}'$ [S1, unofficial]:
    $$ \mathbf{p}' = \mathbf{v} + \mathbf{F} \cdot 5 \cdot \mathcal{N}(\sigma, d_{\perp}^2) $$
    where:
    $$ \mathcal{N}(\sigma, x) = \frac{e^{-x / (2\sigma^2)}}{\sigma\sqrt{2\pi}} $$
    $$ d_{\perp}^2 = \|\mathbf{v} - \mathbf{c}\|^2 - \frac{((\mathbf{v} - \mathbf{c}) \cdot \mathbf{F})^2}{\|\mathbf{F}\|^2} $$
    Here, $\mathbf{F}$ is the uniform force vector, $\sigma = 0.8$ is the falloff width, and $\mathbf{c}$ is the field center [S1, unofficial].
    *   **Guard:** If $\|\mathbf{F}\| = 0$, the deformation is skipped to avoid division by zero [S1, unofficial].
    *   **Guard:** If $((\mathbf{v} - \mathbf{c}) \cdot \mathbf{F}) \le 0$, the deformation is skipped to prevent backward pushing [S1, unofficial].

6.  **Triangle Emission:**
    Iterate through the triangle table entries for the current cube index in steps of 3 [S1, unofficial]. For each triangle defined by three vertex indices:
    *   Retrieve the three deformed vertices $\mathbf{p}'_1, \mathbf{p}'_2, \mathbf{p}'_3$.
    *   Compute the normal $\mathbf{n}$ for the triangle:
        $$ \mathbf{n} = \text{normalize}((\mathbf{p}'_2 - \mathbf{p}'_1) \times (\mathbf{p}'_3 - \mathbf{p}'_1)) $$
    *   Transform the position to clip space. Note: The source implementation uses an unusual matrix multiplication order `mul(UNITY_MATRIX_MVP, unity_WorldToObject)` which is likely incorrect for standard MVP transformation (typically $M_{VP} \cdot \mathbf{p}'$) [S2, unofficial]. Assuming standard transformation, the clip-space position is $\mathbf{p}_{clip} = M_{VP} \cdot \mathbf{p}'$.
    *   Emit the vertex with position $\mathbf{p}_{clip}$, normal $\mathbf{n}$, and UV $(0,0)$ [S2, unofficial].
    *   After emitting three vertices, call `RestartStrip()` [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
SIGMA = 0.8
CENTER = (0, 0, -20)
RADIUS = 4.0
GRID_SIZE = 0.25

# Lookup tables (simplified representation)
# edge_table and tri_table are 2D textures accessed via texelFetch
CUBE_DIFF_INDICES = [
    0, 1, 1, 2, 2, 3, 3, 0,
    4, 5, 5, 6, 6, 7, 7, 4,
    0, 4, 1, 5, 2, 6, 3, 7
]

def gaussian_deformation(p, F, sigma, c):
    """
    Applies Gaussian deformation to point p along force F.
    """
    if length(F) == 0:
        return p
    
    diff = p - c
    dot_prod = dot(diff, F)
    
    # Guard against backward pushing
    if dot_prod <= 0:
        return p
        
    # Calculate perpendicular distance squared
    # d_perp^2 = ||p-c||^2 - ((p-c) . F)^2 / ||F||^2
    norm_F_sq = dot(F, F)
    d_perp_sq = dot(diff, diff) - (dot_prod * dot_prod) / norm_F_sq
    
    # Gaussian kernel
    # N(sigma, x) = exp(-x / (2*sigma^2)) / (sigma * sqrt(2*pi))
    exponent = -d_perp_sq / (2 * sigma * sigma)
    kernel = exp(exponent) / (sigma * sqrt(2 * PI))
    
    return p + F * 5.0 * kernel

def scalar_field(p, c, r, s):
    """
    Evaluates the sphere-based scalar field.
    """
    dist = length(p - c)
    return dist - (r + s * sqrt(3))

def interpolate_vertex(p1, p2, f1, f2):
    """
    Linear interpolation on an edge.
    """
    denom = abs(f1) + abs(f2)
    if denom == 0:
        return (p1 + p2) / 2.0 # Guard against division by zero
    alpha = abs(f1) / denom
    return (1 - alpha) * p1 + alpha * p2

def process_voxel(point_pos, F, M_VP):
    """
    Geometry shader entry point for one voxel.
    point_pos: center of the voxel in world space.
    """
    s = GRID_SIZE
    half_s = s / 2.0
    
    # Define 8 corners relative to center
    corners = []
    for x in [-1, 1]:
        for y in [-1, 1]:
            for z in [-1, 1]:
                corners.append(point_pos + vec3(x * half_s, y * half_s, z * half_s))
    
    # Evaluate scalar field at corners
    field_vals = []
    for p in corners:
        field_vals.append(scalar_field(p, CENTER, RADIUS, s))
    
    # Compute cube index
    cube_index = 0
    for i, f in enumerate(field_vals):
        if f < 0.0:
            cube_index |= (1 << i)
            
    # Fetch edge mask
    edge_mask = texelFetch(edgeTableTex, ivec2(cube_index, 0), 0).r
    
    if edge_mask == 0:
        return # Discard primitive
        
    # Compute intersection points for active edges
    intersections = {}
    for edge_i in range(12):
        if (edge_mask >> edge_i) & 1:
            v1_idx = CUBE_DIFF_INDICES[edge_i * 2]
            v2_idx = CUBE_DIFF_INDICES[edge_i * 2 + 1]
            
            p1 = corners[v1_idx]
            p2 = corners[v2_idx]
            f1 = field_vals[v1_idx]
            f2 = field_vals[v2_idx]
            
            v_undeformed = interpolate_vertex(p1, p2, f1, f2)
            v_deformed = gaussian_deformation(v_undeformed, F, SIGMA, CENTER)
            
            # Transform to clip space
            v_clip = M_VP * vec4(v_deformed, 1.0)
            
            # Store for triangle generation
            intersections[edge_i] = (v_deformed, v_clip)
            
    # Fetch triangle configuration
    tri_config = texelFetch(triTableTex, ivec2(cube_index, 0), 0).r # Simplified: assumes single row or specific layout
    
    # Generate triangles
    # Note: The actual table layout in the source is a 2D texture where each row might contain multiple triangles
    # or the index maps to a specific offset. Based on standard MC, we iterate 3 vertices at a time.
    # Assuming tri_config contains the list of vertex indices for all triangles in this cube case.
    
    # Pseudocode assumes we can iterate through the triangle list for this cube_index
    # In the source, triTableTex is likely indexed by cube_index and a triangle offset.
    # For implementation clarity, we assume a helper function get_triangles(cube_index) returns a list of (v1, v2, v3) edge indices.
    
    triangles = get_triangles_from_texture(triTableTex, cube_index)
    
    for tri in triangles:
        v1_data = intersections[tri[0]]
        v2_data = intersections[tri[1]]
        v3_data = intersections[tri[2]]
        
        p1_clip, p2_clip, p3_clip = v1_data[1], v2_data[1], v3_data[1]
        p1_world, p2_world, p3_world = v1_data[0], v2_data[0], v3_data[0]
        
        # Compute normal
        normal = normalize(cross(p2_world - p1_world, p3_world - p1_world))
        
        # Emit vertices
        emit_vertex(p1_clip, normal, vec2(0,0))
        emit_vertex(p2_clip, normal, vec2(1,0))
        emit_vertex(p3_clip, normal, vec2(0,1))
        restart_strip()
```

### Parameters
*   `gridSize`: Float, voxel edge length. Typical value: 0.25 [S1, unofficial].
*   `i_force`: Uniform `vec3`, deformation direction and magnitude [S1, unofficial].
*   `sigma`: Constant float, Gaussian width. Value: 0.8 [S1, unofficial].
*   `lightPos`: Light position in world space (used in fragment shader) [S2, unofficial].

### Edge Cases and Pitfalls
*   **Zero Force:** If $\mathbf{F} = \mathbf{0}$, the deformation formula divides by $\|\mathbf{F}\|$, causing NaN. The code must guard against this [S1, unofficial].
*   **Backward Pushing:** If the dot product of the displacement vector and force is non-positive, deformation is skipped to prevent artifacts [S1, unofficial].
*   **Division by Zero in Interpolation:** If both scalar field values at an edge's endpoints are exactly 0.0, the interpolation denominator is 0.0. This degenerate case is not explicitly handled in the source code [S1, unofficial].
*   **Matrix Multiplication Error:** The Unity variant uses `mul(UNITY_MATRIX_MVP, unity_WorldToObject)` which is mathematically incorrect for transforming vertices to clip space (should be `UNITY_MATRIX_MVP * vertex`) [S2, unofficial].

## Operating characteristics

*   **Time Complexity:** $O(1)$ per voxel processed by the geometry shader, as it performs a fixed number of texture fetches and arithmetic operations. Total complexity is $O(N)$ for $N$ voxels [S1, unofficial].
*   **Memory Complexity:** Low. The algorithm operates on point primitives and generates triangles on the fly. Lookup tables are stored in GPU texture memory [S1, unofficial].
*   **Strengths:** Real-time deformation of the isosurface without CPU-GPU synchronization overhead for mesh generation. Suitable for dynamic interactions like terrain deformation [S1, unofficial].
*   **Limitations:** The Gaussian deformation is applied per-vertex, which may cause discontinuities if adjacent voxels generate vertices that are deformed differently (though they share the same world position, the deformation depends on the force field which is global, so continuity is maintained if the force field is smooth). The matrix multiplication bug in the Unity variant leads to incorrect rendering [S2, unofficial].
*   **When to use:** Real-time applications requiring dynamic, force-driven mesh deformation from a scalar field, such as interactive terrain or fluid-like surface manipulation.
*   **When not to use:** High-fidelity static mesh generation where CPU-side optimization (e.g., dual contouring, QEF) is preferred. Applications requiring precise control over the deformation field beyond a simple Gaussian brush.

## Background

This method was implemented in the NtslMaterial geometry shader for a custom OpenGL 3.3+ application, demonstrating real-time deformable terrain/mesh interaction [S1, unofficial]. It is a variant of the classic Marching Cubes algorithm [Lorensen and Cline, 1987], adapted for GPU geometry shaders to allow for dynamic deformation. The Gaussian deformation technique is inspired by brush-based modeling techniques, where a force vector influences the surface geometry based on a Gaussian falloff [S1, unofficial].

## Sources

[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] kristofe/UnityGPUMarchingCubes, https://github.com/kristofe/UnityGPUMarchingCubes, fetched 2026-10-26 (unofficial)
