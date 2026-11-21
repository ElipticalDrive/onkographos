# Volume Encoding Gaussians

Also called: Transfer Function-Agnostic 3D Gaussians for Volume Rendering

## Overview
Volume Encoding Gaussians adapts 3D Gaussian Splatting for volume data visualization by removing per-Gaussian color and opacity attributes, training only scalar values to enable interactive transfer function application post-training [S1, unofficial]. It differs from standard Gaussian Splatting Volume Rendering by decoupling the radiometric representation from the geometric density, allowing the same underlying scalar field to be visualized with arbitrary transfer functions without retraining [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A dataset of multi-view images of a volume (e.g., medical scans, fluid simulations) and corresponding camera poses.
- **Output:** A set of 3D Gaussians defined by mean position $\mu$, covariance $\Sigma$, and a scalar value $s$.

**Data Structures:**
Each Gaussian $i$ stores:
- $\mu_i \in \mathbb{R}^3$: Mean position.
- $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix (often parameterized by rotation $R$ and scale $S$).
- $s_i \in \mathbb{R}$: Scalar value (e.g., density, temperature, concentration).
- *Note:* Color $c_i$ and opacity $\alpha_i$ are **not** stored as learnable attributes [S1, unofficial].

**Steps:**
1. **Initialization:** Initialize Gaussians in 3D space, typically using point cloud data or random initialization within the volume bounds.
2. **Rendering:** For each pixel $\mathbf{p}$, project Gaussians to 2D screen space. Compute the 2D Gaussian value $\tilde{\mathcal{G}}_i(\mathbf{p})$ using the projected covariance $\Sigma_{2D}$ [G].
3. **Compositing:** Render the scalar field using front-to-back accumulation. The accumulated scalar $S_f$ at pixel $\mathbf{p}$ is:
   $$ S_f(\mathbf{p}) = \sum_{i=1}^N s_i \tilde{\sigma}_i \tilde{\mathcal{G}}_i(\mathbf{p}) \prod_{j=1}^{i-1} (1 - \tilde{\sigma}_j \tilde{\mathcal{G}}_j(\mathbf{p})) $$
   where $\tilde{\sigma}_i$ is a fixed or learned opacity scaling factor (if not removed entirely, but typically opacity is part of the transfer function or fixed for visualization) [S1, unofficial]. *Clarification:* The method focuses on training $s_i$. The opacity $\alpha$ might be fixed or derived, but the key is that color is absent [S1, unofficial].
4. **Transfer Function Application:** After rendering the scalar field $S_f$, apply a user-defined transfer function $T(s)$ to map the scalar value to color $c$ and final opacity $\alpha$:
   $$ c = T_c(S_f), \quad \alpha = T_\alpha(S_f) $$
   This step is performed post-rendering, enabling interactive visualization [S1, unofficial].
5. **Optimization:** Minimize the difference between the rendered color (after applying a target transfer function) and the ground truth image. The loss is typically:
   $$ \mathcal{L} = \mathcal{L}_{RGB} + \lambda \mathcal{L}_{depth} $$
   where $\mathcal{L}_{RGB}$ is computed using the transfer function applied to the rendered scalars [S1, unofficial].

**Parameters:**
- Learning rate for $\mu, \Sigma, s$: Standard Gaussian Splatting rates [G].
- Transfer Function: User-defined, e.g., linear, exponential, or custom LUT [S1, unofficial].

**Edge Cases:**
- **Degenerate Gaussians:** Standard Gaussian Splatting safeguards (e.g., resetting scale) apply [G].
- **Transfer Function Sensitivity:** Poor transfer function choice may lead to low contrast or saturation, but does not affect the underlying scalar field quality [S1, unofficial].

## Subtopics

### [GVGEN](8091-gvgen.md)
GVGEN is a method for 3D generation that operates in 3D space using a 3D Gaussian volume representation. It is distinguished from sibling methods like GaussianCube, which converts a constant number of Gaussians into a voxelized representation via optimal transport, by maintaining a direct 3D Gaussian volume structure.

## Operating characteristics
- **Time Complexity:** Similar to 3D Gaussian Splatting, $O(N)$ per pixel for compositing, where $N$ is the number of Gaussians. Transfer function application is $O(1)$ per pixel.
- **Memory Complexity:** Reduced compared to standard Gaussian Splatting as color and opacity are not stored per Gaussian, only scalar values [S1, unofficial].
- **Strengths:** Interactive transfer function application; no retraining required for different visualizations; efficient storage [S1, unofficial].
- **Limitations:** Requires a suitable transfer function for each dataset; scalar values may not capture complex radiometric properties directly [S1, unofficial].
- **When to Use:** Volume visualization where multiple transfer functions are desired for the same data; interactive exploration [S1, unofficial].
- **When Not to Use:** When color is intrinsically tied to the geometry in a way that cannot be separated by a scalar field [S1, unofficial].

## Background
Originated from the work "Transfer Function-Agnostic 3D Gaussians for Volume Rendering" [S1, unofficial]. It extends 3D Gaussian Splatting to handle volume data by focusing on scalar field representation rather than direct color rendering [S1, unofficial].

## Sources
[S1] ldyken53/VEG, https://github.com/ldyken53/VEG, fetched 2026-10-26 (unofficial)
