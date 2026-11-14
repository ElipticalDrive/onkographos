# GSDF
Also called: Gaussian SDF, Gaussian Signed Distance Function

## Overview
GSDF is a training optimization technique that transfers explicit surface properties to the discrete 3D Gaussian representation by enforcing consistency with a neural Signed Distance Function (SDF) [S1]. It enhances geometry reconstruction quality and recovers detailed mesh structures, distinguishing itself from standard 3D Gaussian Splatting by explicitly modeling the zero-level set of the surface [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ 3D Gaussians $\{G_i\}$, where each Gaussian $i$ has a center $\mathbf{c}_i$, a covariance matrix $\Sigma_i$ (or rotation quaternion and scale), and an opacity $\alpha_i$. A neural SDF network $f_\theta(\mathbf{x})$ parameterized by weights $\theta$.
*   **Outputs:** Updated Gaussian attributes (centers, covariances, opacities) and SDF weights $\theta$ after optimization.

**Data Structures:**
*   **Gaussian Center:** $\mathbf{c}_i \in \mathbb{R}^3$.
*   **SDF Network:** $f_\theta: \mathbb{R}^3 \rightarrow \mathbb{R}$.

**Numbered Steps:**
1.  **SDF Evaluation:** For each Gaussian $i$, evaluate the neural SDF at its center: $d_i = f_\theta(\mathbf{c}_i)$.
2.  **SDF Consistency Loss:** Compute the loss encouraging Gaussians to lie on the surface (zero distance):
    $$ \mathcal{L}_{sdf} = \frac{1}{N} \sum_{i=1}^{N} |f_\theta(\mathbf{c}_i)| $$
    [S1]
3.  **Normal Estimation:**
    *   Compute the geometric normal $\mathbf{m}_i$ from the SDF gradient: $\mathbf{m}_i = \frac{\nabla f_\theta(\mathbf{c}_i)}{\|\nabla f_\theta(\mathbf{c}_i)\|}$.
    *   Compute the Gaussian normal $\mathbf{n}_i$ from the covariance matrix $\Sigma_i$. If $\Sigma_i = R_i S_i S_i^T R_i^T$, the principal axis is derived from the eigenvector corresponding to the smallest eigenvalue, or directly from the rotation quaternion $q_i$ [S1].
4.  **Normal Consistency Loss:** Minimize the angular difference between the Gaussian orientation and the surface geometry:
    $$ \mathcal{L}_{normal} = \frac{1}{N} \sum_{i=1}^{N} (1 - \mathbf{n}_i \cdot \mathbf{m}_i) $$
    [S1]
5.  **Total Objective:** Combine with the standard photometric rendering loss $\mathcal{L}_{rgb}$:
    $$ \mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{sdf} \mathcal{L}_{sdf} + \lambda_{normal} \mathcal{L}_{normal} $$
    [S1]
6.  **Optimization:** Perform gradient descent on both Gaussian attributes and SDF weights $\theta$.

**Parameters:**
*   $\lambda_{sdf}, \lambda_{normal}$: Weights for the geometric losses. Typical values are not given in the sources.

**Edge Cases and Pitfalls:**
*   **Degenerate Gaussians:** If a Gaussian is far from the surface, $f_\theta(\mathbf{c}_i)$ may be large, leading to unstable gradients.
*   **SDF Smoothness:** The SDF network must be trained to be differentiable and smooth to provide stable normals.
*   **Not given in the sources:** Specific initialization strategies for the SDF network or Gaussian centers.

## Operating characteristics
*   **Time Complexity:** Adds overhead for SDF evaluation and gradient computation for each Gaussian per training step.
*   **Memory Complexity:** Requires storing the SDF network weights $\theta$ in addition to Gaussian attributes.
*   **Strengths:** Recovers more detailed mesh structures and improves geometry reconstruction quality [S2, unofficial].
*   **Limitations:** Requires training an additional neural network (SDF).
*   **When to use:** When high-fidelity surface geometry and mesh extraction are required.
*   **When not:** When only photometric accuracy is needed or computational resources are severely limited.

## Background
GSDF was introduced to leverage neural signed distance functions for surface reconstruction, inspired by similar approaches used in NeRF-based methods [S2, unofficial]. It was extended in subsequent works like 2DGS, which replaced 3D Gaussians with 2D Gaussians for more accurate ray-splat intersection [S2, unofficial]. It is part of a family of methods including 3DGSR and GSPull that encourage consistency between SDF and 3DGS [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
