# Deformable Beta Splatting
Also called: DBS, Liu et al, 2025a

## Overview
Deformable Beta Splatting (DBS) replaces standard Gaussian kernels with an adaptive, bounded Beta function kernel [S1, unofficial]. It solves the limitation of fixed-shape primitives by allowing the kernel's profile to deform during training, enabling better modeling of diverse geometric structures such as solid geometry with sharp boundaries or high-frequency details [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** $N$ primitives with mean $\boldsymbol{\mu}_i$, covariance $\boldsymbol{\Sigma}_i$, opacity $o_i$, color $c_i$, and learnable shape parameter $b_i$ [S1, unofficial].
- **Output:** Pixel color $C(\mathbf{x})$ [S1, unofficial].

**Data Structures:**
Each primitive $i$ stores:
- $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Mean position.
- $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix.
- $o_i \in \mathbb{R}$: Base opacity.
- $c_i \in \mathbb{R}^3$: Color.
- $b_i \in \mathbb{R}$: Learnable Beta shape parameter [S1, unofficial].

**Kernel Definition:**
Density is defined as $\sigma_i(\mathbf{x}) = \mathcal{B}(\mathbf{x}; \boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i, b_i) \cdot o_i$ [S1, unofficial].
The Beta kernel is:
$$ \mathcal{B}(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma}, b) = (1 - r(\mathbf{x}))^{\beta(b)} $$
where the exponent is $\beta(b) = 4 \exp(b)$ [S1, unofficial].
$r(\mathbf{x})$ is the normalized squared Mahalanobis distance, constrained to $[0, 1)$ [S1, unofficial].

**Rendering Steps:**
1. Project $\boldsymbol{\mu}_i$ and $\boldsymbol{\Sigma}_i$ to 2D screen space [G].
2. Cull primitives outside the pixel/tile bounding box [G].
3. Sort primitives front-to-back by depth [G].
4. Evaluate $\sigma_i(\mathbf{x})$ for each pixel [S1, unofficial].
5. Composite color:
   $$ C(\mathbf{x}) = \sum_{i=1}^N c_i \sigma_i(\mathbf{x}) \prod_{j=1}^{i-1} (1 - \sigma_j(\mathbf{x})) $$
   [S1, unofficial].

**Parameters:**
- $b_i \in \mathbb{R}$ controls shape. Negative $b$ produces flatter profiles with sharper boundaries for solid geometry; positive $b$ yields sharper peaks for high-frequency details [S1, unofficial].
- Initialization approximates a scaled Gaussian [S1, unofficial].

**Pitfalls:**
- Clamp $r(\mathbf{x})$ to $[0, 1)$ to prevent NaNs in the power operation [G].
- Optimize $b$ via gradient descent alongside other attributes [G].

## Subtopics

### [Universal Beta Splatting](9234-universal-beta-splatting.md)
A unified framework that generalizes 3D Gaussian Splatting to N-dimensional anisotropic Beta kernels. It sets itself apart by natively incorporating view and temporal dimensions into the kernel's query space, achieving parameter efficiency without auxiliary color encodings. Choose it when modeling view-dependent appearance and scene dynamics is required.

## Operating characteristics
- **Time and memory complexity:** Not given in the sources.
- **Strengths:** Adaptive kernel shapes model diverse geometries more effectively than fixed Gaussian kernels [S1, unofficial].
- **Limitations:** Requires optimization of the additional parameter $b$ [G].
- **When to use:** When scene geometry varies significantly in sharpness or when Gaussian kernels fail to capture high-frequency details or sharp boundaries efficiently [G].

## Background
Deformable Beta Splatting (DBS) was introduced by Liu et al. in 2025 [S1, unofficial]. It builds on the Beta Function Splatting Kernel, which uses the Beta function to control footprint shape [S1, unofficial].

## Sources
[S1] Universal Beta Splatting, https://doi.org/10.48550/arXiv.2510.03312, fetched 2026-10-30 (unofficial)
