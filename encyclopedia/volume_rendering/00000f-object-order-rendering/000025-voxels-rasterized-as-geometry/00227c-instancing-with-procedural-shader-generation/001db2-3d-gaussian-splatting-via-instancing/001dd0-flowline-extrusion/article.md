# Flowline Extrusion


## Overview
Flowline Extrusion is a technique to generate volumetric data, specifically velocity fields, by extruding a path of positions into a flow field using instanced rendering [S1, unofficial]. It solves the problem of visualizing vector fields by accumulating velocity vectors at specific points along a trajectory into a 3D texture [S1, unofficial]. It differs from [Volumetric Spot Noise Rendering](2360-volumetric-spot-noise-rendering.md) by explicitly tracking a path and accumulating directional velocity data rather than generating stochastic noise patterns.

## How it works
**Inputs and Outputs:**
- Input: A list of 3D positions (`PointList`) representing a path.
- Output: A 3D texture (`GL_TEXTURE_3D`) storing RGB half-float values representing the accumulated velocity field.

**Data Structures:**
- **Volume Texture**: Created with `glTexImage3D` using format `GL_RGB16F` (RGB half-float) [S1, unofficial]. Dimensions are `width`, `height`, `depth`. Wrap modes are `GL_CLAMP_TO_EDGE` for S, T, and R [S1, unofficial]. Filters are `GL_LINEAR` for min and mag [S1, unofficial].
- **Renderbuffer**: A single renderbuffer (`colorbuffer`) attached to `GL_COLOR_ATTACHMENT0` of a framebuffer, linked to the volume texture [S1, unofficial].

**Steps:**
1. Initialize the 3D texture and framebuffer as described above [S1, unofficial].
2. Enable blending: `glEnable(GL_BLEND)` [S1, unofficial].
3. Set blend function: `glBlendFunc(GL_ONE, GL_ONE)` for additive accumulation [S1, unofficial].
4. Iterate through each position `i` in the `PointList`:
    a. Determine the next position `next` in the path (wrap to beginning if at end) [S1, unofficial].
    b. Compute velocity vector: $v = \text{next} - i$ [S1, unofficial].
    c. Set the `Center` uniform to the current position $(x, y, z, 0)$ [S1, unofficial].
    d. Set the `Color` uniform to the computed velocity vector $v$ [S1, unofficial].
    e. Draw the splat using `glDrawArraysInstanced` with `GL_TRIANGLE_STRIP`, 4 vertices, and `Size` instances [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume volume is initialized with GL_RGB16F texture and framebuffer attachment
# Assume splat_program is loaded with 'Center' and 'Color' uniforms

glEnable(GL_BLEND)
glBlendFunc(GL_ONE, GL_ONE)

positions = [...] # List of Vector3 points

for i in range(len(positions)):
    current_pos = positions[i]
    
    # Determine next position, wrapping around
    next_idx = (i + 1) % len(positions)
    next_pos = positions[next_idx]
    
    # Compute velocity vector
    velocity = next_pos - current_pos
    
    # Set uniforms
    glUniform4f(glGetUniformLocation(splat_program, "Center"), 
                current_pos.x, current_pos.y, current_pos.z, 0.0)
    glUniform3f(glGetUniformLocation(splat_program, "Color"), 
                velocity.x, velocity.y, velocity.z)
    
    # Render the splat
    # Size is the depth/extent of the splat instance
    glDrawArraysInstanced(GL_TRIANGLE_STRIP, 0, 4, Size)
```

**Edge Cases and Pitfalls:**
- The path must be closed or handled carefully at boundaries; the code wraps to the beginning if the end is reached [S1, unofficial].
- Precision: Using `GL_RGB16F` (half-float) is noted as "egregious usage of memory" but necessary for velocity field accuracy [S1, unofficial].
- For a true distance field, additive blending and Gaussian falloff are incorrect; one should use `GL_MAX` blending and linear falloff [S1, unofficial].

## Subtopics

### [Stream Tube Generation](2707-stream-tube-generation.md)
Stream tube generation solves the ambiguity of line position and orientation in still images by inflating thin stream lines with volume to create shaded tubes. It differs from standard stream lines by computing face normals and indices for a pentagonal cross-section, which consumes more GPU memory but allows for smooth shading that helps viewers perceive 3D orientation.

## Operating characteristics
- **Memory**: Uses significant GPU memory due to `GL_RGB16F` 3D textures for velocity fields [S1, unofficial].
- **Performance**: Relies on instanced rendering (`glDrawArraysInstanced`) which is efficient for repeated splatting [S1, unofficial].
- **Strengths**: Directly accumulates vector data into a volume, suitable for velocity field visualization [S1, unofficial].
- **Limitations**: Additive blending accumulates magnitude, which may not represent true physical flow if not normalized; not suitable for distance fields [S1, unofficial].

## Background
Originated in the context of volumetric splatting techniques for visualizing vector fields, as described in blog posts on procedural rendering [S1, unofficial].

## Sources
[S1] Volumetric Splatting at The Little Grasshopper, https://prideout.net/blog/old/blog/index.html%40p%3D60.html, fetched 2026-10-28 (unofficial)
