# GeoGaussian


## Overview
GeoGaussian is a method for improving the geometric consistency and surface smoothness of 3D Gaussian Splatting representations by modifying the densification and initialization strategies. It addresses the issue of noisy or discontinuous geometry often resulting from standard 3DGS optimization by enforcing normal consistency between neighboring Gaussians and constraining new Gaussians to lie on the tangent planes of existing ones. It sets itself apart from standard 3DGS by prioritizing geometric smoothness over raw appearance fidelity during the densification phase, distinguishing it from siblings like [Negative Gaussian Splatting](9152-negative-gaussian-splatting.md) which modifies opacity kernels, or [Scaffold-GS](9186-scaffold-gs.md) which uses voxel-based scaffolding.

## How it works
GeoGaussian operates within the standard differentiable rasterization pipeline of 3D Gaussian Splatting [Parent](10145-representation-and-kernel-modifications.md). It modifies the **Primitive Attributes** and the **Densification Process** to ensure geometric coherence.

**Inputs and Outputs:**
- **Input:** A set of 3D Gaussians $\{G_i\}$ with positions $\mu_i$, covariances $\Sigma_i$, opacities $\alpha_i$, and spherical harmonic coefficients $c_i$, along with camera poses and 2D image observations.
- **Output:** An optimized set of Gaussians with improved geometric smoothness and normal consistency.

**Data Structures:**
The Gaussian primitive retains the standard 3DGS structure:
- $\mu \in \mathbb{R}^3$: Mean position.
- $\Sigma \in \mathbb{R}^{3 \times 3}$: Covariance matrix (often parameterized via rotation $R$ and scale $S$).
- $\alpha \in [0, 1]$: Opacity.
- $c$: Spherical Harmonic coefficients for view-dependent color.
- **New/Modified Attributes:** Normal vector $n_i$ derived from the covariance matrix or explicitly optimized to enforce smoothness.

**Steps:**
1. **Tangent Plane Densification:** When densification is triggered (e.g., based on gradient magnitude or depth error), new Gaussians are not spawned randomly in 3D space. Instead, they are projected onto the **tangent plane** of an existing parent Gaussian $G_p$.
   - Let $n_p$ be the normal of $G_p$.
   - A new point $\mu_{new}$ is sampled on the tangent plane: $\mu_{new} = \mu_p + v$, where $v \cdot n_p = 0$.
   - This ensures new primitives are geometrically aligned with the local surface.

2. **Normal Consistency Regularization:** To encourage smoothness of geometric properties between neighboring Gaussians, a regularization term is added to the loss function.
   - Let $n_i$ and $n_j$ be the normals of neighboring Gaussians $i$ and $j$.
   - The smoothness loss $L_{smooth}$ is defined as:
     $$ L_{smooth} = \sum_{(i,j) \in \mathcal{N}} (1 - n_i \cdot n_j) $$
     where $\mathcal{N}$ is the set of pairs of neighboring Gaussians (e.g., those sharing a common parent or spatially adjacent).
   - This term penalizes discontinuities in surface orientation.

3. **Plane Constraints:** GeoGaussian introduces plane constraints to further stabilize the geometry. This involves fitting local planes to clusters of Gaussians and penalizing deviations from these planes.
   - For a cluster of Gaussians, a plane equation $ax + by + cz + d = 0$ is estimated.
   - The distance of each Gaussian center $\mu_i$ to this plane is minimized.

4. **Optimization:** The total loss is a weighted sum of the standard photometric loss $L_{photo}$ and the geometric losses:
   $$ L_{total} = L_{photo} + \lambda_{smooth} L_{smooth} + \lambda_{plane} L_{plane} $$
   where $\lambda_{smooth}$ and $\lambda_{plane}$ are hyperparameters.

**Parameters:**
- $\lambda_{smooth}$: Weight for normal consistency. Typical values are small (e.g., $10^{-3}$ to $10^{-2}$) to avoid over-smoothing appearance details.
- $\lambda_{plane}$: Weight for plane constraints.
- Densification threshold: Standard 3DGS thresholds (e.g., gradient magnitude) are used to trigger the tangent plane densification.

**Edge Cases and Pitfalls:**
- **Degenerate Normals:** If a Gaussian is very flat or has high anisotropy, the normal estimation may be unstable. [G] A guard should check the condition number of the covariance matrix before computing normals.
- **Over-smoothing:** Excessive geometric regularization can blur fine details and reduce photometric accuracy.
- **Tangent Plane Ambiguity:** If the parent Gaussian's normal is poorly estimated, new Gaussians will be misaligned. [G] Iterative refinement of normals may be required.

## Subtopics

### [Spatio-temporal Gaussian Tracking](9773-spatio-temporal-gaussian-tracking.md)
Spatio-temporal Gaussian Tracking (SGT) is a method for modeling surface dynamics in dynamic scenes by tracking the attribute shift of each Gaussian at time $t$ using a deformation field $D(\mu, t)$, allowing the rendering of deformed Gaussians $G_t$. It sets itself apart from static representations by explicitly parameterizing temporal deformation through a lightweight, decomposed encoding voxel and a set of small MLPs, prioritizing real-time inference speed over large neural network approximations.

### [Progressive training by scales of Gaussians](9871-progressive-training-by-scales-of-gaussians.md)
This method prevents rendering artifacts caused by 3D Gaussians becoming smaller than a single pixel during optimization by enforcing a minimum projected area through covariance regularization. It differs from standard 3D Gaussian Splatting by introducing a scale-dependent low-pass filter that focuses training on low-frequency details in early stages, ensuring coarse scene structure is learned efficiently before fine details are resolved.

## Operating characteristics
- **Time Complexity:** Densification on tangent planes adds a small overhead to the standard densification step, primarily due to normal computation and plane fitting. The optimization loop adds the cost of computing normal consistency terms, which is linear in the number of neighbor pairs.
- **Memory Complexity:** No significant additional memory is required beyond standard 3DGS, as normals can be derived from covariances or stored as small additional attributes.
- **Performance:** Improves geometric smoothness and reduces noise in reconstructed surfaces. May slightly reduce photometric accuracy if regularization weights are too high.
- **Strengths:** Better surface continuity, reduced artifacts in regions with sparse observations.
- **Limitations:** May blur fine geometric details; requires careful tuning of regularization weights.
- **When to use:** When geometric consistency is more important than raw photometric fidelity, or when dealing with scenes with sparse views.
- **When not to use:** When real-time performance is critical and the overhead of geometric regularization is unacceptable, or when the scene has complex, non-smooth geometry.

## Background
GeoGaussian was proposed in the context of recent advances in 3D Gaussian Splatting, specifically to address the issue of geometric inconsistency in the standard method. It builds on the idea of using geometric constraints to improve the quality of the reconstructed scene. The method is cited as [41] in the source literature [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
