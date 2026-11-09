# Geometry Shader Cube Expansion

Also called: GS Voxels, GS-based voxelizer, Geometry Shader (Voxels), Geometry Shader quads generated each frame from voxels, Point-to-Cube voxelization

## Overview
Geometry Shader Cube Expansion is a technique that converts point streams into geometric primitives by expanding each input voxel (represented as a single point) into a full cube shell composed of up to six faces [S2, S3, unofficial]. It solves the problem of rendering volumetric data by generating visible surface geometry on the GPU, avoiding the memory overhead of storing pre-computed vertex buffers for every face [S2, S3, unofficial]. Unlike [Geometry Shader (Quads)](4369-geometry-shader-quads.md), which may break voxels down to individual face primitives, this method generates the entire cube structure from a single point primitive per voxel, optimizing memory usage by storing only center position, color, and face enable flags [S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
The method takes a stream of points (`GL_POINTS`) as input [S3, unofficial]. Each point represents a voxel and contains its center position, color, and a bitmask indicating which of the six faces (+X, -X, +Y, -Y, +Z, -Z) should be rendered [S2, S3, unofficial]. The output is a stream of triangle strips, with a maximum of 12 vertices emitted per input point (representing up to 6 quads) [S2, S3, unofficial].

**Data Structures:**
The input vertex structure is compact, totaling 25 bytes per voxel [S2, S3, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct PointVertex {
    vec3 position;      // 12 bytes: Voxel center in world or view space
    uint32_t packedColor; // 4 bytes: Color information
    uint8_t enabledFaces; // 1 byte: Bitmask for face visibility
};
```
The `enabledFaces` bitmask uses the following bit assignments [S2, S3, unofficial]:
- Bit 0x01: +X face
- Bit 0x02: -X face
- Bit 0x04: +Y face
- Bit 0x08: -Y face
- Bit 0x10: +Z face
- Bit 0x20: -Z face

**Algorithm Steps:**
1.  **Extract Voxel Data:** In the geometry shader, read the input point's position (`center`), color, and `enabledFaces` bitmask from `gl_in[0]` [S2, S3, unofficial].
2.  **Compute Extent Vectors:** Calculate the half-extent vectors for the voxel in clip space by scaling the Model-View-Projection (MVP) matrix columns by half the voxel size (`voxSize`) [S2, S3, unofficial]:
    $$dx = \frac{MVP[0]}{2.0} \times voxSize$$
    $$dy = \frac{MVP[1]}{2.0} \times voxSize$$
    $$dz = \frac{MVP[2]}{2.0} \times voxSize$$
    Here, $MVP[i]$ refers to the $i$-th column of the MVP matrix [S2, S3, unofficial].
3.  **Face Culling Check:** For each potential face, determine if it is back-facing relative to the camera. A face with normal vector $N$ is culled if $N_{view}.z > 0$ (assuming a right-handed view space where the camera looks down the negative Z axis, or specifically checking the Z component of the transformed normal) [S2, S3, unofficial]. Note: [S1, unofficial] uses a more complex normal calculation involving cross products and view space transformation, but [S2, S3, unofficial] simplify this to checking the Z-component of the extent vector itself in clip/view space [S2, S3, unofficial].
4.  **Generate Visible Faces:** Iterate through the 6 bits of `enabledFaces`. If a bit is set and the corresponding face is not culled, generate a quad for that face [S2, S3, unofficial].
5.  **Emit Quad:** For each visible face, emit 4 vertices forming a triangle strip. The vertices are computed by offsetting the center by the appropriate combinations of $dx, dy, dz$ [S2, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Simulated GLSL geometry shader logic in valid Python syntax
# Shader state simulation
gl_Position = None
fColor = None
gl_in = [None]
gColor = [None]
gEnabledFaces = [None]
voxSize = 0.25
mvp = None

def emit_vertex():
    pass # Simulates GLSL EmitVertex()

def end_primitive():
    pass # Simulates GLSL EndPrimitive()

def add_quad(center, dy, dx):
    """Emits a triangle strip for a single face."""
    global fColor
    fColor = gColor[0]
    # Vertex 0: center + dx - dy
    gl_Position = center + dx - dy
    emit_vertex()
    # Vertex 1: center - dx - dy
    gl_Position = center - dx - dy
    emit_vertex()
    # Vertex 2: center + dx + dy
    gl_Position = center + dx + dy
    emit_vertex()
    # Vertex 3: center - dx + dy
    gl_Position = center - dx + dy
    emit_vertex()
    end_primitive()

def is_culled(normal):
    """Simple back-face culling based on the Z-component of the normal/extent vector."""
    return normal.z > 0

def main():
    global gl_in, gColor, gEnabledFaces, voxSize, mvp
    center = gl_in[0].gl_Position
    faces = gEnabledFaces[0]
    # Calculate extent vectors from MVP matrix
    dx = mvp[0] / 2.0 * voxSize
    dy = mvp[1] / 2.0 * voxSize
    dz = mvp[2] / 2.0 * voxSize
    # Check +X face (Bit 0x01)
    if (faces & 0x01) and not is_culled(dx):
        add_quad(center + dx, dy, dz)
    # Check -X face (Bit 0x02)
    if (faces & 0x02) and not is_culled(-dx):
        add_quad(center - dx, dz, dy)
    # Check +Y face (Bit 0x04)
    if (faces & 0x04) and not is_culled(dy):
        add_quad(center + dy, dz, dx)
    # Check -Y face (Bit 0x08)
    if (faces & 0x08) and not is_culled(-dy):
        add_quad(center - dy, dx, dz)
    # Check +Z face (Bit 0x10)
    if (faces & 0x10) and not is_culled(dz):
        add_quad(center + dz, dx, dy)
    # Check -Z face (Bit 0x20)
    if (faces & 0x20) and not is_culled(-dz):
        add_quad(center - dz, dy, dx)
```

**Parameters:**
- `voxSize`: The side length of the voxel cube. Default value is 0.25 [S2, S3, unofficial].
- `mvp`: The Model-View-Projection matrix [S2, S3, unofficial].

**Edge Cases and Pitfalls:**
- **Culling Logic:** The simple `is_culled` check (`normal.z > 0`) assumes a specific view space orientation. If the camera orientation changes significantly or the coordinate system differs, this culling may fail or cull visible faces [S2, S3, unofficial]. [S1, unofficial] provides a more robust normal calculation using cross products and view space transformation, which is safer for arbitrary camera orientations [S1, unofficial].
- **Interior Voxelization:** This method only generates the shell of the voxel (surface representation). It does not fill the interior, making it unsuitable for deep voxelization or ray-marching inside the volume without additional techniques [S1, unofficial].
- **Performance:** Emitting up to 12 vertices per primitive can be inefficient on some GPU architectures compared to limiting output to 4 vertices per primitive (as seen in [Geometry Shader (Quads)](4369-geometry-shader-quads.md)) [S3, unofficial].

## Subtopics

### [Active Cell Geometry Generation](3887-active-cell-geometry-generation.md)
Generates cube geometry for active voxels in a volume dataset using a flag volume. It builds on this method by handling activity flags rather than raw triangle input.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels. Each voxel is processed independently in the geometry shader [S2, unofficial].
- **Memory Complexity:** 25 bytes per voxel [S2, S3, unofficial]. This is significantly lower than previous techniques which required 96-576 bytes per voxel depending on the number of faces [S2, S3, unofficial].
- **Performance:** On a GTX 980 Ti with 16GB RAM and 1920x1080 resolution, geometry shaders provide low memory usage and render times close to display lists, though display lists may still be faster in raw frame time [S2, S3, unofficial].
- **Strengths:** Low memory footprint, dynamic generation of visible faces, simple implementation [S2, S3, unofficial].
- **Limitations:** Only suitable for shell voxelization (surface), potential aliasing and holes compared to precise ray-casting voxelizers [S1, unofficial], culling logic may need adjustment for different view spaces [S1, S3, unofficial].
- **When to use:** When memory is constrained and only the surface of a voxelized mesh needs to be rendered, such as in voxel art or real-time visualization of sparse volumes [S1, S2, unofficial].
- **When not to use:** When deep voxelization (interior rendering) is required, or when higher geometric accuracy is needed than what cube approximation provides [S1, unofficial].

## Background
The technique was developed as a method to reduce memory usage in voxel rendering by generating geometry on the fly [S2, S3, unofficial]. It was discussed in the context of comparing various voxel rendering techniques, highlighting the trade-offs between memory usage and render time [S2, S3, unofficial]. The author compared this method against a header-only C library (`vx_voxelize`), finding the library produced better results due to more precise surface approximation, as the geometry shader method approximates the mesh surface with cubes centered on triangle centroids, leading to aliasing and holes [S1, unofficial].

## Sources
[S1] Mesh Voxelization - Geeks3D, https://www.geeks3d.com/20160531/mesh-voxelization/, fetched 2026-10-26 (unofficial)
[S2] Quads All the Way Down: Simple Voxel Rendering - Medium, https://medium.com/@calebleak/quads-all-the-way-down-simple-voxel-rendering-fea1e4488e26, fetched 2026-10-27 (unofficial)
[S3] here, https://www.calebleak.com/posts/quads-all-the-way-down-voxel-rendering/, fetched 2026-10-27 (unofficial)
