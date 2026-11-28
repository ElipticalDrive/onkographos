# Continuous LOD (CLOD)

Also called: Continuous Level of Detail

## Overview
Continuous LOD (CLOD) is a multiresolution technique that replaces a high-resolution implicit function or point cloud with a more efficient low-resolution representation by interpolating between them based on the viewer's position [S2]. Unlike traditional distance-based LOD which selects a single model based on distance to the object center, CLOD adapts the representation continuously based on the distance between the viewer and the query point, allowing the transition region to straddle or embed the surface [S2]. It sets itself apart from proxy nodes by making the region of influence dependent on the viewer position, and from discrete LOD methods by eliminating sudden visual changes through smooth blending [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** High-resolution model $h$, low-resolution model $l$, viewer position $e$, transition start radius $r$, transition width $\delta_r$ [S2].
*   **Outputs:** Continuous effective model $f_e(p)$ for any query point $p$ [S2].

**Data Structures:**
*   **Implicit/Mesh Nodes:** Each node stores references to its high-resolution child $h$ and low-resolution child $l$ [S2].
*   **Mesh Vertices (Rasterization):** For transition regions, vertices store a "parent vertex" from the parent chunk's mesh to enable blending in the vertex shader [S3, unofficial].
*   **Point Cloud Batches:** Store bounding box (`Min`, `Max`), offset, and count, alongside separate buffers for low, medium, and high-resolution positions and colors [S1, unofficial].

**Steps:**
1.  **Compute Distance:** Calculate the Euclidean distance $d(p, e) = \|p - e\|$ between the query point $p$ and the viewer $e$ [S2].
2.  **Determine Blend Factor:** Compute the interpolation weight $\alpha(p)$ based on the distance relative to the transition zone boundaries $r$ and $r + \delta_r$ [S2]:
    $$
    \alpha(p) = \begin{cases} 
    0 & \text{if } \|p - e\| \le r \\
    \frac{\|p - e\| - r}{\delta_r} & \text{if } r < \|p - e\| < r + \delta_r \\
    1 & \text{if } \|p - e\| \ge r + \delta_r 
    \end{cases}
    $$
3.  **Interpolate Model:** Evaluate the effective model $f_e(p)$ using the weighted average of the high and low-resolution models, normalized by a factor $\lambda$ [S2]:
    $$
    f_e(p) = \frac{1}{\lambda} \left( (1 - \alpha(p)) h(p) + \alpha(p) l(p) \right)
    $$
    *Note: The sources define $\lambda$ in the formula but do not specify its value or calculation method [S2].*
4.  **Mesh/Vertex Blending:** For rasterized meshes, vertices in the transition region possess positions from both LOD levels [S3, unofficial]. The vertex shader blends these positions using weights $W_0$ and $W_1$ such that $W_0 + W_1 = 1$ [S3, unofficial]. At the boundary between the high-resolution LOD and the transition, weights are $(1, 0)$ [S3, unofficial]. At the boundary between the transition and the low-resolution LOD, weights are $(0, 1)$ [S3, unofficial].
5.  **Point Cloud Filtering (Alternative):** For point clouds, assign each point an LOD value (0–3) based on spatial density [S1, unofficial]. LOD 0 covers 1×1×1 m volumes, LOD 1 covers 0.5×0.5×0.5 m, and LOD 2 covers 0.25×0.25×0.25 m [S1, unofficial]. Average the color of all points within each volume to prevent aliasing [S1, unofficial]. At runtime, iterate over points, perform frustum culling, compare distance to assigned LOD level, and apply a randomized normalized weight for smooth blending before copying visible points to a render buffer [S1, unofficial].

**Parameters:**
*   $r$: The radius defining the start of the transition zone from the viewer. Inside this radius, the high-resolution model is used ($\alpha=0$) [S2].
*   $\delta_r$: The width of the transition zone. Interpolation occurs in the range $r \le d(p, e) \le r + \delta_r$ [S2].

**Edge Cases and Pitfalls:**
*   **Performance:** Computational cost is higher in the transition zone because both high and low-resolution functions must be evaluated [S2].
*   **Memory:** Storing parent vertices for mesh blending requires significantly more memory [S3, unofficial].
*   **GPU Updates:** Mesh data requires more frequent updates on the GPU to maintain correct blending states [S3, unofficial].
*   **Point Cloud Amortization:** If the camera moves too quickly while amortizing LOD generation over multiple frames, empty space appears at the edges of the screen [S1, unofficial].
*   **Non-manifold Geometry:** Surface nets used to extract meshes from SDFs can produce non-manifold geometry near pseudo-singularities, which may break normal-based lighting [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Higher in the transition zone due to dual function evaluation [S2].
*   **Memory Complexity:** Increased due to the storage of parent vertices for mesh blending [S3, unofficial].
*   **Strengths:** Provides seamless, continuous visual transitions without popping artifacts; adapts to viewer position rather than object center [S2].
*   **Limitations:** Higher computational cost in transition regions; increased memory footprint and GPU update frequency for mesh-based rendering [S2][S3, unofficial].
*   **When to use:** When visual continuity is critical and the cost of dual evaluation in transition zones is acceptable [S2][S3, unofficial].
*   **When not to use:** When memory bandwidth is severely constrained or when the transition zone cost is prohibitive for the target hardware [S2][S3, unofficial].

## Background
Continuous LOD nodes were introduced by P. Hubert-Brierre, E. Guérin, A. Peytavie, and E. Galin in their work on accelerating signed distance functions [S2]. The technique is part of a broader family of multiresolution representations for implicit surfaces, building upon the concept of LOD selection via projected width and progressive level transitions [S2]. It addresses the limitations of proxy nodes and traditional distance-based LOD selection by allowing the region of influence to depend on the viewer position [S2].

## Sources
[S1] How we render extremely large point clouds — Magnopus, https://www.magnopus.com/blog/how-we-render-extremely-large-point-clouds, fetched 2026-10-26 (unofficial)
[S2] Accelerating Signed Distance Functions, https://doi.org/10.1111/cgf.70258, fetched 2026-10-26
[S3] I wish I found Surface Nets sooner! : r/VoxelGameDev - Reddit, https://www.reddit.com/r/VoxelGameDev/comments/pklhuo/i_wish_i_found_surface_nets_sooner/, fetched 2026-10-28 (unofficial)
