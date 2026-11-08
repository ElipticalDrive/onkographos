# Rasterizing Proxy Geometry

Also called: Liu et al. proxy rasterization

## Overview
Rasterizing proxy geometry converts volumetric data into simple geometric primitives (lines, planes, or view-dependent cuts) to leverage the graphics pipeline for specific sampling or visibility tasks, rather than full volume rendering. It solves the problem of efficiently performing linear probes, slicing, or hierarchical culling by mapping 3D volumetric queries to 2D rasterization outputs. It differs from standard ray casting by using hardware rasterization to determine visibility and sampling positions, and from naive voxel rendering by using abstract primitives (like a single line) instead of explicit voxel cubes.

## How it works
The method relies on generating specific proxy primitives that represent the desired sampling path or spatial cut through the volume.

**Inputs and Outputs:**
*   **Input:** Volumetric scalar field data, camera parameters (view/projection matrices), and proxy geometry definition (e.g., line endpoints, plane equation).
*   **Output:** A 1D array of sampled values (for lines) or a visibility buffer (for planes/cuts), typically copied to main memory for UI interaction or further processing.

**Data Structures:**
*   **Proxy Geometry:** Defined by vertex positions in object space. For a line probe, this is a 2-vertex line segment. For a plane slice, this is a quad or triangle covering the view frustum.
*   **Frame Buffer:** A back buffer (typically 1D or 2D texture/render target) stores the fragment shader output. For line probes, this is a 1D texture aligned with the rasterized line.

**Steps:**
1.  **Define Proxy Geometry:** Create geometric primitives that represent the sampling path.
    *   *Line Probe:* Define a line segment in object space.
    *   *Plane Slice:* Define a plane intersecting the volume.
    *   *View-Dependent Cut (Liu et al.):* Define geometry representing a cut of octree nodes based on the current view [S1, unofficial].
2.  **Transform and Align:**
    *   For line probes, rotate the object space (or the proxy geometry) so that the desired sampling line is parallel to the image plane (view direction) [S2]. This ensures the rasterized line traverses the volume along the desired axis in screen space.
    *   Apply the Model-View-Projection (MVP) matrix to transform vertices to clip space [G].
3.  **Rasterize:**
    *   Submit the proxy geometry to the graphics pipeline.
    *   Enable depth testing if occlusion is required (e.g., for picking the first surface).
    *   The fragment shader samples the volume at the interpolated world position of each fragment.
    *   Output the sampled value (e.g., density, alpha) to the back frame buffer [S2].
4.  **Readback and Process:**
    *   Copy the relevant portion of the frame buffer (e.g., the 1D line data) from GPU memory to main memory [S2].
    *   Scan the resulting data array for interaction criteria:
        *   First non-zero alpha value (surface detection) [S2].
        *   First gradient with large magnitude (edge detection) [S2].
        *   Sum of alpha values exceeding a threshold (e.g., 1.0 for infinite absorption) [S2].
    *   For hierarchical culling (Liu et al.), reduce the visibility buffer to a list of active page IDs or non-empty segments [S1, unofficial].

**Formulas:**
*   **World Position for Sampling:** The fragment shader computes the world position $P_{world}$ of the fragment to sample the volume. If the proxy line is aligned with the view, the depth buffer value $z_{clip}$ can be unprojected to find the depth in world space.
*   **Line Alignment:** The rotation matrix $R$ is applied such that the line direction vector $v_{line}$ becomes parallel to the view vector $v_{view}$ in clip space [S2].

**Parameters:**
*   **Resolution:** The length of the rasterized line determines the sampling resolution. Higher screen resolution yields finer sampling [S2].
*   **Thresholds:** Alpha thresholds for absorption (e.g., 1.0) [S2].

**Edge Cases and Pitfalls:**
*   **Deformation:** If the volume is deformed, the proxy geometry must account for the deformation. Relying on deforming the mesh of the proxy geometry requires complex approaches; instead, the sampling in the fragment shader should handle the deformation [S2].
*   **Precision:** Copying data from frame buffer to main memory can be a bottleneck. Use efficient readback methods or compute shader reductions if possible [S1, unofficial].
*   **Occlusion:** Depth testing must be enabled if only the front surface is of interest.

## Subtopics

### [Proxy Mesh Rendering for Page Visibility](8386-proxy-mesh-rendering-for-page-visibility.md)
Proxy Mesh Rendering for Page Visibility determines which virtual memory pages are visible in a scene by rendering a simplified proxy mesh to a visibility buffer and reducing that buffer to a list of active page IDs. It differs from standard visibility buffers by encoding depth information to support Level of Detail (LOD) selection and by utilizing a compute shader reduction step to minimize memory bandwidth, addressing the performance bottlenecks of atomic operations in concurrent thread updates.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of fragments rasterized (proportional to screen resolution for lines/planes). The readback and scanning step is also $O(N)$.
*   **Memory Complexity:** $O(N)$ for the frame buffer and readback buffer.
*   **Performance:** Very fast for single-line probes or simple slices as it leverages hardware rasterization. Avoids complex ray traversal logic on the CPU or GPU compute shaders for simple queries.
*   **Strengths:** Simple implementation, leverages existing graphics hardware, efficient for interactive picking and slicing.
*   **Limitations:** Limited to specific sampling paths (lines, planes). Not suitable for full volume rendering. Readback to main memory can be slow if done frequently.
*   **When to use:** Interactive picking, slicing, simple visibility culling.
*   **When not to use:** Full volume rendering, complex multi-path sampling.

## Background
The concept of using proxy geometry for volume visualization was introduced by Liu et al. in "Integrating Dynamic Deformations into Interactive Volume Visualization" (2006) [S2]. They demonstrated using lines and planes as proxy geometry for linear probes and slicing. Later, Liu et al. [LCDP13] proposed rasterizing proxy geometry representing view-dependent cuts of octree nodes for efficient sparse voxel rendering [S1, unofficial]. This approach has been adopted in various systems for interactive volume exploration and hierarchical culling.

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Integrating Dynamic Deformations into Interactive Volume Visualization, https://doi.org/10.2312/vissym.eurovis06.219-226, fetched 2026-10-31
