# Rasterization-based Ray Setup

Also called: Raycast Shader Setup, RenderVolume, bounding box rasterization

## Overview
Rasterization-based ray setup generates ray origins and directions for volume ray casting by rasterizing the volume's bounding box [S2, unofficial]. It maps screen-space fragments to 3D ray parameters efficiently on the GPU [S2, unofficial]. Unlike procedural intersection methods, this approach uses identical ray setup logic for both orthogonal and perspective projections [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume 3D texture, transfer function texture, camera position (`camerapos`), inverted model matrix (`invertedmodel`), and a quad vertex buffer [S1, unofficial].
*   **Output:** Ray start position and direction vector (or start/exit positions) for subsequent ray marching [S2, unofficial].

**Data Structures:**
*   **Vertex Buffer:** Quad with two attributes per vertex: position (2 floats at offset 0 bytes) and UV coordinates (2 floats at offset 8 bytes) [S1, unofficial].
*   **Textures:** Unit 0 holds the volume 3D texture, unit 1 holds the front face depth texture, unit 2 holds the back face depth texture, and unit 3 holds the transfer function texture [S1, unofficial].

**Parameters:**
*   `camerapos`: World space camera position [S1, unofficial].
*   `invertedmodel`: Inverted model matrix for transforming camera space to world space [S1, unofficial].
*   `transferindex`: Uniform index for transfer function lookup [S1, unofficial].

**Steps:**
1.  Bind the raycast program.
2.  Bind the volume 3D texture to texture unit 0.
3.  Bind the front face depth texture to texture unit 1.
4.  Bind the back face depth texture to texture unit 2.
5.  Bind the transfer function texture to texture unit 3.
6.  Set the uniform `transferindex`.
7.  Set the uniform `camerapos` (world space).
8.  Set the uniform `invertedmodel`.
9.  Bind the quad vertex buffer with vertex attribute at offset 0 (2 floats) and UV attribute at offset 8 bytes (2 floats).
10. Draw `GL_TRIANGLE_STRIP` with 4 vertices [S1, unofficial].

**Formulas:**
The ray start position is computed by reconstructing the world-space intersection of the view ray with the front face of the bounding box [G]. The ray direction is computed from the camera position to the fragment's world position [G]. The exact reconstruction depends on the projection matrix used during depth texture generation [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mat_vec_mul(M, v):
    return [sum(M[i][j] * v[j] for j in range(4)) for i in range(4)]

def normalize(v, origin):
    diff = [v[i] - origin[i] for i in range(3)]
    length = sum(d**2 for d in diff)**0.5
    return [d / length for d in diff]

def rasterization_ray_setup(uv, front_depth, camera_pos, inverted_model, inv_proj_matrix):
    clip_x = uv[0] * 2.0 - 1.0
    clip_y = uv[1] * 2.0 - 1.0
    clip_z = front_depth
    clip_w = 1.0

    view_pos = mat_vec_mul(inv_proj_matrix, [clip_x, clip_y, clip_z, clip_w])
    view_pos = [v / view_pos[3] for v in view_pos]
    world_pos = mat_vec_mul(inverted_model, view_pos)[:3]

    ray_start = world_pos
    ray_direction = normalize(world_pos, camera_pos)
    return ray_start, ray_direction
```

**Edge Cases and Pitfalls:**
Not given in the sources. [G]

## Subtopics

### Procedural Ray/Box Intersection
Computes ray-box intersection analytically in the shader. It is chosen when avoiding the extra render pass for depth textures is preferred, but requires different logic for orthogonal versus perspective projections. [Procedural Ray/Box Intersection](9618-procedural-ray-box-intersection.md)

### Early Ray Termination
Stops ray casting when a surface is hit or opacity threshold is reached. It is chosen when computational savings from early exit outweigh the overhead of per-sample checks. [Early Ray Termination](9618-early-ray-termination.md)

### Empty Space Skipping
Skips transparent regions to reduce sample count. It is chosen when the volume contains large transparent areas and hierarchical structures or tighter bounding geometry are available. [Empty Space Skipping](9618-empty-space-skipping.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Requires storage for front and back face depth textures plus the volume texture [S1, unofficial].
*   **Strengths:** Identical logic for orthogonal and perspective projections [S2, unofficial].
*   **Limitations:** Requires an additional render pass to generate depth textures [S2, unofficial].
*   **When to Use:** Suitable for real-time volume rendering where simplicity and compatibility with standard rasterization pipelines are prioritized [S2, unofficial].
*   **When Not to Use:** Not given in the sources. [G]

## Background
The rasterization-based ray setup approach was introduced by Krüger and Westermann in 2003 [S2, unofficial]. It emerged as an alternative to procedural ray/box intersection methods proposed by Röttger et al. (2003) and Green (2004) [S2, unofficial]. This method leverages the GPU's rasterization capabilities to generate ray parameters, simplifying the shader logic and ensuring consistency across different projection types [S2, unofficial].

## Sources
[S1] mversiotech/volume-rendering-demo, https://github.com/mversiotech/volume-rendering-demo, fetched 2026-10-26 (unofficial)
[S2] PDF Volume Rendering - Isenberg, https://tobias.isenberg.cc/graphics/uploads/Class/07-volume-rendering.pdf, fetched 2026-10-28 (unofficial)
