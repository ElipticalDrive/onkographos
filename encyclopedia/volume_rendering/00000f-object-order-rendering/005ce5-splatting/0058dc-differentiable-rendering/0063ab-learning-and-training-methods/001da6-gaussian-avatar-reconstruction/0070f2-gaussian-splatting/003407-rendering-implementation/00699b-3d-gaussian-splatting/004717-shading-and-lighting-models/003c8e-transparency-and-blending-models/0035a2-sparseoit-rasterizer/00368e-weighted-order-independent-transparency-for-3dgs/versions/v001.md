# Weighted Order-Independent Transparency for 3DGS

Also called: OIT-based 3DGS rendering, Weighted OIT, Weighted Order-Independent Transparency (OIT) Rendering

## Overview
Weighted Order-Independent Transparency (OIT) for 3D Gaussian Splatting (3DGS) is a rendering technique that computes a blended image without sorting Gaussians by depth, using a learnable weight function to approximate correct transparency blending [S1, unofficial]. It differs from standard 3DGS rasterization, which requires sorting by distance to camera to correctly accumulate transmittance, by eliminating the sorting step entirely in favor of a weighted average that approximates the order-dependent result [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: A set of $N$ 3D Gaussians $G = \{\bm{\mu}_i, q_i, s_i, o_i, \mathbf{h}_i, w_i \mid i=1,\dots,N\}$, where $\bm{\mu}_i$ is the 3D center, $q_i$ is the quaternion rotation, $s_i$ is the scale, $o_i$ is the base opacity, $\mathbf{h}_i$ are spherical harmonic coefficients, and $w_i$ is the learnable weight [S1, unofficial]. Camera parameters (view matrix $\mathbf{W}$, projection matrix, focal point $\mathbf{f}$) are required for projection [S1, unofficial].
- **Outputs**: Final pixel color $\mathbf{C}$ and transmittance $T$ for each pixel [S1, unofficial].

**Data Structures**
- **Gaussian Attributes**: Each Gaussian $i$ stores:
  - $\bm{\mu}_i \in \mathbb{R}^3$: 3D center position.
  - $q_i \in \mathbb{R}^4$: Rotation quaternion.
  - $s_i \in \mathbb{R}^3$: Scale vector.
  - $o_i \in \mathbb{R}$: Base opacity scalar.
  - $\mathbf{h}_i$: Spherical harmonic coefficients for color.
  - $w_i \in \mathbb{R}$: Learnable weight scalar [S1, unofficial].
- **Projected Attributes** (computed per pixel/tile):
  - $\bm{\mu}'_i \in \mathbb{R}^2$: Projected 2D center.
  - $\bm{\Sigma}'_i \in \mathbb{R}^{2 \times 2}$: Projected 2D covariance matrix.
  - $\alpha_i \in \mathbb{R}$: Final opacity at the pixel.
  - $\mathbf{c}_i \in \mathbb{R}^3$: Gaussian color at the pixel.

**Numbered Steps**

1. **Project Gaussians**: For each Gaussian $i$, compute the 3D-to-2D projection.
   - Compute the 3D covariance matrix $\bm{\Sigma}_i$ from scale $s_i$ and rotation $q_i$.
   - Compute the view transformation matrix $\mathbf{W}$ and the Jacobian matrix $\mathbf{J}$ of the affine approximation of the projective transformation at the Gaussian center [S1, unofficial].
   - Compute the 2D covariance matrix:
     $$ \bm{\Sigma}'_i = \mathbf{J} \mathbf{W} \bm{\Sigma}_i \mathbf{W}^T \mathbf{J}^T $$
     [S1, unofficial]

2. **Compute Final Opacity $\alpha_i$**: For a pixel at position $\mathbf{x}'_i$ (the pixel center in 2D space), calculate the Gaussian's contribution opacity:
   $$ \alpha_i = o_i \cdot \exp\left(-\frac{1}{2} (\mathbf{x}'_i - \bm{\mu}'_i)^T {\bm{\Sigma}'_i}^{-1} (\mathbf{x}'_i - \bm{\mu}'_i)\right) $$
   [S1, unofficial]
   *Note: The source text writes the exponent as $-\frac{1}{2} (\dots)$ inside the opacity formula, implying the exponential function is applied to the quadratic form [S1, unofficial].*

3. **Compute Color $\mathbf{c}_i$**: Evaluate the spherical harmonic function $\mathbf{Y}$ using the direction from the camera focal point $\mathbf{f}$ to the Gaussian center $\bm{\mu}_i$ and the coefficients $\mathbf{h}_i$:
   $$ \mathbf{c}_i = \mathbf{Y}(\|\mathbf{f} - \bm{\mu}_i\|, \mathbf{h}_i) $$
   [S1, unofficial]

4. **Accumulate Weighted Sum**: For each pixel, initialize accumulators for the weighted color sum and the total weight. Iterate over all Gaussians $i=1 \dots N$ that overlap the pixel (or use an active set for optimization [S1, unofficial]):
   - Let $w_i$ be the learnable weight for Gaussian $i$. The weight is defined to be larger when the Gaussian is near the camera [S1, unofficial].
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
- **Learnable Weight $w_i$**: A scalar value associated with each Gaussian. It is learned during optimization and is designed to be larger when the Gaussian is closer to the camera [S1, unofficial]. The exact functional form of $w_i$ is referenced from [14] (Eq. 1 in [S1, unofficial]), but the specific formula is not provided in the source text [S1, unofficial].

**Edge Cases and Pitfalls**
- **Division by Zero**: If $\text{TotalWeight} = 0$ (no Gaussians overlap or all $\alpha_i w_i = 0$), the weighted average term is undefined. In this case, the result should default to $T \mathbf{c}_0$ [S1, unofficial].
- **Precision**: The product $T = \prod (1-\alpha_i)$ can underflow to zero for many overlapping Gaussians, which is mathematically correct but may cause numerical issues if not handled with log-space accumulation in implementation [G].
- **Weight Definition**: The source states $w_i$ is "larger when gaussian near to camera" but does not provide the explicit formula for $w_i$ in the text, citing [14] instead [S1, unofficial]. Implementers must refer to [14] for the exact definition of $w_i$.

## Operating characteristics
- **Time Complexity**: $O(N \cdot P)$ where $N$ is the number of Gaussians and $P$ is the number of pixels, as every Gaussian must be evaluated for every pixel it overlaps. This is generally slower than sorted 3DGS rasterization which can terminate early when transmittance is negligible [S1, unofficial].
- **Memory Complexity**: $O(N)$ to store Gaussian attributes. No sorting buffer is required [S1, unofficial].
- **Strengths**: Eliminates the need for depth sorting, which simplifies parallelization on GPU and avoids sorting artifacts [S1, unofficial].
- **Limitations**: Approximates order-dependent transparency; accuracy depends on the quality of the learned weights $w_i$ [S1, unofficial]. Performance is lower than sorted rasterization due to the lack of early-out optimization based on depth [S1, unofficial].

## Background
- **Origin**: The weighted OIT rendering equation was first defined in [22] and adopted from [14] [S1, unofficial]. It was applied to 3DGS in the context of improving order-independent transparency methods for Gaussian Avatar Reconstruction [S1, unofficial].
- **Development**: The method is part of the "SparseOIT" framework, which proposes acceleration strategies like the active set method to improve the performance of OIT-based 3DGS reconstruction [S1, unofficial].

## Sources
[S1] SparseOIT: Improving Order-Independent Transparency 3DGS via Active Set Method, https://doi.org/10.48550/arXiv.2605.13855, fetched 2026-10-29 (unofficial)
