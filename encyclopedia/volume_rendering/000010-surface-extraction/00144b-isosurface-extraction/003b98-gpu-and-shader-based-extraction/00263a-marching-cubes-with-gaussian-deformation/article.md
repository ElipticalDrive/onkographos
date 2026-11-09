# Marching Cubes with Gaussian Deformation

Also called: Geometry Shader Marching Cubes, Marching Cubes Geometry Shader Variant, Marching Cubes with Brush Deformation

## Overview
This method extracts an isosurface from a scalar field using a Geometry Shader to perform the topology generation and mesh emission directly on the GPU, while applying a spatially varying Gaussian deformation to the resulting vertices [S1, unofficial]. It differs from standard GPU Marching Cubes by integrating a continuous deformation field driven by a user-provided force vector, allowing for real-time, shader-based manipulation of the extracted geometry without CPU-side mesh modification [S1, unofficial]. Unlike methods that rely on CPU-side topology generation or pure compute shader buffer accumulation, this approach leverages the geometry shader's primitive assembly capabilities to emit triangles directly, though it is constrained by the fixed-function nature of the geometry shader stage [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   **Voxel Grid Points:** A point cloud representing the centers of the voxel grid cells. Each point corresponds to a cube in the Marching Cubes algorithm [S1, unofficial].
*   **Scalar Field Parameters:** Center $\mathbf{c}$, radius $r$, and grid size $s$ defining the base implicit sphere [S1, unofficial].
*   **Deformation Force:** A uniform 3D vector $\mathbf{F}$ (i_force) defining the direction and magnitude of the Gaussian deformation [S1, unofficial].
*   **Lookup Tables:** Two 2D integer textures (`edgeTableTex`, `triTableTex`) containing the standard 256-entry Marching Cubes edge and triangle configuration tables [S1, unofficial].

**Outputs:**
*   **Triangle Mesh:** A stream of triangles emitted by the Geometry Shader, with vertices displaced by the Gaussian deformation [S1, unofficial].

### Data Structures

**Marching Cubes Lookup Tables:**
The standard lookup tables are stored in GPU textures for efficient random access.
*   `edgeTableTex`: `isampler2D`, size $256 \times 1$. Stores the edge mask for each of the 256 cube configurations. Accessed via `texelFetch(tex, ivec2(index, 0), 0).r` [S1, unofficial].
*   `triTableTex`: `isampler2D`, size $256 \times 192$ (or similar, depending on implementation, typically storing up to 5 triangles per cube). Stores the vertex indices for the triangles forming the isosurface for each cube configuration. Accessed similarly [S1, unofficial].

**Edge Vertex Mapping:**
A static array `cube_diff_indices` of size 24 maps the 12 edges of a cube to their two endpoint vertex indices.
*   Layout: `int cube_diff_indices[24]`
*   Values: `{0,1, 1,2, 2,3, 3,0, 4,5, 5,6, 6,7, 7,4, 0,4, 1,5, 2,6, 3,7}` [S1, unofficial].
*   Index $i$ (0-11) corresponds to edge $i$. The vertices for edge $i$ are at `cube_diff_indices[i*2]` and `cube_diff_indices[i*2+1]`.

**Shader Structures:**
*   **Vertex Shader Input:** Position in object space.
*   **Geometry Shader Input:** A single point primitive representing the cube center.
*   **Geometry Shader Output:** `TriangleStream` of vertices containing position, normal, and UV coordinates [S2, unofficial].

### Steps

1.  **Voxel Grid Generation (CPU):** Generate a grid of points covering the region of interest. The spacing between points is `gridSize` (typically 0.25f) [S1, unofficial]. Each point is passed to the GPU as a point primitive [S1, unofficial].

2.  **Vertex Shader:**
    *   Transform the input vertex position from object space to world space using `unity_ObjectToWorld` [S2, unofficial].
    *   Pass the normal and texture coordinates unchanged to the Geometry Shader [S2, unofficial].

3.  **Geometry Shader - Cube Index Calculation:**
    *   For each input point (representing a cube center), evaluate the scalar field $f(\mathbf{p})$ at the 8 corners of the cube.
    *   The scalar field is defined as:
        $$f(\mathbf{p}) = \|\mathbf{p} - \mathbf{c}\| - (r + s\sqrt{3})$$
        where $\mathbf{c} = (0, 0, -20)$, $r = 4.0$, and $s$ is the grid size [S1, unofficial].
    *   Construct the cube index: if $f(\mathbf{p}_{corner}) < 0.0$, set the corresponding bit in the index. The corner ordering follows the standard Marching Cubes convention [S1, unofficial].

4.  **Geometry Shader - Edge Interpolation:**
    *   Look up the edge table using the cube index. If the result is 0, discard the primitive (no intersection) [S1, unofficial].
    *   Iterate over the 12 edges. For each edge $i$ where the $i$-th bit of the edge table is set:
        *   Retrieve the two corner positions $\mathbf{p}_1$ and $\mathbf{p}_2$ using `cube_diff_indices` [S1, unofficial].
        *   Compute the interpolated vertex position $\mathbf{v}$ using linear interpolation:
            $$\mathbf{v} = (1 - \alpha)\mathbf{p}_1 + \alpha\mathbf{p}_2$$
            where
            $$\alpha = \frac{|f(\mathbf{p}_1)|}{|f(\mathbf{p}_1)| + |f(\mathbf{p}_2)|}$$
            [S1, unofficial].
        *   Store the interpolated position $\mathbf{v}$ in a temporary array `interpolated_vertices[12]` [S1, unofficial].

5.  **Geometry Shader - Triangle Generation and Deformation:**
    *   Look up the triangle table using the cube index [S1, unofficial].
    *   Iterate through the table in steps of 3. For each valid triangle vertex index $idx$:
        *   Retrieve the undeformed interpolated position $\mathbf{p} = \text{interpolated\_vertices}[idx]$ [S1, unofficial].
        *   Apply the Gaussian deformation to $\mathbf{p}$ to get $\mathbf{p}'$ (see Formula below) [S1, unofficial].
        *   Set the output vertex position to $\mathbf{p}'$.
        *   Emit the vertex [S1, unofficial].
    *   After emitting 3 vertices, call `EndPrimitive()` [S1, unofficial].

6.  **Fragment Shader:**
    *   Compute lighting using a normalized light vector $\mathbf{L}$ derived from the light position and the fragment's world position [S2, unofficial].
    *   Calculate diffuse lighting: $\text{diffuse} = \max(\mathbf{N} \cdot \mathbf{L}, 0)$ [S2, unofficial].
    *   Sample a sprite texture and multiply by the diffuse light intensity [S2, unofficial].

### Formulas

**Gaussian Deformation:**
The deformation displaces a vertex $\mathbf{p}$ along the force vector $\mathbf{F}$ based on its perpendicular distance from the force axis:
$$\mathbf{p}' = \mathbf{p} + \mathbf{F} \cdot 5 \cdot \mathcal{N}(\sigma, d_{\perp}^2)$$
where $\mathcal{N}(\sigma, x)$ is the Gaussian distribution:
$$\mathcal{N}(\sigma, x) = \frac{e^{-x / (2\sigma^2)}}{\sigma\sqrt{2\pi}}$$
and $d_{\perp}^2$ is the squared perpendicular distance from $\mathbf{p}$ to the line defined by $\mathbf{c}$ and $\mathbf{F}$:
$$d_{\perp}^2 = \|\mathbf{p} - \mathbf{c}\|^2 - \frac{((\mathbf{p} - \mathbf{c}) \cdot \mathbf{F})^2}{\|\mathbf{F}\|^2}$$
[Note: The source notes contain a typo in the dot product term, writing $(\mathbf{p} - \mathbf{c}) \cdot \mathbf{F}^2$. The standard projection formula requires the square of the dot product, i.e., $((\mathbf{p} - \mathbf{c}) \cdot \mathbf{F})^2$. This correction is applied here for mathematical validity.] [S1, unofficial].

**Parameters:**
*   `sigma`: Constant float, typically 0.8, controlling the width of the Gaussian falloff [S1, unofficial].
*   `i_force`: Uniform `vec3`, defining the deformation direction and magnitude [S1, unofficial].
*   `gridSize`: Uniform float, defining the voxel edge length. Typical value: 0.25f [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
def vertex_shader(position_obj):
    position_world = mul(unity_ObjectToWorld, position_obj)
    normal_world = normalize(mul(unity_ObjectToWorld_normal, normal_obj))
    return position_world, normal_world

# Geometry Shader
def geometry_shader(point_in):
    # point_in contains the cube center position in world space
    cube_center = point_in.position
    
    # 1. Evaluate scalar field at 8 corners
    # Corners are offset by +/- gridSize/2 from cube_center
    # Assuming standard MC corner ordering
    corners = [
        cube_center + vec3(-0.5, -0.5, -0.5) * gridSize,
        cube_center + vec3( 0.5, -0.5, -0.5) * gridSize,
        cube_center + vec3( 0.5,  0.5, -0.5) * gridSize,
        cube_center + vec3(-0.5,  0.5, -0.5) * gridSize,
        cube_center + vec3(-0.5, -0.5,  0.5) * gridSize,
        cube_center + vec3( 0.5, -0.5,  0.5) * gridSize,
        cube_center + vec3( 0.5,  0.5,  0.5) * gridSize,
        cube_center + vec3(-0.5,  0.5,  0.5) * gridSize
    ]
    
    field_values = [scalar_field(c) for c in corners]
    
    # 2. Compute cube index
    cube_index = 0
    for i, fv in enumerate(field_values):
        if fv < 0.0:
            cube_index |= (1 << i)
            
    # 3. Look up edge table
    edge_mask = texelFetch(edgeTableTex, ivec2(cube_index, 0), 0).r
    
    if edge_mask == 0:
        return # Discard primitive
        
    # 4. Interpolate vertices on edges
    interpolated_vertices = [None] * 12
    
    for i in range(12):
        if (edge_mask >> i) & 1:
            v1_idx = cube_diff_indices[i * 2]
            v2_idx = cube_diff_indices[i * 2 + 1]
            
            p1 = corners[v1_idx]
            p2 = corners[v2_idx]
            
            f1 = field_values[v1_idx]
            f2 = field_values[v2_idx]
            
            # Guard against division by zero if both are 0
            denom = abs(f1) + abs(f2)
            if denom == 0.0:
                alpha = 0.5
            else:
                alpha = abs(f1) / denom
                
            interpolated_vertices[i] = (1 - alpha) * p1 + alpha * p2
            
    # 5. Generate triangles
    # Look up triangle table
    tri_data = texelFetch(triTableTex, ivec2(cube_index, 0), 0) # Simplified access
    
    # Iterate through triangle table entries (typically 5 triangles max)
    # The table stores vertex indices. -1 indicates end of triangle or cube.
    # Standard implementation iterates 3 indices at a time.
    
    tri_idx = 0
    while True:
        i1 = tri_data[tri_idx]
        i2 = tri_data[tri_idx + 1]
        i3 = tri_data[tri_idx + 2]
        
        if i1 == -1:
            break
            
        # Apply deformation to each vertex of the triangle
        p1_deformed = gaussian_deform(interpolated_vertices[i1])
        p2_deformed = gaussian_deform(interpolated_vertices[i2])
        p3_deformed = gaussian_deform(interpolated_vertices[i3])
        
        # Emit vertices
        emit_vertex(p1_deformed)
        emit_vertex(p2_deformed)
        emit_vertex(p3_deformed)
        end_primitive()
        
        tri_idx += 3

def scalar_field(p):
    c = vec3(0, 0, -20)
    r = 4.0
    s = gridSize
    return length(p - c) - (r + s * sqrt(3.0))

def gaussian_deform(p):
    F = i_force
    c = vec3(0, 0, -20)
    sigma = 0.8
    
    # Guard against zero force
    if length(F) == 0.0:
        return p
        
    diff = p - c
    dist_sq = dot(diff, diff)
    proj_sq = (dot(diff, F) ** 2) / dot(F, F)
    d_perp_sq = dist_sq - proj_sq
    
    # Gaussian
    gaussian_val = exp(-d_perp_sq / (2 * sigma * sigma)) / (sigma * sqrt(2 * 3.14159265))
    
    return p + F * 5.0 * gaussian_val

def emit_vertex(position):
    # Transform to clip space
    # Note: Source [S2, unofficial] uses mul(UNITY_MATRIX_MVP, unity_WorldToObject) which is likely incorrect.
    # Standard is mul(UNITY_MATRIX_VP, mul(unity_ObjectToWorld, position))
    # Here we assume correct MVP transformation for output
    clip_pos = mul(UNITY_MATRIX_MVP, position)
    output.pos = clip_pos
    output.normal = normalize(position) # Approximation
    output.uv = vec2(0,0) # Placeholder
    TriangleStream.Append(output)
```

### Edge Cases and Pitfalls
*   **Zero Force Vector:** If $\mathbf{F}$ is zero, the term $\|\mathbf{F}\|^2$ in the denominator of $d_{\perp}^2$ causes division by zero, resulting in NaN positions. The source code does not explicitly guard against this, though it mentions an early return if the dot product is non-positive [S1, unofficial]. Implementers must add a check: `if length(F) < epsilon: return p`.
*   **Zero Field Values:** If both endpoints of an edge have a field value of exactly 0.0, the interpolation denominator $|f(\mathbf{p}_1)| + |f(\mathbf{p}_2)|$ becomes 0.0, causing division by zero. A guard such as `if denom == 0: alpha = 0.5` is required [S1, unofficial].
*   **Matrix Multiplication Order:** The source [S2, unofficial] uses `mul(UNITY_MATRIX_MVP, unity_WorldToObject)` in the Geometry Shader. This is mathematically incorrect for transforming world-space positions to clip space. The correct order is `UNITY_MATRIX_MVP * world_position` or `UNITY_MATRIX_VP * (unity_ObjectToWorld * vertex)`. Using the source's order will produce incorrect rendering [S2, unofficial].
*   **Backward Pushing:** The source notes mention that if the dot product of the displacement vector and force is non-positive, deformation is skipped. This is handled by an early return in the deformation function [S1, unofficial].

## Operating characteristics

*   **Time Complexity:** $O(1)$ per voxel processed by the geometry shader, as it performs a fixed number of texture fetches and arithmetic operations. Total complexity is $O(N)$ for $N$ voxels [S1, unofficial].
*   **Memory Complexity:** Dominated by the storage of the voxel grid points and the lookup table textures. The lookup tables are small (256 entries). The voxel grid size depends on the resolution and region of interest.
*   **Performance:** Real-time performance is achievable on modern GPUs due to the parallel nature of the geometry shader. However, geometry shaders can be less efficient than compute shaders for large volumes due to fixed-function overhead and limited parallelism compared to compute shaders. The method is suitable for interactive applications where deformation is required [S1, unofficial].
*   **Strengths:**
    *   Real-time deformation integrated into the extraction pipeline.
    *   No CPU-GPU synchronization for mesh updates.
    *   Simple implementation using standard OpenGL/GLSL features.
*   **Limitations:**
    *   Geometry shader performance can be a bottleneck for very high-resolution grids.
    *   Limited flexibility compared to compute shader-based approaches (e.g., atomic counter emission) for complex topology changes.
    *   The matrix multiplication bug in the source [S2, unofficial] must be corrected for proper rendering.
*   **When to Use:** Suitable for real-time applications requiring deformable isosurfaces, such as terrain interaction or fluid simulation visualization, where the deformation field is relatively simple and the grid resolution is moderate [S1, unofficial].
*   **When Not to Use:** Not suitable for very high-resolution volumes where compute shaders would offer better performance, or for complex topological changes that require more flexible mesh management than the geometry shader provides.

## Background

The method is based on the classic Marching Cubes algorithm, which was introduced by Lorensen and Cline in 1987. The specific implementation described in the sources is a variant designed for real-time graphics using OpenGL 3.3+ and Geometry Shaders [S1, unofficial]. It was implemented in the NtslMaterial geometry shader for a custom application, demonstrating the feasibility of shader-based isosurface extraction with deformation [S1, unofficial]. The Gaussian deformation technique allows for smooth, localized modifications to the mesh, mimicking the effect of a "brush" or force field acting on the surface [S1, unofficial].

## Sources

[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] kristofe/UnityGPUMarchingCubes, https://github.com/kristofe/UnityGPUMarchingCubes, fetched 2026-10-26 (unofficial)
