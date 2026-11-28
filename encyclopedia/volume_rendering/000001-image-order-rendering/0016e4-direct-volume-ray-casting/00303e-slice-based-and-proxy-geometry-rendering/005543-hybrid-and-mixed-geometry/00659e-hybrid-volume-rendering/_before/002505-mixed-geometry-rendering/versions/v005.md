# Mixed Geometry Rendering

Also called: Combining geometric objects with volumetric data, Direct Volume Rendering with Z-Buffer Geometry, Intermixing Volumes and Geometry, Wireframe and Surface Combination

## Overview
Mixed Geometry Rendering integrates explicit geometric primitives (isosurfaces, bounding boxes, wireframes) with volumetric data in a single image to improve spatial understanding and context [S2, book]. It solves the occlusion and compositing problem between opaque/transparent surfaces and semi-transparent volumes by leveraging hardware depth buffers or explicit sorting [S2, book]. Unlike pure volume rendering which treats the scene as a continuous density field, or pure surface rendering which discards internal structure, this method combines the structural clarity of geometry with the internal detail of volumes [S1].

## How it works
The method adapts based on whether the volume rendering is performed via hardware texture mapping or software ray casting/splatting. Both approaches rely on the standard volume rendering pipeline defined in [Slice-based and Proxy Geometry Rendering](6225-slice-based-and-proxy-geometry-rendering.md), extended with geometric primitives.

### Inputs and Outputs
*   **Inputs**: 3D volumetric scalar field, geometric primitives (triangles, lines, bounding boxes), camera parameters, transfer functions for volume, and lighting parameters for geometry.
*   **Outputs**: A 2D image where geometry and volume are correctly composited.

### Data Structures
*   **Geometry Buffer**: Standard graphics primitive list (vertices, indices).
*   **Depth Buffer (Z-Buffer)**: Hardware buffer storing depth values for opaque geometry.
*   **Volume Texture**: 3D texture array storing scalar values [S2, book].

### Algorithm: Hardware-Based Volume Rendering (Texture Mapping)
This approach uses the GPU's rasterization pipeline for both geometry and volume slices.

1.  **Render Opaque Geometry**: Render all opaque geometric primitives (e.g., bounding boxes, solid isosurfaces) first. This populates the depth buffer with the closest surface depths [S2, book].
2.  **Sort Semi-Transparent Elements**: Sort all semi-transparent geometric primitives and volume slice polygons in back-to-front order relative to the camera. If primitives intersect, they may need to be split to maintain a valid sorted order [S2, book].
3.  **Render Semi-Transparent Geometry**: Render the sorted semi-transparent geometry using alpha blending (`GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA`) [S2, book].
4.  **Render Volume Slices**: Render the volume slices (as defined in the parent article) in back-to-front order. The fragment shader samples the 3D texture to get scalar values, maps them to color/opacity, and blends them into the frame buffer [S1][S2, book].

### Algorithm: Software-Based Volume Rendering (Ray Casting / Splatting)
This approach computes volume data on the CPU or via custom shaders that do not rely on standard texture-mapped slice rasterization for the final compositing, often used for irregular grids or complex lighting.

1.  **Render Opaque Geometry**: Render all opaque geometric primitives. Capture the resulting depth values in the hardware depth buffer [S2, book].
2.  **Volume Rendering with Depth Clipping**:
    *   For **Ray Casting**: For each pixel, retrieve the depth value $z_{geo}$ from the depth buffer. Convert this depth into a distance $t_{geo}$ along the view ray. Perform ray integration for the volume only for $t < t_{geo}$ [S2, book].
    *   For **Splatting/Object-Order**: For each volume sample (splat), determine its depth. Compare this depth to the value stored in the depth buffer at the corresponding pixel. Accumulate the sample's contribution only if the sample is in front of the geometry for that pixel [S2, book].
3.  **Compositing**:
    *   Let $C_{geo}$ be the color from the geometric rendering.
    *   Let $C_{vol}$ be the color computed from volume integration.
    *   Let $\alpha_{vol}$ be the integral of opacity along the ray (or accumulated opacity from splats).
    *   The final pixel color $C_{final}$ is computed using the over operator:
        $$ C_{final} = C_{geo} \cdot (1 - \alpha_{vol}) + C_{vol} $$
        *Note: In the specific implementation described in [S1], the stored color in the pixel (from geometry) is multiplied by $(1 - \alpha_{vol})$ and added to the volume ray color. This effectively places the volume *in front* of the geometry if the volume rendering is done after, or requires careful ordering. However, [S2, book] states the volume rendered image is blended *over* the geometric image, implying the geometry is the background layer.*
    *   Clarification from [S1]: "The stored color in the pixel is then multiplied by one minus the integral of the opacity along the ray. This product is added to color of the ray. This is the finished color of the pixel." This implies the geometry is rendered *first* (background), and the volume is rendered *on top* (foreground), but the volume's opacity determines how much of the background geometry is visible.

### Gradient Computation for Lighting (Wireframe/Structure)
To give geometric feel to the volume or wireframes, the gradient is computed at each voxel/node [S1].

1.  **Compute Differences**: For a point $P$, select three neighboring points to form a local basis. Let $A$ represent the differences in coordinates and values.
2.  **Gradient Formula**: The X component of the gradient $\nabla f_x$ is calculated as the ratio of two determinants [S1]:
    $$ \nabla f_x = \frac{\begin{vmatrix} A_{f0} & A_{y0} & A_{z0} \\ A_{f1} & A_{y1} & A_{z1} \\ A_{f2} & A_{y2} & A_{z2} \end{vmatrix}}{\begin{vmatrix} A_{x0} & A_{y0} & A_{z0} \\ A_{x1} & A_{y1} & A_{z1} \\ A_{x2} & A_{y2} & A_{z2} \end{vmatrix}} $$
    Where $A_{x}, A_{y}, A_{z}$ are coordinate differences and $A_{f}$ is the value difference between the reference point and the neighbor.
3.  **Y and Z Components**: Computed similarly by substituting the value differences into the second and third columns of the numerator determinant, respectively [S1].
4.  **Lighting**: Use the gradient vector as the surface normal in a simple lighting equation (e.g., diffuse shading) to model light interaction [S1].

### Parameters
*   **Resolution**: Performance results cited use a 512x512 pixel window [S1].
*   **Sorting**: Critical for correct alpha blending. Not given in the sources how splitting is performed algorithmically, only that it is required [S2, book].

### Edge Cases and Pitfalls
*   **Wireframe Occlusion**: Wireframe geometry completely occludes volume data behind it, although volume data in front of the wireframe correctly transmits colors from the geometry [S1].
*   **Memory Usage**: High for cell-organized data (e.g., tetrahedral meshes). A model with 381,548 cells required over 50 Megabytes [S1].
*   **Polygon Splitting**: If semi-transparent geometry intersects, simple sorting fails. Polygons must be split to obtain a valid sorted order [S2, book].

## Subtopics

### [Hybrid Scanline A-Buffer Volume Rendering](3616-hybrid-scanline-a-buffer-volume-rendering.md)
Uses scanline-based A-buffer for surfaces and volume rendering for gases, merging via Z-sorted fragments. Chosen when spatial coherence in scanlines can optimize mixed surface/volume compositing.

## Operating characteristics
*   **Time Complexity**:
    *   Volume rendering alone is $O(n^3)$ if every voxel is visited [S2, book].
    *   Performance on Stardent ST1000 (5 Mips processors):
        *   16,896 tetrahedral cells (Bearing Housing): 149.2 seconds for 512x512 image [S1].
        *   381,548 tetrahedral cells (CFD Analysis): 591.6 seconds for 512x512 image [S1].
*   **Memory Complexity**: High for cell-organized data. Over 50 MB for ~380k cells [S1].
*   **Strengths**: Provides clear structural context (bounding boxes, cut planes) alongside internal volume details. Hardware-based approach is efficient for simple geometry.
*   **Limitations**: Wireframes occlude background volume. Sorting semi-transparent geometry is complex. Software ray casting is slow on older hardware.
*   **When to use**: When understanding the spatial relationship between external structures (e.g., CFD grid, bounding box) and internal volume data is critical.
*   **When not to use**: When performance is critical and geometry is complex enough to require expensive sorting/splitting, or when the volume is the sole focus.

## Background
The technique was developed in the late 1980s and early 1990s for scientific visualization, particularly for Finite Element Models (FEM) and Computational Fluid Dynamics (CFD) [S1]. It addresses the limitation of pure volume rendering which can obscure the underlying mesh structure or domain boundaries [S1]. Key early implementations were demonstrated on the Stardent ST1000 [S1]. The VTK book formalizes the hardware and software pipelines for intermixing these data types [S2, book].

## Sources
[S1] Raytracing irregular volume data, https://doi.org/10.1145/99307.99316, fetched 2026-10-25
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
