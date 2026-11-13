# GeoGaussian


## Overview
GeoGaussian is a training optimization and initialization strategy for 3D Gaussian Splatting that improves geometric consistency by modifying the densification process. It addresses the issue of noisy or inconsistent surface reconstruction by constraining the growth of new Gaussians to the local tangent plane of existing primitives and enforcing smoothness of geometric properties between neighbors [S1]. It sets itself apart from standard 3DGS by introducing explicit geometric priors during the densification phase, rather than relying solely on gradient-based opacity and scale thresholds [S1].

## How it works
GeoGaussian modifies the standard 3DGS densification pipeline by integrating geometric constraints derived from the local surface orientation. The method operates on the existing set of 3D Gaussians, each defined by mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$, opacity $\alpha$, and color $\mathbf{c}$ [S1].

**Inputs:**
*   Current set of 3D Gaussians $\mathcal{G} = \{G_i\}$.
*   Camera poses and image observations.
*   Gradient information from the rendering loss.

**Outputs:**
*   Updated set of 3D Gaussians $\mathcal{G}'$ with new primitives added according to geometric constraints.

**Data Structures:**
*   Each Gaussian $G_i$ retains standard attributes: $\boldsymbol{\mu}_i \in \mathbb{R}^3$, $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$ (or rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$), $\alpha_i \in \mathbb{R}$, $\mathbf{c}_i$ [S1].
*   Normal vectors $\mathbf{n}_i$ are derived from the covariance matrix or estimated from the local point cloud structure [S1].

**Steps:**
1.  **Identify Densification Candidates:** Select Gaussians $G_i$ that require densification based on standard criteria (e.g., high gradient magnitude in image space) [S1].
2.  **Compute Tangent Plane:** For each candidate Gaussian $G_i$, compute the local surface normal $\mathbf{n}_i$. The tangent plane is defined as the plane passing through $\boldsymbol{\mu}_i$ with normal $\mathbf{n}_i$ [S1].
3.  **Constrained Densification:** When splitting or cloning $G_i$ to create new Gaussians, constrain the position of the new means $\boldsymbol{\mu}_{new}$ to lie on the tangent plane of $G_i$. This is achieved by projecting the proposed new mean onto the tangent plane:
    $$ \boldsymbol{\mu}_{new} = \boldsymbol{\mu}_i + (\boldsymbol{\mu}_{prop} - \boldsymbol{\mu}_i) - ((\boldsymbol{\mu}_{prop} - \boldsymbol{\mu}_i) \cdot \mathbf{n}_i)\mathbf{n}_i $$
    where $\boldsymbol{\mu}_{prop}$ is the unconstrained proposed position [S1].
4.  **Geometric Smoothness Regularization:** Encourage smoothness of geometric properties (such as normals or positions) between neighboring Gaussians. This is implemented by adding a consistency loss term to the optimization objective that penalizes large differences in geometric attributes between adjacent Gaussians [S1].
5.  **Update:** Add the new Gaussians to the scene and continue optimization [S1].

**Parameters:**
*   Weight of the geometric smoothness loss: Not given in the sources.
*   Threshold for densification: Not given in the sources.

**Edge Cases and Pitfalls:**
*   If the normal estimation is inaccurate, the tangent plane constraint may lead to incorrect placement of new Gaussians [S1].
*   The method assumes a locally planar surface structure; it may not perform well on highly curved or non-planar regions without additional adaptations [S1].

## Subtopics

### [Spatio-temporal Gaussian Tracking](9773-spatio-temporal-gaussian-tracking.md)
Spatio-temporal Gaussian Tracking (SGT) is a method for modeling surface dynamics in dynamic scenes by tracking the attribute shift of each Gaussian at time $t$ using a deformation field $D(\mu, t)$, allowing the rendering of deformed Gaussians $G_t$. It sets itself apart from static representations by explicitly parameterizing temporal deformation through a lightweight, decomposed encoding voxel and a set of small MLPs, prioritizing real-time inference speed over large neural network approximations.

### [Progressive training by scales of Gaussians](9871-progressive-training-by-scales-of-gaussians.md)
This method prevents rendering artifacts caused by 3D Gaussians becoming smaller than a single pixel during optimization by enforcing a minimum projected area through covariance regularization. It differs from standard 3D Gaussian Splatting by introducing a scale-dependent low-pass filter that focuses training on low-frequency details in early stages, ensuring coarse scene structure is learned efficiently before fine details are resolved.

## Operating characteristics
*   **Time Complexity:** Increases training time due to the additional computation of normal vectors and the application of geometric constraints during densification [S1].
*   **Memory Complexity:** No significant additional memory overhead compared to standard 3DGS, as it uses the same primitive representation [S1].
*   **Strengths:** Improves geometric consistency and reduces noise in reconstructed surfaces [S1].
*   **Limitations:** May struggle with highly curved surfaces if the planar assumption is violated [S1].
*   **When to use:** When high-quality geometric reconstruction is prioritized over raw training speed [S1].
*   **When not to use:** For scenes with significant non-planar structures or when computational resources are severely limited [S1].

## Background
GeoGaussian was proposed in the paper "GeoGaussian" [41], as cited in "Recent advances in 3D Gaussian splatting" [S1]. It is part of the broader effort to improve the geometric fidelity of 3D Gaussian Splatting by incorporating explicit geometric priors into the training process [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
