# GPU-based Tiled Ray Casting with Depth Peeling

Also called: Depth Peeling Ray Casting, GPU-based Tiled Ray Casting, GPU-based Tiled Ray Casting using Depth Peeling, Ray Casting for TVSF, Tiled Ray Casting, Tiled Ray Casting using Depth Peeling, Tiled Ray Casting with Depth Peeling

## Overview
This method is a hardware-accelerated ray casting algorithm for volume rendering of time-varying scalar fields on unstructured tetrahedral meshes [S1, S2, unofficial]. It solves the problem of rendering non-convex meshes on GPUs by combining screen tiling for performance with depth peeling to correctly handle ray re-entry into the mesh [S2, unofficial]. It sets itself apart from its sibling [Hardware-Accelerated Volume Rendering](5616-hardware-accelerated-and-gpu-implementations.md) by explicitly addressing non-convexity without convexification, and from [WebGL Path Tracing Volume Rendering](8592-webgl-path-tracing-volume-rendering.md) by using deterministic ray casting on unstructured grids rather than path tracing on regular grids.

## How it works
**Inputs and Outputs:**
*   **Input:** Unstructured tetrahedral mesh (vertices, normals, faces, tetrahedra indices, scalar values), camera parameters (eye position, view direction), and a pre-integrated Lookup Table (LUT).
*   **Output:** Final color and opacity image.

**Data Structures:**
All data is stored in 2D textures (F32x4 format) to optimize GPU memory access and avoid 3D texture limitations [S2, unofficial].

1.  **Mesh Data Storage (144 bytes per tetrahedron):**
    *   **Vertices:** 3 rows per tetrahedron $t$, indexed by $(t_u, t_v)$, $(t_u, t_v+d_v)$, $(t_u, t_v+2d_v)$.
        *   RGB channels: Vertex coordinates $(v_{t,0..2}^{x,y,z})$.
        *   Alpha channel: Fourth vertex coordinate $(v_{t,3}^{x,y,z})$.
    *   **Face Normals:** 3 rows per tetrahedron $t$, indexed by $(t_u, t_v)$, $(t_u, t_v+d_v)$, $(t_u, t_v+2d_v)$.
        *   RGB channels: Normal coordinates $(n_{t,0..2}^{x,y,z})$.
        *   Alpha channel: Fourth normal coordinate $(n_{t,3}^{x,y,z})$.
    *   **Neighbor Data:** 1 row per tetrahedron $t$, indexed by $(t_u, t_v)$.
        *   RGB channels: Adjacency indices $(a_{t,0}, a_{t,1})$.
    *   **Scalar Data:** 1 row per tetrahedron $t$, indexed by $(t_u, t_v+2d_v)$.
        *   RGB channels: Gradient $(g_t^{x,y,z})$.
        *   Alpha channel: Scalar value $(s_t)$.
    *   *Note:* $t_u, t_v$ split the tetrahedron index because GPU 2D texture indices are limited (e.g., 4096 per dimension) [S2, unofficial].

2.  **Traversal Data Storage:**
    *   **Current Cell:** Indexed by raster position.
        *   R channel: Tetrahedron index $t_u$.
        *   G channel: Tetrahedron index $t_v$.
        *   B channel: Intersection parameter $\lambda$.
        *   A channel: Scalar field value $s(e + \lambda r)$.
    *   **Color/Opacity:** Indexed by raster position.
        *   RGB channels: Accumulated color.
        *   A channel: Accumulated opacity.

**Algorithm Steps:**

1.  **Data Packing:** Pack mesh data (vertices, normals, faces, tetrahedra) and pre-integrated tables into 2D textures [S2, unofficial].
2.  **Tile Partitioning:** Partition the image screen into fixed-size disjoint tiles to exploit spatial and memory cache coherence [S2, unofficial].
3.  **Phase 1: First Intersection:**
    *   Render boundary faces with Z-buffer enabled.
    *   Vertex shader sets tetrahedron index $(t_u, t_v)$ at each vertex.
    *   Fragment shader computes the first intersection in Object Coordinate System (OCS) by applying inverse modeling transformation to eye ($e_{ocs}$) and ray direction ($r_{ocs}$).
    *   Retrieve mesh data from 2D textures.
    *   Store intersection $\lambda$, scalar field, and tetrahedron indices in traversal structures [S2, unofficial].
4.  **Phase 2: Depth Peeling Levels:**
    *   Use a multi-pass approach (one layer per pass) because GPUs cannot read/write to the same texture [S2, unofficial].
    *   Render boundary faces with back-face culling ON.
    *   Fragment shader compares candidate $\lambda$ with value stored in previous layer. If candidate $\lambda \le$ stored $\lambda$, discard fragment.
    *   Apply a bias value in the comparison test to reduce numerical artifacts [S2, unofficial].
    *   This generates $n$ layers of visible faces. The first layer is the initial set of visible faces; subsequent layers contain the next visible faces, discarding faces from previous layers [S2, unofficial].
5.  **Phase 3a: Ray Casting (Ping-Pong Rendering):**
    *   Use a pair of traversal textures switched each pass [S2, unofficial].
    *   Render screen-aligned rectangle (or tile-aligned subregions).
    *   For each active ray, compute the next intersection along ray direction in OCS.
    *   Look for minimum $\lambda$ (exit point) among positive denominators.
    *   Compute adjacent tetrahedron at intersection point using neighbor data.
    *   If no adjacent tetrahedron (special index), store indicating ray left mesh.
    *   Accumulate color using pre-computed color information from entry/exit points via LUT [S2, unofficial].
    *   **Occlusion Test:** After $k$ ray casting passes, perform Z-update and Occlusion Test to amortize cost. Use early-z tests to avoid computation for rays that miss or have left the mesh [S2, unofficial].
6.  **Phase 3b: Layer Advancement:**
    *   If the next depth peeling layer contains visible faces, issue a second ray casting pass using the information stored in that layer as starting points [S2, unofficial].
    *   Repeat Phase 3a and 3b until all rays leave the mesh and the following depth peeling layer does not contain visible faces [S2, unofficial].
7.  **Phase 4: Display:**
    *   Render screen-aligned rectangle sampling the color texture generated by ray-casting pass [S2, unofficial].

**Formulas:**

*   **Ray-Tetrahedron Intersection:**
    $$ \lambda_i = \frac{(v - e) \cdot n_{t,i}}{r \cdot n_{t,i}} \quad \text{where } v = v_{t,3-i} $$
    Only positive $\lambda_i$ values are considered.
    *   Entry point $\lambda_{in}$: Maximum $\lambda$ among values with negative denominator.
    *   Exit point $\lambda_{out}$: Minimum $\lambda$ among values with positive denominator.
    *   If $\lambda_{in} > \lambda_{out}$, no intersection exists [S2, unofficial].

*   **Scalar Field Interpolation:**
    $$ s(x) = g_t \cdot (x - x_0) + s(x_0) $$
    Where $g_t$ is the gradient of the scalar field, $x_0$ is a reference point (e.g., a vertex), and $s(x_0)$ is the scalar value at $x_0$ [S2, unofficial].

*   **Front-to-Back Color Composition:**
    $$ C_k = C'_{k-1} + (1 - \alpha_{k-1})C_k $$
    Where $C_k$ is the current color, $C'_{k-1}$ is the accumulated color from previous steps, and $\alpha_{k-1}$ is the accumulated opacity [S2, unofficial].

**Parameters:**
*   **Pre-integrated 3D LUT:** Indexed by distances between entry and exit points (normalized using the length of the greatest edge of the mesh) and scalar field values at both points. Returns color $(r,g,b)$ and opacity $\alpha$ [S2, unofficial].
*   **k:** Number of ray casting passes after which the Z-update and Occlusion Test is performed [S2, unofficial].
*   **Bias value:** Used in depth peeling comparison test to reduce artifacts due to numerical calculations [S2, unofficial].

**Pitfalls:**
*   **Non-convex meshes:** Rays may exit and re-enter the mesh. Depth peeling handles this, avoiding convexification [S2, unofficial].
*   **GPU Memory Limits:** 2D texture indices are limited (up to 4096 per dimension), requiring tetrahedral index splitting [S2, unofficial].
*   **MRT Limitations:** Current GPUs (at time of source) do not allow writing to the frame-buffer while using Multiple Render Targets (MRTs), requiring a separate buffer for color accumulation [S2, unofficial].
*   **Branching:** GPUs have limited/expensive branching, requiring fetching all vertices/normals of a tetrahedron consecutively in texture memory [S2, unofficial].
*   **Read/Write Constraints:** GPUs cannot read and write to the same texture, requiring multi-pass depth peeling and ping-pong rendering for ray casting [S2, unofficial].

## Operating characteristics
*   **Memory:** Constrained by GPU memory available for storing mesh data in textures. Saves up to 30% memory compared to previous 3D texture layouts [S2, unofficial].
*   **Performance:** Substantially faster than Weiler et al.'s technique on the same hardware [S2, unofficial].
*   **Scaling:** Tiling allows variable number of intersection passes per tile, approximating minimum required passes and reducing processed fragments [S2, unofficial].
*   **Strengths:** Handles general non-convex meshes without convexification; efficient 2D texture storage; exploits spatial coherence via tiling [S2, unofficial].
*   **Limitations:** Limited by GPU memory (smaller datasets than CPU-based methods like HAVS); requires multiple passes for depth peeling and ray traversal [S1, S2, unofficial].

## Background
*   **Origin:** Proposed by Christian A. Pagot, Joao L. D. Comba, and Claudio T. Silva [S2, unofficial].
*   **History:** Builds on the technique published at IEEE Visualization 2003 by Weiler et al. and the idea of using boundary fragments from Bunyk et al. [S2, unofficial].
*   **Usage:** Implemented in C++ using DirectX 9.0 and HLSL. Tested on ATI Radeon 9700 Pro 128MB and NVIDIA GeForce 6800 GT 256MB [S2, unofficial]. Used for rendering time-varying scalar fields on unstructured meshes, such as EEG brain data and procedural meshes [S1, unofficial].

## Sources
[S1] Volume Rendering of Time-Varying Scalar Fields on Unstructured Meshes, https://www.sci.utah.edu/~stevec/papers/tvrc.pdf, fetched 2026-10-27 (unofficial)
[S2] GPU-based Tiled Ray Casting using Depth Peeling, https://sci.utah.edu/~csilva/papers/gpu_volume_ray_casting.pdf, fetched 2026-10-29 (unofficial)
