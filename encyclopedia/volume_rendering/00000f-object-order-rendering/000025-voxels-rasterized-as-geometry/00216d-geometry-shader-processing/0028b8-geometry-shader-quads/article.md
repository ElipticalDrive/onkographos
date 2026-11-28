# Geometry Shader (Quads)

Also called: GS Quads, Geometry Shader quads generated each frame from faces

## Overview
This method extends the geometry shader voxel rendering approach by decomposing voxels into individual faces, where each input `GL_POINT` represents a single quad rather than a whole voxel [S1, unofficial][S2, unofficial]. It differs from [Geometry Shader Cube Expansion](1074-geometry-shader-cube-expansion.md) by outputting only 4 vertices per primitive instead of 12, which reduces buffer space requirements and improves compatibility with GPU architectures, at the cost of requiring one input point per face rather than one per voxel [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
The input is a stream of points, where each point corresponds to a single visible face of a voxel [S1, unofficial][S2, unofficial]. The output is a stream of quads, with exactly 4 vertices generated per input point [S1, unofficial][S2, unofficial].

**Data Structures:**
The input vertex structure is defined as [S1, unofficial][S2, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct PointQuad {
    vec3 position;      // Center position of the voxel
    uint32_t packedColor; // Color data
    uint8_t faceIdx;    // Index indicating which of the 6 faces to generate
};
```
The `faceIdx` selects one of the 6 possible face orientations [S2, unofficial].

**Constant Arrays:**
Two constant arrays define the local axis vectors for the 6 face orientations [S1, unofficial][S2, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```glsl
const vec3 dxs[6] = vec3[6](
    vec3(0, 1, 0), vec3(0, 0, 1), vec3(0, 0, 1),
    vec3(1, 0, 0), vec3(1, 0, 0), vec3(0, 1, 0)
);
const vec3 dys[6] = vec3[6](
    vec3(0, 0, 1), vec3(0, 1, 0), vec3(1, 0, 0),
    vec3(0, 0, 1), vec3(0, 1, 0), vec3(1, 0, 0)
);
```

**Parameters:**
- `voxSize`: The side length of the voxel.
- `mvp`: The Model-View-Projection matrix uniform.

**Steps:**
1. Retrieve the center position from `gl_in[0].gl_Position` [S1, unofficial][S2, unofficial].
2. Retrieve the `faceIdx` from the input vertex attributes [S2, unofficial].
3. Calculate the offset vectors `dx` and `dy` in clip space using the selected indices from `dxs` and `dys`, scaled by half the voxel size and transformed by the MVP matrix [S2, unofficial]:
   $$ dx = mvp \times \text{vec4}(dxs[faceIdx] \times \frac{voxSize}{2.0}, 0.0) $$
   $$ dy = mvp \times \text{vec4}(dys[faceIdx] \times \frac{voxSize}{2.0}, 0.0) $$
4. Call `AddQuad` with the center and the calculated offsets [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Geometry Shader main function
def main():
    # 1. Retrieve center position and color from input
    center = gl_in[0].gl_Position
    color = gl_in[0].packedColor
    faceIdx = gl_in[0].faceIdx

    # 2. Define constant arrays for 6 face orientations
    dxs = [
        (0, 1, 0), (0, 0, 1), (0, 0, 1),
        (1, 0, 0), (1, 0, 0), (0, 1, 0)
    ]
    dys = [
        (0, 0, 1), (0, 1, 0), (1, 0, 0),
        (0, 0, 1), (0, 1, 0), (1, 0, 0)
    ]

    # 3. Calculate offset vectors in clip space
    dx = mvp * vec4(dxs[faceIdx] / 2.0 * voxSize, 0.0)
    dy = mvp * vec4(dys[faceIdx] / 2.0 * voxSize, 0.0)

    # 4. Generate exactly 4 vertices for the quad
    AddQuad(center, dx, dy, color)

def AddQuad(center, dx, dy, color):
    # Vertex 0: center - dx - dy
    gl_Position = center - dx - dy
    fColor = color
    emit_vertex()

    # Vertex 1: center + dx - dy
    gl_Position = center + dx - dy
    fColor = color
    emit_vertex()

    # Vertex 2: center + dx + dy
    gl_Position = center + dx + dy
    fColor = color
    emit_vertex()

    # Vertex 3: center - dx + dy
    gl_Position = center - dx + dy
    fColor = color
    emit_vertex()

    end_primitive()

# GLSL built-in stubs for Python-like pseudocode
def emit_vertex(): pass
def end_primitive(): pass
```
*Note: The source states that `AddQuad` generates exactly 4 vertices per primitive [S2, unofficial].*

**Edge Cases and Pitfalls:**
- **Vertex Output Limit:** The technique is limited to 4 vertices per primitive, down from 12 in the voxel-based technique [S1, unofficial][S2, unofficial].
- **Input Data:** Requires one input point per face, which increases the input data size compared to the voxel-based approach [S2, unofficial].
- **Face Indexing:** The `faceIdx` must correctly map to the `dxs` and `dys` arrays to generate the correct face orientation [S2, unofficial].

## Subtopics

### [Geometry Shader Cube Expansion](1074-geometry-shader-cube-expansion.md)
Generates a full cube per voxel. Sets it apart by outputting 12 vertices per primitive instead of 4. Choose when rendering entire voxel volumes at once is preferred over face-level control.

### [Velocity Visualization via Geometry Shader Extrusion](1581-velocity-visualization-via-geometry-shader-extrusion.md)
Extrudes points into lines or quads based on velocity vectors. Sets it apart by using vector data to drive geometry expansion. Choose for flow or motion visualization.

### [Geometry Shader Triangle Slicing (Z-Pass)](1681-geometry-shader-triangle-slicing-z-pass.md)
Uses triangle slicing for depth-based rendering. Sets it apart by leveraging Z-pass depth testing for slice generation. Choose for specific depth-based volumetric effects.

### [GPU Particle Rendering via Geometry Shader](4444-gpu-particle-rendering-via-geometry-shader.md)
Renders particles as billboards or sprites. Sets it apart by focusing on point-based particle systems rather than voxel faces. Choose for particle effects.

### [Recursive Geometry Shader Beam Generation](8395-recursive-geometry-shader-beam-generation.md)
Generates beams recursively using feedback loops. Sets it apart by using recursion for procedural geometry. Choose for dynamic beam or light effects.

## Operating characteristics
- **Performance:** Reduces buffer space needs compared to the voxel-based technique (4 vertices vs 12) [S1, unofficial][S2, unofficial]. Plays better with current GPU architectures [S1, unofficial][S2, unofficial]. Frame times are close to other optimized techniques after significant optimization [S2, unofficial].
- **Strengths:** Low memory usage, good render time, improved GPU compatibility [S1, unofficial][S2, unofficial].
- **Limitations:** Requires more input data (one point per face) [S2, unofficial].
- **Time Complexity:** Not given in the sources. [G]
- **Memory Complexity:** Not given in the sources. [G]
- **Output Buffer Size:** Not given in the sources. [G]

## Background
Originated as an extension to the geometry shader voxel rendering technique, breaking voxels down into individual faces [S1, unofficial][S2, unofficial]. The technique was developed to address the high memory usage and setup time of display lists, offering a low-memory alternative with competitive render times [S1, unofficial][S2, unofficial].

## Sources
[S1] Quads All the Way Down: Simple Voxel Rendering - Medium, https://medium.com/@calebleak/quads-all-the-way-down-simple-voxel-rendering-fea1e4488e26, fetched 2026-10-27 (unofficial)
[S2] here, https://www.calebleak.com/posts/quads-all-the-way-down-voxel-rendering/, fetched 2026-10-27 (unofficial)
