# Flowline Extrusion
## Overview
Flowline Extrusion is a technique to extrude a path of positions into a velocity field by looping over positions, computing velocity vectors, and splatting them using instanced rendering [S1, unofficial]. It generates volumetric velocity data from discrete path points by converting vector information into additive splat contributions [S1, unofficial]. It sets itself apart from [3D Gaussian Splatting via Instancing](1586-3d-gaussian-splatting-via-instancing.md) by explicitly encoding the velocity vector as the color of the splat rather than just density, and from [Volumetric Spot Noise Rendering](2360-volumetric-spot-noise-rendering.md) by relying on deterministic path integration rather than stochastic noise functions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of 3D positions representing a flow path [S1, unofficial].
*   **Output:** A 3D texture storing the accumulated velocity field [S1, unofficial].

**Data Structures:**
*   **Volume Texture:** A 3D texture with format `GL_RGB16F`, dimensions `width × height × depth`, data type `GL_HALF_FLOAT`, and `GL_CLAMP_TO_EDGE` wrapping [S1, unofficial].
*   **Point List:** A vector of 3D points (`VectorMath::Point3`) [S1, unofficial].

**Parameters:**
*   **Blend Function:** `GL_ONE`, `GL_ONE` (additive blending) [S1, unofficial].
*   **Splat Geometry:** A quad (4 vertices) rendered as `GL_TRIANGLE_STRIP` [S1, unofficial].
*   **Instance Count:** Determined by the depth of the splat [S1, unofficial].
*   **Typical Values:** Not given in the sources.

**Steps:**
1. Initialize a 3D texture with `GL_RGB16F` format and `GL_CLAMP_TO_EDGE` wrapping [S1, unofficial].
2. Enable blending and set the blend function to `GL_ONE`, `GL_ONE` [S1, unofficial].
3. Iterate through each position `i` in the `PointList` [S1, unofficial].
4. Determine the next position `next` in the path; if `i` is the last element, wrap to the beginning [S1, unofficial].
5. Compute the velocity vector: $v = \text{next} - i$ [S1, unofficial].
6. Set the fragment shader uniform `Center` to the current position $(x, y, z, 0)$ [S1, unofficial].
7. Set the fragment shader uniform `Color` to the computed velocity vector $v$ [S1, unofficial].
8. Draw the splat using instanced rendering with `GL_TRIANGLE_STRIP`, 4 vertices, and the instance count [S1, unofficial].

**Formulas:**
*   $v = \text{next} - i$, where $v$ is the velocity vector, $\text{next}$ is the subsequent position in the path, and $i$ is the current position [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flowline_extrusion(positions, splat_program, texture_handle, instance_count):
    # Initialize 3D texture with GL_RGB16F format and GL_CLAMP_TO_EDGE wrapping
    # (Texture initialization step: Not given in the sources)
    
    # Enable additive blending
    set_render_state(blending=True)
    set_blend_equation(source_factor=ONE, destination_factor=ONE)
    
    # Bind the volume texture to the shader
    bind_texture(unit=0, target=TEXTURE_3D, handle=texture_handle)
    
    # Use the splat program
    set_shader_program(splat_program)
    
    center_loc = get_uniform_location(splat_program, "Center")
    color_loc = get_uniform_location(splat_program, "Color")
    
    num_points = len(positions)
    
    for idx in range(num_points):
        current_pos = positions[idx]
        
        # Get next position, wrapping around if at the end
        next_idx = (idx + 1) % num_points
        next_pos = positions[next_idx]
        
        # Compute velocity vector
        velocity = next_pos - current_pos
        
        # Set uniforms
        set_uniform_4f(center_loc, current_pos.x, current_pos.y, current_pos.z, 0.0)
        set_uniform_3f(color_loc, velocity.x, velocity.y, velocity.z)
        
        # Draw instanced splats
        draw_arrays_instanced(mode=TRIANGLE_STRIP, first=0, count=4, instance_count=instance_count)
        
    # Cleanup
    set_shader_program(None)
    set_render_state(blending=False)
```

**Edge Cases and Pitfalls:**
*   **Memory Usage:** Using `GL_RGB16F` for velocity fields consumes significant memory [S1, unofficial].
*   **Distance Fields:** For a true distance field, use `GL_MAX` blending and a linear falloff in the fragment shader instead of Gaussian and additive blending [S1, unofficial].

## Subtopics

### [Stream Tube Generation](2707-stream-tube-generation.md)
Solves ambiguity of line position and orientation in still images of vector fields by inflating thin stream lines into volumetric tubes. Sets itself apart from Flowline Extrusion by requiring face and normal computation, which consumes more GPU memory and vertices. Choose when spatial structure and lighting are needed to convey geometry [G].

### [Flow Volumes](7690-flow-volumes.md)
Generates volumetric primitives (prisms and tetrahedra) from advected streamlines, rendered using volume rendering to simulate smoke or dye. Sets itself apart from Stream Tube Generation by using a surface-based initialization that naturally handles flow divergence and splitting, rather than relying on a single centerline with a fixed radius. Choose when simulating volumetric effects like smoke or dye is required [G].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of positions in the path, as the method iterates once over the input list [S1, unofficial].
*   **Memory Complexity:** Dominated by the 3D texture size (`width × height × depth × 6` bytes for `RGB16F`), which consumes significant memory [S1, unofficial].
*   **Strengths:** Simple implementation; directly encodes vector direction and magnitude in the splat color [S1, unofficial].
*   **Limitations:** High memory usage for high-resolution volumes; additive blending may cause saturation if splats overlap excessively [S1, unofficial].
*   **When to use:** When generating velocity fields from known paths is required [S1, unofficial].
*   **When not to use:** When memory is constrained or when a true distance field representation is needed (use `GL_MAX` blending instead) [S1, unofficial].

## Background
Originated in the context of volumetric splatting techniques for visualizing vector fields, as described in "Volumetric Splatting" by Pride [S1, unofficial]. It builds on the concept of rendering 3D Gaussians via instancing but adapts the output to represent velocity vectors [S1, unofficial].

## Sources
[S1] Volumetric Splatting at The Little Grasshopper, https://prideout.net/blog/old/blog/index.html%40p%3D60.html, fetched 2026-10-28 (unofficial)
