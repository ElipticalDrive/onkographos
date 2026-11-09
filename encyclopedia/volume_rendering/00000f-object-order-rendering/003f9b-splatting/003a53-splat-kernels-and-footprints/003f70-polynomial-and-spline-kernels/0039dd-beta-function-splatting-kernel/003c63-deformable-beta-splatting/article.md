# Deformable Beta Splatting
Also called: Liu et al, 2025a

## Overview
Deformable Beta Splatting (DBS) is an explicit radiance field rendering method that replaces the standard Gaussian kernel with an adaptive, bounded Beta function kernel [S1, unofficial]. It solves the limitation of fixed-shape primitives by allowing the kernel's profile to deform during training, enabling better modeling of diverse geometric structures such as solid geometry with sharp boundaries or high-frequency details [S1, unofficial]. Unlike its sibling Universal Beta Splatting (UBS), which generalizes to N-dimensional anisotropic kernels for view and temporal dimensions, DBS focuses on the spatial deformation of the Beta kernel shape via a single learnable parameter [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A set of $N$ primitives, each defined by a mean position $\boldsymbol{\mu}_i$, a covariance matrix $\boldsymbol{\Sigma}_i$, an opacity $o_i$, and a color $c_i$. Additionally, each primitive has a learnable scalar parameter $b_i \in \mathbb{R}$ that controls the kernel shape.
- **Output:** Pixel color $C(\mathbf{x})$ and opacity $\sigma(\mathbf{x})$ for a given camera ray origin $\mathbf{x}$.

**Data Structures:**
Each primitive $i$ stores:
- $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Mean position.
- $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix (typically decomposed into rotation and scale for optimization).
- $o_i \in \mathbb{R}$: Base opacity.
- $c_i \in \mathbb{R}^3$: Color.
- $b_i \in \mathbb{R}$: Learnable Beta shape parameter.

**Kernel Definition:**
The density contribution of a primitive is defined as:
$$ \sigma_i(\mathbf{x}) = \mathcal{B}(\mathbf{x}; \boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i, b_i) \cdot o_i $$
where the Beta kernel $\mathcal{B}$ is:
$$ \mathcal{B}(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma}, b) = (1 - r(\mathbf{x}))^{\beta(b)} $$
Here, $r(\mathbf{x})$ is the normalized squared Mahalanobis distance:
$$ r(\mathbf{x}) = (\mathbf{x} - \boldsymbol{\mu})^\top \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu}) $$
Note: The source defines $r(\mathbf{x}) \in [0, 1)$, implying a normalization or clipping is applied to ensure the base $(1-r)$ remains non-negative. The exponent parameter is:
$$ \beta(b) = 4 \exp(b) $$

**Rendering Steps:**
1. **Projection:** For each primitive, project $\boldsymbol{\mu}_i$ and $\boldsymbol{\Sigma}_i$ to 2D screen space using the camera projection matrix and Jacobian.
2. **Footprint Culling:** Discard primitives whose 2D bounding box does not overlap the current pixel or tile.
3. **Sorting:** Sort the remaining primitives in front-to-back order based on their depth along the ray.
4. **Density Evaluation:** For each pixel, iterate through the sorted primitives. For primitive $i$, compute $r(\mathbf{x})$ and then $\beta(b_i)$. Calculate the kernel value $\mathcal{B}$ and the final density $\sigma_i(\mathbf{x}) = \mathcal{B} \cdot o_i$.
5. **Alpha Compositing:** Accumulate color and opacity using standard front-to-back alpha compositing:
   $$ C(\mathbf{x}) = \sum_{i=1}^N c_i \sigma_i(\mathbf{x}) \prod_{j=1}^{i-1} (1 - \sigma_j(\mathbf{x})) $$
   $$ \alpha(\mathbf{x}) = 1 - \prod_{j=1}^N (1 - \sigma_j(\mathbf{x})) $$

**Parameters:**
- $b_i$: Learnable scalar. Negative values produce flatter profiles with sharper boundaries (suitable for solid geometry). Positive values produce sharper peaks (suitable for high-frequency details) [S1, unofficial].
- Initialization: The Beta kernel closely approximates a scaled Gaussian when initialized, suggesting $b$ should be initialized to a value that yields a Gaussian-like shape [S1, unofficial].

**Pitfalls:**
- Ensure $r(\mathbf{x}) < 1$ to avoid complex numbers or NaNs in the power operation. If $r(\mathbf{x}) \ge 1$, the kernel value should be clamped to 0 [G].
- The parameter $b$ must be optimized via gradient descent alongside other primitive attributes.

## Subtopics

### [Universal Beta Splatting](9234-universal-beta-splatting.md)
A unified framework that generalizes 3D Gaussian Splatting to N-dimensional anisotropic Beta kernels, incorporating view and temporal dimensions into the kernel's query space. Choose this when you need to model view-dependent appearance and scene dynamics without auxiliary encodings.

## Operating characteristics
- **Time Complexity:** Similar to 3D Gaussian Splatting, dominated by the sorting and compositing steps $O(N \log N)$ per pixel, where $N$ is the number of primitives. The kernel evaluation is slightly more complex due to the exponential and power operations but remains constant time per primitive.
- **Memory Complexity:** $O(N)$ per primitive, with an additional scalar $b_i$ per primitive.
- **Strengths:** Adaptive kernel shapes allow for better representation of diverse geometries (sharp vs. smooth) compared to fixed Gaussians. Parameter efficiency is improved compared to methods requiring auxiliary color encodings.
- **Limitations:** Requires optimization of the additional parameter $b$. The bounded nature of the Beta kernel may require careful handling of the $r(\mathbf{x}) \ge 1$ case.
- **When to use:** When scene geometry varies significantly in sharpness or when Gaussian kernels fail to capture high-frequency details or sharp boundaries efficiently.

## Background
Deformable Beta Splatting (DBS) was introduced by Liu et al. in 2025 [S1, unofficial]. It is part of the broader development of explicit radiance field rendering methods that seek to improve upon the limitations of 3D Gaussian Splatting by introducing more flexible kernel functions. The method builds on the Beta Function Splatting Kernel, which uses the Beta function to control footprint shape.

## Sources
[S1] Universal Beta Splatting, https://doi.org/10.48550/arXiv.2510.03312, fetched 2026-10-30 (unofficial)
