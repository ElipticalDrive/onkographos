# Intersection using Rasterization

Also called: GPU Rasterization Intersection, Rasterization-based Intersection

## Overview
This method computes the intersection between view-rays and a volume boundary by leveraging the parallel rasterization hardware of a GPU [S1]. It solves the problem of determining ray entry and exit points for volume rendering without explicit geometric intersection tests [S1]. Unlike analytic ray-box intersection methods, it maps screen-space pixels to 3D volume coordinates by rendering a unit cube, yielding identical ray setup logic for both orthogonal and perspective projections [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Camera parameters (eye position, direction, projection matrix), volume bounding box defined in texture coordinates $[0,1]^3$.
*   **Output:** For each pixel, the 3D texture coordinates $(u, v, w)$ of the ray entry point (and optionally exit point) on the volume boundary.

**Data Structures**
*   **Geometry:** A simple cube (6 faces) representing the volume boundaries. Vertices are defined in normalized device coordinates or unit space $[0,1]^3$.
*   **Attributes:** Each vertex carries color attributes $(R, G, B)$ corresponding to its 3D texture coordinates. For a unit cube, the vertex at $(u, v, w)$ has color $(u, v, w)$.

**Steps**
1.  **Setup Geometry:** Define a cube where each vertex position corresponds to a corner of the unit volume. Assign the vertex's 3D coordinates as its RGB color values.
2.  **Configure Rasterization:** Enable back-face culling to ensure only the front-facing surfaces (entry points) are rendered. For exit points, render the back faces or use a separate pass.
3.  **Render:** Rasterize the cube from the camera's perspective. The GPU interpolates the color attributes across the triangle surfaces.
4.  **Readback:** The color value of each rasterized pixel directly represents the 3D texture coordinates of the intersection point.

**Formulas**
The color $C$ of a rasterized pixel is the interpolated vertex color. For a vertex $V_i$ with position $P_i = (u_i, v_i, w_i)$, the color attribute is $C_i = (u_i, v_i, w_i)$. The interpolated color $C_{pixel}$ at a pixel is:
$$ C_{pixel} = \frac{\sum \alpha_i C_i}{\sum \alpha_i} $$
where $\alpha_i$ are the barycentric weights. Since the cube is convex and the mapping is linear, $C_{pixel}$ equals the 3D texture coordinates of the intersection point.

**Parameters**
*   **Culling:** Back-face culling is essential to find the first intersection.
*   **Precision:** Limited by the color buffer precision (e.g., 8-bit per channel).

**Edge Cases**
*   **Precision Loss:** 8-bit color buffers may introduce quantization errors in the intersection coordinates. Floating-point framebuffers can mitigate this.
*   **Degenerate Input:** If the camera is inside the volume, back-face culling may fail to return the correct entry point without special handling.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Define cube vertices in unit space [0,1]^3
# Each vertex has position P and color C = P
vertices = [
    (0,0,0, 0,0,0), (1,0,0, 1,0,0), (1,1,0, 1,1,0), (0,1,0, 0,1,0),
    (0,0,1, 0,0,1), (1,0,1, 1,0,1), (1,1,1, 1,1,1), (0,1,1, 0,1,1)
]
# Define triangles for the cube faces
triangles = [
    (0,1,2), (0,2,3), # Front
    (5,4,7), (5,7,6), # Back
    (4,0,3), (4,3,7), # Left
    (1,5,6), (1,6,2), # Right
    (3,2,6), (3,6,7), # Top
    (4,5,1), (4,1,0)  # Bottom
]

# Render pass
set_rasterization_state(cull_mode=BACK_FACE)
set_vertex_data(vertices)
set_index_data(triangles)
render_to_framebuffer()
# The resulting framebuffer contains the intersection texture coordinates
```

## Subtopics

### [Fragment Shader Volume Ray Casting](9618-fragment-shader-volume-ray-casting.md)
Uses this method to decouple ray entry/exit calculation from volumetric integration.

### [View-Ray Intersection via Hardware Tessellation](13938-view-ray-intersection-via-hardware-tessellation.md)
Uses rasterization primitives to generate precise entry/exit points for volume blocks.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel, dominated by rasterization overhead.
*   **Memory Complexity:** Low; requires only vertex buffers and a framebuffer.
*   **Performance:** Highly efficient on GPUs due to parallel rasterization.
*   **Strengths:** Simple implementation, leverages existing GPU hardware, works for any projection type.
*   **Limitations:** Limited by framebuffer precision, requires two passes for entry/exit, not suitable for complex non-convex volumes.

## Background
Originated as a technique to accelerate volume ray casting by replacing explicit ray-box intersection tests with hardware rasterization [S1]. It is commonly used in GPU-based volume rendering implementations [S1].

## Sources
[S1] A Comparison between GPU-based Volume Ray Casting Implementations: Fragment Shader, Compute Shader, OpenCL and CUDA, https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
