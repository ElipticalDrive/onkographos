# Circle Splat Rendering
## Overview
Circle Splat Rendering is an object-order volume rendering technique that projects volumetric primitives as 2D quads (splats) in screen space to leverage hardware-accelerated rasterization [S1, unofficial]. It differs from standard point-splatting by explicitly constructing a quad geometry with a size derived from the voxel grid scale and a configurable splat parameter, allowing for consistent coverage and efficient frustum culling in clip space [S1, unofficial].
## How it works
### Inputs and Outputs
**Inputs:**
*   Vertex data per voxel: position $\mathbf{p}$, color $\mathbf{c}$, and normal $\mathbf{n}$ [S1, unofficial].
*   Uniforms: Model-View matrix $MV$, Projection matrix $P$, entity $scale$, grid $gridLength$, splat size parameter $splatSize$, and culling flags [S1, unofficial].

**Outputs:**
*   Rasterized image with accumulated color and depth [S1, unofficial].

### Data Structures
The input data is typically stored in a Vertex Buffer Object (VBO) containing interleaved or separate arrays for:
*   `position`: `vec3` (float, 3 components) [S1, unofficial].
*   `color`: `vec4` (float, 4 components) [S1, unofficial].
*   `normal`: `vec3` (float, 3 components) [S1, unofficial].

### Steps
1.  **Transform to View Space:** Multiply the input voxel position by the Model-View matrix to obtain view-space coordinates [S1, unofficial].
    $$ \mathbf{p}_{view} = MV \cdot \mathbf{p}_{input} $$
2.  **Transform to Clip Space:** Multiply the view-space position by the Projection matrix and perform perspective division to normalize coordinates [S1, unofficial].
    $$ \mathbf{p}_{clip} = P \cdot \mathbf{p}_{view} $$
    $$ \mathbf{p}_{ndc} = \frac{\mathbf{p}_{clip}}{\mathbf{p}_{clip}.w} $$
3.  **Calculate Splat Offset:** Compute the physical offset for the quad size in view space. The offset is scaled by the ratio of the entity scale to the grid length and multiplied by $\sqrt{2}$ to account for diagonal coverage [S1, unofficial].
    $$ offset = splatSize \times \frac{scale}{gridLength} \times \sqrt{2} $$
    Where $\sqrt{2} \approx 1.414213$ [S1, unofficial].
4.  **Compute Clip-Space Half-Edge:** Determine the distance in clip space between the center of the splat and the midpoint of its right edge to establish the culling boundary [S1, unofficial].
    *   Calculate the right edge midpoint in view space: $\mathbf{p}_{right\_view} = \mathbf{p}_{view}.xy + (0.5 \times offset, 0)$.
    *   Transform this midpoint to clip space: $\mathbf{p}_{right\_clip} = P \cdot \mathbf{p}_{right\_view}$.
    *   Normalize: $\mathbf{p}_{right\_ndc} = \frac{\mathbf{p}_{right\_clip}}{\mathbf{p}_{right\_clip}.w}$.
    *   Compute distance: $d_{half} = \text{distance}(\mathbf{p}_{ndc}, \mathbf{p}_{right\_ndc})$.
5.  **Frustum Culling:** Define a culling border in normalized device coordinates (NDC) as $1 + d_{half}$. Discard the primitive if the NDC center position exceeds this border in any axis [S1, unofficial].
    $$ \text{if } |x_{ndc}| > 1 + d_{half} \lor |y_{ndc}| > 1 + d_{half} \text{ then discard} $$
6.  **Back-Face Culling (Optional):** If enabled, compute the view-space normal $\mathbf{n}_{view} = MV \cdot (\mathbf{n}, 0)$ and discard if the dot product with the view direction (vector from camera to voxel) is negative [S1, unofficial].
7.  **Emit Quad Vertices:** Generate a triangle strip consisting of four vertices representing the quad. The vertices are defined in view space by offsetting the center position by $\pm 0.5 \times offset$ in X and Y, then transformed to clip space for `gl_Position` [S1, unofficial].
    *   Bottom-Left: $\mathbf{p}_{view} + (-0.5 \cdot offset, -0.5 \cdot offset, 0)$
    *   Bottom-Right: $\mathbf{p}_{view} + (0.5 \cdot offset, -0.5 \cdot offset, 0)$
    *   Top-Left: $\mathbf{p}_{view} + (-0.5 \cdot offset, 0.5 \cdot offset, 0)$
    *   Top-Right: $\mathbf{p}_{view} + (0.5 \cdot offset, 0.5 \cdot offset, 0)$
    *   Pass texture coordinates $(0,0), (1,0), (0,1), (1,1)$ and color/normal to the fragment shader [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader / Geometry Shader Logic (Conceptual)

def process_voxel(voxel_pos_input, voxel_color, voxel_normal, MV, P, scale, grid_length, splat_size, cull_invisible_faces):
    # 1. Transform to View Space
    voxel_pos_view = MV * voxel_pos_input
    
    # 2. Transform to Clip Space and Normalize
    voxel_pos_clip = P * voxel_pos_view
    voxel_pos_ndc = voxel_pos_clip / voxel_pos_clip.w
    
    # 3. Calculate Splat Offset
    sqrt2 = 1.414213
    offset = splat_size * (scale / grid_length) * sqrt2
    
    # 4. Compute Clip-Space Half-Edge for Culling
    # Right edge midpoint in view space
    right_edge_view_x = voxel_pos_view.x + 0.5 * offset
    right_edge_view_y = voxel_pos_view.y
    right_edge_view_z = voxel_pos_view.z
    right_edge_view_w = voxel_pos_view.w
    
    right_edge_clip = P * vec4(right_edge_view_x, right_edge_view_y, right_edge_view_z, right_edge_view_w)
    right_edge_ndc = right_edge_clip / right_edge_clip.w
    
    half_edge_length = distance(voxel_pos_ndc.xy, right_edge_ndc.xy)
    culling_border = 1.0 + half_edge_length
    
    # 5. Frustum Culling
    if (abs(voxel_pos_ndc.x) > culling_border or 
        abs(voxel_pos_ndc.y) > culling_border):
        return None # Discard
    
    # 6. Back-Face Culling
    if cull_invisible_faces:
        normal_view = MV * vec4(voxel_normal, 0.0)
        view_direction = normalize(-voxel_pos_view.xyz) # Camera is at origin in view space
        if dot(view_direction, normal_view.xyz) < 0:
            return None # Discard
            
    # 7. Emit Quad Vertices
    vertices = []
    offsets = [
        (-0.5 * offset, -0.5 * offset),
        ( 0.5 * offset, -0.5 * offset),
        (-0.5 * offset,  0.5 * offset),
        ( 0.5 * offset,  0.5 * offset)
    ]
    tex_coords = [
        (0.0, 0.0),
        (1.0, 0.0),
        (0.0, 1.0),
        (1.0, 1.0)
    ]
    
    for i in range(4):
        ox, oy = offsets[i]
        vx = voxel_pos_view.x + ox
        vy = voxel_pos_view.y + oy
        vz = voxel_pos_view.z
        vw = voxel_pos_view.w
        
        clip_v = P * vec4(vx, vy, vz, vw)
        ndc_v = clip_v / clip_v.w
        
        vertices.append({
            'position': ndc_v,
            'color': voxel_color,
            'normal': voxel_normal,
            'tex_coord': tex_coords[i]
        })
        
    return vertices
```

### Parameters
*   `splatSize`: A uniform scaling factor for the primitive size [S1, unofficial].
*   `scale`: The entity scale factor [S1, unofficial].
*   `gridLength`: The length of the grid unit [S1, unofficial].
*   `cullInvisibleFaces`: Boolean flag for back-face culling [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** Perspective division must be performed carefully to avoid division by zero if a voxel is exactly at the camera origin [S1, unofficial].
*   **Culling Border:** The culling border includes the half-edge length to ensure the entire quad is discarded if its center is outside the frustum, preventing artifacts from partially visible splats near the edge [S1, unofficial].
*   **Diagonal Scaling:** The use of $\sqrt{2}$ in the offset calculation assumes the splat covers the diagonal of the voxel grid cell, which affects the visual density and overlap of adjacent splats [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels $N$, $O(N)$, as each voxel is processed independently in the geometry shader [S1, unofficial].
*   **Memory Complexity:** Linear with respect to $N$ for storing voxel data [S1, unofficial].
*   **Performance:** Relies on GPU rasterization for high performance, enabling real-time interaction [S1, unofficial].
*   **Strengths:** Simple implementation, hardware-accelerated, efficient culling [S1, unofficial].
*   **Limitations:** Fixed quad shape may not adapt to anisotropic view transformations as well as EWA splatting; potential for aliasing if splat size is not tuned to resolution [S1, unofficial].

## Background
Circle Splat Rendering, referred to as `square_splat` in the source implementation, is part of a custom SVO (Sparse Voxel Octree) renderer [S1, unofficial]. It is designed to render volumetric data by projecting voxels as 2D primitives, leveraging the graphics pipeline's ability to handle large numbers of simple shapes efficiently [S1, unofficial].

## Sources
[S1] mmalesadev/svo_renderer, https://github.com/mmalesadev/svo_renderer, fetched 2026-10-26 (unofficial)
