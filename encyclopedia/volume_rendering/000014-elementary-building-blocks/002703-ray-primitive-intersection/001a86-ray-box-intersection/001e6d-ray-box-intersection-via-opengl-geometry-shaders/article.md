# Ray-Box Intersection via OpenGL Geometry Shaders

Also called: CubeIntersection, Front/Back Hit Textures

## Overview
This method computes the entry and exit points of rays intersecting a volume's axis-aligned bounding box (AABB) by rendering the box geometry to textures using OpenGL geometry shaders [S1, unofficial]. It solves the problem of providing fragment shaders with precise start and end texture coordinates for volume traversal without complex CPU-side ray-box intersection calculations [S2, unofficial]. It sets itself apart from sibling methods by using a single-pass geometry shader approach to compute these points, whereas alternatives like [Two-Pass Ray Entry/Exit Determination](2669-two-pass-ray-entry-exit-determination.md) use a two-pass rendering approach with separate FBOs for front and back hits [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Camera projection matrix, model-view matrix, volume bounding box geometry (cube), and a boolean flag `m_bFront` to select front (entry) or back (exit) hit computation.
- **Outputs:** Two 2D textures (`TEXTURE_FRONT_HIT` and `TEXTURE_BACK_HIT`) storing the 3D world-space hit positions (or texture coordinates) for each pixel.

**Data Structures:**
- **Hit Textures:** 2D textures of size `Width x Height` (screen resolution). Format is `GL_RGBA16F`, internal format `GL_RGBA16F`, type `GL_FLOAT` [S1, unofficial].
- **Framebuffer Object (FBO):** Contains a color attachment (the hit texture) and a depth renderbuffer of format `GL_DEPTH_COMPONENT16` [S1, unofficial].

**Steps:**
1. Create an FBO with a color attachment (`TEXTURE_FRONT_HIT` or `TEXTURE_BACK_HIT`) of format `GL_RGBA16F` and a depth renderbuffer of format `GL_DEPTH_COMPONENT16` [S1, unofficial].
2. Clear the color buffer and the depth buffer [S1, unofficial].
3. Enable `GL_CULL_FACE` and `GL_DEPTH_TEST` [S1, unofficial].
4. If `m_bFront` is true, cull `GL_BACK` faces; otherwise, cull `GL_FRONT` faces [S1, unofficial].
5. Set up the geometry shader program. The vertex shader passes the vertex position and normal. The geometry shader receives the triangle vertices, computes the ray-box intersection for the ray defined by the camera origin and the fragment's screen position, and outputs the hit position. The fragment shader writes the hit position to the color attachment [S1, unofficial].
6. Draw the cube geometry using the hit shader program, passing the Model-View-Projection (MVP) matrix as a uniform [S1, unofficial].
7. Repeat steps 1-6 for the opposite hit (front or back) by toggling `m_bFront` and using the other FBO/texture [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_ray_box_hits(camera_origin, camera_direction, mvp_matrix, cube_vertices, width, height):
    # Create FBOs and textures for front and back hits
    front_fbo = create_fbo(width, height, format=GL_RGBA16F, depth_format=GL_DEPTH_COMPONENT16)
    back_fbo = create_fbo(width, height, format=GL_RGBA16F, depth_format=GL_DEPTH_COMPONENT16)
    
    front_texture = create_texture(width, height, format=GL_RGBA16F, type=GL_FLOAT)
    back_texture = create_texture(width, height, format=GL_RGBA16F, type=GL_FLOAT)
    
    attach_texture_to_fbo(front_fbo, front_texture, attachment=GL_COLOR_ATTACHMENT0)
    attach_texture_to_fbo(back_fbo, back_texture, attachment=GL_COLOR_ATTACHMENT0)
    
    # Load shader program with geometry shader
    shader_program = load_shader_program(vertex_shader_code, geometry_shader_code, fragment_shader_code)
    
    # Compute front hits (entry points)
    bind_fbo(front_fbo)
    gl_clear_color_buffer()
    gl_clear_depth_buffer()
    gl_enable(GL_CULL_FACE)
    gl_enable(GL_DEPTH_TEST)
    gl_cull_face(GL_BACK)  # Cull back faces to get front hits
    
    use_shader(shader_program)
    set_uniform(shader_program, "mvp_matrix", mvp_matrix)
    set_uniform(shader_program, "camera_origin", camera_origin)
    draw_cube(cube_vertices)
    
    # Compute back hits (exit points)
    bind_fbo(back_fbo)
    gl_clear_color_buffer()
    gl_clear_depth_buffer()
    gl_cull_face(GL_FRONT)  # Cull front faces to get back hits
    
    use_shader(shader_program)
    set_uniform(shader_program, "mvp_matrix", mvp_matrix)
    set_uniform(shader_program, "camera_origin", camera_origin)
    draw_cube(cube_vertices)
    
    # Unbind FBOs
    bind_fbo(0)
    
    return front_texture, back_texture
```

**Geometry Shader Logic (Conceptual):**
The geometry shader receives a triangle. For each vertex, it computes the ray direction from the camera origin to the vertex. It then performs a ray-box intersection test (using the slab method from [Ray-box intersection](776-ray-box-intersection.md)) to find the entry and exit points. The hit position is written to the output. The fragment shader then writes this position to the color attachment [S1, unofficial].

**Parameters:**
- `m_bFront`: Boolean flag determining whether to compute front (entry) or back (exit) hits [S1, unofficial].

**Edge Cases and Pitfalls:**
- The depth buffer is cleared along with the color buffer, but the depth test is enabled, requiring correct depth writing from the geometry shader [S1, unofficial].
- Precision issues may arise from using 16-bit floating point textures for storing hit positions [S1, unofficial].
- The method assumes the volume bounding box is axis-aligned and aligned with the world axes [S1, unofficial].

## Subtopics

### [Two-Pass Ray Entry/Exit Determination](2669-two-pass-ray-entry-exit-determination.md)
This method determines the 3D volume texture coordinates for the entry and exit points of a ray passing through a volume's bounding box by rendering the box geometry to textures using OpenGL. It solves the problem of providing the fragment shader with the precise start and end texture coordinates required for volume traversal without complex CPU-side ray-box intersection calculations. It sets itself apart from sibling methods by using a two-pass rendering approach where the first pass stores exit coordinates in a separate Framebuffer Object (FBO), and the second pass retrieves them, whereas alternatives like [Ray-Box Intersection via OpenGL Geometry Shaders](1768-ray-box-intersection-via-opengl-geometry-shaders.md) compute these points in a single pass using geometry shaders.

## Operating characteristics
- **Time Complexity:** O(N) where N is the number of pixels in the screen resolution, as each pixel requires a ray-box intersection computation in the geometry shader.
- **Memory Complexity:** O(W * H) for storing the hit textures, where W and H are the screen dimensions.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance scales linearly with screen resolution.
- **Strengths:** Offloads ray-box intersection computation to the GPU, reducing CPU overhead. Simple to implement using standard OpenGL features.
- **Limitations:** Requires OpenGL geometry shaders, which may not be available on all hardware. Precision issues with 16-bit textures. Limited to axis-aligned bounding boxes.
- **When to use:** When GPU-based ray-box intersection is desired and OpenGL geometry shaders are available.
- **When not to use:** When high precision is required (use 32-bit textures or CPU-side computation) or when geometry shaders are not supported.

## Background
This method is described in the VRCUDA project, an unofficial implementation of volume rendering using CUDA and OpenGL [S1, unofficial]. It is used in the context of GPU-accelerated volume rendering to efficiently compute ray-box intersections for volume traversal.

## Sources
[S1] franjaviersans/VRCUDA, https://github.com/franjaviersans/VRCUDA, fetched 2026-10-27 (unofficial)
