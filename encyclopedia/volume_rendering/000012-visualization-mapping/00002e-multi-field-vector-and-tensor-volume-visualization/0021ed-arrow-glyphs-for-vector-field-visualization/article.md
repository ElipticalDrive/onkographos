# Arrow Glyphs for Vector Field Visualization
Also called: Arrow Glyphs
## Overview
Arrow glyphs resolve the direction ambiguity inherent in line-based vector field visualizations by using arrow-shaped primitives that explicitly encode flow direction. Unlike [Line Glyphs for Vector Field Visualization](2569-line-glyphs-for-vector-field-visualization.md), which suffer from orientation ambiguity, arrow glyphs use triangular faces to allow for 3D shading and clear directional cues. This method is implemented entirely on the GPU to avoid CPU-GPU data transfer, resulting in performance approximately 2x faster than line glyphs (about 10 ms for typical scenes) due to the larger visual size of arrows allowing for lower sampling density.
## How it works
### Inputs and Outputs
**Inputs:**
- `vectorFieldTex`: A 3D texture storing vector data. Each texel is a `float4` where $(x, y, z)$ is the vector direction and $w$ is the magnitude.
- `position`: The 3D world position where the glyph is centered.
- `glyphsCount`: A `uint2` specifying the number of glyphs in the x and y dimensions of the sampling plane.
- `worldSize`: A `float2` defining the physical size of the sampling plane.
- `x`: A float parameter defining the z-coordinate (depth) of the sampling plane.

**Outputs:**
- `outVertices`: A buffer of `float3` with 9 vertices per glyph.
- `outFaces`: A buffer of `uint3` with 6 faces (triangles) per glyph.
- `outVertexNormals`: A buffer of `float3` with 9 normals per glyph.
- `outVertexColors`: A buffer of `float3` with 9 colors per glyph.

### Data Structures
Each glyph consists of 9 vertices and 6 triangular faces. The vertex layout is defined as follows:
- Vertices 0–4 form the arrowhead (cone).
- Vertices 5–8 form the arrow shaft (base).
- Vertex 0 is the tip.
- Vertices 1, 2, 3, 4 are the base of the cone.
- Vertices 5, 6, 7, 8 are the corners of the rectangular shaft base.

### Algorithm
1. **Sample Vector:** Read the vector field at the current position $(x, y, z)$ from `vectorFieldTex`. Let the result be `vector` ($x, y, z, w$).
2. **Compute Forward Axis:** Normalize the directional components to get the forward unit vector:
   $$ \mathbf{f} = \text{normalize}(vector.xyz) $$
3. **Compute Perpendicular Axes:**
   - Compute a perpendicular vector $\mathbf{x}$ to $\mathbf{f}$. This requires a helper function `findPerpendicular` that returns a unit vector orthogonal to $\mathbf{f}$.
   - Compute $\mathbf{y}$ as the cross product:
     $$ \mathbf{y} = \text{normalize}(\mathbf{f} \times \mathbf{x}) $$
4. **Define Face Indices:** For a glyph with global vertex index `vertId`, define the 6 triangles:
   - Face 0: $(v_0, v_1, v_2)$
   - Face 1: $(v_0, v_2, v_3)$
   - Face 2: $(v_0, v_3, v_4)$
   - Face 3: $(v_0, v_4, v_1)$
   - Face 4: $(v_5, v_6, v_7)$
   - Face 5: $(v_5, v_7, v_8)$
5. **Assign Normals:**
   - $n_0 = \mathbf{f}$ (Tip)
   - $n_1 = \mathbf{x}$
   - $n_2 = \mathbf{y}$
   - $n_3 = -\mathbf{x}$
   - $n_4 = -\mathbf{y}$
   - $n_5 = -\mathbf{f}$ (Base center direction)
   - $n_6 = -\mathbf{f}$
   - $n_7 = -\mathbf{f}$
   - $n_8 = -\mathbf{f}$
6. **Scale Axes:**
   - Scale the forward vector by the magnitude: $\mathbf{f}_{scaled} = \mathbf{f} \times vector.w$.
   - Scale the perpendicular axes by a fixed factor of 0.1: $\mathbf{x}_{scaled} = \mathbf{x} \times 0.1$, $\mathbf{y}_{scaled} = \mathbf{y} \times 0.1$.
7. **Compute Vertex Positions:**
   - $v_0 = \text{position} - \mathbf{f}_{scaled}$ (Tip)
   - $v_1 = \text{position} + \mathbf{x}_{scaled}$
   - $v_2 = \text{position} + \mathbf{y}_{scaled}$
   - $v_3 = \text{position} - \mathbf{x}_{scaled}$
   - $v_4 = \text{position} - \mathbf{y}_{scaled}$
   - $v_5 = \text{position} + \mathbf{x}_{scaled}$
   - $v_6 = \text{position} + \mathbf{y}_{scaled}$
   - $v_7 = \text{position} - \mathbf{x}_{scaled}$
   - $v_8 = \text{position} - \mathbf{y}_{scaled}$
8. **Assign Color:** Sample the magnitude color lookup table using `vector.w` and assign the resulting RGB color to all 9 vertices.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def glyph_arrows_kernel(position, vector_field_tex, magnitude_lut, glyphs_count, world_size, z_plane):
    # Calculate grid coordinates based on position and world size
    # This is a simplified representation of the CUDA kernel logic
    # assuming 'id' is the unique glyph index
    
    # 1. Sample vector
    vector = sample_3d_texture(vector_field_tex, position)
    
    # 2. Compute forward axis
    forward = normalize(vector.xyz)
    
    # 3. Compute perpendicular axes
    # findPerpendicular returns a unit vector orthogonal to 'forward'
    # Common implementation: if abs(forward.x) < 0.9, perp = cross(forward, (1,0,0))
    # else perp = cross(forward, (0,1,0))
    x_axis = normalize(find_perpendicular(forward))
    y_axis = normalize(cross(forward, x_axis))
    
    # 4. Define face indices
    vert_id = id * 9
    face_id = id * 6
    
    faces = [
        (vert_id, vert_id + 1, vert_id + 2),
        (vert_id, vert_id + 2, vert_id + 3),
        (vert_id, vert_id + 3, vert_id + 4),
        (vert_id, vert_id + 4, vert_id + 1),
        (vert_id + 5, vert_id + 6, vert_id + 7),
        (vert_id + 5, vert_id + 7, vert_id + 8)
    ]
    
    # 5. Assign normals
    normals = [
        forward,       # 0: Tip
        x_axis,        # 1
        y_axis,        # 2
        -x_axis,       # 3
        -y_axis,       # 4
        -forward,      # 5: Base
        -forward,      # 6
        -forward,      # 7
        -forward       # 8
    ]
    
    # 6. Scale axes
    forward_scaled = forward * vector.w
    x_axis_scaled = x_axis * 0.1
    y_axis_scaled = y_axis * 0.1
    
    # 7. Compute vertex positions
    vertices = [
        position - forward_scaled, # 0: Tip
        position + x_axis_scaled,  # 1
        position + y_axis_scaled,  # 2
        position - x_axis_scaled,  # 3
        position - y_axis_scaled,  # 4
        position + x_axis_scaled,  # 5
        position + y_axis_scaled,  # 6
        position - x_axis_scaled,  # 7
        position - y_axis_scaled   # 8
    ]
    
    # 8. Assign color
    color = sample_1d_texture(magnitude_lut, vector.w)
    colors = [color] * 9
    
    return vertices, faces, normals, colors
```

### Parameters
- **Head Width Scaling Factor:** 0.1. This scales the magnitude of the perpendicular axes ($\mathbf{x}$ and $\mathbf{y}$) to determine the width of the arrow head and shaft.

### Edge Cases and Pitfalls
- **Zero Vector:** If the sampled vector magnitude is 0, the forward vector is undefined. The implementation must handle this by either skipping the glyph or defaulting to a neutral orientation.
- **Perpendicular Vector Selection:** The `findPerpendicular` function must handle cases where the forward vector is parallel to standard basis vectors to avoid zero-length cross products.
- **Precision:** Using shared VBOs (Vertex Buffer Objects) avoids CPU-GPU transfer but requires careful memory management on the GPU to prevent overflow for large glyph counts.

## Operating characteristics
- **Time Complexity:** The kernel computes vertices, normals, and indices for all arrows in parallel. The source notes that arrow glyphs are roughly 2x faster than line glyphs, taking about 10 ms for typical scenes. This speedup is attributed to the larger visual size of arrows, which allows for a lower sampling density while maintaining visual clarity.
- **Memory Complexity:** Requires storage for 9 vertices, 6 faces, 9 normals, and 9 colors per glyph. All data is computed on the GPU.
- **Strengths:** Removes direction ambiguity; allows for 3D shading due to triangular faces; fast performance due to GPU-only computation and lower density requirements.
- **Limitations:** More complex geometry than lines (9 vertices vs 2); requires careful handling of perpendicular vector generation.

## Background
Arrow glyphs are a technique for visualizing 3D vector fields, introduced in the context of real-time visualization using CUDA. The method was presented by Marek Fiser in an unofficial blog post detailing the implementation of vector field visualization techniques. The approach builds on the limitations of line glyphs, specifically their ambiguity in direction, by introducing explicit directional cues through arrow geometry.

## Sources
[S1] Realtime visualization of 3D vector fields using CUDA, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
