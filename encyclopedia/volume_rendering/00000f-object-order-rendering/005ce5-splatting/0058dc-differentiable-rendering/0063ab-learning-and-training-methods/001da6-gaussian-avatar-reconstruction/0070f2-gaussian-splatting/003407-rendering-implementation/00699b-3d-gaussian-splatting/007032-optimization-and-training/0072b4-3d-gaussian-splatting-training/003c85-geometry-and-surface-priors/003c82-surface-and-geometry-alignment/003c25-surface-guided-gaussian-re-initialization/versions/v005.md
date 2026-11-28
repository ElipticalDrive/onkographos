# Surface-guided Gaussian Re-initialization

Also called: GVA

## Overview
Surface-guided Gaussian Re-initialization (GVA) is a training strategy designed to resolve the unbalanced aggregation of 3D Gaussians caused by densification and splitting operations during the optimization of dynamic human avatars from monocular video [S1][S2, unofficial]. It makes trained Gaussians better fit the input monocular video by anchoring their positions to a canonical surface geometry, distinguishing it from unconstrained splatting methods [S1][S2, unofficial].

## How it works
**Inputs:**
- A set of 3D Gaussians $\{G_i\}$ with current parameters (position $\mathbf{c}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, spherical harmonics coefficients).
- A canonical human body mesh $\mathcal{M}$ (e.g., SMPL or similar parametric model).
- Camera poses and image observations.

**Data Structures:**
- **Canonical Mesh:** Defined by vertices $\mathbf{V}$ and faces $\mathcal{F}$. Each face $f \in \mathcal{F}$ is a triangle with vertices $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$.
- **Barycentric Coordinates:** For any point $\mathbf{p}$ projected onto a face, represented as $(\lambda_1, \lambda_2, \lambda_3)$ where $\sum \lambda_k = 1$ and $\lambda_k \ge 0$.
- **Surface Normal:** $\mathbf{n}_f$ for face $f$, computed as the normalized cross product of edge vectors.

**Steps:**
1. **Ray Casting / Projection:** For each Gaussian center $\mathbf{c}_i$ in the canonical space, determine its closest point on the canonical mesh $\mathcal{M}$. This is typically done by projecting $\mathbf{c}_i$ onto the mesh surface to find the intersecting face $f$ and barycentric coordinates $(\lambda_1, \lambda_2, \lambda_3)$ [S1][S2, unofficial].
2. **Surface Point Calculation:** Compute the projected point $\mathbf{p}_{surf}$ on the mesh surface using the barycentric interpolation:
   $$ \mathbf{p}_{surf} = \lambda_1 \mathbf{v}_1 + \lambda_2 \mathbf{v}_2 + \lambda_3 \mathbf{v}_3 $$
3. **Normal Displacement:** Calculate the surface normal $\mathbf{n}_f$ at face $f$. The Gaussian position is adjusted or constrained relative to this surface. In GVA, the position is determined by the barycentric point and a displacement along the normal direction [S1][S2, unofficial]. Let $d_i$ be the signed distance or displacement parameter. The re-initialized position $\mathbf{c}_i'$ is:
   $$ \mathbf{c}_i' = \mathbf{p}_{surf} + d_i \mathbf{n}_f $$
   Note: In some implementations, $d_i$ might be optimized or fixed to zero to strictly enforce surface alignment, effectively "snapping" the Gaussian to the mesh [S1][S2, unofficial].
4. **Covariance Adjustment:** The orientation of the Gaussian is aligned with the surface normal. The rotation matrix or quaternion of the Gaussian is updated such that one principal axis aligns with $\mathbf{n}_f$, ensuring the Gaussian sheet lies tangent to the surface [G]. This reduces the "floating" artifacts and unbalanced aggregation.
5. **Re-initialization:** Replace the original Gaussian parameters with the re-initialized ones. This step is performed periodically during training to counteract the drift and clustering issues caused by standard densification/splitting heuristics [S1][S2, unofficial].

**Parameters:**
- Displacement weight: Often zero or small, depending on whether the Gaussian is strictly constrained to the surface or allowed slight offset [S1][S2, unofficial].
- Frequency of re-initialization: Not given in the sources.

**Edge Cases:**
- **Degenerate Faces:** If the mesh has degenerate triangles, barycentric coordinates may be undefined. [G] Handle by skipping or using a fallback projection.
- **Off-Mesh Gaussians:** If a Gaussian is far from the mesh, the closest point search may fail or snap to an incorrect region. [G] Implement a distance threshold; if distance > threshold, ignore or re-project using a broader search.

## Subtopics

## Operating characteristics
- **Time Complexity:** Depends on the mesh traversal for closest point projection. Typically $O(N)$ per Gaussian if using efficient spatial hashing or BVH for mesh intersection.
- **Memory Complexity:** Requires storing the canonical mesh and its acceleration structure.
- **Strengths:** Resolves unbalanced aggregation of Gaussians; improves geometric fidelity to the input monocular video; reduces floating artifacts.
- **Limitations:** Relies on the accuracy of the canonical mesh; may restrict the Gaussian's ability to model non-surface details (e.g., hair, loose clothing) if strictly constrained.
- **When to use:** Dynamic human avatar reconstruction from monocular video where surface consistency is critical.
- **When not to use:** Scenes without a clear underlying surface or when modeling volumetric objects not attached to a mesh.

## Background
GVA was proposed in the context of 3D Gaussian Splatting for dynamic human avatar reconstruction. It addresses the issue of Gaussian aggregation and drift during training from monocular video. The method builds on the idea of embedding Gaussians onto a canonical human body mesh, similar to SplattingAvatar and GoMAvatar [S1][S2, unofficial]. It is cited in recent advances surveys of 3D Gaussian Splatting [S1][S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
