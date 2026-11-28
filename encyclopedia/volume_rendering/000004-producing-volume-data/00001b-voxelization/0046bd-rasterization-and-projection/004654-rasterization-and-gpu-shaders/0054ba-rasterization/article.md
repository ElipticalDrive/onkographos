# Rasterization


## Overview
Rasterization is a rendering technique that exploits the geometric properties of objects (primarily meshes) to quickly create a 2D image from 3D data [S1, unofficial]. It solves the problem of mapping continuous geometric primitives to a discrete pixel buffer by projecting vertices and interpolating face attributes, rather than simulating light transport [S1, unofficial]. It is distinguished from ray tracing by scaling with the number of visible vertices rather than pixel count, making it highly parallelizable and favored for high-performance applications like video games [S1, unofficial][S2]. However, implementing physically-based rendering (PBR) numerical methods atop rasterization is markedly harder or impossible compared to ray-tracing [S2].

## How it works
**Inputs:**
*   A set of vertices $V$ and faces $F$ (triangles or quads) defining the geometry [S1, unofficial].
*   An extrinsic camera matrix $E$ (world-to-camera transformation) [S1, unofficial].
*   An intrinsic camera matrix $K$ (projection parameters) [S1, unofficial].
*   A 2D image buffer (framebuffer) with associated depth buffer (z-buffer) [S1, unofficial][S2].

**Outputs:**
*   A discretized 2D image buffer containing color and depth values for each pixel [S2].

**Data Structures:**
*   **Vertices:** Array of 3D points $(x, y, z)$ in world space [G].
*   **Faces:** List of indices referencing vertices, defining surface connectivity [G].
*   **Z-buffer:** 2D array storing depth values for occlusion testing [S1, unofficial].

**Steps:**
1.  **Transform:** Transform each vertex $v$ from world coordinates to camera coordinates using the extrinsic matrix $E$ [S1, unofficial].
2.  **Cull:** Discard vertices or faces that are outside the view frustum or have incorrect normal orientation (back-face culling) to reduce processing load [S1, unofficial].
3.  **Project:** Project the remaining camera-space vertices to 2D image coordinates using the intrinsic matrix $K$ [S1, unofficial].
4.  **Interpolate and Rasterize:** For each visible face, interpolate attributes (such as depth) across the 2D projected triangle. Store the topmost (closest to camera) face values in the z-buffer, overwriting previous values if the new depth is smaller [S1, unofficial].

**Formulas:**
*   Camera transformation: $v_{cam} = E \cdot v$, where $v$ is a vertex in world space and $v_{cam}$ is its position in camera space [S1, unofficial].
*   Image projection: $p_{image} = K \cdot v_{cam}$, where $p_{image}$ is the projected 2D image coordinate [S1, unofficial].

**Parameters:** Not given in the sources.

**Edge cases and pitfalls:**
*   Harder to capture complex lighting effects, shadows, and reflections compared to ray casting [S1, unofficial].
*   Implementing physically-based rendering (PBR) numerical methods atop rasterization is markedly harder or impossible in some cases compared to ray-tracing [S2].
*   Precision and boundary handling during perspective division and barycentric interpolation require standard graphics safeguards [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def rasterize(vertices, faces, E, K, image_width, image_height):
    # Initialize output buffers
    color_buffer = np.zeros((image_height, image_width, 3))
    z_buffer = np.full((image_height, image_width), np.inf)

    for face in faces:
        # 1. Transform vertices to camera space
        v_cam = E @ vertices[face]

        # 2. Cull: skip if outside view frustum or back-facing [G]
        if v_cam[:, 2] <= 0:
            continue

        # 3. Project to 2D image coordinates
        v_proj = K @ v_cam
        # Perspective division [G]
        v_img = v_proj[:, :2] / v_proj[:, 2]

        # 4. Interpolate attributes and rasterize
        # Determine bounding box of projected triangle [G]
        min_x, max_x = int(np.floor(min(v_img[:, 0]))), int(np.ceil(max(v_img[:, 0])))
        min_y, max_y = int(np.floor(min(v_img[:, 1]))), int(np.ceil(max(v_img[:, 1])))
        min_x, max_x = max(0, min_x), min(image_width - 1, max_x)
        min_y, max_y = max(0, min_y), min(image_height - 1, max_y)

        for y in range(min_y, max_y + 1):
            for x in range(min_x, max_x + 1):
                # Interpolate depth and color across the 2D projected triangle [G]
                # Standard barycentric interpolation computes coverage and attributes
                # If inside triangle and depth < z_buffer[y, x]:
                #     z_buffer[y, x] = interpolated_depth
                #     color_buffer[y, x] = interpolated_color
                pass
    return color_buffer, z_buffer
```

## Subtopics

### [Rasterization-based Voxelization](908-rasterization-based-voxelization.md)
Uses rasterization to generate voxel data from meshes; chosen when GPU-accelerated mesh-to-volume conversion is needed.

### [Mesh-to-Voxel Geometry Shader](2800-mesh-to-voxel-geometry-shader.md)
GPU-based approach using geometry shaders for voxelization; chosen for on-the-fly volume generation during rendering.

### [Scanline and Geometric Primitives](10941-scanline-and-geometric-primitives.md)
Algorithmic approach to rasterizing geometric shapes; chosen for CPU-side or legacy pipeline implementations.

### [Mesh Rasterization and Triangle Intersection](11783-mesh-rasterization-and-triangle-intersection.md)
Focuses on triangle intersection logic within rasterization; chosen when precise primitive clipping and coverage testing are required.

### [Rasterization-based Voxelization Shader](13854-rasterization-based-voxelization-shader.md)
Shader implementation of rasterization for voxel generation; chosen for compute-heavy, parallel voxelization workloads.

### [LPV Voxelization Sampling](15815-lpv-voxelization-sampling.md)
Sampling techniques for Local Persistent Volumes using rasterization; chosen for dynamic lighting in voxel-based engines.

### [Proxy-GS Cluster Rasterizer](19873-proxy-gs-cluster-rasterizer.md)
Uses proxy geometry and cluster culling for efficient rasterization; chosen for massive scene culling and draw-call reduction.

### [Rasterization and GPU Shaders](11755-rasterization-and-gpu-shaders.md)
Parent article covering GPU shader integration; links to broader pipeline context.

## Operating characteristics
*   **Performance:** Scales with the number of visible vertices and primitives, not the number of pixels [S1, unofficial]. This makes it significantly faster than ray casting for scenes with many pixels but manageable geometry [S1, unofficial].
*   **Parallelization:** Highly parallelizable on GPUs, favoring its use in real-time applications like video games [S2].
*   **Limitations:** Struggles with accurate global illumination, shadows, and reflections without additional techniques [S1, unofficial]. PBR implementation is restricted compared to ray tracing [S2].
*   **Differentiability:** Can be made differentiable through "soft" rasterization techniques, enabling use in optimization and learning-based rendering [S1, unofficial].

## Background
Rasterization is an alternative to ray casting, primarily used with meshes described by vertices and faces [S1, unofficial]. It was developed as a method to quickly create images by exploiting geometric properties rather than emulating the physical image formation process [S1, unofficial]. Modern approaches have extended it to be differentiable for neural rendering applications [S1, unofficial].

## Sources
[S1] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S2] Differentiable visual computing for inverse problems and machine learning, https://doi.org/10.1038/s42256-023-00743-0, fetched 2026-10-27
