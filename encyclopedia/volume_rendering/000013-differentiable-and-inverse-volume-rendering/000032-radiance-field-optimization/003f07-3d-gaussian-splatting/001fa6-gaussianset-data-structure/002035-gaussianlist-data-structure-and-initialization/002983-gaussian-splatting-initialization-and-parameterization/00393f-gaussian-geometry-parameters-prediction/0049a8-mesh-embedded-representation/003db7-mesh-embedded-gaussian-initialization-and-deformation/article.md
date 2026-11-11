# Mesh-Embedded Gaussian Initialization and Deformation

Also called: Gaussian Embedding, Triangle Embedding

## Overview
This method anchors 3D Gaussians to a triangle mesh via barycentric coordinates and a normal displacement, allowing deformation tracking through mesh topology rather than direct vertex positions [S1, unofficial]. It differs from sibling methods like [Mesh-Embedded Gaussian Splatting](9553-mesh-embedded-gaussian-splatting.md) by explicitly parameterizing the Gaussian mean as a displacement from the mesh surface, and from [GaMeS](12542-games-gaussian-mesh-splatting.md) by using barycentric interpolation for rotation and area-based scaling rather than face-vertex convex combinations.

## How it works
**Inputs and Outputs:**
*   **Input:** A canonical triangle mesh with vertices $V$, per-vertex normals $n$, and per-vertex quaternions $q$ (for dynamic scenes).
*   **Output:** A set of Gaussians defined by mean $\mu$, rotation $q$, scale $s$, and color $c$.

**Data Structures:**
Each Gaussian $i$ is defined by an embedding $E_i = \{k, u, v, d\}$:
*   $k$: Triangle index.
*   $u, v$: Barycentric coordinates.
*   $d$: Normal displacement scalar.

**Initialization:**
1.  Randomly select $N$ pairs of triangle indices $k$ and barycentric coordinates $(u, v)$ on the canonical mesh.
2.  Initialize the displacement $d$ for all Gaussians to zero.
3.  Set initial barycentric coordinates $(u, v)$ to the sampled values.
4.  Initialize other Gaussian properties (color, etc.) based on the canonical position.

**Canonical Space Calculation:**
For a Gaussian with embedding $E_i = \{k, u, v, d\}$ in the canonical space:
1.  Compute the position $P$ on triangle $k$ using barycentric interpolation of vertices $V_1, V_2, V_3$:
    $$P = \mathcal{V}(k, u, v) = u V_1 + v V_2 + (1 - u - v) V_3$$
2.  Compute the interpolated normal $n$ using per-vertex normals $n_1, n_2, n_3$:
    $$n = \mathcal{N}(k, u, v) = u n_1 + v n_2 + (1 - u - v) n_3$$
3.  Compute the Gaussian mean $\mu$:
    $$\mu = P + d \cdot n$$

**Deformation to Posed Space (Frame $t$):**
For dynamic scenes, the Gaussian parameters are transformed from canonical to posed space using the mesh deformation.

1.  **Rotation:**
    *   Compute per-vertex quaternions $q_1, q_2, q_3$ for the triangle vertices in the posed space. These are typically derived from the rotation matrix $R_k$ (constructed from tangent, bitangent, and normal) converted to quaternion, then area-weighted averaged from surrounding neighbor triangles [S1, unofficial].
    *   Interpolate the rotation $\delta q_{i,t}$ using barycentric weights:
        $$\delta q_{i,t} = u q_1 + v q_2 + (1 - u - v) q_3$$
    *   Apply the interpolated rotation to the canonical rotation $q_i$:
        $$q_{i,t} = \delta q_{i,t} * \bar{q}_i$$
        where $\bar{q}_i$ is the canonical rotation quaternion.

2.  **Scale:**
    *   Compute the area of the embedded triangle in the posed space ($A_{pose}$) and canonical space ($A_{cano}$).
    *   Update the scale $s_{i,t}$ based on the area change ratio:
        $$s_{i,t} = \left( \frac{A_{pose}}{A_{cano}} \right) s_i$$

**Color:**
Spherical harmonics are turned off for simplicity, using a single color value per Gaussian [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Crossing:** If barycentric updates cause a Gaussian to cross a triangle boundary, the embedding must be re-expressed in the neighboring triangle to maintain differentiability. Failure to handle this breaks gradient flow [S1, unofficial].
*   **Sticking Artifacts:** Disabling the "Walking on Triangle" mechanism causes Gaussians to stick and pile up on triangle boundaries, leading to artifacts when animated by novel poses [S1, unofficial].

## Subtopics

### Walking on Triangle
A mechanism that constrains Gaussians to move within their corresponding mesh triangles, preventing them from moving beyond triangle boundaries [S1, unofficial]. It works by clipping the UV values of the Gaussians to ensure they remain within the bounds of their corresponding mesh triangles [S1, unofficial]. Disabling this mechanism causes Gaussians to stick and pile up on the boundaries of mesh triangles, leading to artifacts when animated by novel poses [S1, unofficial]. For the 'bala' head avatar, disabling walking on triangle reduces PSNR from 30.04 to 29.91, SSIM from 0.938 to 0.933, and increases LPIPS from 0.062 to 0.070. For the 'male-3-casual' full-body avatar, disabling walking on triangle reduces PSNR from 33.01 to 32.48, SSIM from 0.982 to 0.979, and increases LPIPS from 0.020 to 0.024 [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
*   **Bala Head Avatar:**
    *   With Walking on Triangle: PSNR 30.04, SSIM 0.938, LPIPS 0.062.
    *   Without Walking on Triangle: PSNR 29.91, SSIM 0.933, LPIPS 0.070.
*   **Male-3-Casual Full-Body Avatar:**
    *   With Walking on Triangle: PSNR 33.01, SSIM 0.982, LPIPS 0.020.
    *   Without Walking on Triangle: PSNR 32.48, SSIM 0.979, LPIPS 0.024.

**Strengths and Limitations:**
*   **Strengths:** Ensures that modifying the underlying mesh automatically updates the Gaussian representation. Allows for relightable modeling with explicit shape representations.
*   **Limitations:** Requires handling of boundary crossings to maintain differentiability. Disabling boundary constraints leads to significant performance drops and visual artifacts.

**When to Use:**
Use when dynamic mesh deformation is required and the Gaussian representation needs to track the mesh topology closely.

## Background
Originated in SplattingAvatar [S1, unofficial]. The method builds on the principles of mesh-embedded representations, where Gaussian geometry is defined as a function of mesh attributes.

## Sources
[S1] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
