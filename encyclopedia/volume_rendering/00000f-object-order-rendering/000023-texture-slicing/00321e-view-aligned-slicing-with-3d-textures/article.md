# View-Aligned Slicing with 3D Textures

Also called: View-Aligned Slicing, View-Orthogonal Slicing, View-Aligned Quad Slicing

## Overview
View-aligned slicing with 3D textures is a hardware-accelerated volume rendering technique that renders the volume as a series of parallel planes orthogonal to the current view direction. Unlike object-aligned slicing, which renders slices parallel to the volume axes, this method ensures that each slice covers the entire visible extent of the volume in screen space, minimizing overdraw and maintaining consistent pixel coverage regardless of camera orientation. It builds on [Texture slicing](25-texture-slicing.md) by using 3D textures for data storage and view-aligned proxy geometry for rasterization.

## How it works
**Inputs and Outputs:**
- **Input:** 3D scalar volume data, current modelview matrix, volume bounding box (or center and size), and a slicing interval (distance between slices).
- **Output:** A composited 2D image in the frame buffer.

**Data Structures:**
- **3D Texture:** Stores the scalar volume data. The texture format is typically `GL_RGBA` (pre-classified) or `GL_COLOR_INDEX8_EXT` (paletted). Dimensions match the volume resolution $(N_x, N_y, N_z)$.
- **Modelview Matrix:** A $4 \times 4$ column-major float array `mat[16]` representing the current camera transformation.
- **Proxy Geometry:** A set of polygons (quads or triangle fans) representing the slicing planes.

**Steps:**

1.  **Retrieve View Orientation:**
    Get the current modelview matrix. Extract the right, up, and eye (view direction) vectors from the matrix columns. Assuming a column-major layout where vectors are stored in columns 0, 1, and 2:
    $$
    \begin{aligned}
    \mathbf{r} &= (mat[0], mat[4], mat[8]) \\
    \mathbf{u} &= (mat[1], mat[5], mat[9]) \\
    \mathbf{e} &= (mat[2], mat[6], mat[10])
    \end{aligned}
    $$
    [S1, unofficial]

2.  **Define Slice Plane:**
    Determine the position of the current slice. Let $d$ be the distance from the volume center along the view direction $\mathbf{e}$. The plane equation is defined by normal $\mathbf{n} = \mathbf{e}$ and distance $D = \mathbf{c} \cdot \mathbf{e} - d$, where $\mathbf{c}$ is the volume center. Alternatively, for view-aligned quads, the plane is implicitly defined by the center point $\mathbf{c}$ and the normal $\mathbf{e}$.
    [S1, unofficial]

3.  **Generate Proxy Geometry:**
    There are two common approaches to generate the slice geometry:

    **Approach A: View-Aligned Quad (Simple)**
    Create a quad centered at the slice position, oriented orthogonal to the view.
    Let $\mathbf{c}$ be the center of the slice (e.g., volume center offset by $d \cdot \mathbf{e}$).
    Let $s$ be the `quad_size` (half-width/height of the quad in world space).
    The four vertices are:
    $$
    \begin{aligned}
    \mathbf{v}_0 &= \mathbf{c} + (-\mathbf{r} - \mathbf{u}) \cdot s \\
    \mathbf{v}_1 &= \mathbf{c} + (\mathbf{r} - \mathbf{u}) \cdot s \\
    \mathbf{v}_2 &= \mathbf{c} + (\mathbf{r} + \mathbf{u}) \cdot s \\
    \mathbf{v}_3 &= \mathbf{c} + (-\mathbf{r} + \mathbf{u}) \cdot s
    \end{aligned}
    $$
    [S1, unofficial]

    **Approach B: Bounding Box Intersection (Precise)**
    Compute the intersection of the slicing plane with the volume's bounding box.
    1. Define the plane equation $(a,b,c,d)$ using the view vector $(a,b,c)$ and the distance from the center.
    2. Intersect the plane with each of the 12 edges of the volume's axis-aligned bounding box.
    3. Collect the intersection points.
    4. Sort the intersection points in counter-clockwise order around the plane normal.
    5. Render the sorted vertices as a triangle fan.
    [S1, unofficial]

4.  **Rasterize and Sample:**
    Rasterize the proxy geometry. The fragment shader samples the 3D texture at the interpolated texture coordinates. The texture coordinates are typically derived from the world-space position of the fragment mapped into the volume's local coordinate system.

5.  **Composite:**
    Use hardware alpha blending to composite the fragment color and opacity into the frame buffer. The blending equation is:
    $$ C_{dest} = C_{src} \cdot A_{src} + C_{dest} \cdot (1 - A_{src}) $$
    [S3, book]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_view_aligned_slices(volume_texture, modelview_matrix, volume_center, quad_size, num_slices, slice_spacing):
    """
    Renders volume using view-aligned quads.
    """
    # 1. Extract basis vectors from modelview matrix (column-major)
    # mat is a 16-element list/array
    mat = modelview_matrix
    
    # Right vector (column 0)
    right = Vector3(mat[0], mat[4], mat[8])
    # Up vector (column 1)
    up = Vector3(mat[1], mat[5], mat[9])
    # Eye/View vector (column 2)
    eye = Vector3(mat[2], mat[6], mat[10])
    
    # 2. Iterate through slices
    for i in range(num_slices):
        # Calculate slice distance from center
        distance = (i - num_slices / 2.0) * slice_spacing
        
        # Center of the current slice in world space
        slice_center = volume_center + eye * distance
        
        # 3. Compute quad vertices
        # vec0 = center + (-right - up) * size
        v0 = slice_center + (-right - up) * quad_size
        # vec1 = center + (right - up) * size
        v1 = slice_center + (right - up) * quad_size
        # vec2 = center + (right + up) * size
        v2 = slice_center + (right + up) * quad_size
        # vec3 = center + (-right + up) * size
        v3 = slice_center + (-right + up) * quad_size
        
        # 4. Render the quad
        # The texture coordinates must be mapped such that the quad covers the volume extent
        # in the texture space. This usually requires transforming the vertices into 
        # volume-local coordinates or using a custom vertex shader to compute texcoords.
        # For simplicity, assuming a standard quad with texcoords (0,0) to (1,1) 
        # mapped to the volume bounds in the shader.
        draw_quad(v0, v1, v2, v3)
        
    # 5. Enable blending for compositing
    enable_blend()
    set_blend_func(SRC_ALPHA, ONE_MINUS_SRC_ALPHA)
```

**Parameters:**
- `quad_size`: Half-width/height of the view-aligned quad in world units. Determines the extent of the slice geometry. [S1, unofficial]
- `slice_spacing`: Distance between consecutive slices.
- `num_slices`: Number of slices to render.

**Edge Cases and Pitfalls:**
- **Rotational Artifacts:** Rotating the volume without updating the view-aligned slice geometry correctly causes visual artifacts such as popping or the volume disappearing. The slice geometry must be recomputed every frame based on the current modelview matrix. [S1, unofficial]
- **Precision:** Floating-point precision errors in matrix extraction or vertex computation can cause gaps between slices or incorrect occlusion.
- **Boundary Handling:** If the quad size is too small, parts of the volume may not be covered. If too large, unnecessary overdraw occurs.

## Subtopics

### Paletted 3D Texture Rendering
A rendering technique using a 3D texture where the index is identical to the scalar value, pointing to a color/opacity palette (transfer function) applied during runtime. The 3D texture is created with format `GL_COLOR_INDEX8_EXT`, where the index size is a byte identical to the scalar value. A 1D transfer function lookup table is defined using the OpenGL extension `glColorTableEXT`. The palette size is 256 entries, each 4 bytes (RGBA), corresponding to the 8-bit index. This allows simple and fast changes to the transfer function at runtime, unlike pre-classified textures which require complete redefinition of texture data. [S2, unofficial]

## Operating characteristics
- **Time Complexity:** $O(N_s \cdot N_p)$, where $N_s$ is the number of slices and $N_p$ is the number of pixels covered by the slice geometry.
- **Memory Complexity:** $O(N_x \cdot N_y \cdot N_z)$ for the 3D texture storage.
- **Performance:** Fast for moderately sized data sets due to hardware acceleration. Interactive exploration is possible. [S2, unofficial]
- **Strengths:** Consistent pixel coverage regardless of view angle; minimal overdraw compared to object-aligned slicing when viewing from arbitrary angles; simple implementation.
- **Limitations:** Requires frequent geometry regeneration on camera rotation; may suffer from popping if not updated correctly; limited to linear sampling unless higher-order interpolation is used in the shader.
- **When to use:** When interactive rotation is required and the volume data is stored in 3D textures.
- **When not to use:** When the volume is extremely large and memory bandwidth is a bottleneck, or when non-linear slicing planes are required.

## Background
Originating from early OpenGL volume rendering techniques, view-aligned slicing leverages the fixed-function pipeline's ability to rasterize geometry and sample textures. It was popularized in the early 2000s as a method to achieve interactive frame rates on consumer graphics hardware. The technique is discussed in community forums and educational resources on direct volume rendering. [S1, unofficial][S2, unofficial]

## Sources
[S1] view aligned slices - OpenGL: Advanced Coding, https://community.khronos.org/t/view-aligned-slices/38447, fetched 2026-10-28 (unofficial)
[S2] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
