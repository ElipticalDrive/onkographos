# GeoGaussian


## Overview
GeoGaussian is a training optimization and initialization strategy for 3D Gaussian Splatting that improves geometric consistency and surface reconstruction quality. It addresses the issue of noisy or inconsistent geometry in standard 3DGS by densifying Gaussians on the tangent plane of existing primitives and enforcing smoothness of geometric properties between neighboring Gaussians [S1]. It sets itself apart from siblings like Negative Gaussian Splatting by focusing on positive geometric propagation and smoothness rather than negative space representation, and from Scaffold-GS by using local tangent plane densification rather than global scaffold structures [S1].

## How it works
GeoGaussian modifies the standard 3D Gaussian Splatting optimization process by introducing two key mechanisms: tangent plane densification and geometric smoothness regularization.

**Inputs and Outputs:**
- **Input:** A set of 3D Gaussians $\mathcal{G} = \{G_i\}$, where each $G_i$ is defined by position $\mathbf{p}_i$, covariance $\boldsymbol{\Sigma}_i$ (or rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$), opacity $\alpha_i$, and color $\mathbf{c}_i$.
- **Output:** An updated set of Gaussians with improved geometric consistency.

**Data Structures:**
- Each Gaussian $G_i$ retains the standard 3DGS attributes: $\mathbf{p}_i \in \mathbb{R}^3$, $\mathbf{R}_i \in SO(3)$, $\mathbf{s}_i \in \mathbb{R}^3$, $\alpha_i \in [0,1]$, $\mathbf{c}_i$.
- Normal vectors $\mathbf{n}_i$ are derived from the covariance matrix or estimated from the local geometry.

**Steps:**

1. **Tangent Plane Densification:**
   For each Gaussian $G_i$, compute its tangent plane at $\mathbf{p}_i$ using the normal $\mathbf{n}_i$. New Gaussians are spawned on this tangent plane rather than in the full 3D volume. This ensures that new primitives align with the local surface geometry, reducing noise and improving surface fidelity.
   - Let $\mathbf{T}_i$ be the basis vectors of the tangent plane orthogonal to $\mathbf{n}_i$.
   - New positions $\mathbf{p}_{new}$ are sampled as $\mathbf{p}_{new} = \mathbf{p}_i + u \mathbf{t}_1 + v \mathbf{t}_2$, where $\mathbf{t}_1, \mathbf{t}_2$ are tangent basis vectors and $u, v$ are small offsets.

2. **Geometric Smoothness Regularization:**
   Encourage smoothness of geometric properties (such as normals and positions) between neighboring Gaussians. This is achieved by adding a regularization term to the loss function.
   - Let $\mathcal{N}(i)$ be the set of neighbors of Gaussian $G_i$.
   - The smoothness loss $\mathcal{L}_{smooth}$ is defined as:
     $$ \mathcal{L}_{smooth} = \sum_{i} \sum_{j \in \mathcal{N}(i)} w_{ij} \left( \| \mathbf{n}_i - \mathbf{n}_j \|^2 + \| \mathbf{p}_i - \mathbf{p}_j \|^2 \right) $$
     where $w_{ij}$ is a weight based on the distance or similarity between $G_i$ and $G_j$.

3. **Optimization:**
   Update the Gaussian parameters using gradient descent, minimizing the combined loss:
   $$ \mathcal{L}_{total} = \mathcal{L}_{render} + \lambda \mathcal{L}_{smooth} $$
   where $\mathcal{L}_{render}$ is the standard rendering loss (e.g., L1 or SSIM) and $\lambda$ is a hyperparameter controlling the strength of the smoothness regularization.

**Parameters:**
- $\lambda$: Weight for smoothness regularization. Typical value: Not given in the sources.
- Tangent plane offset scale: Controls the spread of new Gaussians on the tangent plane. Typical value: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Degenerate Normals:** If the normal estimation is inaccurate, the tangent plane may be misaligned, leading to poor densification. [G] Ensure robust normal estimation, possibly using local PCA or neighbor averaging.
- **Over-smoothing:** High $\lambda$ may lead to overly smooth geometry, losing fine details. [G] Tune $\lambda$ based on scene complexity.

## Subtopics

### [Spatio-temporal Gaussian Tracking](9773-spatio-temporal-gaussian-tracking.md)
Spatio-temporal Gaussian Tracking (SGT) is a method for modeling surface dynamics in dynamic scenes by tracking the attribute shift of each Gaussian at time $t$ using a deformation field $D(\mu, t)$, allowing the rendering of deformed Gaussians $G_t$. It sets itself apart from static representations by explicitly parameterizing temporal deformation through a lightweight, decomposed encoding voxel and a set of small MLPs, prioritizing real-time inference speed over large neural network approximations.

### [Progressive training by scales of Gaussians](9871-progressive-training-by-scales-of-gaussians.md)
This method prevents rendering artifacts caused by 3D Gaussians becoming smaller than a single pixel during optimization by enforcing a minimum projected area through covariance regularization. It differs from standard 3D Gaussian Splatting by introducing a scale-dependent low-pass filter that focuses training on low-frequency details in early stages, ensuring coarse scene structure is learned efficiently before fine details are resolved.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Improves geometric consistency and surface reconstruction quality; reduces noise in the reconstructed geometry.
- **Limitations:** May require careful tuning of the smoothness regularization parameter; computationally more expensive than standard 3DGS due to neighbor calculations and additional loss terms.
- **When to use:** When high-quality geometric reconstruction is required, such as in architectural scenes or objects with smooth surfaces.
- **When not to use:** When real-time performance is critical and geometric accuracy is less important than rendering speed.

## Background
GeoGaussian was proposed in the paper "GeoGaussian: Geometrically Consistent Gaussian Splatting for Dynamic Scene Representation" [S1]. It is part of the broader effort to improve the geometric fidelity of 3D Gaussian Splatting, which is known for its high-quality rendering but sometimes suffers from noisy or inconsistent geometry. The method builds on the shared principles of 3D Gaussian Splatting, focusing on modifying the training process to enforce geometric smoothness.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
