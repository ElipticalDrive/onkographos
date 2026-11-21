# Weighted Order-Independent Transparency for 3DGS

Also called: OIT-based 3DGS rendering, Weighted OIT, Weighted Order-Independent Transparency (OIT) Rendering

## Overview
Weighted Order-Independent Transparency (OIT) for 3D Gaussian Splatting computes a blended image without sorting Gaussians by depth, using a learnable weight function to approximate correct transparency blending [S1, unofficial]. It differs from standard 3DGS rasterization, which requires sorting by distance to camera to correctly accumulate transmittance, by eliminating the sorting step entirely in favor of a weighted average [S1, unofficial]. The method builds on [SparseOIT Rasterizer](7562-sparseoit-rasterizer.md) [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: A set of $N$ 3D Gaussians $G = \{\bm{\mu}_i, q_i, s_i, o_i, \mathbf{h}_i, w_i \mid i=1,\dots,N\}$, where $\bm{\mu}_i$ is the 3D center, $q_i$ is the quaternion rotation, $s_i$ is the scale, $o_i$ is the base opacity, $\mathbf{h}_i$ are spherical harmonic coefficients, and $w_i$ is the learnable weight [S1, unofficial]. Camera parameters (view matrix $\mathbf{W}$, projection matrix, focal point $\mathbf{f}$) are required for projection [S1, unofficial].
- **Outputs**: Final pixel color $\mathbf{C}$ and transmittance $T$ for each pixel [S1, unofficial].

**Data Structures**
- **Gaussian Attributes**: Each Gaussian $i$ stores:
  - $\bm{\mu}_i \in \mathbb{R}^3$: 3D center position [S1, unofficial].
  - $q_i \in \mathbb{R}^4$: Rotation quaternion [S1, unofficial].
  - $s_i \in \mathbb{R}^3$: Scale vector [S1, unofficial].
  - $o_i \in \mathbb{R}$: Base opacity scalar [S1, unofficial].
  - $\mathbf{h}_i$: Spherical harmonic coefficients for color [S1, unofficial].
  - $w_i \in \mathbb{R}$: Learnable weight scalar [S1, unofficial].
- **Projected Attributes** (computed per pixel/tile):
  - $\bm{\mu}'_i \in \mathbb{R}^2$: Projected 2D center [S1, unofficial].
  - $\bm{\Sigma}'_i \in \mathbb{R}^{2 \times 2}$: Projected 2D covariance matrix [S1, unofficial].
  - $\alpha_i \in \mathbb{R}$: Final opacity at the pixel [S1, unofficial].
  - $\mathbf{c}_i \in \mathbb{R}^3$: Gaussian color at the pixel [S1, unofficial].

**Numbered Steps**

1. **Project Gaussians**: For each Gaussian $i$, compute the 3D-to-2D projection [S1, unofficial].
   - Compute the 3D covariance matrix $\bm{\Sigma}_i$ from scale $s_i$ and rotation $q_i$ [S1, unofficial].
   - Compute the view transformation matrix $\mathbf{W}$ and the Jacobian matrix $\mathbf{J}$ of the affine approximation of the projective transformation at the Gaussian center [S1, unofficial].
   - Compute the 2D covariance matrix:
     $$ \bm{\Sigma}'_i = \mathbf{J} \mathbf{W} \bm{\Sigma}_i \mathbf{W}^T \mathbf{J}^T $$
     [S1, unofficial]

2. **Compute Final Opacity $\alpha_i$**: For a pixel at position $\bm{x}'_i$ (the pixel center in 2D space), calculate the Gaussian's contribution opacity:
   $$ \alpha_i = o_i \cdot \left(-\frac{1}{2} (\bm{x}'_i - \bm{\mu}'_i)^T {\bm{\Sigma}'_i}^{-1} (\bm{x}'_i - \bm{\mu}'_i)\right) $$
   [S1, unofficial]

3. **Compute Color $\mathbf{c}_i$**: Evaluate the spherical harmonic function $\mathbf{Y}$ using the direction from the camera focal point $\mathbf{f}$ to the Gaussian center $\bm{\mu}_i$ and the coefficients $\mathbf{h}_i$:
   $$ \mathbf{c}_i = \mathbf{Y}(\|\mathbf{f} - \bm{\mu}_i\|, \mathbf{h}_i) $$
   [S1, unofficial]

4. **Accumulate Weighted Sum**: For each pixel, initialize accumulators for the weighted color sum and the total weight [S1, unofficial]. Iterate over all Gaussians $i=1 \dots N$ that overlap the pixel (or use an active set for optimization [S1, unofficial]):
   - Let $w_i$ be the learnable weight for Gaussian $i$ [S1, unofficial].
   - Accumulate:
     $$ \text{WeightedColor} = \sum_{i=1}^{N} \mathbf{c}_i \alpha_i w_i $$
     $$ \text{TotalWeight} = \sum_{i=1}^{N} \alpha_i w_i $$
   [S1, unofficial]

5. **Compute Transmittance $T$**: Calculate the product of $(1 - \alpha_i)$ for all overlapping Gaussians:
   $$ T = \prod_{i=1}^{N} (1 - \alpha_i) $$
   [S1, unofficial]

6. **Compute Final Color $\mathbf{C}$**: Combine the background color $\mathbf{c}_0$, the transmittance $T$, and the weighted average of the Gaussian colors:
   $$ \mathbf{C} = T \mathbf{c}_0 + (1 - T) \frac{\text{WeightedColor}}{\text{TotalWeight}} $$
   [S1, unofficial]

**Parameters**
- **Learnable Weight $w_i$**: A scalar value associated with each Gaussian [S1, unofficial]. It is a learnable weight larger when the Gaussian is near the camera [S1, unofficial]. The exact functional form of $w_i$ is defined in Eq. 1 of reference [14], but the specific formula is not provided in the source text [S1, unofficial].

**Edge Cases and Pitfalls**
- **Division by Zero**: If $\text{TotalWeight} = 0$ (no Gaussians overlap or all $\alpha_i w_i = 0$), the weighted average term is undefined [G]. In this case, the result should default to $T \mathbf{c}_0$ [G].
- **Precision**: The product $T = \prod (1-\alpha_i)$ can underflow to zero for many overlapping Gaussians, which is mathematically correct but may cause numerical issues if not handled with log-space accumulation in implementation [G].
- **Weight Definition**: The source states $w_i$ is "larger when gaussian near to camera" but does not provide the explicit formula for $w_i$ in the text, citing [14] instead [S1, unofficial]. Implementers must refer to [14] for the exact definition of $w_i$ [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources [S1, unofficial].
- **Memory Complexity**: Not given in the sources [S1, unofficial].
- **Strengths**: Eliminates the need for depth sorting, which simplifies parallelization on GPU and avoids sorting artifacts [S1, unofficial].
- **Limitations**: Approximates order-dependent transparency; accuracy depends on the quality of the learned weights $w_i$ [S1, unofficial].

## Background
- **Origin**: The weighted OIT rendering equation was first defined in [22] and adopted from [14] [S1, unofficial]. It was applied to 3DGS in the context of improving order-independent transparency methods for Gaussian Avatar Reconstruction [S1, unofficial].
- **Development**: The method is part of the "SparseOIT" framework, which proposes acceleration strategies like the active set method to improve the performance of OIT-based 3DGS reconstruction [S1, unofficial].

## Sources
[S1] SparseOIT: Improving Order-Independent Transparency 3DGS via Active Set Method, https://doi.org/10.48550/arXiv.2605.13855, fetched 2026-10-29 (unofficial)
