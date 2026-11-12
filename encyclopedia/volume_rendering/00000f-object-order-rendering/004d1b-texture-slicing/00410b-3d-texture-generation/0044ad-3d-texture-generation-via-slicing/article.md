# 3D Texture Generation via Slicing

Also called: 3D Texture Slicing, Slice-based Volume Generation

## Overview
3D Texture Generation via Slicing is a method for constructing a 3D texture from volume data by approximating slices of that data [S1, unofficial]. It solves the problem of populating a 3D texture resource by rendering 2D slices of a scene to a 2D texture, reading the resulting pixels, and assembling them into a 3D texture [S2, unofficial]. It is described as one of the simplest methods for volume visualization [S1, unofficial]. It differs from [AR mapping to 3D texture](884-ar-mapping-to-3d-texture.md) by relying on sequential 2D render targets rather than augmented reality coordinate mapping, and from [Post-packing Optimisation](11679-post-packing-optimisation.md) by performing the assembly during generation rather than optimizing an existing packed structure.

## How it works
**Inputs:** Volume data (represented as a mesh or scene), width of the target 3D texture (assumed cubic, $N \times N \times N$).
**Outputs:** A 3D texture with format R8 containing the volume data.

**Data Structures:**
1.  `Uint8Array` of size $N \times N \times N$ (or $width^3$) to hold the assembled 3D data [S2, unofficial].
2.  2D Render Target (Framebuffer) with dimensions $N \times N$ [S2, unofficial].
3.  3D Texture object with format `R8` and dimensions $N \times N \times N$ [S2, unofficial].

**Steps:**
1.  Initialize a `Uint8Array` named `data3D` of size $width \times width \times width$ [S2, unofficial].
2.  Loop $i$ from $0$ to $width - 1$:
    a.  Set the material slice parameter to $i / width$ [S2, unofficial].
    b.  Render the mesh to the 2D render target [S2, unofficial].
    c.  Read pixels from the render target [S2, unofficial].
    d.  Extract the red channel from the pixel data. For a pixel at index $j$ in the flattened 2D array, the red channel is at `pixels[4*j]` [S2, unofficial]. Store these in `pixelsRed`.
    e.  Copy `pixelsRed` into `data3D` at the offset corresponding to slice $i$ [S2, unofficial].
3.  Create a 3D texture [S2, unofficial].
4.  Upload `data3D` to the 3D texture with format `R8` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_3d_texture_via_slicing(width, render_scene, create_texture, upload_texture):
    # Initialize 3D data array
    data3D = [0] * (width * width * width)
    
    # Create a 2D render target (framebuffer)
    render_target = create_2d_render_target(width, width)
    
    for i in range(width):
        # Set slice parameter for the current iteration
        # The slice value is normalized from 0.0 to 1.0
        slice_value = i / width
        render_scene.set_material_slice(slice_value)
        
        # Render the mesh to the 2D render target
        render_to_target(render_target, render_scene)
        
        # Read pixels from the render target
        # Assuming RGBA format, 4 bytes per pixel
        pixels = read_pixels(render_target)
        
        # Extract red channel
        pixels_red = []
        for j in range(width * width):
            # pixels is a flat array of bytes
            # Red channel is at index 4*j
            pixels_red.append(pixels[4 * j])
            
        # Copy to 3D data array
        # Offset for slice i is i * width * width
        offset_start = i * width * width
        offset_end = offset_start + width * width
        data3D[offset_start:offset_end] = pixels_red
        
    # Create 3D texture
    texture = create_texture(width, width, width, format="R8")
    
    # Upload data
    upload_texture(texture, data3D)
    
    return texture
```

**Parameters:**
-   `width`: The dimension of the cubic 3D texture. Typical values depend on the desired resolution [S2, unofficial].

**Edge Cases and Pitfalls:**
-   **Precision:** Reading pixels back from the GPU to CPU memory can introduce precision loss if the intermediate format does not match the source data precision [S2, unofficial].
-   **Boundaries:** The loop assumes a cubic volume ($width \times width \times width$). Non-cubic volumes require adjustment to the offset calculation [S2, unofficial].
-   **Degenerate Input:** If the mesh does not cover the full slice, the resulting texture may contain uninitialized or zero values [S2, unofficial].

## Subtopics

### 3D Texture Generation via Vertex Shader Iteration
A technique to render to a specific slice of a 3D texture in a single draw call by iterating over texels of a source depth buffer in the vertex shader and projecting them into the 3D volume [S3, unofficial]. It avoids the performance penalty of multiple draw calls (one per slice) and redundant sampling of the depth texture for every slice [S3, unofficial].

**Steps:**
1.  Draw points, with the number of points equal to the depth buffer resolution or less [S3, unofficial].
2.  In the vertex shader, fetch the depth buffer value for the current point [S3, unofficial].
3.  Project the fetched depth value into the cube covered by the 3D texture to determine the Z-coordinate (slice) [S3, unofficial].
4.  Set the target slice for rendering using the `gl_Layer` variable in the vertex shader [S3, unofficial].
5.  In the fragment shader, perform the desired calculations (e.g., processing color textures) and write to the 3D texture slice [S3, unofficial].

**Parameters:**
-   `gl_Layer`: A built-in variable used to specify the layer/slice index for rendering to a 3D texture or array texture [S3, unofficial].

**Pitfalls:**
-   The `gl_Layer` variable in the vertex shader is not part of the core OpenGL specification prior to version 4.2; it requires the `GL_AMD_vertex_shader_layer` extension or a geometry shader pass-through [S3, unofficial].

## Operating characteristics
**Time Complexity:** $O(N^3)$ for the slicing method, as it processes each voxel once through $N$ render passes [S2, unofficial]. The vertex shader iteration method reduces the number of draw calls to $O(1)$ but still processes $O(N^2)$ points per frame if the depth buffer is fully iterated [S3, unofficial].
**Memory Complexity:** $O(N^3)$ to store the 3D texture and intermediate pixel buffers [S2, unofficial].
**Measured Performance:** The standard slicing method is described as "terribly slow" when implemented with manual draw calls per slice due to the overhead of switching framebuffers and redundant sampling [S3, unofficial]. The vertex shader iteration method is significantly faster by avoiding these penalties [S3, unofficial].
**Strengths:** Simple to implement; works with standard 2D rendering pipelines [S1, unofficial].
**Limitations:** High CPU-GPU bandwidth usage due to reading pixels back; multiple draw calls for the standard method [S2, unofficial].
**When to use:** When simplicity is prioritized over performance, or when the volume data is static and generation is done offline [S1, unofficial].
**When not to use:** When real-time updates are required, or when the volume is large and the overhead of multiple draw calls is prohibitive [S3, unofficial].

## Background
The method is described as one of the simplest methods for volume visualization [S1, unofficial]. The vertex shader iteration variant was discussed in a 2013 Stack Exchange post as a solution to the inefficiency of rendering to 3D textures slice-by-slice [S3, unofficial].

## Sources
[S1] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27 (unofficial)
[S2] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
[S3] Efficiently rendering to 3D texture, https://gamedev.stackexchange.com/questions/63956/efficiently-rendering-to-3d-texture, fetched 2026-10-26 (unofficial)
