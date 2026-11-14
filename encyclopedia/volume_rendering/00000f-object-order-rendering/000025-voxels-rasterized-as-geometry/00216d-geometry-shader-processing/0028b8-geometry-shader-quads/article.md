# Geometry Shader (Quads)
Also called: GS Quads, Geometry Shader quads generated each frame from faces

## Overview
This method renders volumetric data by generating one quad per visible voxel face, where each input point represents a single face rather than a whole voxel [S1, S2, unofficial]. It differs from [Geometry Shader Cube Expansion](1074-geometry-shader-cube-expansion.md) by outputting only 4 vertices per primitive instead of 12, reducing buffer space and improving GPU architecture compatibility, at the cost of requiring more input data (one point per face vs. one per voxel) [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of `GL_POINTS`. Each point contains the voxel center position, packed color, and a face index indicating which of the 6 possible faces to generate [S1, S2, unofficial].
*   **Output:** A stream of 4 vertices per primitive representing the visible faces [S1, S2, unofficial].

**Data Structures:**
The input vertex structure is defined as [S1, S2, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct PointQuad {
    vec3 position;      // Voxel center in world space
    uint32_t packedColor; // Color associated with the face/voxel
    uint8_t faceIdx;    // Index 0-5 indicating which face to generate
};
```
*   `faceIdx` maps to one of the 6 face orientations [S2, unofficial]. The specific mapping is defined by the `dxs` and `dys` arrays below [S1, S2, unofficial].

**Numbered Steps:**
1.  **Retrieve Input:** Read the center position and face index from the input point `gl_in[0]` [S1, S2, unofficial].
2.  **Select Face Vectors:** Use `faceIdx` to select the corresponding `dx` and `dy` direction vectors from constant lookup tables [S1, S2, unofficial].
3.  **Transform Offsets:** Transform the half-size direction vectors by the MVP matrix to account for view and projection [S1, S2, unofficial].
4.  **Generate Vertices:** Emit 4 vertices forming a quad centered at the voxel center, offset by the transformed `dx` and `dy` vectors [S1, S2, unofficial].

**Formulas:**
*   `offset = mvp * vec4(direction * (vox_size / 2.0), 0.0)` [S1, S2, unofficial].
*   `direction` is selected from `dxs` or `dys` based on `faceIdx` [S1, S2, unofficial].
*   `vox_size` is the physical dimension of a single voxel [S1, S2, unofficial].
*   `mvp` is the Model-View-Projection matrix [S1, S2, unofficial].

**Parameters:**
*   `voxSize`: The physical size of a voxel (scalar) [S1, S2, unofficial].
*   `mvp`: The Model-View-Projection matrix (uniform) [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Vertex Limit:** This method outputs 4 vertices per primitive, down from 12 in the voxel-based geometry shader, reducing buffer space and improving GPU architecture compatibility [S1, S2, unofficial].
*   **Input Data Volume:** Requires more input points than voxel-based approaches because each face is a separate point [S2, unofficial].
*   **Culling:** The source mentions back-face culling using `IsCulled` based on the Z-component of the transformed offset vector in clip/view space (`normal.z > 0`), not the view-space normal [S1, unofficial]. The simplified code above does not include culling [S1, unofficial].

**Pseudocode:**
The following implementation assumes the geometry shader receives one point per face [S2, unofficial]. It uses the constant arrays provided in the sources [S1, S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants for the 6 face orientations
dxs = [
    (0, 1, 0), (0, 0, 1), (0, 0, 1),
    (1, 0, 0), (1, 0, 0), (0, 1, 0)
]
dys = [
    (0, 0, 1), (0, 1, 0), (1, 0, 0),
    (0, 0, 1), (0, 1, 0), (1, 0, 0)
]

def add_quad(center, offset_dx, offset_dy):
    # Emits 4 vertices forming a quad
    emit_vertex(center + (-offset_dx - offset_dy))
    emit_vertex(center + (offset_dx + offset_dy))
    emit_vertex(center + (-offset_dx + offset_dy))
    emit_vertex(center + (offset_dx - offset_dy))
    end_primitive()

def main():
    center = gl_in[0].gl_Position
    face_idx = gl_in[0].faceIdx
    local_dx = dxs[face_idx]
    local_dy = dys[face_idx]
    # Transform half-size offsets to clip space
    offset_dx = mvp * vec4(local_dx * (vox_size / 2.0), 0.0)
    offset_dy = mvp * vec4(local_dy * (vox_size / 2.0), 0.0)
    add_quad(center, offset_dx, offset_dy)
```

## Subtopics

### Geometry Shader Cube Expansion
Generates a full cube from a single point [G]. Sets it apart by outputting 12 vertices per primitive instead of 4 [S1, unofficial]. Use when minimizing input point count is prioritized over geometry shader throughput [G]. [1074-geometry-shader-cube-expansion.md]

### Velocity Visualization via Geometry Shader Extrusion
Extrudes points into lines or quads based on velocity vectors [G]. Sets it apart by animating flow data rather than rendering static voxels [G]. Use for flow visualization [G]. [1581-velocity-visualization-via-geometry-shader-extrusion.md]

### Geometry Shader Triangle Slicing (Z-Pass)
Uses triangles and depth testing for volume rendering [G]. Sets it apart by prioritizing correct depth sorting for transparency [G]. Use when transparency and correct depth sorting are critical [G]. [1681-geometry-shader-triangle-slicing-z-pass.md]

### GPU Particle Rendering via Geometry Shader
Renders particles as billboards [G]. Sets it apart by targeting unstructured particle systems rather than structured voxel grids [G]. Use for particle systems [G]. [4444-gpu-particle-rendering-via-geometry-shader.md]

### Recursive Geometry Shader Beam Generation
Generates recursive beam structures [G]. Sets it apart by using feedback loops for procedural geometry [G]. Use for procedural beam or trail effects [G]. [8395-recursive-geometry-shader-beam-generation.md]

## Operating characteristics
*   **Time Complexity:** $O(F)$, where $F$ is the number of visible faces [G]. Each face is processed in constant time by the geometry shader [G].
*   **Memory Complexity:** $O(F)$ for input buffers [G]. The output buffer size is proportional to the number of faces [G].
*   **Performance:** Reduces buffer space needs and improves GPU architecture compatibility compared to cube expansion by limiting vertex output to 4 per primitive [S2, unofficial]. Frame times are close to display lists and VAOs after optimization [S1, S2, unofficial].
*   **Strengths:** Low memory usage, good render time, simple geometry shader logic [S1, S2, unofficial].
*   **Limitations:** Higher input data volume (one point per face) [S2, unofficial]. Requires careful culling to avoid rendering back faces [S1, S2, unofficial].

## Background
This technique is an extension of the geometry shader voxel rendering method, specifically designed to optimize vertex output [S1, S2, unofficial]. It was presented by Caleb Leak in "Quads All the Way Down: Simple Voxel Rendering" [S1, S2, unofficial]. The approach breaks voxels down into individual faces, with each `GL_POINT` representing a single quad, simplifying the geometry shader logic and reducing vertex count per primitive [S1, S2, unofficial].

## Sources
[S1] Quads All the Way Down: Simple Voxel Rendering, https://medium.com/@calebleak/quads-all-the-way-down-simple-voxel-rendering-fea1e4488e26, fetched 2026-10-27 (unofficial)
[S2] here, https://www.calebleak.com/posts/quads-all-the-way-down-voxel-rendering/, fetched 2026-10-27 (unofficial)
