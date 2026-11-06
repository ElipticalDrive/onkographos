# Texture-Based Volume Rendering

Also called: TBVR, Proxy Geometry Rendering, Slice-Based Volume Rendering (SBVR)

## Overview
Texture-Based Volume Rendering (TBVR) is a GPU-accelerated technique that renders volumetric data by rasterizing a set of 2D polygon primitives (proxy geometry) through the volume, which are textured with the volumetric data and composited in sorted order using fragment shaders [S1, S3, S4, S5, unofficial]. It solves the problem of efficiently mapping 3D scalar fields to 2D screens by leveraging the graphics pipeline's texture sampling and alpha blending hardware, avoiding the need for explicit ray-marching loops in shaders [S2, book][S4, unofficial]. It is distinguished from direct ray casting by replacing per-pixel ray integration with parallel rasterization of view-aligned or object-aligned slices, trading memory efficiency for rendering speed [S1, unofficial][S2, book].

## How it works
TBVR implements the emission-absorption optical model by treating the volume as a texture resource and the rendering process as a sequence of 2D slice renderings [S1, S3, unofficial]. The method follows three main stages: Initialize, Update, and Draw [S2, book][S6].

### Inputs and Outputs
*   **Input:** A 3D scalar volume grid $V$ of dimensions $D_x \times D_y \times D_z$, a transfer function $T$ mapping scalar values to color and opacity, and camera parameters (view direction, projection).
*   **Output:** A 2D image frame buffer with accumulated color and opacity.

### Data Structures
1.  **Volume Texture:** The scalar data is stored in GPU memory as either:
    *   A 3D texture object (single texture unit, supports trilinear interpolation) [S1, unofficial][S2, book][S4, unofficial].
    *   A stack of 2D textures (multiple texture units, bilinear interpolation per slice) [S1, unofficial][S2, book][S5, unofficial].
2.  **Transfer Function Texture:** A 1D texture mapping scalar values to RGBA [S2, book][S6].
3.  **Proxy Geometry:** A set of polygons (quads or triangle fans) representing slices through the volume bounding box. Vertices are stored in vertex arrays with associated texture coordinates [S2, book][S6].

### Steps

#### 1. Initialize
*   Load volume data into CPU memory [S2, book][S6].
*   Process data if necessary (e.g., compute gradients for shading, downsample) [S2, book][S6].
*   Create the transfer function lookup table (texture) [S2, book][S6].
*   Create fragment shaders for data lookup, transfer function application, and optional shading [S2, book][S6].
*   Upload volume data to GPU memory as a 3D texture or 2D texture stack [S1, unofficial][S2, book][S4, S5, unofficial].

#### 2. Update (per frame or parameter change)
*   **Compute Proxy Geometry:**
    *   Define slicing planes orthogonal to the viewing direction, spaced by a fixed distance $\Delta t$ [S2, book][S4, unofficial].
    *   Intersect these planes with the volume's axis-aligned bounding box (AABB) to determine the polygon vertices [S2, book][S6].
    *   Sort the resulting vertices in clockwise or counter-clockwise order around the polygon centroid [S2, book][S6].
    *   Construct the polygon primitive (e.g., as a triangle fan from the centroid) [S3, unofficial].
    *   Assign 3D texture coordinates to each vertex. For a 3D texture, coordinates map the AABB to $[0,1]^3$ [S2, book][S6]. For 2D stacks, coordinates map to the specific 2D slice [S2, book].
*   **Sort Slices:** Sort the proxy polygons back-to-front relative to the camera view [S1, unofficial][S2, book][S6].
*   **Refresh Textures:** If rendering mode or transfer function parameters change, update the transfer function texture. If sampling rate changes, apply opacity correction to the transfer function to maintain consistent visual density [S2, book][S6].

#### 3. Draw
*   Set up rendering state:
    *   Disable lighting and face culling [S2, book][S6].
    *   Enable alpha blending. For the Over operator (back-to-front), set source factor to 1 and destination factor to $1 - \text{Source Alpha}$ [S1, unofficial][S2, book].
    *   Disable depth buffer writing if blending with opaque geometry, or enable depth testing with write disabled [S2, book][S6].
*   Bind volume texture (3D or 2D stack) and transfer function texture (1D) to texture units [S2, book][S6].
*   For each slice polygon in sorted order:
    *   Draw the polygon.
    *   The rasterizer interpolates texture coordinates across the polygon [S2, book].
    *   The fragment shader executes:
        1.  Lookup scalar value $v$ from volume texture using interpolated coordinates [S2, book][S6].
        2.  Lookup color $C$ and opacity $\alpha$ from transfer function texture using $v$ [S2, book][S6].
        3.  (Optional) Apply shading (e.g., Blinn-Phong) using gradients estimated from neighboring voxels [S1, unofficial][S2, book].
        4.  Output $(C, \alpha)$ to the compositing stage [S2, book][S6].
*   Restore rendering state [S2, book][S6].

### Formulas
The compositing is handled by hardware alpha blending. For the Over operator (back-to-front), the hardware performs:
$$
C_{out} = C_{src} + (1 - \alpha_{src}) C_{dst}
$$
$$
\alpha_{out} = \alpha_{src} + (1 - \alpha_{src}) \alpha_{dst}
$$
where $C_{src}, \alpha_{src}$ are the fragment's color and opacity, and $C_{dst}, \alpha_{dst}$ are the current frame buffer values [S1, unofficial][S2, book].

The transfer function lookup is a texture access:
$$
(C_i, \alpha_i) = \text{tex1D}(v_i, \text{TFTexture})
$$
where $v_i$ is the interpolated scalar value at sample $i$ [S2, book][S6].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Logic (simplified)
def fragment_shader(tex_coord_3d, tf_texture, volume_texture, shading_enabled):
    # 1. Sample volume data
    scalar_value = texture(volume_texture, tex_coord_3d)
    
    # 2. Apply transfer function
    color, opacity = texture(tf_texture, scalar_value)
    
    # 3. Optional Shading
    if shading_enabled:
        # Estimate gradient using central differences on volume texture
        # Note: Requires access to neighboring texels, typically done via 
        # multiple texture lookups or precomputed gradient textures
        normal = estimate_normal(volume_texture, tex_coord_3d)
        color = apply_blinn_phong(color, normal, light_position, view_direction)
        
    return color, opacity

# Main Render Loop
def render(volume_texture, tf_texture, camera, bbox):
    # 1. Generate Proxy Geometry
    slices = []
    num_slices = compute_num_slices(bbox, camera.view_direction, step_size)
    for i in range(num_slices):
        plane_z = i * step_size
        # Intersect plane with bbox to get polygon vertices
        vertices = intersect_plane_with_bbox(plane_z, camera.view_direction, bbox)
        # Sort vertices around centroid
        centroid = mean(vertices)
        sorted_vertices = sort_clockwise(vertices, centroid)
        # Assign texture coordinates
        tex_coords = assign_3d_tex_coords(sorted_vertices, bbox)
        slices.append((sorted_vertices, tex_coords))
    
    # 2. Sort slices back-to-front
    slices.sort(key=lambda s: distance_to_camera(s.centroid, camera))
    
    # 3. Draw
    setup_blending(source=1, dest=1-src_alpha)
    disable_depth_write()
    bind_texture(volume_texture, unit=0)
    bind_texture(tf_texture, unit=1)
    
    for vertices, tex_coords in slices:
        draw_polygon(vertices, tex_coords)
        
    restore_blending()
    enable_depth_write()
```

### Subtopics
*   **Object-Aligned Slices:** Uses three stacks of 2D textures aligned with the X, Y, and Z axes. The stack closest to the viewpoint is selected. It is fast but suffers from 3x memory usage, popping artifacts during stack swaps, and intensity variations as the camera moves [S1, S3, S4, S5, unofficial].
*   **View-Aligned Slices:** Computes slicing planes orthogonal to the current view direction. It is more memory efficient (no replication) and provides consistent sampling distance, but requires per-frame geometry computation [S1, S3, S4, unofficial].
*   **3D Texture vs. 2D Stack:** 3D textures allow trilinear interpolation and avoid replication but may have slower access speeds than 2D textures. 2D stacks use bilinear interpolation, causing artifacts at slice edges [S2, book][S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering time is proportional to the number of slices $N$ and the area of the proxy polygons. Geometry computation is $O(N)$ per frame.
*   **Memory Complexity:**
    *   View-Aligned: $O(V)$ for the volume texture, where $V$ is the number of voxels.
    *   Object-Aligned: $O(3V)$ due to data replication along three axes [S1, S3, S4, unofficial].
*   **Performance:** Generally faster than direct ray casting on older GPUs due to hardware rasterization and blending. 2D texture stacks are faster than 3D textures for access but require more memory management [S2, book][S5, unofficial].
*   **Strengths:** Leverages standard graphics pipeline; simple implementation; good performance on hardware with limited shader capabilities [S1, S4, unofficial].
*   **Limitations:** Artifacts from discrete slicing (popping, intensity variation); memory overhead for object-aligned; dependency on sampling rate [S1, unofficial][S2, book][S4, S5, unofficial].
*   **When to use:** Real-time visualization on hardware with limited ray-marching support; when memory bandwidth is a bottleneck for ray casting.
*   **When not to use:** When high-quality trilinear interpolation is critical and 3D texture performance is poor; when memory is extremely constrained and object-aligned artifacts are unacceptable.

## Background
TBVR was developed in the early 2000s to exploit the emerging 3D texture and fragment shader capabilities of consumer GPUs [S1, unofficial][S2, book][S4, unofficial]. It was popularized by works such as Krüger and Westermann (2003) and the NVIDIA GPU Gems Chapter 39 [S1, unofficial][S2, book]. It represents a shift from CPU-based volume rendering to GPU-accelerated techniques, utilizing the graphics pipeline's existing texture mapping and blending units [S4, unofficial][S6].

## Sources
[S1] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
[S2] Chapter 39. Volume Rendering Techniques | NVIDIA Developer, https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-39-volume-rendering-techniques, fetched 2026-10-26 (book)
[S3] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S4] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S5] Volume Rendering Lecture Nov-30-2005 (Spanish), https://www.ciroduran.com/personal2/wp-content/uploads/2009/07/clase-volumerendering.pdf, fetched 2026-10-29 (unofficial)
[S6] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
