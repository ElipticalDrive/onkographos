# GeoGaussian
## Overview
GeoGaussian is a training optimization and initialization method for 3D Gaussian Splatting that improves geometric consistency by densifying primitives on the tangent planes of existing Gaussians and enforcing smoothness of geometric properties between neighbors [S1][S2, unofficial]. It addresses the irregular distribution and geometric discontinuities common in standard 3D Gaussian Splatting by incorporating plane constraints and normal consistency into the densification process [S1][S2, unofficial].
## How it works
GeoGaussian modifies the standard densification and optimization pipeline of 3D Gaussian Splatting. It retains the primitive definition of position $\mathbf{p}_i$, covariance $\boldsymbol{\Sigma}_i$ (parameterized by rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$), opacity $\alpha_i$, and color $\mathbf{c}_i$ [Parent].

**Inputs:**
- Current set of 3D Gaussians $\mathcal{G} = \{G_i\}$.
- Training images and camera poses.
- Gradient information from the rendering loss.

**Outputs:**
- Updated set of 3D Gaussians with improved geometric coherence.

**Data Structures:**
- Each Gaussian $i$ stores its position $\mathbf{p}_i$, rotation matrix $\mathbf{R}_i$, scale vector $\mathbf{s}_i$, opacity $\alpha_i$, and color coefficients $\mathbf{c}_i$.
- The covariance matrix is $\boldsymbol{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T$, where $\mathbf{S}_i = \text{diag}(\mathbf{s}_i)$ [Parent].

**Steps:**
1. **Tangent Plane Densification:** When a Gaussian $i$ requires densification (based on standard gradient thresholds), new Gaussians are not spawned randomly or uniformly in space. Instead, they are placed on the tangent plane of the parent Gaussian $i$. The tangent plane is defined by the normal vector $\mathbf{n}_i$ derived from the orientation of $\mathbf{R}_i$. New positions $\mathbf{p}_{new}$ are sampled such that $(\mathbf{p}_{new} - \mathbf{p}_i)^T \mathbf{n}_i = 0$ [S1][S2, unofficial].
2. **Geometric Smoothness Regularization:** To encourage smoothness of geometric properties between neighboring Gaussians, a regularization term is added to the loss function. This term penalizes large differences in normals or positions between adjacent Gaussians. Let $\mathcal{N}(i)$ be the set of neighbors of Gaussian $i$. The smoothness loss $L_{smooth}$ can be formulated as:
   $$ L_{smooth} = \sum_{i} \sum_{j \in \mathcal{N}(i)} w_{ij} \| \mathbf{n}_i - \mathbf{n}_j \|^2 $$
   where $\mathbf{n}_i$ is the normal of Gaussian $i$ and $w_{ij}$ is a weighting factor based on proximity or overlap [S1][S2, unofficial].
3. **Optimization:** The total loss is the sum of the rendering loss (photometric error) and the geometric smoothness loss. The Gaussians are optimized using gradient descent, updating $\mathbf{p}_i, \mathbf{R}_i, \mathbf{s}_i, \alpha_i, \mathbf{c}_i$ [S1][S2, unofficial].

**Parameters:**
- Densification threshold: Standard 3DGS thresholds apply, but applied to the tangent plane context [S1].
- Smoothness weight: Not given in the sources.
- Neighbor definition: Not given in the sources.

**Edge Cases and Pitfalls:**
- Degenerate normals: If a Gaussian is highly anisotropic or has undefined orientation, the tangent plane is ill-defined. [G] A small epsilon or fallback to a default normal may be required.
- Neighbor selection: Defining "neighboring" Gaussians efficiently is critical. [G] Spatial hashing or k-d trees are typically used to find neighbors.
- Precision: Ensuring new Gaussians lie exactly on the tangent plane may require projection steps. [G] Project $\mathbf{p}_{new} \leftarrow \mathbf{p}_{new} - ((\mathbf{p}_{new} - \mathbf{p}_i)^T \mathbf{n}_i) \mathbf{n}_i$.
## Subtopics

### [Spatio-temporal Gaussian Tracking](9773-spatio-temporal-gaussian-tracking.md)
Spatio-temporal Gaussian Tracking (SGT) is a method for modeling surface dynamics in dynamic scenes by tracking the attribute shift of each Gaussian at time $t$ using a deformation field $D(\mu, t)$, allowing the rendering of deformed Gaussians $G_t$. It sets itself apart from static representations by explicitly parameterizing temporal deformation through a lightweight, decomposed encoding voxel and a set of small MLPs, prioritizing real-time inference speed over large neural network approximations.

### [Progressive training by scales of Gaussians](9871-progressive-training-by-scales-of-gaussians.md)
This method prevents rendering artifacts caused by 3D Gaussians becoming smaller than a single pixel during optimization by enforcing a minimum projected area through covariance regularization. It differs from standard 3D Gaussian Splatting by introducing a scale-dependent low-pass filter that focuses training on low-frequency details in early stages, ensuring coarse scene structure is learned efficiently before fine details are resolved.

## Operating characteristics
Time and memory complexity: Not given in the sources.
Measured performance with its hardware, resolution and scene: Not given in the sources.
Scaling: Not given in the sources.
Strengths and limitations: Improves geometric consistency and smoothness of the reconstructed surface [S1][S2, unofficial]. May introduce additional computational overhead due to neighbor search and smoothness regularization [G].
When to use it and when not: Use when geometric accuracy and surface smoothness are prioritized over raw rendering speed or simplicity. Not given in the sources for when not to use.
## Background
Origin: GeoGaussian was introduced in the paper "GeoGaussian: Geometry-aware Gaussian Splatting for Scene Rendering" [S1][S2, unofficial].
History and development: It is part of the recent advances in 3D Gaussian Splatting that focus on improving the geometric properties of the representation [S1][S2, unofficial]. It builds on the standard 3D Gaussian Splatting framework [Parent].
Where it is used: Used in applications requiring high-quality geometric reconstruction from multi-view images [S1][S2, unofficial].
## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
