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
*   `position`: `vec4` (float, 4 components) [S1, unofficial].
*   `color`: `vec4` (float, 4 components) [S1, unofficial].
*   `normal`: `vec3` (float, 3 components) [S1, unofficial].

### Steps
1.  **Transform to View Space:** Multiply the input voxel position by the Model-View matrix to obtain view-space coordinates [S1, unofficial].
    $$ \mathbf{p}_{view} = MV \cdot \mathbf{p}_{input} $$
2.  **Transform to Clip Space:** Multiply the view-space position by the Projection matrix and perform perspective division to normalize coordinates [S1, unofficial].
    $$ \mathbf{p}_{clip} = P \cdot \mathbf{p}_{view} $$
    $$ \mathbf{p}_{ndc} = \frac{\mathbf{p}_{clip}}{\mathbf{p}_{clip}.w} $$
3.  **Calculate Splat Offset:** Compute the physical offset for the quad size in view space [S1, unofficial]. The offset is scaled by the ratio of the entity scale to the grid length and multiplied by $\sqrt{2}$ to account for diagonal coverage [S1, unofficial].
    $$ offset = splatSize \times \frac{scale}{gridLength} \times \sqrt{2} $$
    Where $\sqrt{2} \approx 1.414213$ [S1, unofficial].
4.  **Compute Clip-Space Half-Edge:** Determine the distance in clip space between the center of the splat and the midpoint of its right edge to establish the culling boundary [S1, unofficial].
    *   Calculate the right edge midpoint in view space: $\mathbf{p}_{right\_view} = \mathbf{p}_{view}.xy + (0.5 \times offset, 0)$ [S1, unofficial].
    *   Transform this midpoint to clip space: $\mathbf{p}_{right\_clip} = P \cdot \mathbf{p}_{right\_view}$ [S1, unofficial].
    *   Normalize: $\mathbf{p}_{right\_ndc} = \frac{\mathbf{p}_{right\_clip}}{\mathbf{p}_{right\_clip}.w}$ [S1, unofficial].
    *   Compute distance: $d_{half} = \text{distance}(\mathbf{p}_{ndc}, \mathbf{p}_{right\_ndc})$ [S1, unofficial].
5.  **Frustum Culling:** Define a culling border in normalized device coordinates (NDC) as $1 + d_{half}$ [S1, unofficial]. Discard the primitive if the NDC center position exceeds this border in any axis [S1, unofficial].
    $$ \text{if } |x_{ndc}| > 1 + d_{half} \lor |y_{ndc}| > 1 + d_{half} \text{ then discard} $$
6.  **Back-Face Culling (Optional):** If enabled, compute the view-space normal $\mathbf{n}_{view} = MV \cdot (\mathbf{n}, 0)$ and discard if the dot product with the view direction (vector from camera to voxel) is negative [S1, unofficial].
7.  **Emit Quad Vertices:** Generate a triangle strip consisting of four vertices representing the quad [S1, unofficial]. The vertices are defined in view space by offsetting the center position by $\pm 0.5 \times offset$ in X and Y, then transformed to clip space for `gl_Position` [S1, unofficial].
    *   Bottom-Left: $\mathbf{p}_{view} + (-0.5 \cdot offset, -0.5 \cdot offset, 0)$ [S1, unofficial].
    *   Bottom-Right: $\mathbf{p}_{view} + (0.5 \cdot offset, -0.5 \cdot offset, 0)$ [S1, unofficial].
    *   Top-Left: $\mathbf{p}_{view} + (-0.5 \cdot offset, 0.5 \cdot offset, 0)$ [S1, unofficial].
    *   Top-Right: $\mathbf{p}_{view} + (0.5 \cdot offset, 0.5 \cdot offset, 0)$ [S1, unofficial].
    *   Pass texture coordinates $(0,0), (1,0), (0,1), (1,1)$ and color/normal to the fragment shader [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def process_voxel(voxel_pos_input, voxel_color, voxel_normal, MV, P, scale, grid_length, splat_size, cull_invisible_faces):
    # 1. Transform to View Space
    voxel_pos_view = MV * voxel_pos_input
    
    # 2. Transform to Clip Space and Normalize
    voxel_pos_clip = P * voxel_pos_view
    voxel_pos_ndc = [c / voxel_pos_clip[3] for c in voxel_pos_clip]
    
    # 3. Calculate Splat Offset
    sqrt2 = 1.414213
    offset = splat_size * (scale / grid_length) * sqrt2
    
    # 4. Compute Clip-Space Half-Edge for Culling
    right_edge_view = [voxel_pos_view[0] + 0.5 * offset, voxel_pos_view[1], voxel_pos_view[2], voxel_pos_view[3]]
    right_edge_clip = P * right_edge_view
    right_edge_ndc = [c / right_edge_clip[3] for c in right_edge_clip]
    
    half_edge_length = math.sqrt((voxel_pos_ndc[0] - right_edge_ndc[0])**2 + (voxel_pos_ndc[1] - right_edge_ndc[1])**2)
    culling_border = 1.0 + half_edge_length
    
    # 5. Frustum Culling
    if (math.fabs(voxel_pos_ndc[0]) > culling_border or 
        math.fabs(voxel_pos_ndc[1]) > culling_border):
        return None  # Discard
    
    # 6. Back-Face Culling
    if cull_invisible_faces:
        normal_view = MV * [voxel_normal[0], voxel_normal[1], voxel_normal[2], 0.0]
        view_direction = [-voxel_pos_view[0], -voxel_pos_view[1], -voxel_pos_view[2]]
        dot_product = view_direction[0]*normal_view[0] + view_direction[1]*normal_view[1] + view_direction[2]*normal_view[2]
        if dot_product < 0:
            return None  # Discard
            
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
        vx = voxel_pos_view[0] + ox
        vy = voxel_pos_view[1] + oy
        vz = voxel_pos_view[2]
        vw = voxel_pos_view[3]
        
        clip_v = P * [vx, vy, vz, vw]
        ndc_v = [c / clip_v[3] for c in clip_v]
        
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
Not given in the sources.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels $N$, $O(N)$, as each voxel is processed independently in the geometry shader [S1, unofficial].
*   **Memory Complexity:** Linear with respect to $N$ for storing voxel data [S1, unofficial].
*   **Performance:** Relies on GPU rasterization for high performance, enabling real-time interaction [S1, unofficial].
*   **Strengths:** Simple implementation, hardware-accelerated, efficient culling [S1, unofficial].

## Background
Circle Splat Rendering, referred to as `square_splat` in the source implementation, is part of a custom SVO (Sparse Voxel Octree) renderer [S1, unofficial]. It is designed to render volumetric data by projecting voxels as 2D primitives, leveraging the graphics pipeline's ability to handle large numbers of simple shapes efficiently [S1, unofficial].

## Sources
[S1] mmalesadev/svo_renderer, https://github.com/mmalesadev/svo_renderer, fetched 2026-10-26 (unofficial)
