# Mesh-Embedded Gaussian Initialization and Deformation

Also called: Gaussian Embedding, Triangle Embedding

## Overview
This method anchors 3D Gaussians to a triangle mesh using barycentric coordinates and normal displacement, enabling deformation tracking through mesh topology rather than direct vertex positions [S1, unofficial]. It differs from standard Gaussian Splatting by decoupling Gaussian positions from fixed world coordinates, allowing them to deform with the underlying mesh, and from [Textured-GS](9436-textured-gs.md) by using explicit geometric embeddings rather than texture maps for spatial definition.

## How it works
**Inputs:** A canonical triangle mesh with vertices $V$, per-vertex normals $n$, and per-vertex quaternions $q$ (computed from tangent frames); a set of Gaussian embeddings $E$.
**Outputs:** Deformed Gaussian parameters (position $\mu$, rotation $q$, scale $s$) for a target pose.

**Data Structures:**
*   **Embedding $E_i$**: A tuple $(k, u, v, d)$ where:
    *   $k$: Triangle index.
    *   $u, v$: Barycentric coordinates ($u \ge 0, v \ge 0, u+v \le 1$).
    *   $d$: Normal displacement scalar.
*   **Mesh Attributes**:
    *   $V_k = \{V_1, V_2, V_3\}$: Vertices of triangle $k$.
    *   $n_k = \{n_1, n_2, n_3\}$: Per-vertex normals of triangle $k$.
    *   $q_k = \{q_1, q_2, q_3\}$: Per-vertex quaternions of triangle $k$.

**Initialization:**
1.  Randomly select $10,000$ pairs of triangle indices and barycentric coordinates on the canonical mesh [S1, unofficial].
2.  Set the embedding's $u, v$ to the selected barycentric coordinates.
3.  Initialize all displacements $d$ to zero [S1, unofficial].
4.  Compute initial position $\mu$ using the canonical mesh geometry (see Formulas).
5.  Initialize other Gaussian properties (opacity, color) as per standard Gaussian Splatting, but disable spherical harmonics for simplicity [S1, unofficial].

**Deformation Steps for Frame $t$:**
1.  **Compute Interpolated Position:**
    Calculate the base position $P$ on triangle $k$ using barycentric interpolation of vertices:
    $$P = \mathcal{V}(k, u, v) = u V_1 + v V_2 + (1 - u - v) V_3$$
    Calculate the interpolated normal $n$:
    $$n = \mathcal{N}(k, u, v) = u n_1 + v n_2 + (1 - u - v) n_3$$
    The Gaussian mean $\mu$ is then:
    $$\mu = P + d \cdot n$$
    [S1, unofficial]

2.  **Compute Interpolated Rotation:**
    For the target pose, compute the rotation matrix $R_k$ for triangle $k$ based on its tangent, bitangent, and normal in both canonical and posed spaces. Convert to quaternions $q_1, q_2, q_3$ for the triangle vertices.
    Compute the barycentric interpolated rotation $\delta q_{i,t}$:
    $$\delta q_{i,t} = u q_1 + v q_2 + (1 - u - v) q_3$$
    Normalize $\delta q_{i,t}$ if necessary (quaternion interpolation usually requires slerp or normalization to maintain unit length, though the source specifies linear combination) [S1, unofficial].
    The final Gaussian rotation $q_{i,t}$ is:
    $$q_{i,t} = \delta q_{i,t} * \bar{q}_i$$
    where $\bar{q}_i$ is the canonical rotation of the Gaussian and $*$ denotes quaternion multiplication [S1, unofficial].

3.  **Compute Interpolated Scale:**
    Calculate the area of the triangle in the posed space ($A_{pose}$) and canonical space ($A_{cano}$).
    The scale $s_{i,t}$ is:
    $$s_{i,t} = \left( \frac{A_{pose}}{A_{cano}} \right) s_i$$
    where $s_i$ is the canonical scale [S1, unofficial].

4.  **Walking on Triangle (Constraint):**
    To prevent Gaussians from moving beyond their triangle boundaries, clip the barycentric coordinates $(u, v)$ to the valid triangle domain ($u \ge 0, v \ge 0, u+v \le 1$) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** Disabling the "walking on triangle" constraint causes Gaussians to stick and pile up on triangle boundaries, leading to visible artifacts during novel pose animation [S1, unofficial].
*   **Precision:** Barycentric interpolation assumes linear variation of normals and rotations across the triangle. Large triangles may introduce interpolation errors.
*   **Degenerate Input:** If the mesh has non-manifold edges or degenerate triangles, barycentric coordinates may be undefined or unstable.

## Subtopics

### Walking on Triangle
A mechanism that constrains Gaussians to move within their corresponding mesh triangles by clipping UV values to prevent movement beyond triangle boundaries [S1, unofficial]. It prevents Gaussians from sticking and piling up on triangle boundaries during animation [S1, unofficial]. Disabling it reduces performance: for the 'bala' head avatar, PSNR drops from 30.04 to 29.91, SSIM from 0.938 to 0.933, and LPIPS increases from 0.062 to 0.070; for 'male-3-casual', PSNR drops from 33.01 to 32.48, SSIM from 0.982 to 0.979, and LPIPS increases from 0.020 to 0.024 [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear in the number of Gaussians for interpolation and deformation calculation. Triangle area computation is constant per Gaussian if precomputed or cached.
*   **Memory Complexity:** $O(N)$ for $N$ Gaussians, storing $(k, u, v, d)$ per Gaussian.
*   **Performance:** The method provides high-fidelity deformation tracking for avatars. The ablation study shows that the "walking on triangle" constraint is critical for maintaining PSNR and SSIM, particularly for full-body avatars [S1, unofficial].
*   **Strengths:** Explicit geometric control; avoids the need for dense Gaussian distributions to approximate surfaces; handles deformation naturally via mesh topology.
*   **Limitations:** Requires a valid input mesh; spherical harmonics are disabled for simplicity, potentially limiting view-dependent color accuracy [S1, unofficial].

## Background
Originated in the SplattingAvatar paper [S1, unofficial], this method builds on the idea of embedding Gaussians in a continuous space defined by a mesh. It addresses the limitation of standard Gaussian Splatting in handling articulated or deforming geometry by anchoring Gaussians to a mesh structure.

## Sources
[S1] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
