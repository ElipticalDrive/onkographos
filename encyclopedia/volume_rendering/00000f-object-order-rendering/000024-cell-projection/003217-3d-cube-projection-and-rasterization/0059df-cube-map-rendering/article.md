# Cube Map Rendering

Also called: Cubemap-space ray marching, Cube Map-space rendering

## Overview
Cube Map Rendering is an object-order volume rendering technique that projects the 3D scene onto the six faces of a cube map, performs ray marching or rasterization within that cubemap space, and then maps the result back to the screen. It solves the problem of efficiently rendering volumetric data or complex geometry by leveraging the GPU's native cube map sampling hardware. Unlike [2D Raycasting with Z-Buffering](8233-2d-raycasting-with-z-buffering.md), which operates directly in screen space, this method decouples the volume traversal from the final pixel projection, allowing for optimized lighting passes and consistent sampling rates across the field of view.

## How it works
The method utilizes the graphics pipeline to render a cube map representation of the volume.

**Inputs and Outputs:**
- **Inputs:** 3D volume data (or geometry), camera position, camera orientation, cube map resolution ($S \times S$ per face), and shader programs.
- **Outputs:** A 2D screen image derived from the cube map faces.

**Data Structures:**
- **Cube Map:** A 2D texture array or 6 separate 2D textures, each of size $S \times S$.
- **Constant Buffers (CBVs):**
    - Per-object data (e.g., volume bounds, density scale).
    - Per-frame data (e.g., camera position, view matrix).
    - Sample resolution ($CB\_SAMPLE\_RES$).
- **Shader Resource View (SRV):** Bound to the cube map at a specific Level of Detail (LOD).
- **Sampler:** Linear filtering, clamp addressing mode.

**Steps:**
1.  **Setup Shaders:** Set Vertex Shader `VS_CUBE` and Pixel Shader `PS_CUBE` [S1, unofficial].
2.  **Set Topology:** Configure primitive topology to `TRIANGLESTRIP` [S1, unofficial].
3.  **Configure Rasterizer:** Set rasterizer state to `CULL_FRONT` [S1, unofficial].
4.  **Configure Depth/Blend:** Set depth stencil state to `NONE` and blend state to `PREMULTIPLIED` [S1, unofficial].
5.  **Bind Resources:**
    - Bind CBVs for per-object, per-frame, and sample resolution to slots 0 in both VS and PS [S1, unofficial].
    - Bind the cube map SRV to slot 0 in PS [S1, unofficial].
    - Bind a `LINEAR_CLAMP` sampler to slot 0 in PS [S1, unofficial].
6.  **Render Faces:** Draw 4 vertices with 6 instances, corresponding to the 6 faces of the cube [S1, unofficial].
7.  **Switch Culling:** Set rasterizer state to `CULL_BACK` for subsequent rendering passes if required [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode based on [S1, unofficial]
def render_cube_map_command_list(command_list, fluid, frame_index):
    # Clear render target and depth
    command_list.clear_depth_stencil_view(depth_view, clear_flag=DEPTH, depth=1.0f)
    command_list.set_render_targets([render_target_view], [depth_view])
    
    # Set viewport and scissor
    viewport = Viewport(0.0f, 0.0f, width, height)
    scissor_rect = RectRange(0, 0, width, height)
    command_list.rasterizer_set_viewports([viewport])
    command_list.rasterizer_set_scissor_rects([scissor_rect])
    
    # Render light probe if active
    if light_probe_active:
        light_probe.render_environment(command_list, frame_index)
        
    # Select rendering method
    if render_method == RAY_MARCH_MERGED:
        fluid.render(command_list, frame_index, mode=Fluid::RAY_MARCH_CUBEMAP)
    elif render_method == RAY_MARCH_SEPARATE:
        fluid.render(command_list, frame_index, mode=Fluid::OPTIMIZED)
    elif render_method == RAY_MARCH_DIRECT_MERGED:
        fluid.render(command_list, frame_index, mode=Fluid::RAY_MARCH_DIRECT)
    # ... other cases

def fluid_render_cubemap(command_list, frame_index, mode):
    # 1. Set Shaders
    command_list.set_vertex_shader(VS_CUBE)
    command_list.set_pixel_shader(PS_CUBE)
    
    # 2. Set Topology
    command_list.set_primitive_topology(TRIANGLESTRIP)
    
    # 3. Set Rasterizer State (Cull Front)
    command_list.set_rasterizer_state(CULL_FRONT)
    
    # 4. Set Depth/Blend State
    command_list.set_depth_stencil_state(NONE)
    command_list.set_blend_state(PREMULTIPLIED)
    
    # 5. Bind CBVs
    command_list.set_constant_buffer_vs(slot=0, buffer=per_object_cb)
    command_list.set_constant_buffer_vs(slot=0, buffer=per_frame_cb)
    command_list.set_constant_buffer_vs(slot=0, buffer=sample_res_cb)
    command_list.set_constant_buffer_ps(slot=0, buffer=per_object_cb)
    command_list.set_constant_buffer_ps(slot=0, buffer=per_frame_cb)
    command_list.set_constant_buffer_ps(slot=0, buffer=sample_res_cb)
    
    # 6. Bind SRV and Sampler
    command_list.set_shader_resource_view_ps(slot=0, resource=cube_map_srv)
    command_list.set_sampler_ps(slot=0, sampler=LINEAR_CLAMP)
    
    # 7. Draw
    # 4 vertices, 6 instances (one for each cube face)
    command_list.draw_instanced(vertex_count=4, instance_count=6)
    
    # 8. Switch Culling for subsequent operations
    command_list.set_rasterizer_state(CULL_BACK)
```

**Parameters:**
- $S$: Cube map face resolution. Typical values depend on hardware capabilities and memory constraints. Not given in the sources.
- LOD: Level of Detail for the cube map texture. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point precision in the vertex shader when transforming cube vertices can affect alignment. [G] Ensure consistent winding order to avoid culling issues.
- **Boundaries:** The `LINEAR_CLAMP` sampler handles out-of-bounds access by clamping, which may introduce artifacts at the edges of the cube faces if the volume does not perfectly fit the cube. [G] Normalize coordinates carefully to avoid sampling outside the valid volume bounds.

## Subtopics

### Equirectangular Rendering via Cube Map Projection
This variant renders a 360-degree equirectangular image by projecting the scene onto a cube map and then mapping the cube faces to the equirectangular format. It sets up rotation matrices for the six cube faces relative to the base camera orientation using $R_f = \text{rotation\_matrix}(\text{axis}, \text{angle\_deg})$ [S2, unofficial]. For each face, the face pose is computed as $face\_pose = m \otimes base\_poses$ where $m[:3,:3] = R_f^T$ [S2, unofficial]. The 3D vector is mapped to 2D UV coordinates on the hit face using specific lambda functions (e.g., for the front face: $u = v.x/v.z, v = v.y/v.z$) [S2, unofficial]. Sampling is performed using `F.grid_sample` with a grid in the range $[-1, 1]$ and `align_corners=True` [S2, unofficial]. The intrinsics tensor is of shape $(B, 4)$ with values $[S/2, S/2, S/2, S/2]$, representing $f_x, f_y, c_x, c_y$ for a 90-degree FOV [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the cube map resolution $S$ and the number of samples per ray. The rendering of the 6 faces is constant in terms of draw calls (1 draw call with 6 instances) [S1, unofficial].
- **Memory Complexity:** Requires storage for the cube map texture ($6 \times S \times S \times \text{channels}$) and constant buffers.
- **Performance:** Efficient for hardware that supports cube map sampling. The use of `TRIANGLESTRIP` and instancing reduces CPU overhead [S1, unofficial].
- **Strengths:** Leverages GPU hardware for texture sampling; decouples volume traversal from screen-space projection.
- **Limitations:** Requires additional memory for the cube map; mapping back to screen space may introduce distortion if not handled correctly.
- **When to use:** Suitable for environments where cube map sampling is available and efficient, such as in real-time rendering of volumetric effects or environment mapping.
- **When not to use:** When memory is constrained or when the scene geometry is highly irregular and does not fit well into a cube map structure.

## Background
Cube Map Rendering is used in various graphics applications, including real-time fluid simulation and environment mapping. The specific implementation described here is part of the FluidX12 project, which demonstrates cubemap-space ray marching for fluid rendering [S1, unofficial]. The variant involving equirectangular projection is used in the spirulae-splat project for 3D Gaussian Splatting [S2, unofficial].

## Sources
[S1] StarsX/FluidX12, https://github.com/StarsX/FluidX12, fetched 2026-10-26 (unofficial)
[S2] notlasandu/spirulae-splat, https://github.com/notlasandu/spirulae-splat, fetched 2026-10-27 (unofficial)
