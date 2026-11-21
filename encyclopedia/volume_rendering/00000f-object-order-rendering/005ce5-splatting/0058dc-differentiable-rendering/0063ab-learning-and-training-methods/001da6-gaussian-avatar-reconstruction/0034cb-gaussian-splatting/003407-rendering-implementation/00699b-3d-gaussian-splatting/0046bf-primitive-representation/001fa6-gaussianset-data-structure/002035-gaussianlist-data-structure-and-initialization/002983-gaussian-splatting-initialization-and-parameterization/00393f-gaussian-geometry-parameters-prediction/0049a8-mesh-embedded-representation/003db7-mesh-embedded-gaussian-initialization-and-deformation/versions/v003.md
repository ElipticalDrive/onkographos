# Mesh-Embedded Gaussian Initialization and Deformation

Also called: Gaussian Embedding, Triangle Embedding

## Overview
This method initializes and deforms 3D Gaussians by anchoring them to a canonical triangle mesh via barycentric coordinates and normal displacement [S1, unofficial]. It solves the problem of maintaining geometric consistency and differentiability during animation by tracking deformation through mesh topology rather than direct vertex positions [S1, unofficial]. It differs from sibling methods like [Mesh-Embedded Gaussian Splatting](9553-mesh-embedded-gaussian-splatting.md) by explicitly defining the "walking on triangle" constraint mechanism to prevent Gaussians from crossing triangle boundaries during optimization [S1, unofficial].

## How it works
**Inputs:** Canonical triangle mesh $M_{cano}$ with vertices $V$, per-vertex normals $n$, and per-vertex quaternions $q$ (for deformation tracking); number of Gaussians $N$ (typically 10,000).
**Outputs:** Set of $N$ Gaussians, each defined by embedding $E_i = \{k, u, v, d\}$ and derived properties (mean $\mu$, rotation $q$, scale $s$, color).

**Data Structures:**
*   **Embedding $E_i$**: A tuple containing:
    *   $k$: Integer triangle index in the canonical mesh.
    *   $u, v$: Float barycentric coordinates within triangle $k$.
    *   $d$: Float normal displacement from the mesh surface.
*   **Gaussian Properties**:
    *   $\mu$: Mean position (3D vector).
    *   $q$: Orientation (quaternion).
    *   $s$: Scale (vector or scalar, derived from area).
    *   Color: View-independent (Spherical Harmonics turned off) [S1, unofficial].

**Initialization Steps:**
1.  **Select Triangles and Barycentrics:** Randomly select $N$ pairs of triangle indices $k$ and barycentric coordinates $(u, v)$ from the canonical mesh $M_{cano}$ [S1, unofficial].
2.  **Initialize Displacement:** Set the normal displacement $d$ for all Gaussians to zero [S1, unofficial].
3.  **Compute Canonical Position:** For each Gaussian $i$, compute the canonical mean $\mu_{cano}$ using Phong interpolation of the mesh vertices and normals, plus the displacement [S1, unofficial]:
    $$
    P(k, u, v) = u V_1 + v V_2 + (1 - u - v) V_3
    $$
    $$
    n(k, u, v) = u n_1 + v n_2 + (1 - u - v) n_3
    $$
    $$
    \mu_{cano} = P(k, u, v) + d \cdot n(k, u, v)
    $$
    where $V_1, V_2, V_3$ are the vertices of triangle $k$, and $n_1, n_2, n_3$ are their corresponding per-vertex normals [S1, unofficial].
4.  **Initialize Rotation and Scale:**
    *   Compute per-vertex quaternions $q_1, q_2, q_3$ for the triangle vertices based on the area-weighted average of surrounding neighbor triangles' rotation matrices from canonical to posed space [S1, unofficial].
    *   Interpolate the rotation $\delta q_{i,0}$ for the Gaussian using barycentric weights:
        $$
        \delta q_{i,0} = u q_1 + v q_2 + (1 - u - v) q_3
        $$
    *   Initialize the canonical rotation $q_{cano}$ (details on $\bar{q}$ initialization not specified in sources, assumed to be identity or derived from local frame) [S1, unofficial].
    *   Initialize scale $s_{cano}$ based on the area of the canonical triangle $A_{cano}$ (details on base scale derivation not specified) [S1, unofficial].

**Deformation Steps (for a posed frame $t$):**
1.  **Interpolate Rotation:** Compute the interpolated rotation $\delta q_{i,t}$ from the mesh vertices at frame $t$:
    $$
    \delta q_{i,t} = u q_{1,t} + v q_{2,t} + (1 - u - v) q_{3,t}
    $$
    where $q_{1,t}, q_{2,t}, q_{3,t}$ are the per-vertex quaternions at frame $t$ [S1, unofficial].
2.  **Apply Rotation:** Update the Gaussian rotation $q_{i,t}$ by multiplying the interpolated mesh rotation with the canonical rotation $\bar{q}_i$ (the rotation in canonical space):
    $$
    q_{i,t} = \delta q_{i,t} * \bar{q}_i
    $$
    where $*$ denotes quaternion multiplication [S1, unofficial].
3.  **Apply Scale:** Update the scale $s_{i,t}$ based on the ratio of the triangle area in the posed space $A_{pose}$ to the canonical space $A_{cano}$:
    $$
    s_{i,t} = \left( \frac{A_{pose}}{A_{cano}} \right) \bar{s}_i
    $$
    where $\bar{s}_i$ is the scale in canonical space [S1, unofficial].
4.  **Walking on Triangle (Constraint):** To maintain differentiability and prevent Gaussians from crossing triangle boundaries, clip the barycentric coordinates $(u, v)$ to ensure they remain within the bounds of triangle $k$ (i.e., $u \ge 0, v \ge 0, u+v \le 1$) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Cross-Triangle Updates:** If an optimization step moves the embedding point across a triangle boundary, the embedding must be re-expressed in the neighboring triangle to maintain differentiability. Failure to handle this causes Gaussians to stick to boundaries and pile up [S2, unofficial].
*   **Disabling Walking on Triangle:** Disabling the constraint causes Gaussians to stick and pile up on triangle boundaries, leading to artifacts when animated by novel poses [S1, unofficial].

## Subtopics

### Walking on Triangle
A mechanism that constrains Gaussians to move within their corresponding mesh triangles by clipping UV values to prevent movement beyond triangle boundaries [S1, unofficial]. It is chosen to maintain gradient flow and prevent artifacts during animation [S1, unofficial]. Disabling it reduces performance: for the 'bala' head avatar, PSNR drops from 30.04 to 29.91, SSIM from 0.938 to 0.933, and LPIPS increases from 0.062 to 0.070; for 'male-3-casual', PSNR drops from 33.01 to 32.48, SSIM from 0.982 to 0.979, and LPIPS increases from 0.020 to 0.024 [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Initialization is $O(N)$ where $N$ is the number of Gaussians. Deformation is $O(N)$ per frame, involving barycentric interpolation of quaternions and area calculations.
*   **Memory Complexity:** $O(N)$ to store embeddings and Gaussian properties.
*   **Performance:** The method provides high-quality relightable modeling with explicit shape representations [S1, unofficial]. However, it compromises performance in terms of pixel-wise metrics compared to some implicit methods [S1, unofficial].
*   **Strengths:** Maintains geometric consistency through mesh topology; differentiable deformation; prevents artifacts via walking on triangle constraint.
*   **Limitations:** Relies on the quality of the canonical mesh and per-vertex quaternion computation; requires handling of cross-triangle updates to avoid sticking.
*   **When to use:** When animating avatars or deformable objects where explicit shape representation and relighting are important.
*   **When not to use:** When pixel-wise metrics are the sole priority and mesh topology is not available or reliable.

## Background
Originated in the SplattingAvatar paper [S1, unofficial], this method builds on the principles of mesh-embedded representations [12559-mesh-embedded-representation.md]. It uses Phong surface interpolation [38] for position and normal calculation, and follows the rotation tracking approach proposed by Zielonka et al. [53]. The method turns off spherical harmonics for color to simplify the representation [S1, unofficial].

## Sources
[S1] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
