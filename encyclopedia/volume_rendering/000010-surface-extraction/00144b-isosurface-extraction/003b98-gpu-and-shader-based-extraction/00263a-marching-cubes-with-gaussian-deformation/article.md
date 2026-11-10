# Marching Cubes with Gaussian Deformation

Also called: Geometry Shader Marching Cubes, Marching Cubes Geometry Shader Variant, Marching Cubes with Brush Deformation

## Overview
This method extracts an isosurface from a scalar field using a GPU geometry shader, applying a Gaussian-based deformation to the resulting mesh vertices along a user-defined force vector [S1, unofficial]. It differs from standard Marching Cubes by integrating the deformation directly into the primitive generation stage, allowing for real-time, procedural modification of the surface topology without pre-computing a deformed scalar field [S1, unofficial]. It builds on [GPU and Shader-Based Extraction](9036-gpu-and-shader-based-extraction.md) by utilizing the geometry shader's ability to generate new primitives and modify vertex attributes on the fly.

## How it works

### Inputs and Outputs
**Inputs:**
*   A point cloud representing the voxel grid centers, where each point corresponds to a cube in the scalar field [S1, unofficial].
*   A scalar field function $f(\mathbf{p})$ defined analytically (e.g., a sphere function) [S1, unofficial].
*   Uniform parameters: grid size ($s$), deformation force vector ($\mathbf{F}$), and Gaussian width ($\sigma$) [S1, unofficial].
*   Lookup tables for edge and triangle configurations stored in 2D integer textures [S1, unofficial].

**Outputs:**
*   A `TriangleStream` of deformed mesh vertices in clip space [S1, unofficial].

### Data Structures

**Lookup Tables:**
The standard Marching Cubes edge and triangle tables are stored in 2D textures to allow efficient GPU access via `texelFetch` [S1, unofficial].
*   `edgeTableTex`: Stores the 256x16 edge configuration table. Accessed as `texelFetch(edgeTableTex, ivec2(cubeIndex, 0), 0).r` [S1, unofficial].
*   `triTableTex`: Stores the 256x5 triangle configuration table. Accessed as `texelFetch(triTableTex, ivec2(cubeIndex, 0), 0).r` [S1, unofficial].

**Edge Vertex Mapping:**
A static array `cube_diff_indices` of size 24 maps the 12 cube edges to their two endpoint vertex indices [S1, unofficial]. The layout is:
`{0,1, 1,2, 2,3, 3,0, 4,5, 5,6, 6,7, 7,4, 0,4, 1,5, 2,6, 3,7}` [S1, unofficial].

**Shader Structures:**
*   **Vertex Shader Input/Output:** Position in object space, normal, and texture coordinates [S2, unofficial].
*   **Geometry Shader Input:** A single point (representing the cube center) [S1, unofficial].
*   **Geometry Shader Output:** A structure containing `pos` (clip space), `normal`, and `tex0` (UVs) [S2, unofficial].

### Formulas

**Scalar Field Evaluation:**
The scalar field value at a point $\mathbf{p}$ is computed as [S1, unofficial]:
$$ f(\mathbf{p}) = \|\mathbf{p} - \mathbf{c}\| - (r + s\sqrt{3}) $$
where $\mathbf{c} = (0, 0, -20)$ is the sphere center, $r = 4.0$ is the radius, and $s$ is the grid size (voxel edge length) [S1, unofficial].

**Linear Interpolation:**
The intersection point $\mathbf{v}$ on an edge between vertices $\mathbf{p}_1$ and $\mathbf{p}_2$ is calculated as [S1, unofficial]:
$$ \mathbf{v} = (1 - \alpha)\mathbf{p}_1 + \alpha\mathbf{p}_2 $$
where $\alpha = \frac{|f(\mathbf{p}_1)|}{|f(\mathbf{p}_1)| + |f(\mathbf{p}_2)|}$ [S1, unofficial].

**Gaussian Deformation:**
The undeformed vertex position $\mathbf{p}$ is displaced to $\mathbf{p}'$ by [S1, unofficial]:
$$ \mathbf{p}' = \mathbf{p} + \mathbf{F} \cdot 5 \cdot \mathcal{N}(\sigma, d_{\perp}^2) $$
where $\mathcal{N}(\sigma, x) = \frac{e^{-x / (2\sigma^2)}}{\sigma\sqrt{2\pi}}$ is the Gaussian distribution, and $d_{\perp}^2$ is the squared perpendicular distance from $\mathbf{p}$ to the force vector line passing through $\mathbf{c}$ [S1, unofficial]:
$$ d_{\perp}^2 = \|\mathbf{p} - \mathbf{c}\|^2 - \frac{((\mathbf{p} - \mathbf{c}) \cdot \mathbf{F})^2}{\|\mathbf{F}\|^2} $$
Note: The source notes contain a typo in the dot product term ($\mathbf{F}^2$ instead of $(\mathbf{p}-\mathbf{c})\cdot\mathbf{F}$), but the geometric definition of perpendicular distance requires the projection of the displacement vector onto the force vector [S1, unofficial].

### Parameters
*   `gridSize` ($s$): Voxel edge length. Typical value: 0.25 [S1, unofficial].
*   `sigma` ($\sigma$): Gaussian width controlling deformation falloff. Typical value: 0.8 [S1, unofficial].
*   `i_force` ($\mathbf{F}$): Uniform `vec3` defining the direction and magnitude of the deformation brush [S1, unofficial].
*   `lightPos`: Light position in world space for fragment shading [S2, unofficial].

### Steps

1.  **Vertex Shader:** Transform the input point position from object space to world space using `unity_ObjectToWorld`. Pass normal and texcoords unchanged [S2, unofficial].
2.  **Geometry Shader Entry:** For each input point (cube center), compute the cube index by evaluating $f(\mathbf{p})$ at the 8 corners of the cube defined by the grid size [S1, unofficial]. If $f(\mathbf{p}_{corner}) < 0.0$, set the corresponding bit in the cube index [S1, unofficial].
3.  **Edge Intersection:** Look up the edge table using the cube index. If the result is 0, discard the primitive [S1, unofficial]. Otherwise, iterate over the 12 edges. If the $i$-th bit is set, compute the interpolated vertex position $\mathbf{v}$ using linear interpolation [S1, unofficial].
4.  **Deformation:** For each interpolated vertex $\mathbf{v}$, apply the Gaussian deformation formula to obtain $\mathbf{v}'$ [S1, unofficial].
5.  **Triangle Generation:** Look up the triangle table using the cube index. Iterate through the table in steps of 3. For each valid triangle vertex index, retrieve the corresponding deformed position, set the output position, and emit the vertex [S1, unofficial].
6.  **Primitive Emission:** After emitting three vertices, end the primitive [S1, unofficial].
7.  **Fragment Shader:** Compute lighting using a normalized light vector $\mathbf{L}$ derived from `lightPos` and the input position. Calculate diffuse lighting as $\max(\mathbf{N} \cdot \mathbf{L}, 0)$ [S2, unofficial]. Sample a sprite texture and multiply by the diffuse light [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
function vertex_shader(position, normal, texcoord):
    world_pos = mul(unity_ObjectToWorld, position)
    return world_pos, normal, texcoord

# Geometry Shader
function geometry_shader(point):
    # 1. Compute Cube Index
    cube_index = 0
    for i in range(8):
        corner_pos = get_cube_corner(point.pos, i, gridSize)
        if scalar_field(corner_pos) < 0.0:
            cube_index |= (1 << i)
    
    # 2. Check Edge Table
    edge_config = texel_fetch(edge_table_tex, (cube_index, 0))
    if edge_config == 0:
        return # Discard
    
    # 3. Compute Intersections
    intersections = []
    for edge in range(12):
        if (edge_config >> edge) & 1:
            p1, p2 = get_edge_vertices(point.pos, edge, gridSize)
            f1 = scalar_field(p1)
            f2 = scalar_field(p2)
            
            # Handle degenerate case where both are 0
            if abs(f1) + abs(f2) == 0:
                continue
                
            alpha = abs(f1) / (abs(f1) + abs(f2))
            v = lerp(p1, p2, alpha)
            intersections.append(v)
    
    # 4. Generate Triangles
    tri_config = texel_fetch(tri_table_tex, (cube_index, 0))
    if tri_config == 0:
        return

    # Iterate through triangle table
    # Note: The table stores vertex indices for the 12 edges. 
    # We map these to the computed intersections.
    # The table format is typically a list of edge indices, terminated by -1.
    
    tri_idx = 0
    while tri_idx < 3:
        edge_idx = tri_config[tri_idx]
        if edge_idx == -1:
            break
        # Get the two vertices for this edge from the edge_diff_indices mapping
        # Actually, the tri_table usually stores vertex indices of the cube (0-7) 
        # or edge indices. The source notes imply mapping via cube_diff_indices.
        # Assuming standard MC table structure where entries are edge indices:
        
        # Retrieve the two edge endpoints for this triangle
        # This part depends on the specific table layout. 
        # Standard MC tables list 3 edge indices per triangle.
        
        # For this implementation, we assume the table provides indices into 
        # the 'intersections' array or directly into edge definitions.
        # Based on S1, we iterate the table and retrieve undeformed positions.
        
        # Let's assume the table gives us the 3 vertices of the triangle 
        # in terms of the 8 cube corners or edge intersections.
        # The source code snippet implies: "For each valid triangle vertex index, 
        # retrieve the undeformed position, apply deformation..."
        
        # Simplified logic based on S1 description:
        v1 = get_triangle_vertex(cube_index, tri_config, tri_idx, 0)
        v2 = get_triangle_vertex(cube_index, tri_config, tri_idx, 1)
        v3 = get_triangle_vertex(cube_index, tri_config, tri_idx, 2)
        
        # Apply Deformation
        v1_deformed = gaussian_deform(v1, point.pos, i_force, sigma)
        v2_deformed = gaussian_deform(v2, point.pos, i_force, sigma)
        v3_deformed = gaussian_deform(v3, point.pos, i_force, sigma)
        
        # Transform to Clip Space
        # Note: S2 notes a potential matrix error. Standard is MVP * WorldPos.
        # S1 uses OpenGL, S2 uses Unity. We follow S1's OpenGL context.
        clip_v1 = mul(UNITY_MATRIX_MVP, v1_deformed) # S2 syntax, adapted for S1 context
        clip_v2 = mul(UNITY_MATRIX_MVP, v2_deformed)
        clip_v3 = mul(UNITY_MATRIX_MVP, v3_deformed)
        
        emit_vertex(clip_v1)
        emit_vertex(clip_v2)
        emit_vertex(clip_v3)
        end_primitive()
        
        tri_idx += 3

# Helper Functions
function scalar_field(p):
    c = vec3(0, 0, -20)
    r = 4.0
    s = gridSize
    return length(p - c) - (r + s * sqrt(3))

function gaussian_deform(p, center, force, sigma):
    if length(force) == 0:
        return p # Guard against NaN
    
    # Perpendicular distance squared
    diff = p - center
    proj = dot(diff, force) / length(force)
    d_perp_sq = length(diff) - proj # Note: S1 formula has a typo in the dot product term. 
                                     # Correct geometric projection is (diff . F)^2 / |F|^2.
                                     # The source notes: d_perp^2 = ||p-c||^2 - ((p-c).F^2)/||F||^2
                                     # We implement the corrected geometric version.
    d_perp_sq = length(diff) - (dot(diff, force)**2) / length(force)
    
    # Gaussian
    gaussian_val = exp(-d_perp_sq / (2 * sigma**2)) / (sigma * sqrt(2 * pi))
    
    return p + force * 5 * gaussian_val

function get_cube_corner(center, i, gridSize):
    # Returns the 8 corners of the cube centered at 'center' with edge length 'gridSize'
    # Bit i determines the corner
    x = (i & 1) * gridSize - gridSize/2
    y = (i & 2) * gridSize/2 - gridSize/2 # Corrected bit shift for y
    z = (i & 4) * gridSize/4 - gridSize/2 # Corrected bit shift for z
    return center + vec3(x, y, z)
```

### Edge Cases and Pitfalls
*   **Zero Force Vector:** If $\mathbf{F}$ is zero, the division by $\|\mathbf{F}\|$ in the deformation formula causes a NaN. The source code does not explicitly guard against this, so an implementation should check `length(force) == 0` [S1, unofficial].
*   **Degenerate Interpolation:** If both scalar field values at an edge's endpoints are exactly 0.0, the interpolation denominator becomes 0.0, causing a division by zero. This case should be handled by skipping the edge or defaulting to one endpoint [S1, unofficial].
*   **Matrix Multiplication Order:** In Unity-based implementations, the geometry shader may incorrectly use `mul(UNITY_MATRIX_MVP, unity_WorldToObject)`. The standard transformation is `VP * World * Vertex`. Using the inverse order produces incorrect clip-space positions [S2, unofficial].
*   **Backward Pushing:** If the dot product of the displacement vector and force is non-positive, the deformation is skipped to prevent backward pushing. This is handled by an early return in the deformation logic [S1, unofficial].

## Operating characteristics

*   **Time Complexity:** $O(1)$ per voxel processed by the geometry shader, as it performs a fixed number of texture fetches and arithmetic operations regardless of grid resolution. Total complexity is $O(N)$ for $N$ voxels [S1, unofficial].
*   **Memory Complexity:** Depends on the storage of the lookup tables in textures (constant size) and the input point cloud ($O(N)$) [S1, unofficial].
*   **Strengths:** Real-time deformation of isosurfaces without pre-computation; efficient GPU-based implementation using geometry shaders [S1, unofficial].
*   **Limitations:** Limited to specific scalar field definitions (e.g., sphere function in the source); potential for numerical instability with zero force or degenerate edges; matrix transformation errors in certain engine contexts [S1, S2, unofficial].
*   **When to Use:** Real-time deformable terrain or mesh interaction where a Gaussian brush effect is desired [S1, unofficial].
*   **When Not to Use:** When high-precision scalar field evaluation is required for complex geometries, or when the force vector might be zero without proper guarding [S1, unofficial].

## Background

*   **Origin:** This method was implemented in the NtslMaterial geometry shader for real-time deformable terrain/mesh interaction in a custom OpenGL 3.3+ application [S1, unofficial].
*   **History:** The implementation is part of the `ntsl-graphics` project, which explores shader-based graphics techniques [S1, unofficial]. A related Unity-based implementation exists in `UnityGPUMarchingCubes`, though it focuses more on sprite rendering and may contain matrix transformation errors [S2, unofficial].
*   **Usage:** Used in custom OpenGL applications for interactive 3D modeling and terrain deformation [S1, unofficial].

## Sources

[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] kristofe/UnityGPUMarchingCubes, https://github.com/kristofe/UnityGPUMarchingCubes, fetched 2026-10-26 (unofficial)
