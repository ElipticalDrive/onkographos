# Splat Rasterization via Raycasting

Also called: EWA Splat Rasterization via Ray-Plane Intersection, GPU-based raycasting of quadric surfaces, Splat rasterization

## Overview
This method rasterizes Ellipsoid-Warping (EWA) splats on general-purpose graphics hardware by treating point primitives as screen-space squares and casting a viewing ray for each fragment to test inclusion within the splat's volume [S2, unofficial]. It distinguishes itself from polygon-based bounding approximations (e.g., Botsch et al.) by providing a correct upper bound to splat extents that remains stable for large splats near the near plane, while avoiding the numerical instabilities of explicit conic centralization or matrix inversions [S1][S2, unofficial].

## How it works
**Inputs:**
*   Splat parameters in camera space: center $\mathbf{c}_0$, orthonormal basis vectors $\mathbf{u}_0, \mathbf{v}_0, \mathbf{w}_0$ (where $\mathbf{w}_0$ is the normal, often reduced to zero length for the degenerate ellipsoid representation) [S1].
*   Projection matrix $P$ [S1].
*   Reconstruction kernel $k(\cdot)$ [S1].
*   Cutoff radius for the kernel [S1].

**Outputs:**
*   Fragment weight $w_f$ and depth value for the splat at each sample position [S1].

**Data Structures:**
*   Clip space plane representation: A plane is represented as a 4-vector $\mathbf{h} = (h_x, h_y, h_z, h_w)^T$ [S1].
*   Splat matrix in clip space: $\mathbf{S} = \begin{pmatrix} \mathbf{u}' & \mathbf{v}' & \mathbf{0} & \mathbf{c}' \end{pmatrix}$, where $\mathbf{u}', \mathbf{v}', \mathbf{c}'$ are the transformed basis vectors and center [S1].

**Steps:**

1.  **Rasterization Setup (Bounding Box Calculation):**
    Compute the tight axis-aligned bounding rectangle (AABB) in screen space for the projected splat. This is done by treating the splat as a degenerate ellipsoid (unit sphere in local space with $\mathbf{w}_0$ length zero) [S1].
    *   Map the viewing planes $\mathbf{h}_x = (-1, 0, 0, x)^T$ and $\mathbf{h}_y = (0, -1, 0, y)^T$ from clip space to the local splat coordinate system using the inverse transpose of the projection matrix: $\bar{\mathbf{h}}_c = (\mathbf{P}\mathbf{S})^{-T} \mathbf{h}_c$ for $c \in \{x, y\}$ [S1].
    *   Solve for $x$ and $y$ such that the distance from the origin to the plane $\bar{\mathbf{h}}_x$ (or $\bar{\mathbf{h}}_y$) in the local $(u_0, v_0, w_0)$ space is exactly 1 (tangential to the unit sphere) [S1].
    *   This avoids explicit inversion of $\mathbf{S}$ or $\mathbf{P}$, ensuring stability for near-degenerate splats [S1].
    *   The resulting bounds are $x_{1,2} = p_1 \pm h_1$ and $y_{1,2} = p_2 \pm h_2$ [S1].
    *   Clamp the bounding box extents $h_i$ to a lower bound of 1 to prevent division by zero or degenerate traversal [S1].

2.  **Fragment Generation:**
    Draw the splat using point primitives rendered as screen-space squares covering the computed bounding box [S2, unofficial].

3.  **Raycasting and Inclusion Test (Fragment Shader):**
    For each fragment $(x, y)$ within the bounding box:
    *   Represent the viewing ray through $(x, y)$ as the intersection of two planes in clip space: $\mathbf{h}_x = (-1, 0, 0, x)^T$ and $\mathbf{h}_y = (0, -1, 0, y)^T$ [S1].
    *   Transform these planes into the splat's local coordinate system using the precomputed matrix $\mathbf{T} = (\mathbf{P}\mathbf{S})^{-T}$: $\mathbf{k} = \mathbf{T} \mathbf{h}_x$ and $\mathbf{l} = \mathbf{T} \mathbf{h}_y$ [S1].
    *   Intersect the projected planes $\mathbf{k}$ and $\mathbf{l}$ with the splat's object-space plane (defined by normal $\mathbf{w}_0$ and center $\mathbf{c}_0$) to find the intersection point $\mathbf{s}$ in local space [S1].
    *   Compute the squared distance $\rho^2$ from $\mathbf{s}$ to the splat's center (origin in local space) [S1].

4.  **Weight Computation:**
    *   If $\rho^2$ is within the specified cutoff radius, the sample is inside the splat.
    *   Compute the fragment weight: $w_f = k(\rho^2)$ [S1].
    *   Apply the EWA screen-space filter approximation by clamping $\rho^2$:
        $$ \rho^2_{clamped} = \min\left(\rho^2, k(x,y) - (\mathbf{c}_{00} \cdot (x,y))^2\right) $$
        *(Note: The source formula notation $k(x,y) - (c_{00} x, c_{00} y)^2$ is interpreted as a clamping term derived from the EWA filter geometry, limiting the effective radius based on screen-space coordinates)* [S1].

**Parameters:**
*   Cutoff radius: Determined by the kernel support (e.g., radius where $k(r^2) \approx 0$) [S1].
*   Lower bound for bounding box extents: 1 [S1].

**Edge Cases and Pitfalls:**
*   **Near-plane splats:** Polygon-based approximations (Botsch et al.) can underestimate extents for large splats near the near plane, causing artifacts. This raycasting method provides a correct upper bound [S2, unofficial].
*   **Degenerate splats:** Explicit matrix inversions in conic-based approaches become unstable. This method uses inverse transpose mapping, remaining stable for near-degenerate splats [S1].
*   **Precision:** The method relies on accurate plane transformations. Numerical stability is maintained by avoiding direct inversion of the splat matrix $\mathbf{S}$ [S1].

## Subtopics

### [Weyrich et al. Splat Extent Calculation](1440-weyrich-et-al-splat-extent-calculation.md)
Computes tight AABBs for EWA splats without matrix inversions, ensuring stability for near-degenerate cases. Choose when tight bounds are critical for performance.

### [GPU Splat Rasterization via Vertex/Fragment Shaders](2152-gpu-splat-rasterization-via-vertex-fragment-shaders.md)
Implements the raycasting logic on GPUs using vertex/fragment shaders, optimizing for minimal instruction count (3 instructions). Choose for high-performance GPU rendering.

## Operating characteristics
*   **Time Complexity:** Per-fragment cost is low, requiring only a few MAD (Multiply-Accumulate) operations and a texture fetch in the fragment shader [S2, unofficial]. The setup cost is higher than simple square approximations but lower than full polygon clipping for tiny primitives [S1][S2, unofficial].
*   **Memory Complexity:** Minimal additional memory; requires storing the transformed plane coefficients or matrix $\mathbf{T}$ per splat [S1].
*   **Performance:** On NVIDIA hardware, this approach is highly efficient due to the low instruction count in the fragment shader [S2, unofficial].
*   **Strengths:** Correct upper bounds for extents, stability for near-degenerate and near-plane splats, no explicit matrix inversions [S1][S2, unofficial].
*   **Limitations:** Higher per-fragment computational cost compared to simple square rasterization (though still low) [S1]. Requires programmable shaders; not suitable for fixed-function pipelines without specialization [S2, unofficial].
*   **When to use:** When rendering surface splats on GPUs where accuracy near the near plane and stability for degenerate splats are required [S1][S2, unofficial].
*   **When not to use:** When fixed-function hardware with specialized splat rasterization units is available, or when the absolute lowest per-fragment cost is prioritized over accuracy for large near-plane splats.

## Background
This technique was developed to address the limitations of existing GPU splatting methods, which often relied on approximations that could fail for large splats near the near plane or become numerically unstable [S1][S2, unofficial]. It builds on the EWA splatting framework [S1] and adapts it for modern programmable graphics pipelines by leveraging raycasting within the fragment shader [S2, unofficial]. The method is part of the broader field of surface splatting, which aims to render smooth surfaces from scattered data points [S1][S2, unofficial].

## Sources
[S1] A hardware architecture for surface splatting, https://doi.org/10.1145/1275808.1276490, fetched 2026-10-28
[S2] sebastianlipponer/surface_splatting, https://github.com/sebastianlipponer/surface_splatting, fetched 2026-10-28 (unofficial)
