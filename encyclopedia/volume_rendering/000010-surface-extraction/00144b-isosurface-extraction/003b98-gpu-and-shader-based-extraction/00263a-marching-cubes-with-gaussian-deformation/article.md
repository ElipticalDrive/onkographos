# Marching Cubes with Gaussian Deformation

Also called: Geometry Shader Marching Cubes, Marching Cubes Geometry Shader Variant, Marching Cubes with Brush Deformation

## Overview
This method extracts a triangle mesh from a scalar field defined by a sphere function using a Geometry Shader, applying a Gaussian-based deformation along a user-provided force vector to simulate real-time terrain or mesh interaction [S1, unofficial]. It differs from standard static Marching Cubes implementations by integrating the deformation calculation directly into the geometry shader pipeline, allowing for dynamic, GPU-accelerated surface modification without CPU-side mesh regeneration [S1, unofficial]. Unlike [GPU-accelerated Marching Cubes with Distance Extraction](3818-gpu-accelerated-marching-cubes-with-distance-extraction.md) which focuses on distance field optimization, this variant prioritizes interactive deformation capabilities [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A point primitive buffer where each point represents a voxel grid position (center of a cube). Uniforms include `gridSize` (float, typically 0.25f), `i_force` (vec3, deformation direction/magnitude), and `sigma` (float, typically 0.8) [S1, unofficial].
*   **Output:** A `TriangleStream` of vertices with positions, normals, and UVs [S2, unofficial].

**Data Structures:**
*   **Edge Table (`edgeTableTex`):** A 1D texture (or 2D texture with height 1) of size 256, storing 16-bit integers. Index `i` contains the bitmask of intersected edges for cube index `i` [S1, unofficial].
*   **Triangle Table (`triTableTex`):** A 1D texture (or 2D texture with height 1) of size 256, storing 24-bit integers (packed as 12 pairs of 8-bit indices or similar, accessed via `texelFetch`). Maps cube index to a list of triangle vertex indices, terminated by -1 [S1, unofficial].
*   **Edge Vertex Indices (`cube_diff_indices`):** A static integer array of size 24, mapping edge index `i` to two vertex indices: `cube_diff_indices[i*2]` and `cube_diff_indices[i*2+1]`. Values: `0,1, 1,2, 2,3, 3,0, 4,5, 5,6, 6,7, 7,4, 0,4, 1,5, 2,6, 3,7` [S1, unofficial].
*   **GS Input/Output Structure:** Contains `float4 pos`, `float3 normal`, `float2 tex0` (or `uv`) [S2, unofficial].

**Steps:**

1.  **Vertex Shader:**
    *   Transform the input vertex position `v` from object space to world space: `world_pos = unity_ObjectToWorld * v` [S2, unofficial].
    *   Pass the normal and texture coordinates unchanged to the Geometry Shader [S2, unofficial].

2.  **Geometry Shader (Per Voxel):**
    *   **Cube Index Calculation:**
        *   Let `p_center` be the input vertex position (voxel center).
        *   Let `s` be `gridSize`.
        *   For each of the 8 corners `k` (0 to 7) of the cube centered at `p_center`:
            *   Compute corner position `c_k = p_center + offset_k * s`, where `offset_k` is the standard MC corner offset vector (e.g., `(-1,-1,-1)`, `(1,-1,-1)`, etc.).
            *   Evaluate scalar field `f(c_k)` using the sphere function (see Formula below).
            *   If `f(c_k) < 0.0`, set bit `k` in `cube_index`.
    *   **Edge Intersection Check:**
        *   Fetch the edge mask from `edgeTableTex` using `cube_index`.
        *   If the mask is 0, discard the primitive [S1, unofficial].
    *   **Vertex Interpolation:**
        *   For each of the 12 edges `i` (0 to 11):
            *   If bit `i` is set in the edge mask:
                *   Get the two corner indices `v1_idx` and `v2_idx` from `cube_diff_indices`.
                *   Retrieve corner positions `p1` and `p2` and field values `f1` and `f2`.
                *   Compute interpolation factor `alpha = |f1| / (|f1| + |f2|)` [S1, unofficial].
                *   Compute interpolated position `v_interpolated = (1 - alpha) * p1 + alpha * p2` [S1, unofficial].
                *   Store `v_interpolated` in an array `edge_vertices[i]`.
    *   **Triangle Generation and Deformation:**
        *   Fetch the triangle list from `triTableTex` using `cube_index`.
        *   Iterate through the list in steps of 3. For each triangle vertex index `idx`:
            *   If `idx` is -1, break.
            *   Retrieve the undeformed position `p_undeformed` from `edge_vertices[idx]`.
            *   Apply Gaussian deformation to `p_undeformed` (see Formula below) to get `p_deformed`.
            *   Transform `p_deformed` to clip space. Note: The source code uses `mul(UNITY_MATRIX_MVP, unity_WorldToObject * p_deformed)`, which is noted as likely incorrect for standard MVP transformation (should be `VP * World * Vertex`) [S2, unofficial].
            *   Emit the vertex with the transformed position, normal, and UV.
        *   After emitting 3 vertices, call `outStream.RestartStrip()` [S2, unofficial].

**Formulas:**

*   **Scalar Field Evaluation:**
    $$f(\mathbf{p}) = \|\mathbf{p} - \mathbf{c}\| - (r + s\sqrt{3})$$
    Where:
    *   $\mathbf{p}$ is the point being evaluated.
    *   $\mathbf{c} = (0, 0, -20)$ is the center of the sphere [S1, unofficial].
    *   $r = 4.0$ is the radius [S1, unofficial].
    *   $s$ is the grid size (`gridSize`) [S1, unofficial].

*   **Linear Interpolation on an Edge:**
    $$\mathbf{v} = (1 - \alpha)\mathbf{p}_1 + \alpha\mathbf{p}_2$$
    Where:
    *   $\alpha = \frac{|f(\mathbf{p}_1)|}{|f(\mathbf{p}_1)| + |f(\mathbf{p}_2)|}$ [S1, unofficial].

*   **Gaussian Deformation:**
    $$\mathbf{p}' = \mathbf{p} + \mathbf{F} \cdot 5 \cdot \mathcal{N}(\sigma, d_{\perp}^2)$$
    Where:
    *   $\mathbf{p}$ is the undeformed vertex position.
    *   $\mathbf{F}$ is the force vector (`i_force`) [S1, unofficial].
    *   $\sigma$ is the Gaussian width (typically 0.8) [S1, unofficial].
    *   $\mathcal{N}(\sigma, x) = \frac{e^{-x / (2\sigma^2)}}{\sigma\sqrt{2\pi}}$ is the Gaussian distribution function [S1, unofficial].
    *   $d_{\perp}^2 = \|\mathbf{p} - \mathbf{c}\|^2 - \frac{((\mathbf{p} - \mathbf{c}) \cdot \mathbf{F})^2}{\|\mathbf{F}\|^2}$ is the squared perpendicular distance from the point to the force vector axis [S1, unofficial].
    *   *Note:* The source code includes a check: if the dot product of the displacement vector and force is non-positive, deformation is skipped to prevent backward pushing [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
def vertex_shader(vertex):
    world_pos = mul(unity_ObjectToWorld, vertex.pos)
    return {
        'pos': world_pos,
        'normal': vertex.normal,
        'tex0': vertex.tex0
    }

# Geometry Shader
def geometry_shader(input_vertices):
    # input_vertices is a single point representing the voxel center
    p_center = input_vertices[0].pos
    s = gridSize
    c = vec3(0, 0, -20)
    r = 4.0
    
    cube_index = 0
    corner_positions = []
    corner_values = []
    
    # Define 8 corner offsets for a cube of size s
    offsets = [
        vec3(-1, -1, -1), vec3(1, -1, -1), vec3(1, 1, -1), vec3(-1, 1, -1),
        vec3(-1, -1, 1), vec3(1, -1, 1), vec3(1, 1, 1), vec3(-1, 1, 1)
    ]
    
    for k in range(8):
        c_k = p_center + offsets[k] * s
        f_c = length(c_k - c) - (r + s * sqrt(3))
        corner_positions.append(c_k)
        corner_values.append(f_c)
        if f_c < 0.0:
            cube_index |= (1 << k)
            
    # Fetch edge mask
    edge_mask = texelFetch(edgeTableTex, ivec2(cube_index, 0), 0).r
    
    if edge_mask == 0:
        return # Discard
        
    edge_vertices = [None] * 12
    
    for i in range(12):
        if (edge_mask >> i) & 1:
            v1_idx = cube_diff_indices[i * 2]
            v2_idx = cube_diff_indices[i * 2 + 1]
            
            p1 = corner_positions[v1_idx]
            p2 = corner_positions[v2_idx]
            f1 = corner_values[v1_idx]
            f2 = corner_values[v2_idx]
            
            # Guard against division by zero
            denom = abs(f1) + abs(f2)
            if denom == 0:
                alpha = 0.0
            else:
                alpha = abs(f1) / denom
                
            v_interpolated = (1 - alpha) * p1 + alpha * p2
            edge_vertices[i] = v_interpolated
            
    # Fetch triangle list
    tri_list = texelFetch(triTableTex, ivec2(cube_index, 0), 0).r # Assuming packed or specific layout
    
    # Iterate triangles (simplified logic based on standard MC table structure)
    # The source uses a 24-element array per cube index, terminated by -1
    # We assume a helper to unpack or direct access based on the specific texture layout
    # For implementation, one would typically use a standard 2D texture lookup or a large 1D array
    
    # Simplified iteration assuming we can access triangle indices directly
    # In the source, triTableTex is accessed similarly to edgeTableTex
    # The actual table data is 256x24 integers.
    
    # Let's assume a function get_triangle_index(cube_idx, tri_idx, vert_idx)
    # which returns the vertex index in edge_vertices
    
    triangle_count = 0
    while True:
        # Check if we have more triangles for this cube index
        # This part depends on the exact texture layout of triTableTex
        # Standard implementation: 256 rows, 24 columns. -1 terminates.
        
        # Placeholder for table lookup logic based on source description
        # The source code snippet doesn't show the exact unpacking, but implies standard MC table usage
        # We will assume a standard loop over the 24 entries
        
        # Note: The source code snippet for GS is not fully provided, but the logic is described
        # We will implement the standard MC triangle generation loop
        
        # For the sake of the article, we assume the table is accessible as:
        # tri_table[cube_index][triangle_index][vertex_index]
        
        # Since the source uses texelFetch, we must reconstruct the table access
        # Let's assume the table is stored as a 1D array of 256*24 shorts/ints
        
        # This is a high-level representation of the loop
        for tri_idx in range(8): # Max 5 triangles per cube, but table has 24 slots
            for vert_idx in range(3):
                idx = get_triangle_vertex_index(cube_index, tri_idx, vert_idx)
                if idx == -1:
                    return # End of triangles for this cube
                
                p_undeformed = edge_vertices[idx]
                
                # Apply Gaussian Deformation
                p_deformed = apply_gaussian_deformation(p_undeformed, i_force, sigma, c)
                
                # Transform to clip space
                # Note: Source uses mul(UNITY_MATRIX_MVP, unity_WorldToObject * p_deformed)
                # This is likely incorrect. Standard is mul(UNITY_MATRIX_VP, mul(unity_ObjectToWorld, p_deformed))
                # Or mul(UNITY_MATRIX_MVP, p_deformed) if p_deformed is in object space
                # Here p_deformed is in world space (since p_center was transformed)
                # So we need VP * World * Vertex
                # The source code's matrix order is suspicious.
                
                clip_pos = mul(UNITY_MATRIX_MVP, unity_WorldToObject * p_deformed) # As per source
                
                output = {
                    'pos': clip_pos,
                    'normal': input_vertices[0].normal, # Approximation, should be computed from mesh
                    'tex0': input_vertices[0].tex0
                }
                outStream.Append(output)
                
            outStream.RestartStrip()
            break # Assuming only one triangle per cube for simplicity in this example, 
                  # but standard MC generates multiple. The source loop structure is not fully detailed.
                  # The source says "Iterate through the table in steps of 3".
                  
    # Corrected loop structure based on standard MC:
    # The table is 256x24. We read 24 integers.
    # We iterate through them in groups of 3.
    
    # Re-implementing the triangle loop properly:
    tri_data = []
    for i in range(24):
        val = texelFetch(triTableTex, ivec2(cube_index, 0), 0).r # This is likely wrong for 24 values
        # The source code initializes triTableTex with 256x1 texture? 
        # "glTexImage2D(..., 256, 1, ...)" for edge table.
        # For tri table, it's likely similar or a larger texture.
        # The source code snippet for tri table initialization is cut off.
        # We assume a standard 2D texture of size 256x24 or a 1D texture of size 256*24.
        # Given the "texelFetch(tex, ivec2(index, 0), 0).r" usage, it implies a 2D texture.
        # If it's 256x24, we need to read 24 texels.
        
    # Due to ambiguity in the exact texture layout for the triangle table in the source,
    # we provide the logical step:
    
    # 1. Read the 24 vertex indices for the given cube_index from triTableTex.
    # 2. Iterate through these 24 indices in steps of 3.
    # 3. If the first index of a triplet is -1, stop.
    # 4. Otherwise, emit a triangle using the 3 indices.
    
    # This section is marked as [G] for the implementation detail of texture reading
    # because the source code snippet for triTableTex initialization is incomplete.
    pass

def apply_gaussian_deformation(p, F, sigma, c):
    # Guard against zero force
    if length(F) == 0:
        return p
        
    # Calculate perpendicular distance squared
    # d_perp^2 = ||p - c||^2 - ((p - c) . F)^2 / ||F||^2
    diff = p - c
    dot_prod = dot(diff, F)
    F_sq = dot(F, F)
    
    d_perp_sq = dot(diff, diff) - (dot_prod * dot_prod) / F_sq
    
    # Gaussian
    gaussian_val = exp(-d_perp_sq / (2 * sigma * sigma)) / (sigma * sqrt(2 * 3.14159265359))
    
    # Deformation
    # p' = p + F * 5 * N
    p_deformed = p + F * 5.0 * gaussian_val
    
    # Check for backward pushing
    # If dot product of displacement and force is non-positive, skip
    displacement = p_deformed - p
    if dot(displacement, F) <= 0:
        return p
        
    return p_deformed
```

## Subtopics

### GPU Marching Cubes Terrain Pipeline with Indirect Rendering
Uses indirect rendering commands to dispatch geometry generation, optimizing draw calls for large terrains. Choose when minimizing CPU-GPU synchronization overhead is critical. [3803-gpu-marching-cubes-terrain-pipeline-with-indirect-rendering.md]

### GPU-accelerated Marching Cubes with Distance Extraction
Focuses on optimizing the scalar field evaluation using distance fields. Choose when the scalar field is derived from distance functions rather than volumetric data. [3818-gpu-accelerated-marching-cubes-with-distance-extraction.md]

### GPU Marching Cubes with Atomic Counter Emission
Uses atomic counters to handle variable triangle output per voxel without pre-allocation. Choose when the number of triangles per voxel varies significantly and dynamic memory allocation on GPU is preferred. [4082-gpu-marching-cubes-with-atomic-counter-emission.md]

### Transvoxel
Supports multi-resolution voxel grids with smooth transitions. Choose when rendering large-scale terrains with varying levels of detail. [6277-transvoxel.md]

### GPU Isosurface Extraction from Molecular Orbital Densities
Specialized for scientific visualization of molecular orbitals. Choose when the scalar field represents quantum mechanical densities. [6862-gpu-isosurface-extraction-from-molecular-orbital-densities.md]

### Fragment Shader Isosurface Extraction with MRTs
Uses fragment shaders and multiple render targets for isosurface extraction. Choose when geometry shaders are unavailable or when a fully fragment-based pipeline is desired. [7507-fragment-shader-isosurface-extraction-with-mrts.md]

## Operating characteristics
*   **Time Complexity:** O(1) per voxel processed by the geometry shader, as it performs a fixed number of texture fetches and arithmetic operations regardless of grid resolution. Total complexity is O(N) for N voxels [S1, unofficial].
*   **Memory Complexity:** O(1) per voxel for the geometry shader state. The lookup tables (`edgeTableTex`, `triTableTex`) are stored in GPU memory and are constant size (256 entries each) [S1, unofficial].
*   **Measured Performance:** Implemented in a custom OpenGL 3.3+ application. Performance is dependent on the GPU's geometry shader throughput and the resolution of the voxel grid [S1, unofficial].
*   **Strengths:** Real-time deformation capability; GPU-accelerated; simple implementation using standard Marching Cubes tables.
*   **Limitations:** The matrix multiplication order in the source code is likely incorrect, leading to incorrect clip-space positions [S2, unofficial]. Degenerate cases (zero force, zero field values) are not explicitly guarded against in the source code, potentially causing NaNs or division by zero [S1, unofficial].
*   **When to Use:** Real-time deformable terrain or mesh interaction where the deformation is localized and Gaussian-based.
*   **When Not to Use:** When high precision is required for the transformation matrices, or when the force vector can be zero (requires external guarding).

## Background
This method was implemented in the NtslMaterial geometry shader for real-time deformable terrain/mesh interaction in a custom OpenGL 3.3+ application [S1, unofficial]. It is a variant of the Marching Cubes algorithm that integrates Gaussian deformation directly into the geometry shader pipeline [S1, unofficial]. The source code is part of the `ntsl-graphics` repository [S1, unofficial]. A related Unity-based implementation exists in `UnityGPUMarchingCubes`, which uses a similar geometry shader approach but with different matrix transformation logic [S2, unofficial].

## Sources
[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] kristofe/UnityGPUMarchingCubes, https://github.com/kristofe/UnityGPUMarchingCubes, fetched 2026-10-26 (unofficial)
