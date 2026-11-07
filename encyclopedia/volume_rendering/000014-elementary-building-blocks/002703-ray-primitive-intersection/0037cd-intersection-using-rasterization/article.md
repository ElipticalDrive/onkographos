# Intersection using Rasterization

Also called: GPU Rasterization Intersection, Rasterization-based Intersection

## Overview
This method leverages the parallel rasterization hardware of a GPU to compute the entry and exit points of view rays through a volumetric dataset, avoiding explicit analytical intersection tests [S1]. It differs from analytical methods like [Ray-box intersection](776-ray-box-intersection.md) by converting the geometric problem into a rendering task, where the volume boundaries are rendered as a simple primitive to generate intersection coordinates directly in the framebuffer [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset with defined spatial bounds; a camera model defining view rays for each pixel.
*   **Output:** A framebuffer containing the 3D texture coordinates (or world-space positions) of the ray entry and exit points for each pixel.

**Data Structures:**
*   **Volume Boundary Geometry:** A unit cube (or axis-aligned bounding box) representing the extent of the volume. The vertices are mapped to texture coordinates in the range $[0, 1]^3$.
*   **Framebuffer:** A render target storing the color of the rasterized pixels. Since the color values are mapped to the volume's texture coordinates, the RGB components of the framebuffer pixel represent the 3D intersection point $(u, v, w)$ in texture space.

**Steps:**
1.  **Setup View Frustum:** Configure the camera projection matrix to match the view rays used for volume rendering.
2.  **Render Boundary Cube:** Render a simple cube (or two triangles per face) representing the volume's bounding box.
    *   Apply back-face culling to ensure only the front-facing surface is rendered, yielding the entry point [S1].
    *   To obtain the exit point, render the same cube again with front-face culling (or reverse the winding order) [S1].
3.  **Interpolate Texture Coordinates:** The GPU rasterizer interpolates the vertex texture coordinates across the screen space pixels. Because the cube's vertices are assigned texture coordinates corresponding to the volume's $[0, 1]^3$ space, the interpolated color at each pixel represents the 3D texture coordinate of the intersection [S1].
4.  **Readback:** Read the framebuffer contents. The red, green, and blue channels of the pixel at $(x, y)$ correspond to the $u, v, w$ texture coordinates of the intersection point.

**Formulas:**
Let the volume bounds be $[x_{min}, x_{max}] \times [y_{min}, y_{max}] \times [z_{min}, z_{max}]$.
The vertex attributes for the boundary cube are set such that the texture coordinate $\mathbf{t} = (u, v, w)$ for a vertex corresponds to its normalized position within the volume.
For a pixel $p$, the output color $\mathbf{C}_p = (R, G, B)$ is interpreted as:
$$ u = R, \quad v = G, \quad w = B $$
The actual 3D position $\mathbf{P}$ in the volume space is:
$$ \mathbf{P} = \mathbf{P}_{min} + \mathbf{C}_p \cdot (\mathbf{P}_{max} - \mathbf{P}_{min}) $$
where $\mathbf{P}_{min} = (x_{min}, y_{min}, z_{min})$ and $\mathbf{P}_{max} = (x_{max}, y_{max}, z_{max})$.

**Parameters:**
*   **Culling Mode:** Back-face culling for entry; Front-face culling for exit.
*   **Texture Coordinate Mapping:** Linear mapping from vertex position to $[0, 1]^3$.

**Edge Cases and Pitfalls:**
*   **Precision:** The accuracy is limited by the framebuffer bit depth (e.g., 8-bit or 16-bit float). Sub-pixel accuracy is not available [S1].
*   **Degenerate Input:** If the volume has zero thickness in any dimension, the cube is degenerate and rasterization may fail or produce undefined results.
*   **Occlusion:** This method only computes the first intersection with the bounding box. It does not handle internal structures or holes within the volume unless the boundary geometry is complex.

## Subtopics

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel relative to the number of primitives, as it relies on fixed-function rasterization hardware. The cost is dominated by the fill rate of the GPU.
*   **Memory Complexity:** Requires a framebuffer for the intersection coordinates. No additional acceleration structures (like BVHs) are needed for the intersection test itself.
*   **Performance:** Highly efficient on modern GPUs due to parallel rasterization [S1]. It avoids the branching and arithmetic overhead of analytical ray-box intersection tests.
*   **Strengths:** Simple implementation; leverages existing graphics pipeline; fast for simple bounding volumes.
*   **Limitations:** Low precision due to framebuffer resolution and bit depth; only suitable for convex, simple bounding volumes; cannot handle complex internal geometry or multiple intersections without additional passes.
*   **When to use:** When a quick estimate of the entry/exit points for a simple volume is needed, and high precision is not critical.
*   **When not to use:** For complex non-convex volumes, or when high-precision intersection points are required for accurate sampling.

## Background
This technique is described in comparative studies of GPU-based volume ray casting implementations, highlighting rasterization as a viable alternative to compute-shader or CUDA-based analytical intersection methods [S1]. It utilizes standard OpenGL rasterization capabilities to offload the intersection calculation to the fixed-function pipeline.

## Sources
[S1] A Comparison between GPU-based Volume Ray Casting Implementations: Fragment Shader, Compute Shader, OpenCL and CUDA, https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
