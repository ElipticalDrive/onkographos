# Parallelization for Backpropagation
Also called: Per-splat Backpropagation

## Overview
Parallelization for Backpropagation is a strategy for computing gradients in 3D Gaussian Splatting that shifts the computational granularity from per-pixel to per-splat (per-Gaussian) [S1]. It solves the high atomic contention and false sharing inherent in standard per-pixel backpropagation, where thousands of threads simultaneously attempt to update the same Gaussian parameters [S1]. By restructuring the backward pass, it significantly accelerates training time while maintaining numerical equivalence to the standard approach [S1].

## How it works
The method replaces the standard pixel-parallel gradient accumulation with a per-splat iteration order.

**Inputs:**
- Set of 3D Gaussians $\mathcal{G}$ with parameters $\theta_i$ (position, covariance, opacity, color, SH coefficients).
- Camera parameters for the current view.
- Loss function $\mathcal{L}$ and its gradient with respect to the rendered image $\frac{\partial \mathcal{L}}{\partial \mathbf{c}}$.

**Outputs:**
- Gradients $\frac{\partial \mathcal{L}}{\partial \theta_i}$ for all Gaussians $i \in \mathcal{G}$.

**Data Structures:**
- A list of Gaussians, each containing its current parameters and an accumulator for gradients.
- A list of pixels or tiles, storing the gradient of the loss with respect to the pixel color.

**Steps:**
1. **Forward Pass:** Render the image using the standard alpha-blending rasterizer to produce pixel colors $\mathbf{c}_j$ and transmittance values $T_j$ for each pixel $j$. Store the depth-sorted list of splats contributing to each pixel [S1].
2. **Pixel Gradient Computation:** Compute the gradient of the loss with respect to the pixel colors, $\frac{\partial \mathcal{L}}{\partial \mathbf{c}_j}$, using the standard loss function (e.g., L1 or SSIM) [S1].
3. **Per-Splat Gradient Accumulation:** Iterate over each Gaussian $i$ in the scene:
    a. Identify the set of pixels $\mathcal{P}_i$ that the Gaussian $i$ contributes to (i.e., pixels where Gaussian $i$ is within the tile and depth-sorted correctly).
    b. For each pixel $j \in \mathcal{P}_i$, compute the local gradient contribution. This involves backpropagating through the alpha-blending equation for pixel $j$ to find $\frac{\partial \mathcal{L}}{\partial \mathbf{c}_i}$, the gradient of the loss with respect to the color contribution of Gaussian $i$ at pixel $j$.
    c. Compute the gradient of the loss with respect to the Gaussian parameters $\theta_i$ using the chain rule: $\frac{\partial \mathcal{L}}{\partial \theta_i} += \sum_{j \in \mathcal{P}_i} \frac{\partial \mathcal{L}}{\partial \mathbf{c}_i} \frac{\partial \mathbf{c}_i}{\partial \theta_i}$.
    d. Accumulate these gradients into the Gaussian's gradient buffer. Since each Gaussian is processed by a single thread (or a small group), atomic operations are minimized or eliminated if the accumulation is done locally before a final reduction [S1].
4. **Parameter Update:** Apply the accumulated gradients to update the Gaussian parameters using an optimizer (e.g., Adam) [S1].

**Formulas:**
The alpha-blending equation for pixel $j$ is:
$$ \mathbf{c}_j = \sum_{i \in \mathcal{S}_j} \alpha_{i,j} T_{i,j} \mathbf{c}_i $$
where $\mathcal{S}_j$ is the set of splats contributing to pixel $j$, $\alpha_{i,j}$ is the opacity of Gaussian $i$ at pixel $j$, $T_{i,j}$ is the transmittance before Gaussian $i$, and $\mathbf{c}_i$ is the color of Gaussian $i$.

The gradient with respect to Gaussian $i$'s parameters is:
$$ \frac{\partial \mathcal{L}}{\partial \theta_i} = \sum_{j \in \mathcal{P}_i} \frac{\partial \mathcal{L}}{\partial \mathbf{c}_j} \frac{\partial \mathbf{c}_j}{\partial \mathbf{c}_i} \frac{\partial \mathbf{c}_i}{\partial \theta_i} $$
where $\frac{\partial \mathbf{c}_j}{\partial \mathbf{c}_i} = \alpha_{i,j} T_{i,j}$ [S1].

**Parameters:**
- Tile size: Typically 16x16 pixels, leading to a reduction in atomic operations by a factor of 256 [S1].

**Edge Cases and Pitfalls:**
- **Memory Usage:** The per-splat approach requires storing the list of contributing pixels for each Gaussian, which increases VRAM usage compared to the per-pixel approach [S1].
- **Numerical Equivalence:** The method must be implemented carefully to ensure numerical equivalence with the standard approach, avoiding precision errors from different accumulation orders [S1].

## Subtopics

### [Per-Gaussian Backward Pass](9590-per-gaussian-backward-pass.md)
A specific implementation of the per-splat strategy that iterates over Gaussians to compute gradients, reducing atomic contention.

## Operating characteristics
- **Time Complexity:** Reduces training time by 4–5× compared to standard per-pixel backpropagation in budgeted settings [S1].
- **Memory Complexity:** Increases VRAM usage due to the need to store pixel contributions for each Gaussian [S1].
- **Strengths:** Eliminates atomic contention and false sharing, leading to significant speedups on GPU hardware [S1].
- **Limitations:** Higher memory footprint may limit the scene size that can be processed on a single GPU [S1].
- **When to Use:** When training speed is critical and sufficient VRAM is available [S1].

## Background
The method was introduced in "Taming 3DGS: High-Quality Radiance Fields with Limited Resources" as part of a broader effort to optimize the 3D Gaussian Splatting training pipeline [S1]. It addresses the scalability issues of the original 3DGS implementation by rethinking the parallelization strategy of the backward pass [S1].

## Sources
[S1] Taming 3DGS: High-Quality Radiance Fields with Limited Resources, https://doi.org/10.1145/3680528.3687694, fetched 2026-10-30
