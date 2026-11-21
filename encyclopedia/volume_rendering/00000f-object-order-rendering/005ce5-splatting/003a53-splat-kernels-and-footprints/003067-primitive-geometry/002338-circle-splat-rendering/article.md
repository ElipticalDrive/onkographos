# Circle Splat Rendering

## Overview
Circle Splat Rendering is a volume rendering technique where volumetric entities (voxels) are projected as 2D quads in screen space. It solves the problem of efficiently rasterizing large datasets by mapping object-space coordinates to screen-space footprints via projection matrices. It is distinguished from sibling methods like [Line Capsule Primitive](3131-line-capsule-primitive.md) and [Tetrahedral Splatting](3181-tetrahedral-splatting.md) by its use of fixed-offset square quads rather than parametric or complex geometric primitives.

## How it works
**Inputs:**
*   Model-View matrix ($MV$) and Projection matrix ($P$).
*   Voxel position in clip space ($gl\_in[0].gl\_Position$).
*   Voxel color and normal attributes.
*   Uniforms: `splatSize` (scalar), `scale` (entity scale), `gridLength` (grid scale), `cullInvisibleFaces` (boolean).

**Outputs:**
*   Triangle strip of 4 vertices representing the splat quad.
*   Interpolated color and normal for fragment shading.

**Data Structures:**
The implementation uses a geometry shader stage. Input is a single point (representing a voxel center). Output is a `triangle_strip` with `max_vertices = 4`.

**Steps:**
1.  **Transform to View Space:** Multiply the input clip-space position by the Model-View matrix to get view-space coordinates.
    $$ \mathbf{p}_{view} = MV \cdot \mathbf{p}_{clip\_input} $$
2.  **Transform to Normalized Device Coordinates (NDC):** Multiply by the Projection matrix and perform perspective division.
    $$ \mathbf{p}_{ndc} = \frac{P \cdot \mathbf{p}_{view}}{w} $$
3.  **Calculate Voxel Offset:** Compute the physical size of the splat in view space.
    $$ \text{voxelOffset} = \text{splatSize} \times \frac{\text{scale}}{\text{gridLength}} \times \sqrt{2} $$
    Where $\sqrt{2} \approx 1.414213$.
4.  **Compute Clip-Space Half-Edge:** Determine the distance in NDC from the center to the right edge midpoint to establish the splat's screen-space extent.
    $$ \mathbf{p}_{right\_mid} = \frac{P \cdot (\mathbf{p}_{view}.xy + \text{voxelOffset} \cdot [0.5, 0], \mathbf{p}_{view}.zw)}{w} $$
    $$ \text{halfEdge} = \text{distance}(\mathbf{p}_{ndc}, \mathbf{p}_{right\_mid}) $$
5.  **Frustum Culling:** Define a culling border slightly larger than the half-edge to prevent gaps. Discard the primitive if the NDC center exceeds this border.
    $$ \text{border} = 1 + \text{halfEdge} $$
    If $|\mathbf{p}_{ndc}.x| > \text{border}$ or $|\mathbf{p}_{ndc}.y| > \text{border}$, discard.
6.  **Back-Face Culling (Optional):** If `cullInvisibleFaces` is true, compute the view-space normal and discard if facing away from the camera.
    $$ \mathbf{n}_{view} = MV \cdot [\mathbf{n}, 0] $$
    If $\mathbf{n}_{view} \cdot (\mathbf{0} - \mathbf{p}_{view}) < 0$, discard.
7.  **Emit Vertices:** Generate four vertices for a triangle strip. The vertices are defined in view space using the offset, then transformed to clip space.
    *   Bottom-Left: $\mathbf{p}_{view} + [-0.5, -0.5] \cdot \text{voxelOffset}$
    *   Bottom-Right: $\mathbf{p}_{view} + [0.5, -0.5] \cdot \text{voxelOffset}$
    *   Top-Left: $\mathbf{p}_{view} + [-0.5, 0.5] \cdot \text{voxelOffset}$
    *   Top-Right: $\mathbf{p}_{view} + [0.5, 0.5] \cdot \text{voxelOffset}$
    Each vertex is transformed by $P$ for `gl_Position`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_splat(voxel_pos_clip, voxel_color, voxel_normal, MV, P, splat_size, scale, grid_length, cull_invisible_faces):
    # 1. Transform to View Space
    voxel_pos_view = MV @ voxel_pos_clip
    
    # 2. Transform to NDC
    voxel_pos_ndc = (P @ voxel_pos_view) / voxel_pos_view.w
    
    # 3. Calculate Voxel Offset
    voxel_offset = splat_size * (scale / grid_length) * 1.414213
    
    # 4. Compute Clip-Space Half-Edge for Culling
    # Right edge midpoint in view space
    right_edge_mid_view = vec4(voxel_pos_view.x + 0.5 * voxel_offset, 
                               voxel_pos_view.y, 
                               voxel_pos_view.z, 
                               voxel_pos_view.w)
    # Transform to NDC
    right_edge_mid_ndc = (P @ right_edge_mid_view) / right_edge_mid_view.w
    
    half_edge = distance(voxel_pos_ndc, right_edge_mid_ndc)
    frustum_border = 1.0 + half_edge
    
    # 5. Frustum Culling
    if abs(voxel_pos_ndc.x) > frustum_border or abs(voxel_pos_ndc.y) > frustum_border:
        return None
        
    # 6. Back-Face Culling
    if cull_invisible_faces:
        normal_view = MV @ vec4(voxel_normal, 0.0)
        view_dir = normalize(-voxel_pos_view.xyz)
        if dot(normal_view.xyz, view_dir) < 0:
            return None
            
    # 7. Emit Vertices
    vertices = []
    offsets = [(-0.5, -0.5), (0.5, -0.5), (-0.5, 0.5), (0.5, 0.5)]
    tex_coords = [(0, 0), (1, 0), (0, 1), (1, 1)]
    
    for i in range(4):
        ox, oy = offsets[i]
        # Define vertex in view space
        v_view = vec4(voxel_pos_view.x + ox * voxel_offset,
                      voxel_pos_view.y + oy * voxel_offset,
                      voxel_pos_view.z,
                      voxel_pos_view.w)
        # Transform to clip space
        v_clip = P @ v_view
        vertices.append((v_clip, tex_coords[i], voxel_color, voxel_normal))
        
    return vertices
```

## Operating characteristics
*   **Complexity:** $O(1)$ per voxel for geometry generation.
*   **Memory:** Minimal overhead; requires storage for MV/P matrices and uniform parameters.
*   **Strengths:** Simple implementation; efficient rasterization via standard triangle strips; effective frustum culling.
*   **Limitations:** Square footprints may alias or leave gaps between adjacent splats if not properly blended or if `splatSize` is not tuned to the grid resolution.

## Background
This method is implemented in the `svo_renderer` project, which uses a geometry shader to expand point primitives into square quads for volume visualization.

## Sources
[S1] mmalesadev/svo_renderer, https://github.com/mmalesadev/svo_renderer, fetched 2026-10-26 (unofficial)
