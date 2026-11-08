# 3DGUT
Also called: 3D Gaussian Uncentred Transform
## Overview
3DGUT is a Gaussian projection technique that employs an unscented transform to improve the affine approximation of the projection operator, providing more precise Gaussian projection onto the image plane [S1][S2, unofficial]. It addresses the accuracy limitations of the standard linearization used in Gaussian Splatting by capturing higher-order statistical moments of the projection, thereby reducing approximation errors in the resulting 2D covariance [S1][S2, unofficial].

## How it works
3DGUT modifies the standard Gaussian Splatting projection pipeline by replacing the first-order Taylor expansion (affine approximation) of the projection function with an unscented transform. This allows for a more accurate estimation of the mean and covariance of the projected Gaussian.

**Inputs:**
- 3D Gaussian parameters: Mean $\mu \in \mathbb{R}^3$, Covariance $\Sigma \in \mathbb{R}^{3 \times 3}$ (symmetric positive definite).
- Camera parameters: View matrix $V$, Projection matrix $P$, Intrinsics $K$.

**Outputs:**
- 2D Gaussian parameters: Mean $\mu_{2D} \in \mathbb{R}^2$, Covariance $\Sigma_{2D} \in \mathbb{R}^{2 \times 2}$.

**Data Structures:**
- Sigma Points: A set of $2n+1$ points (where $n=3$ for 3D Gaussians) derived from the mean and covariance.
- Weights: Scalar weights associated with each sigma point for mean and covariance calculation.

**Steps:**
1. **Generate Sigma Points:** Calculate $2n+1$ sigma points $\chi_i$ from the 3D mean $\mu$ and covariance $\Sigma$. Let $L$ be the lower triangular Cholesky factor of $\Sigma$ such that $\Sigma = LL^T$. The points are defined as:
   $$ \chi_0 = \mu $$
   $$ \chi_i = \mu + (\sqrt{(n+\kappa)L})_i \quad \text{for } i=1,\dots,n $$
   $$ \chi_{i+n} = \mu - (\sqrt{(n+\kappa)L})_i \quad \text{for } i=1,\dots,n $$
   where $\kappa$ is a scaling parameter (typically $\kappa = 3-n$ for optimal performance in 3D) and $(\sqrt{(n+\kappa)L})_i$ denotes the $i$-th column of the matrix square root scaled by $(n+\kappa)$.

2. **Propagate Sigma Points:** Apply the full non-linear projection function $f(\cdot)$ to each sigma point $\chi_i$. This involves transforming the 3D point to camera coordinates, projecting to homogeneous coordinates, and dividing by depth to get 2D image coordinates.
   $$ \tilde{\chi}_i = f(\chi_i) $$

3. **Calculate Weighted Mean:** Compute the projected mean $\mu_{2D}$ using the weighted sum of the projected sigma points.
   $$ \mu_{2D} = \sum_{i=0}^{2n} W_i^m \tilde{\chi}_i $$
   where $W_0^m = \frac{\kappa}{n+\kappa}$ and $W_i^m = \frac{1}{2(n+\kappa)}$ for $i > 0$.

4. **Calculate Weighted Covariance:** Compute the projected covariance $\Sigma_{2D}$ using the weighted outer products of the deviations from the mean.
   $$ \Sigma_{2D} = \sum_{i=0}^{2n} W_i^c (\tilde{\chi}_i - \mu_{2D})(\tilde{\chi}_i - \mu_{2D})^T + Q $$
   where $W_0^c = \frac{\kappa}{n+\kappa} - 1 + \beta$ and $W_i^c = \frac{1}{2(n+\kappa)}$ for $i > 0$. $\beta$ is a parameter (typically $\beta=2$ for Gaussian distributions) and $Q$ is a small regularization term to ensure positive definiteness if necessary.

**Parameters:**
- $\kappa$: Scaling parameter. Typical value: $3-n$ (i.e., 0 for $n=3$).
- $\beta$: Incorporates prior knowledge of the distribution. Typical value: 2 for Gaussian distributions.

**Edge Cases and Pitfalls:**
- **Positive Definiteness:** The resulting $\Sigma_{2D}$ must remain positive semi-definite. Numerical errors in the unscented transform can occasionally violate this. A small regularization term $Q$ or a check to enforce positive definiteness is required [G].
- **Depth Zero:** If any sigma point projects to a depth of zero or negative (behind the camera), the projection is invalid. These points should be handled by clamping depth or excluding them, though this is rare for well-formed Gaussians [G].
- **Computational Cost:** Generating and propagating $2n+1$ points is more expensive than a single affine approximation, though still efficient compared to full quadrature.

## Subtopics

### [Unnormalized 3D Gaussian Function](1026-unnormalized-3d-gaussian-function.md)
Defines the base Gaussian density. 3DGUT operates on these functions but changes the projection step.

### [Row-constant hoisting for Gaussian Splatting Rasterization](1195-row-constant-hoisting-for-gaussian-splatting-rasterization.md)
Optimizes rasterization. 3DGUT is a pre-rasterization projection method.

### [2D Gaussian Projection with Covariance Stabilization](1198-2d-gaussian-projection-with-covariance-stabilization.md)
Another projection refinement. 3DGUT uses unscented transform; this method uses stabilization terms.

### [Gaussian in the Wild (GS-W)](1386-gaussian-in-the-wild-gs-w.md)
Handles wild data. 3DGUT can be integrated into GS-W for better projection accuracy.

### [4D Gaussian Splatting](1458-4d-gaussian-splatting.md)
Adds time. 3DGUT can be applied per-frame.

### [Transformer-based Reconstruction](1794-transformer-based-reconstruction.md)
Alternative reconstruction. 3DGUT is a rendering/projection technique.

### [3D Gaussian Ellipsoid Rendering](1869-3d-gaussian-ellipsoid-rendering.md)
Alternative shape. 3DGUT is specific to Gaussians.

### [3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)](2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md)
Rendering backend. 3DGUT provides the projected Gaussians.

### [3D Gaussian Splatting Rendering (Software)](2805-3d-gaussian-splatting-rendering-software.md)
Rendering backend. 3DGUT provides the projected Gaussians.

### [3D Gaussian Splatting Rasterization](2812-3d-gaussian-splatting-rasterization.md)
Rasterization step. 3DGUT provides the 2D Gaussians.

## Operating characteristics
- **Time Complexity:** $O(n^3)$ for Cholesky decomposition and sigma point generation, plus $O(n)$ projections. For $n=3$, this is constant and small. More expensive than standard affine projection ($O(1)$) but negligible compared to rasterization.
- **Memory Complexity:** $O(n^2)$ to store sigma points and weights. Minimal overhead.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of Gaussians.
- **Strengths:** More accurate projection than affine approximation, especially for large or highly anisotropic Gaussians. Reduces artifacts caused by projection errors.
- **Limitations:** Slightly higher computational cost than affine projection. Requires careful handling of numerical stability.
- **When to use:** When high fidelity is required and projection artifacts are visible.
- **When not:** When performance is critical and standard projection is sufficient.

## Background
3DGUT was introduced to address the limitations of the affine approximation in Gaussian Splatting projection. It leverages the unscented transform, a technique from state estimation, to provide a more precise Gaussian projection onto the image plane [S1][S2, unofficial]. The method is cited in recent surveys on 3D Gaussian Splatting as a way to improve rendering accuracy [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
