# Normal Approximation with Finite Differences

Also called: Smooth normal sampling

## Overview
Normal Approximation with Finite Differences approximates the surface normal at a ray-scalar field intersection point by estimating the gradient via finite differences [S1, unofficial]. It distinguishes itself from basic gradient methods by explicitly handling voxel boundary discontinuities through neighbor voxel queries, ensuring smooth normals across voxel edges [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Intersection point $P$ within a voxel, voxel dimensions ($dx, dy, dz$), epsilon $\epsilon$ (typically slightly less than half the voxel size), and a scalar field $T$ accessible via trilinear interpolation [S1, unofficial].
*   **Output:** Normalized surface normal vector $\hat{N}$ [S1, unofficial].

**Data Structures:**
*   **Scalar Field $T$:** A 3D grid of scalar values. Access requires trilinear interpolation $T(x, y, z)$ [S1, unofficial].
*   **Voxel Neighbor Map:** Implicit or explicit mapping of voxel coordinates to their neighbors to handle boundary queries [S1, unofficial].

**Steps:**
1.  **Initialize Epsilon:** Set $\epsilon$ to a value slightly less than half the voxel size to ensure sampling points remain within or adjacent to the current voxel [S1, unofficial].
2.  **Compute Gradient Components:** For each axis $i \in \{x, y, z\}$, calculate the central difference approximation of the gradient component $N_i$:
    $$ N_i = T(P_i + \epsilon) - T(P_i - \epsilon) $$
    where $P_i$ is the coordinate of $P$ along axis $i$, and $T$ is the trilinear interpolation function [S1, unofficial].
3.  **Handle Boundary Conditions:**
    *   If the sampling point $P_i \pm \epsilon$ falls outside the current voxel boundaries, query the respective neighboring voxel [S1, unofficial].
    *   Sample the distance inside the neighbor using trilinear interpolation relative to that neighbor's local coordinates [S1, unofficial].
    *   If the neighbor query fails (e.g., at the volume boundary), return the extrapolated distance from the current voxel [S1, unofficial].
4.  **Normalize:** Compute the final normal $\hat{N} = \frac{(N_x, N_y, N_z)}{\|(N_x, N_y, N_z)\|}$ [S1, unofficial].

**Pitfalls:**
*   The approximation is continuous inside the voxel but not on the edges, causing artifacts if neighbor queries are not handled correctly [S1, unofficial].
*   Edge cases where sampling requires non-existent data must be handled via clamping or extrapolation to avoid undefined behavior [S2, unofficial].

## Subtopics

### Surface Normal Calculation via Central Difference
Calculates the surface normal $\hat{n}$ using a central difference scheme to approximate the gradient $\nabla \rho$, requiring six calls to trilinear interpolation [S2, unofficial]. It uses the formula $\hat{n} \approx [\frac{\rho(P_x+dx)-\rho(P_x-dx)}{2dx}, \frac{\rho(P_y+dy)-\rho(P_y-dy)}{2dy}, \frac{\rho(P_z+dz)-\rho(P_z-dz)}{2dz}]$ [S2, unofficial]. It is chosen for its balance of performance and visual quality; higher-order $O(h^4)$ variants offer marginal improvement only when zoomed in extremely closely with correct Phong shading parameters, justifying the use of this $O(h^2)$ method for performance [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per intersection, requiring 6 trilinear interpolation calls for the standard central difference scheme [S2, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory, assuming the scalar field is accessible [S2, unofficial].
*   **Performance:** On datasets ranging from $32 \times 32 \times 32$ to $1024 \times 768 \times 256$, the standard $O(h^2)$ scheme provides real-time performance. Higher-order $O(h^4)$ schemes (requiring 12 interpolations) slowed renderers by 0–2 FPS, with negligible visual difference [S2, unofficial].
*   **Strengths:** Smooth normals across voxel boundaries; efficient implementation on GPUs using texture addressing modes like `cudaAddressModeClamp` [S2, unofficial].
*   **Limitations:** Discontinuities at voxel edges if neighbor queries are not implemented; accuracy limited to $O(h^2)$ for the standard scheme [S1, unofficial][S2, unofficial].

## Background
The method originates from voxel-based surface rendering techniques, specifically those described in [PSL*98] and implemented in systems like the Prediction Buffer renderer [S1, unofficial][S2, unofficial]. It leverages the gradient of the scalar field to provide Phong shading normals, which are required for realistic lighting but are not directly provided by voxel traversal algorithms that only identify the containing voxel [S2, unofficial].

## Sources
[S1] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
[S2] GPU Accelerated Voxel Traversal using the Prediction Buffer, https://people.cs.vt.edu/yongcao/publication/pdf/Braley10.pdf, fetched 2026-10-25 (unofficial)
