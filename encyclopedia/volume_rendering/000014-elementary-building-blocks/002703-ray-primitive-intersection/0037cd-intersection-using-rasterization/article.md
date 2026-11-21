# Intersection using Rasterization

Also called: GPU Rasterization Intersection, Rasterization-based Intersection

## Overview
This method computes the intersection between view-rays and a volume boundary by leveraging the parallel rasterization hardware of a GPU [S1]. It solves the problem of determining ray entry and exit points for volume rendering without explicit geometric intersection tests [S1]. Unlike analytic ray-box intersection methods, it maps screen-space pixels to 3D volume coordinates by rendering a unit cube, yielding identical ray setup logic for both orthogonal and perspective projections [G].

## How it works
**Inputs and Outputs**
*   **Input:** Camera parameters (eye position, direction, projection matrix), volume bounding box defined in texture coordinates $[0,1]^3$ [G].
*   **Output:** For each pixel, the 3D texture coordinates $(u, v, w)$ of the ray entry point (and optionally exit point) on the volume boundary [S1].

**Data Structures**
*   **Geometry:** A simple cube (6 faces) representing the volume boundaries. Vertices are defined in unit space $[0,1]^3$ [S1].
*   **Attributes:** Each vertex carries color attributes $(R, G, B)$ corresponding to its 3D texture coordinates. For a unit cube, the vertex at $(u, v, w)$ has color $(u, v, w)$ [S1].

**Steps**
1.  **Setup Geometry:** Define a cube where each vertex position corresponds to a corner of the unit volume. Assign the vertex's 3D coordinates as its RGB color values [S1].
2.  **Configure Rasterization:** Enable back-face culling to ensure only the front-facing surfaces (entry points) are rendered. For exit points, render the back faces or use a separate pass [S1].
3.  **Render:** Rasterize the cube from the camera's perspective. The GPU interpolates the color attributes across the triangle surfaces [S1].
4.  **Readback:** The color value of each rasterized pixel directly represents the 3D texture coordinates of the intersection point [S1].

**Formulas**
The color $C$ of a rasterized pixel is the interpolated vertex color. For a vertex $V_i$ with position $P_i = (u_i, v_i, w_i)$, the color attribute is $C_i = (u_i, v_i, w_i)$. The interpolated color $C_{pixel}$ at a pixel is:
$$ C_{pixel} = \frac{\sum \alpha_i C_i}{\sum \alpha_i} $$
where $\alpha_i$ are the barycentric weights [G]. Since the cube is convex and the mapping is linear, $C_{pixel}$ equals the 3D texture coordinates of the intersection point [S1].

**Parameters**
*   **Culling:** Back-face culling is essential to find the first intersection [S1].
*   **Precision:** Limited by the color buffer precision (e.g., 8-bit per channel) [G].

**Edge Cases**
*   **Precision Loss:** 8-bit color buffers may introduce quantization errors in the intersection coordinates. Floating-point framebuffers can mitigate this [G].
*   **Degenerate Input:** If the camera is inside the volume, back-face culling may fail to return the correct entry point without special handling [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# 1. Define unit cube vertices in [0,1]^3
vertices = [
    (0,0,0), (1,0,0), (1,1,0), (0,1,0),
    (0,0,1), (1,0,1), (1,1,1), (0,1,1)
]
# 2. Define triangles with consistent CCW winding for outward normals
# Winding must match the culling mode to correctly identify entry vs exit faces
triangles = [
    (0,1,2), (0,2,3), # Front (z=0)
    (5,4,7), (5,7,6), # Back (z=1)
    (4,0,3), (4,3,7), # Left (x=0)
    (1,5,6), (1,6,2), # Right (x=1)
    (3,2,6), (3,6,7), # Top (y=1)
    (4,5,1), (4,1,0)  # Bottom (y=0)
]
# 3. Configure GPU rasterization pipeline
vertex_buffer = allocate_buffer(vertices)
index_buffer = allocate_buffer(triangles)
framebuffer = allocate_framebuffer(width, height, format=RGB32F)
set_culling_mode(framebuffer, BackFaces)
# 4. Rasterize and read intersection coordinates
rasterize_pipeline(vertex_buffer, index_buffer, framebuffer)
intersection_coords = framebuffer.read_pixel()
```

## Subtopics

### [Fragment Shader Volume Ray Casting](9618-fragment-shader-volume-ray-casting.md)
A volume rendering technique that decouples ray entry/exit calculation from volumetric integration using two rendering passes. It is chosen when high-performance GPU integration is required without custom compute shaders [G].

### [View-Ray Intersection via Hardware Tessellation](13938-view-ray-intersection-via-hardware-tessellation.md)
Computes intersection points between view-rays and volume block boundaries by reformulating zero-finding as a surface approximation problem. It is chosen when precise entry/exit points for non-convex or block-structured volumes are needed, deferring ray-marching to a later stage [G].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel, dominated by rasterization overhead [G].
*   **Memory Complexity:** Low; requires only vertex buffers and a framebuffer [G].
*   **Performance:** Highly efficient on GPUs due to parallel rasterization [S1].
*   **Strengths:** Simple implementation, leverages existing GPU hardware, works for any projection type [G].
*   **Limitations:** Limited by framebuffer precision, requires two passes for entry/exit, not suitable for complex non-convex volumes [G].

## Background
Originated as a technique to accelerate volume ray casting by replacing explicit ray-box intersection tests with hardware rasterization [S1]. It is commonly used in GPU-based volume rendering implementations [S1].

## Sources
[S1] A Comparison between GPU-based Volume Ray Casting Implementations: Fragment Shader, Compute Shader, OpenCL and CUDA, https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
